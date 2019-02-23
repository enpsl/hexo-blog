---
layout:     post
title:      "Dva快速入门"
subtitle:   "Dva快速入门"
date:       2019-02-24 22:30:00
author:     "Psl"
catalog:    true
tags:
  - Dva
  - react
  - redux
---

## 安装初始化项目

```bash
npm install -g dva-cli
# 或
yarn global add dva-cli
```

```bash
dva -v
dva-cli version 0.10.0
dva version 2.4.1
roadhog version 2.4.9
@local
```

```bash
dva new dva-quickstart
yarn install
```
## 目录说明
<img src="/img/in-post/2019-02-23/2.png" align=left width=100%/>

src文件夹内容：
```bash
src/
├── assets
│   └── yay.jpg
├── components
│   └── Example.js
├── index.css
├── index.js
├── models
│   └── example.js
├── router.js
├── routes
│   ├── IndexPage.css
│   └── IndexPage.js
├── services
│   └── example.js
└── utils
    └── request.js
```
入口文件index.js默认内容
```jsx harmony
import dva from 'dva';
import './index.css';

// 1. Initialize
const app = dva();

// 2. Plugins
// app.use({});

// 3. Model
// app.model(require('./models/example').default);

// 4. Router
app.router(require('./router').default);

// 5. Start
app.start('#root');

```
路由文件默认内容：
```jsx harmony
import React from 'react';
import { Router, Route, Switch } from 'dva/router';
import IndexPage from './routes/IndexPage';

function RouterConfig({ history }) {
  return (
    <Router history={history}>
      <Switch>
        <Route path="/" exact component={IndexPage} />
      </Switch>
    </Router>
  );
}

export default RouterConfig;
```

执行yarn start 即可看到页面效果
![](/img/in-post/2019-02-23/1.png)

## 计数器项目

### Router切换
<img src="/img/in-post/2019-02-23/3.png" align=left width=100%/>

初始化项目的路由默认为HashRouter，路由链接不太友好更换为HashRouter
替换index.js文件内容
```jsx harmony
import dva from 'dva';
import { createBrowserHistory as createHistory } from 'history';
import './index.css';

// 1. Initialize
const app = dva({
  history: createHistory()
});

// 2. Plugins
// app.use({});

// 3. Model
// app.model(require('./models/example').default);

// 4. Router
app.router(require('./router').default);

// 5. Start
app.start('#root');

```

再次访问[http://localhost:8000](http://localhost:8000),路由正常显示
<img src="/img/in-post/2019-02-23/4.png" align=left width=100%/>

### 编写Counter组件

新建 src/components/Counter.js

```jsx harmony
import React from 'react'

const Counter = ({count}) => {
  return (
    <div>
      <h1>1</h1>
      <button>+</button>
    </div>
  );
};
export default Counter
```

新建 src/routes/CounterPage.js

```jsx harmony
import React from 'react'
import Counter from '../components/Counter'

export const CounterPage = (props) => {
  return (
    <div>
      <p>counter</p>
      <Counter/>
    </div>
  )
};

export default CounterPage
```
### 配置路由
在src/router.js中配置路由
```jsx harmony
import CounterPage from './routes/CounterPage';

<Route path="/counter" exact component={CounterPage} />
```

页面效果
![](/img/in-post/2019-02-23/5.png)

### 定义model

新建 src/models/counter.js, copy models下面的Example.js内容到counter.js中，然后修改state和namespace内容

```jsx harmony
export default {

  namespace: 'counter',

  state: {
    count: 2
  },

  subscriptions: {
    setup({ dispatch, history }) {  // eslint-disable-line
    },
  },

  effects: {
    *fetch({ payload }, { call, put }) {  // eslint-disable-line
      yield put({ type: 'save' });
    },
  },

  reducers: {
    save(state, action) {
      return { ...state, ...action.payload };
    },
  },

};
```

> dva中namespace作用类似于redux项目中combineReducers中的key，用来使各个组件中的state划分更清晰

![](/img/in-post/2019-02-23/6.png)

引入到入口文件index.js文件中
```jsx harmony
app.model(require('./models/counter').default);
```

#### model中的API介绍:

参照[Dva API介绍](https://dvajs.com/api/#reducers)可了解到:

##### namespace
model 的命名空间，同时也是他在全局 state 上的属性，只能用字符串，不支持通过 . 的方式创建多层命名空间。

##### state
初始值，优先级低于传给 dva() 的 opts.initialState。

比如：
```jsx harmony
const app = dva({
  initialState: { count: 1 },
});
app.model({
  namespace: 'count',
  state: 0,
});
```

此时，在 app.start() 后 state.count 为 1 。

##### reducers

以 key/value 格式定义 reducer。用于处理同步操作，唯一可以修改 state 的地方。由 action 触发。格式为 (state, action) => newState 或 [(state, action) => newState, enhancer]。
详见:https://github.com/dvajs/dva/blob/master/packages/dva-core/test/reducers.test.js

##### effects

以 key/value 格式定义 effect。用于处理异步操作和业务逻辑，不直接修改 state。由 action 触发，可以触发 action，可以和服务器交互，可以获取全局 state 的数据等等。格式为*(action, effects) => void或[*(action, effects) => void, { type }].
type 类型有：
1. takeEvery
2. takeLatest
3. throttle
4. watcher

详见：https://github.com/dvajs/dva/blob/master/packages/dva-core/test/effects.test.js

##### subscriptions

以 key/value 格式定义 subscription。subscription 是订阅，用于订阅一个数据源，然后根据需要 dispatch 相应的 action。在 app.start() 时被执行，数据源可以是当前的时间、服务器的 websocket 连接、keyboard 输入、geolocation 变化、history 路由变化等等。格式为({ dispatch, history }, done) => unlistenFunction。

>注意：如果要使用 app.unmodel()，subscription 必须返回 unlisten 方法，用于取消数据订阅。

打开redux控制台:
![](/img/in-post/2019-02-23/7.png)

可看到state.counter.count
> counter对应model的namespace,count对应state对象中的count


### connect起来

修改src/components/Counter.js的内容

```jsx harmony
import React from 'react'
import {connect} from "dva";
const Counter = (props) => {
  return (
    <div>
      <h1>{props.counter.count}</h1>
      <button>+</button>
    </div>
  );
};
const mapStateToProps = (state) => {
  // 取出models下的命名空间counter
  return {
    counter: state.counter
  }
};
export default connect(mapStateToProps)(Counter)
```

>这一步做的主要是把models/counter和当前组件通过mapStateToProps连接然后在无状态Counter组件中通过props来接收

可以看到如果在models中的state定义的对象如果结构很长的话,Counter组件中的props链式对象会写的很长，当然我们可以通过es6中的对象析构来优化代码

优化后的代码:

```jsx harmony
import React, {Fragment} from 'react'
import {connect} from "dva";

const Counter = ({count}) => {
  return (
    <Fragment>
      <h1>{count}</h1>
      <button>+</button>
    </Fragment>
  );
};

const mapStateToProps = (state) => {
  // 取出models下的命名空间counter
  const { count } = state.counter;
  return {
    count
  }
};
export default connect(mapStateToProps)(Counter)
```

重新访问[http://localhost:8000/counter](http://localhost:8000/counter),依然正常显示

![](/img/in-post/2019-02-23/8.png)

#### 定义propTypes(数据类型检查)
```jsx harmony
import PropTypes from 'prop-types'
Counter.propTypes = {
  count: PropTypes.number
};
```

### dispatch
修改Counter组件代码，给button增加一个点击事件
```jsx harmony
const Counter = ({count, dispatch}) => {
  return (
    <Fragment>
      <h1>{count}</h1>
      <button onClick={() => {dispatch({type: "counter/add"})}}>+</button>
    </Fragment>
  );
};
```
>type中必须以namespace/action 的形式传递，否则在reducer中会捕获不到

按钮点击后会dispatch一个action{type: "counter/add"}到reducer中
控制台效果：
![](/img/in-post/2019-02-23/9.png)

### 编写reducers

修改src/models/counter.js,修改reducers部分代码
```jsx harmony
reducers: {
    add(state, action) {
      console.log(action);
      return {
        count: state.count+1
      }
    }
  }
```
修改src/components/Counter.js,增加Dispatch传递的参数
```jsx harmony
<button onClick={() => {dispatch({type: "counter/add", names: "psl"})}}>+</button>
```
页面效果：
![](/img/in-post/2019-02-23/10.png)
有时候，当项目非常复杂的时候，为了便于管理reducers中的函数，reducers中的函数也可以这么写

```jsx harmony
reducers: {
    'add'(state, action) {
      console.log(action);
      return {
        count: state.count+1
      }
    }
  }
```
增加type层级
```jsx harmony
<button onClick={() => {dispatch({type: "counter/add/num", names: "psl"})}}>+</button>
```
```jsx harmony
reducers: {
    'add/num'(state, action) {
      console.log(action);
      return {
        count: state.count+1
      }
    }
  }
```
页面效果：
![](/img/in-post/2019-02-23/11.png)

### 异步动作

#### 修改effects
dva中的异步动作要放到effects中来触发,放到effects中可以使用call,put等方法,也就是redux-saga中的一些方法
修改src/components/Counter.js,新增加一个button

```jsx harmony
<button onClick={() => {dispatch({type: "counter/asyncAdd"})}}>asyncAdd+</button>
```
修改src/models/counter.js中的effects
```jsx harmony
effects: {
    *asyncAdd({ payload }, { call, put }) {  // eslint-disable-line
      yield put({ type: 'add/num' });
    },
  },
```

>yield put({ type: 'add/num' }) 会触发reducers中的add方法，由于是同命名空间下，所以不用加入namespace层级

查看redux控制台效果
![](/img/in-post/2019-02-23/12.png)

effects中asyncAdd方法中的payload可以接收dispatch过来的额外的参数
例如:

```jsx harmony
<button onClick={() => {dispatch({type: "counter/asyncAdd", payload: "payload"})}}>asyncAdd+</button>
```

```jsx harmony
effects: {
    *asyncAdd({ payload }, { call, put }) {  // eslint-disable-line
      console.log(payload);
      yield put({ type: 'add/num' });
    },
  },
```
查看redux控制台效果
![](/img/in-post/2019-02-23/13.png)

#### 延时函数
修改src/models/counter.js中的effects

```jsx harmony
import {delay} from 'dva/saga'
effects: {
    *asyncAdd({ payload }, { call, put }) {  // eslint-disable-line
      yield call(delay, 1000);
      yield put({ type: 'add/num' });
    },
  },
```


#### select

select 用于取state中的数据
引入select:

```jsx harmony
effects: {
    *asyncAdd({ payload }, { call, put, select }) {  // eslint-disable-line
      const counter = yield select(state => state.counter);
      console.log(counter);
      yield call(delay, 1000);
      yield put({ type: 'add/num' });
    },
  },
```
查看控制台效果
![](/img/in-post/2019-02-23/14.png)

几种不同的写法:
```jsx harmony
// const counter = yield select(state => state.counter);
// const counter = yield select(({counter}) => counter);
// const counter = yield select(_ => _.counter);
const { counter } = yield select(_ => _);
console.log(counter);
```

### 路由跳转

#### withRouter
修改src/components/Counter.js

```jsx harmony
import React, {Fragment} from 'react'
import {connect} from "dva";
import PropTypes from 'prop-types'
import {withRouter} from 'dva/router'

const Counter = ({count, dispatch, history}) => {
  return (
    <Fragment>
      <h1>{count}</h1>
      <button onClick={() => {history.push("/")}}>go home</button>
      <button onClick={() => {dispatch({type: "counter/add/num", names: "psl"})}}>+</button>
      <button onClick={() => {dispatch({type: "counter/asyncAdd", payload: "payload"})}}>asyncAdd+</button>
    </Fragment>
  );
};

Counter.propTypes = {
  count: PropTypes.number
};

const mapStateToProps = (state) => {
  // 取出models下的命名空间counter
  const { count } = state.counter;
  return {
    count
  }
};
export default connect(mapStateToProps)(withRouter(Counter))
```

#### 在effects中的路由跳转

修改src/models/counter.js

引入routerRedux:
```jsx harmony
import {routerRedux} from 'dva/router'
```
在effects中增加redirect方法:
```jsx harmony
*redirect({ _ }, { call, put }) {
  yield put(routerRedux.push("/"));
}
```

修改src/components/Counter.js中的Counter组件

```jsx harmony
const Counter = ({count, dispatch, history}) => {
  return (
    <Fragment>
      <h1>{count}</h1>
      <button onClick={() => {history.push("/")}}>go home</button>
      <button onClick={() => {dispatch({type: "counter/redirect"})}}>go home by effects</button>
      <button onClick={() => {dispatch({type: "counter/add/num", names: "psl"})}}>+</button>
      <button onClick={() => {dispatch({type: "counter/asyncAdd", payload: "payload"})}}>asyncAdd+</button>
    </Fragment>
  );
};
```

#### 带参数的跳转
修改src/models/counter.js

引入query-string:
```jsx harmony
import {queryString} from 'query-string'
```
修改effects中的redirect
```jsx harmony
*redirect({ _ }, { call, put }) {
  // yield put(routerRedux.push("/"));
  yield put(routerRedux.push({
    pathname: '/',
    search: queryString.stringify({
      from: "psl"
    })
  }));
}
```
点击按钮后跳转到[http://localhost:8000/?from=psl](http://localhost:8000/?from=psl)

### subscriptions

#### dispatch

监听窗口改变触发add/num方法:
```jsx harmony
subscriptions: {
    setup({ dispatch, history }) {  // eslint-disable-line
      window.onresize = () => {
        dispatch({type: 'add/num'});
      }
    },
},
```
>setup可以为多个，可以同时订阅多个监听事件

#### history

````jsx harmony
setupHistory({ dispatch, history }) {  // eslint-disable-line
  history.listen((location) => {
    console.log(location)
  })
},
````
控制台输出：

![](/img/in-post/2019-02-23/15.png)

可以发现location中包含pathname,search,hash,state等参数，我们可以通过析构对象来做一些特殊事情

例如：
```jsx harmony
setupHistory({ dispatch, history }) {  // eslint-disable-line
  history.listen(({pathname}) => {
    if (pathname === "/counter") {
      dispatch({type: 'add/num'});
    }
  })
},
```




