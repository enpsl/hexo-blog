---
layout:     post
title:      "Redux 学习(一)"
subtitle:   "Redux 学习(一)"
date:       2018-11-24 22:30:00
author:     "Psl"
catalog:    true
tags:
  - react
  - redux
---

## Reudx

由官网可看出：A JavaScript library for building user interfaces， React是一个非常简单的视图型框架

![](/img/in-post/2019-02-02/1.png)

Redux是一个流行的JavaScript框架，为应用程序提供一个可预测的状态容器。Redux基于简化版本的Flux框架，Flux是Facebook开发的一个框架。在标准的MVC框架中，数据可以在UI组件和存储之间双向流动，而Redux严格限制了数据只能在一个方向上流动

在Redux中，所有的数据（比如state）被保存在一个被称为store的容器中 → 在一个应用程序中只能有一个。store本质上是一个状态树，保存了所有对象的状态。任何UI组件都可以直接从store访问特定对象的状态。要通过本地或远程组件更改状态，需要分发一个action。分发在这里意味着将可执行信息发送到store。当一个store接收到一个action，它将把这个action代理给相关的reducer。reducer是一个纯函数，它可以查看之前的状态，执行一个action并且返回一个新的状态。

Redux=Reducer+Flux

## Redux的工作流程

![](/img/in-post/2019-02-02/2.png)

## Store创建

创建reducer
```react
const defaultState = {
    inputValue: "123",
    list: [1,2]
};

export default (state = defaultState, action) => {
    return state
}
```
创建Store并引入reducer
```react
import {createStore} from "redux";
import reducer from "./reducer";

const store = createStore(reducer);

export default store;
```

## ReduxTools使用
![](/img/in-post/2019-02-02/3.png)

使用方式详见[redux-devtools-extension](https://github.com/zalmoxisus/redux-devtools-extension#usage)

![](/img/in-post/2019-02-02/4.png)

## Action 和 Reducer
action 传递给reducer:
```react
const action = {
            type: 'change_input_value',
            value: e.target.value
        };
//传给store
store.dispatch(action);
```
store变化订阅

```react
store.subscribe(this.handleStoreChange)
```

reducer修改state:
```react
if (action.type === 'change_input_value') {
    const newState = JSON.parse(JSON.stringify(state));
    newState.inputValue = action.value;
    return newState
}
```

## 发送异步请求

react发送异步请求一般都是在componentDidMount中发送的，选择原因可总结为如下几点：

**constructor()**

constructor()中获取数据的话，如果时间太长，或者出错，组件就渲染不出来，整个页面都没法渲染了。

constructor是作组件state初绐化工作，并不是设计来作加载数据的。

**componentWillMount()**

如果使用SSR（服务端渲染）,componentWillMount会执行2次，一次在服务端，一次在客户端。而componentDidMount不会。

constructor可以完成state初始化，componentWillMount使用的很少，目前16版本加入了UNSAFE来标识componentWillMount，新的生命周期static getDerivedStateFromProps()   也会替代这个。

React16之后采用了Fiber架构，只有componentDidMount声明周期函数是确定被执行一次的，类似ComponentWillMount的生命周期钩子都有可能执行多次，所以不加以在这些生命周期中做有副作用的操作，比如请求数据之类。

**render()**

无限render

**componentDidMount()**

确保已经render过一次。提醒我们正确地设置初始状态，这样就不会得到导致错误的"undefined"状态。

componentDidMount方法中的代码，是在组件已经完全挂载到网页上才会调用被执行，所以可以保证数据的加载。此外，在这方法中调用setState方法，会触发重渲染。所以，官方设计这个方法就是用来加载外部数据用的，或处理其他的副作用代码。

```react
axios.get("/list.json").then((res) => {
    const data = res.data;
    const action = {
        type: 'init_axios_data',
        data
    };
    //传给store
    store.dispatch(action);
});
```

```react
if (action.type === 'init_axios_data') {
    const newState = JSON.parse(JSON.stringify(state));
    newState.list = action.data;
    return newState
}
```

## Redux-thunk

### 什么是Redux-thunk中间件

原生的redux的store的dispatch方法只能接收一个对象，Redux Thunk对dispatch方法做了一个升级
它允许我们写一个返回function而非action对象的action creators。Redux Thunk可以用来延迟dispatch一个action或者只在某些特定场景下才dispatch。内部函数接受store的dispatch方法和getState方法作为参数。


![](/img/in-post/2019-02-02/5.png)

### 使用：

```react
import { createStore, applyMiddleware } from 'redux';
import thunk from 'redux-thunk';
import rootReducer from './reducers/index';

// Note: this API requires redux@>=3.1.0
const store = createStore(
  rootReducer,
  applyMiddleware(thunk)
);
```

兼容redux-devtools-extension
```react
const composeEnhancers = window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__ ? window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__({}) : compose;

const enhancer = composeEnhancers(
    applyMiddleware(thunk),
);
const store = createStore(reducer, enhancer);
```

分离componentDidMount代码

```react
import {getTodoList} from "../store/actions"
componentDidMount() {
    store.dispatch(getTodoList())
}
```

创建actions：
```react
import axios from "axios"

export const getTodoList = () => {
    return (dispatch) => {
        axios.get("/list.json").then((res) => {
            const data = res.data;
            const action = {
                type: 'init_axios_data',
                data
            };
            dispatch(action);
        });
    }
};
```