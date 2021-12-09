---
title: TypeScript 装饰器
date: 2019/10/16
description: 介绍 TypeScript 当中装饰器的使用
tag: decorator, TypeScript, tutorial, Chinese
author: Wendell
---

# TypeScript 装饰器

TypeScript 有一个强大但是却不那么新手友好的功能，那就是[装饰器](https://www.typescriptlang.org/docs/handbook/decorators.html)。 你肯定用过 Angular 实现的很多装饰器，比如装饰类的 `@Component`，装饰属性的 `@ViewChild`，以及装饰方法的 `@HostListenner`，但是你尝试过自己写一个装饰器吗？它们粗看起来似乎很神奇 🍄，但实际上它们只是一些 JavaScript 函数，能够帮助我们来注释代码或者是修改代码的行为——这种做法我们通常称为[**元编程**](https://baike.baidu.com/item/%E5%85%83%E7%BC%96%E7%A8%8B/6846171?fr=aladdin)。

一共有五种装饰器的方法，我们会通过举例子的方式一一讲解它们。

<!-- more -->

- 类声明
- 属性
- 方法
- 参数
- accessor

> 装饰器能够很好的抽象代码。尽管用装饰器来封装所有东西看起来很有诱惑力，但是它们最合适的用场还是来包装可能会多处复用的**稳定的逻辑**。

## 类装饰器 Class Decorator

类装饰器使得开发者能够拦截类的构造方法 constructor。注意：当我们声明一个类时，装饰器就会被调用，而不是等到类实例化的时候。

注：装饰器最为强大的功能之一是它能够反射元数据（reflect metada），一般开发者很少会需要这个功能，但是在例如 Angular 这样的框架中，它很适合用来分析代码来得到最终的 bundle。

### 例子

当你装饰一个类的时候，装饰器并不会对该类的子类生效，让我们来冻结一个类来彻底避免别的程序员不小心忘了这个特性。

```ts
@Frozen
class IceCream {}

function Frozen(constructor: Function) {
  Object.freeze(constructor)
  Object.freeze(constructor.prototype)
}

console.log(Object.isFrozen(IceCream)) // true

class FroYo extends IceCream {} // 报错，类不能被扩展
```

## 属性装饰器 Property Decorator

> 这里的例子都用到了**装饰器工厂模式**。我们将装饰器本身封装在另外一个函数中，这样就能给装饰器传递变量了，例如 `@Cool('stuff')`。而当你不想给装饰器传参，把外层那个函数去掉就好了 `@Cool`。

属性装饰器极其有用，因为它可以监听对象状态的变化。为了充分了解接下来这个例子，建议你先熟悉一下 JavaScript 的属性描述符（[PropertyDescriptor](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)）。

### 例子

在这个例子中我们将会用两个冰淇淋 emoji 来包裹 flavor 属性的值。此时属性装饰器就可以使得我们在对属性赋值之前或在取属性之后附加一些操作，就像**中间件**那样。

```ts
export class IceCreamComponent {
  @Emoji()
  flavor = 'vanilla'
}

// Property Decorator
function Emoji() {
  return function(target: Object, key: string | symbol) {
    let val = target[key]

    const getter = () => {
      return val
    }
    const setter = next => {
      console.log('updating flavor...')
      val = `🍦 ${next} 🍦`
    }

    Object.defineProperty(target, key, {
      get: getter,
      set: setter,
      enumerable: true,
      configurable: true,
    })
  }
}
```

## 方法装饰器 Method Decorator

我们可以使用方法装饰器来覆写一个方法，改变它的执行流程，以及在它执行前后额外运行一些代码。

### 例子

下面这个例子会在执行真正的代码之前弹出一个确认框。如果用户点击了取消，方法就会被跳过。注意，这里我们装饰了一个方法两次，这两个装饰器会从上到下地执行。

```ts
export class IceCreamComponent {
  toppings = []

  @Confirmable('Are you sure?')
  @Confirmable('Are you super, super sure? There is no going back!')
  addTopping(topping) {
    this.toppings.push(topping)
  }
}

// Method Decorator
function Confirmable(message: string) {
  return function(
    target: Object,
    key: string | symbol,
    descriptor: PropertyDescriptor
  ) {
    const original = descriptor.value

    descriptor.value = function(...args: any[]) {
      const allow = confirm(message)

      if (allow) {
        const result = original.apply(this, args)
        return result
      } else {
        return null
      }
    }

    return descriptor
  }
}
```

## 为 Angular 实现 React Hooks 🤯

你或许听说过 [React Hooks](https://reactjs.org/docs/hooks-intro.html) 如何彻底改变了 React 的开发生态。Angular 能不能用这样的方式来写出同样优雅、简明的代码呢？事实上，完全可以，而且从一开始就可以。

![React hooks game changer results](https://fireship.io/lessons/ts-decorators-by-example/img/react-hooks-gamechanger.png)

### UseState 属性装饰器

在 React 中，调用 useState hook 会返回给你一个响应式的变量 `count` 和一个 setter `setCount`。

```jsx
import { useState } from 'react'

function Example() {
  // Declare a new state variable, which we'll call "count"
  const [count, setCount] = useState(0)

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </div>
  )
}
```

我们可以使用一个属性装饰器来实现**相似**的效果。在组件的 `count` 属性上声明该装饰器，这个装饰器就会帮我们定义 `count`，同时也会帮我们定义 `setCount` 这个 setter。

用起来就像这样：

```ts
import { BehaviorSubject } from 'rxjs'

@Component({
  selector: 'app-root',
  template: `
    <p>You clicked {{ count }} times</p>
    <button (click)="setCount(count + 1)">Click Me</button>
  `,
})
export class HookComponent {
  @UseState(0) count
  setCount
}
```

而该装饰器的实现不过五行代码，我们只需要设置好初始值以及相应的 setter 即可：

```ts
function UseState(seed: any) {
  return function(target, key) {
    target[key] = seed
    target[`set${key.replace(/^\w/, c => c.toUpperCase())}`] = val =>
      (target[key] = val)
  }
}
```

### UseEffect 方法装饰器

[Effect hook](https://reactjs.org/docs/hooks-effect.html) 所做的事情只是简单的把组件生命周期中的 `componentDidMount` 和 `componentDidUpdate` 这两个钩子合并到了同一个回调中。

```jsx
useEffect(() => {
  // Update the document title using the browser API
  document.title = `You clicked ${count} times`
})
```

用一个方法装饰器很容易就能模拟，我们只需要将 Angular 对应的生命周期方法 `ngOnInit` 和 `ngAfterViewChecked` 指向该方法的属性描述符的 `value` 即可。

```ts
@Component(...)
export class AppComponent {
  @UseEffect()
  onEffect() {
    document.title = `You clicked ${this.count.value} times`;
  }
}

function UseEffect() {
  return function (target, key, descriptor) {
    target.ngOnInit = descriptor.value;
    target.ngAfterViewChecked = descriptor.value;
  };
}
```
