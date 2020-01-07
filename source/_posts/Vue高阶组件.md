---
title: Vue高阶组件
date: 2020-01-07 16:41:18
categories: 
- 技术
tags:
- vue
- javascript
- HoC
---

高阶组件(`HOC`)全称 `Higher-Order Components`, 在`React`中复用代码最主要的方式，也是[官方推荐](http://react.html.cn/docs/higher-order-components.html)的一种模式。然而，在`Vue`生态中，却很少采用，而是推荐[mixin]((https://cn.vuejs.org/v2/guide/mixins.html))方式。
本文不去对比这`hoc`和`mixin`这两种实现的优劣，旨在探索用`hoc`的方式去实现`vue`的高阶组件，并从中得到为何`vue`少于用`hoc`实现组件的复用。

## 什么是高阶组件
说高阶组件之前，先来看下什么是`高阶函数`和`组件`
 
在维基百科中对[高阶函数](https://zh.wikipedia.org/wiki/%E9%AB%98%E9%98%B6%E5%87%BD%E6%95%B0)的定义。简单来说，就是传入参数有函数，返回的参数是一个函数。

组件，我们对比下`react`和`vue`: 

对于`react`,组件的本质第一反应是`class`, 而`class`的本质是函数,当然纯函数组件也是函数。
对于`vue`,组件的本质是表面来看是`object`, 其实`Vue`中组件是一个被包装的函数，并不简单的就是我们定义组件的时候传入的对象。 具体请参考[组件的本质](http://hcysun.me/vue-design/zh/essence-of-comp.html#%E7%BB%84%E4%BB%B6%E7%9A%84%E4%BA%A7%E5%87%BA%E6%98%AF%E4%BB%80%E4%B9%88)

所以，什么是高阶组件，其实就是高阶函数。对于`vue`，形式不太一样，需要参数一个对象，然后返回一个对象。

## 实现一个简单的vue高阶组件

需求：记录一个组件活了多久，即统计下组件从`mounted`开始，到`destroyed`一共耗时。

``` javascript
// WithLifeTime.js
export default function WithLifeTime(WrappedComponent) {
    return {
        mounted() {
            this._startTime = +new Date
        },
        destroyed() {
            console.log(
              `该组件的生命周期时长为： ${+new Date() - this._startTime}ms`
            );
        },
        render(h) {
            return h(WrappedComponent);
        }
    }
}
```

当`WrappedComponent`不需要和父组件进行任何通信的时候，发现，并没有任何问题。但是孤立的组件其实是没有太大的意义的。那么，需要怎么通信呢?

## 父子组件通信

父子组件通信的方法很多，仅讨论`vue`本身实现，主要是：

1. `props` 和 `$emit`
2. `v-slot`
3. `provide` & `inject` 

其中，`1`和`2`在做`HOC`时需要考虑，`3`不需要考虑

## 解决 `props` 和 `$emit` 通信方式
需在高阶组件中申明`props`，否则实例化时，`this.$props`将是空对象。

```javascript
// WithLifeTime.js
export default function WithLifeTime(WrappedComponent) {
    return {
      mounted() {
        this._startTime = +new Date();
      },
      props: WrappedComponent.props, // 不申明props,实例化时，this.$props将是空对象
      destroyed() {
        console.log(
          `该组件的生命周期时长为： ${+new Date() - this._startTime}ms`
        );
      },
      render(h) {
        return h(WrappedComponent, {
          on: this.$listeners,
          attrs: this.$attrs,
          props: this.$props
        });
      }
    };
}
```

上面组件完成了

> 1、透传 props
> 2、透传没有被声明为 props 的属性
> 3、透传事件

## 解决高阶组件`v-slot`的传递

先看下 `BaseComponent.vue`的内容，该组件是被高阶组件包裹的组件，用于测试`HOC`组件功能是否正确。

``` html
<!-- BaseComponent.vue -->

<template>
  <div class="base-component">
    <button @click="$emit('click', $event)">props: {{text}}</button>
    <slot name="namedSlot" childToParent="fromBaseComponent.vue"></slot>
    <slot></slot>
  </div>
</template>

<script>
export default {
 name: 'BaseComponent',
 props: {
   text: String
 }
}
</script>
```

测试代码，分别渲染`BaseComponent.vue`和被`WithLifeTime包裹后的BaseComponent.vue`
```html
<!-- HoC.vue -->

<template>
  <div class="hoc" style="margin: 50px;">
    <base-component text="baseComponent">
      <p>baseComponent 默认插槽</p>
      <template v-slot:namedSlot="{childToParent}">
        <p>baseComponent 具名插槽; {{ childToParent }}</p>
      </template>
    </base-component>
    <hr />
    <wrapper-base-component text="wrapperBaseComponent">
      <p>wrapperBaseComponent 默认插槽</p>
      <template v-slot:namedSlot="{childToParent}">
        <p>wrapperBaseComponent 具名插槽; {{ childToParent }}</p>
      </template>
    </wrapper-base-component>
  </div>
</template>

<script>
import BaseComponent from "./BaseComponent.vue";
import WithLifeTime from "./WithLifeTime";

export default {
  name: "Hoc",
  components: {
    BaseComponent,
    WrapperBaseComponent: WithLifeTime(BaseComponent)
  }
};
</script>
```
在`WithLifeTime.js`未对`slot`做处理的情况下，看下运行结果，可以看出，`slot`未生效。

![](noslot.png)

下面，我们更改下`WithLifeTime`
```javascript
export default function WithLifeTime(WrappedComponent) {
  return {
    mounted() {
      this._startTime = +new Date();
    },
    props: WrappedComponent.props, // 不申明props,实例化时，this.$props将是空对象
    destroyed() {
      console.log(
        `该组件的生命周期时长为： ${+new Date() - this._startTime}ms`
      );
    },
    render(h) {
      // 将 this.$scopedSlots 格式化为数组
      const children = Object.keys(this.$scopedSlots).reduce(
        (arr, key) => arr.concat(this.$scopedSlots[key]()),
        []
      );

      return h(
        WrappedComponent,
        {
          on: this.$listeners,
          attrs: this.$attrs,
          props: this.$props
        },
        children
      );
    }
  };
}

```
查看运行结果，有两个问题：
1. 默认插槽和具名插槽顺序有问题
2. scopedSlots的值没穿进去

![](slotwrong.png)

先来看`第一个`，顺序问题：
`vue`在渲染具名`slot`的时候，会对比`$vnode.context`（用来表示渲染该组件的上下文）,`BaseComponent.vue`中的具名slot渲染成的`vnode`拿到的`context`是`HoC`的上下文；但是`BaseComponent.vue`的上下文环境是在`WithLifeTime`中。所以传入的具名组件被当成了默认组件。
更改后的`WithLifeTime`

```javascript
// WithLifeTime.js

export default function WithLifeTime(WrappedComponent) {
  return {
    mounted() {
      this._startTime = +new Date();
    },
    props: WrappedComponent.props, // 不申明props,实例化时，this.$props将是空对象
    destroyed() {
      console.log(
        `该组件的生命周期时长为： ${+new Date() - this._startTime}ms`
      );
    },
    render(h) {
      // 将 this.$scopedSlots 格式化为数组
      const children = Object.keys(this.$scopedSlots)
        .reduce((arr, key) => arr.concat(this.$scopedSlots[key]()), [])
        .map(vnode => {
          vnode.context = this._self;
          return vnode;
        });

      return h(
        WrappedComponent,
        {
          on: this.$listeners,
          attrs: this.$attrs,
          props: this.$props
        },
        children
      );
    }
  };
}

```
结果：
![](scopedSlot值未传递进去.png)

`scopedSlots`值没传递下去（先待定，找时间补上）

## 后记

文中的[源码](https://github.com/erichmlyh/vueDemo/tree/master/components/HoC)已上传`github`

本文只提及部分关于`vue HOC`的实现，对于`指令`、`style`、`class`、`domProps`等未做讨论，所以对于项目实践中，`vue HOC`应用相对没那么广泛。在`vue 3.0`的`rfcs`中，`vnode`做出了相应的调整,且看之后发展进程。

对于组件的复用，在`React 16.8`后引入`React Hook`， `vue 3.0`又提出 `Composition API`。 或许借鉴，但总归是殊途同归。


提及或相关但未展开的内容：
> 1. 技术没有银弹，`HOC`也不例外，有兴趣可以看下[Michael Jackson - Never Write Another HoC](https://www.youtube.com/watch?reload=9&v=BcVAq3YFiuc)
> 2. 为何`react`生态没有采用`minxin`，可参考[Mixins Considered Harmful](https://reactjs.org/blog/2016/07/13/mixins-considered-harmful.html)
> 3. `vue`中，对具名`slot`关于`context`处理的[源码](https://github.com/vuejs/vue/blob/dev/src/core/instance/render-helpers/resolve-slots.js#L23-L34)部分
> 4. `vue`中，`_self`指向自己[源码](https://github.com/vuejs/vue/blob/dev/src/core/instance/init.js#L50-L51)部分
> 5. `vue`的`vNode`定义在[源码](https://github.com/vuejs/vue/blob/dev/src/core/vdom/vnode.js#L10)中的位
> 6. `vue 3.0`中，对于`context`将不绑定到`vNode`实例上，这意味着更加灵活的组件声明位置（不止在.vue文件中，不需要到处传递h参数）; 详见[vue rfcs](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0008-render-function-api-change.md), 其中原文内容： `In 3.0 we have found ways to make VNodes context-free. They can now be created anywhere using the globally imported h function, so it only needs to be imported once in any file.`
> 7. 社区关于`vue HOC` 的讨论 [Discussion: Best way to create a HOC](https://github.com/vuejs/vue/issues/6201)