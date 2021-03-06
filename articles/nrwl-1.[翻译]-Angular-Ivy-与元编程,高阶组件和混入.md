# [翻译] Angular Ivy 与元编程，高阶组件和混入(mixins)

> 原文链接：[Metaprogramming, Higher-Order Components and Mixins with Angular Ivy](https://blog.nrwl.io/metaprogramming-higher-order-components-and-mixins-with-angular-ivy-75748fcbc310)

> 原文作者：[Victor Savkin](https://blog.nrwl.io/@vsavkin)

> 原技术博文由 [Victor Savkin](https://blog.nrwl.io/@vsavkin) 撰写发布，他是 [Narwhal](nrwl.io) 的创始人之一，`Narwhal` 为那些希望快速将其应用投入生产的大型团队提供Angular咨询服务。

> 为了行文方便，下文均以`我`指代原作者。

<p align="center"> 
    <img src="../assets/nrwl-1/1.jpeg">
</p>

Angular 社区中的每个开发者都对新一代 Angular 渲染引擎 `Ivy` 的即将发布感到十分激动。

- Ivy 让 Angular 更快速地运行。
- Ivy 让摇树更加优化(使用Ivy编译出的 hello-world 应用大小低于4kb)。
- Ivy 简化了生成的代码，让 Angular 程序更容易调试。
- Ivy 精简了构建管道(不再需要元数据或 ngfactory 文件)，使得诸如懒加载单个组件之类的任务变得更加简单。

虽然我对所有这些新特性都感到十分兴奋，但在内心深处，我最为兴奋的是其他的一些东西。

**Ivy 让 Angular 变得动态化，他支持元编程，并使得混入和高阶组件之类的实现变得更加轻松**

## 如何使用 Ivy 进行元编程

让我们看一个例子，从 NgRx store 中读取数据。现在我们像这样实现它：

```typescript
@Component({
  selector: 'todos-cmp',
  template: `
    <div *ngFor="let t of todos|async">
        {{t.description}}
    </div>
  `
})
class TodosComponent {
  todos: Observable<Todo[]> = this.store.pipe(select('todos'));
  constructor(private store: Store<AppState>) {}
}
```

为了了解如何使用 Ivy 进行元编程，让我们先看看这个组件编译后的样子：

```typescript
var TodosComponent = /** @class */ (function () {
  function TodosComponent(store) {
    this.store = store;
    this.todos = this.store.pipe(select('todos'));
  }

  TodosComponent.ngComponentDef = defineComponent({
    type: TodosComponent,
    selectors: [["todos-cmp"]],
    factory: function TodosComponent_Factory(t) {
      return new (t || TodosComponent)(directiveInject(Store));
    },
    consts: 2,
    vars: 3,
    template: function TodosComponent_Template(rf, ctx) {
      if (rf & 1) {
        pipe(1, "async");
        template(0, TodosComponent_div_Template_0, 2, 1, null, _c0);
      } if (rf & 2) {
        elementProperty(0, "ngForOf", bind(pipeBind1(1, 1, ctx.todos)));
      }
    },
    encapsulation: 2
  });

  return TodosComponent;
}());
```

关于以上编译后内容可以写一整篇的博客描述它的数据结构以及 Angular 使用它的方式，还可以撰写一系列的博客来介绍编译器如何使用 `incremental DOM`技术将模板呈现为指令集。这些都是很有趣的主题，但是他们不是本篇博客的主题。

在本篇博客中我们将聚焦于数据结构的工厂部分：

```typescript
factory: function TodosComponent_Factory(t) {
  return new (t || TodosComponent)(directiveInject(Store));
},
```

Angular 使用工厂方法创造`TodosComponent`，`directiveInject`将于节点注入器树（node injector tree）上移动(想象于 DOM 上移动的过程)并试图找到`Store`服务。如果无法在节点注射器树上找到该服务，那么`directiveInject`将会去模块注入器树（module injector tree）上移动寻找服务。

## Dynamism(活力/动态)

Angular在运行过程中初始化和渲染组件时会使用`ngComponentDef`，这意味着我们可以写一个装饰器以修改这个数据结构：

```typescript
function FromStore(config: {[k: string]: string}) {
  return (clazz: any) => {
    const originalFactory = clazz.ngComponentDef.factory;
    clazz.ngComponentDef.factory = () => {
      const cmp = originalFactory(clazz.ngComponentDef.type);
      const store = directiveInject(Store);
      Object.keys(config).forEach((key: string) => {
         cmp[key] = store.pipe(select(config[key]));
      });
      return cmp;
    };
  };
}
```

然后我们像如下这样将其应用于我们的组件类：

```typescript
@Component({
  selector: 'todos-cmp',
  template: `
    <div *ngFor="let t of todos|async">
        {{t.description}}
    </div>
  `
})
@FromStore({todos: 'todos’})
class TodosComponent {
    todos: Observable<Todo[]>;
}
```

这样的方式虽然可行但却带来另一个问题-它破坏了摇树优化。为什么？因为摇树优化工具会将`FromStore`视为一个有副作用的函数。作为开发者我们知道`FromStore`仅更改`TodosComponent`本身，但是摇树器并不知晓这一状况。这是一个大问题，因为更好的摇树优化是 Ivy 引擎关键优势之一。

**不过我们仍然有一个解决方案**

## 特性（Features）

解决方案是定义一个可以接收`ComponentDef`并向其添加行为的函数：

```typescript
function fromStore(config: {[k: string]: string}) {
  return (def: ComponentDef<any>) => {
    const originalFactory = def.factory; // original factory creating a component
    def.factory = () => {
      const cmp = originalFactory(def.type); // component instance
      const store = directiveInject(TodoStore); // using DI to get the store
      config.forEach((key: string) => {
        cmp[key] = store.pipe(select(config[key]));
      });
      return cmp; // return the patched component
    };
  };
}
```

此函数需要添加到`ngComponentRef`属性中：

```typescript
var TodosComponent = /** @class */ (function () {
  function TodosComponent(store) {
    this.store = store;
    this.todos = this.store.pipe(select('todos'));
  }

  TodosComponent.ngComponentDef = defineComponent({
    type: TodosComponent,
    selectors: [["todos-cmp"]],
    factory: function TodosComponent_Factory(t) {
      return new (t || TodosComponent)(directiveInject(Store));
    },
    consts: 2,
    vars: 3,
    template: function TodosComponent_Template(rf, ctx) {
      if (rf & 1) {
        pipe(1, "async");
        template(0, TodosComponent_div_Template_0, 2, 1, null, _c0);
      } if (rf & 2) {
        elementProperty(0, "ngForOf", bind(pipeBind1(1, 1, ctx.todos)));
      }
    },
    encapsulation: 2,
    features: [fromStore({todos})] // <---------------------- Add it here
  });

  return TodosComponent;
}());
```

这些特性与上面定义的装饰器一样服务于同一目标。他允许我们修改`ngComponentDef`的数据结构，但是其与装饰器最大的区别在于: 他不会破坏摇树优化。

为什么我们要将`fromStore`添加到编译后代码中？

Ivy 引擎本身支持这些特性，但是`将@Component转换为ngComponentDef`的编译器并不支持。根据上述论据，我们不能将其添加到组件装饰器中，但是下面的 API (或类似的东西)将会变成现实：

```typescript
@Component({
  selector: 'todos-cmp',
  template: `
    <div *ngFor="let t of todos|async">
        {{t.description}}
    </div>
  `,
  features: [fromStore({todos: 'todos'})]
})
class TodosComponent {
    todos: Observable<Todo[]>;
}
```

## 高阶组件和混入

基于此功能以及动态生成模板的能力，我们可以实现`将组件包装到另一个组件中`的功能，或将行为混合到现存组件中。换句话说，Ivy 不仅仅完成了对于高阶组件和混入的实现，更让其实现方式简单易行。

## 为什么要使用元编程

我们可以用元编程实现两件事：

- 我们可以封装既存的代码模式让我们的组件更加整洁。有趣的是: 在 Angular2 尚未发布之前，我就写过一组特殊的装饰器，它们依赖于规约并大大缩减组件的声明，我称之为`嬉皮士模式`。 现在，Ivy 引擎让这一切再次成为可能。

- 我们可以尝试新的框架功能，而无需对框架本身进行更改。

- 我们可以在不引入破坏性改变的基础上对`library`进行更新。

但是这样也带来了代价：

- 代码可能更加难以读懂。在上面的示例中，如果你不知道`FromStore`的作用，可能会难于理解`todos observable`的创建过程。

- 代码变得难以工具化。上面的代码可能没有那么糟糕，`WebStorm/VSCode`还是可以在模板中使用自动补全功能，但是在其他地方可能因为过火的行为导致我们失去 Angular 已有的工具化便利。

### 谨慎！

Angular团队目前的重点是让Ivy向后完全兼容，而关于如何进行元编程的官方文档将在 Ivy 之后发布。

## 总结

Ivy 引擎给 Angular 带来了许多好处，他让框架变得更快更小更简单。通过`dynamism`，他也让 Angular 变得更加灵活，我们可以用 Angular 来进行元编程，实现高阶组件和混入的功能。