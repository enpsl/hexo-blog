---
layout:     post
title:      "React 学习(一)"
subtitle:   "React 学习(一)"
date:       2018-11-22 22:30:00
author:     "Psl"
header-img: "img/post-bg-desk.jpg"
catalog:    true
tags:
  - react
  - Es6
---

## JSX语法
简单的jsx示例:

```react
import React, {Component} from 'react'

class JSX extends Component{
    render() {
        return <div>jsx语法</div>
    }
}
export default JSX
```
> jsx语法中 return后面不能有双引号或单引号，并且必须被包含在一个大的根节点中

典型错误示例:
```react
import React, {Component} from 'react'

class JSX extends Component{
    render() {
        return (
            <div>jsx语法</div>
            <ul>
                <li>1</li>
                <li>2</li>
            </ul>
        )
    }
}
export default JSX
```
![](/img/in-post/2019-01-26/1.jpg)

如何解决：

外层引入一个div标签或者Fragment占位符

```react
import React, {Component, Fragment} from 'react'

class JSX extends Component{
    render() {
        return (
            <Fragment>
                <div>jsx语法</div>
                <ul>
                    <li>1</li>
                    <li>2</li>
                </ul>
            </Fragment>
        )
    }
}
export default JSX
```

>推荐使用Fragment， 推荐使用Fragment占位符不生成任何虚拟dom节点

## React中的响应式设计思想和事件绑定
## 事件绑定
看一个小例子，给input框赋值一个默认值，并能够根据输入改变默认值
```react
import React, {Component, Fragment} from 'react'
class TodoList extends Component{
    constructor(props) {
        super(props)
        this.state = {
            inputValue: "hello!!!!!",
            list: [],
        }
    }
    render() {
        return (
            <Fragment>
                <div>
                    <input
                        value={this.state.inputValue}
                        onChange={this.handleInputChange.bind(this)}
                    />
                    <button>提交</button>
                </div>
                <ul>
                    <li>学英语</li>
                    <li>learn react</li>
                </ul>
            </Fragment>
        )
    }
    handleInputChange(e) {
        this.state.inputValue = e.target.value;
    }
}
export default TodoList
```
代码分析：首先程序初始化的时候inputValue为"hello!!!!!",render函数执行的时候把input的内容渲染成数据里面inputValue的值
然后给input绑定了一个onchange事件，同时用es6的bind方法传递react的this对象到handleInputChange中，在handleInputChange去
改变state中inputValue的值为手动输入的对象

## 实现一个todolist

#### todolist新增功能
利用在React中state数据变化dom也会变化的特性：

```react
import React, {Component, Fragment} from 'react'

class TodoList extends Component{
    constructor(props) {
        super(props);
        this.state = {
            inputValue: "",
            list: [],
        }
    }
    render() {
        return (
            <Fragment>
                <div>
                    <input
                        value={this.state.inputValue}
                        onChange={this.handleInputChange.bind(this)}
                    />
                    <button onClick={this.handleButtonClick.bind(this)}>提交</button>
                </div>
                <ul>
                    {
                        this.state.list.map((item, index) => {
                            return <li>{item}</li>
                        })
                    }
                </ul>
            </Fragment>
        )
    }
    handleInputChange(e) {
        this.setState({
            inputValue: e.target.value,
        });
    }
    handleButtonClick(e) {
        this.setState({
            list: [...this.state.list, this.state.inputValue],
            inputValue: "",
        });
    }
}
export default TodoList
```

> Warning: Each child in an array or iterator should have a unique "key" prop. 错误问题
es5的map方法中必须指定一个key值否则会报错

```react
<ul>
    {
        this.state.list.map((item, index) => {
            return <li key={index}>{item}</li>
        })
    }
</ul>
```

#### todolist删除功能

```react
<ul>
    {
        this.state.list.map((item, index) => {
            return <li
                key={index}
                onClick={this.handleDelete.bind(this, index)}
            >{item}</li>
        })
    }
</ul>
handleDelete (index) {
        //immutable
        //state不允许我们做任何的改变，对react性能优化有影响
        const list = [...this.state.list];
        list.splice(index, 1)
        this.setState({
            list: list
        })
    }
```

>在handleDelete方法中，不推荐使用this.state.list.splice(index, 1),原因见注释

#### label属性
```react
<label htmlFor="insertArea">输入内容</label>
<input
    id="insertArea"
    value={this.state.inputValue}
    onChange={this.handleInputChange.bind(this)}
/>
```

#### 组件拆分

我们把li标签中的内容注释掉并提取出来封装为一个组件
```react
import ToDoItem from "./ToDoItem";
<ul>
    {
        this.state.list.map((item, index) => {
            return (
                <Fragment>
                    <ToDoItem/>
                    {
                        /*<li
                            key={index}
                            onClick={this.handleDelete.bind(this, index)}
                        >{item}</li>*/
                    }
                </Fragment>
            );
        })
    }
    </ul>
```
```react
import React, {Component, Fragment} from 'react'

class ToDoItem extends Component{
    render() {
        return <li>item</li>
    }
}
export default ToDoItem
```

> 此时点击提交按钮每次新增的都是一个固定的item值，这也不是我们想要的

#### 组件动态传值
我们可以利用组件传值解决上面的问题，代码做如下修改
传递数据:
```react
<ToDoItem
    content={item}
    index={index}
/>
```
在ToDoItem中接收：
```react
return <li>{this.props.content}</li>
```

传递方法:
```react
<ToDoItem
    content={item}
    index={index}
    //把todolist这个组件的this对象强制传递给todoitem,这样在todoitem中才能调用todolist的handleDelete方法成功
    deleteItem={this.handleDelete.bind(this)}
/>
```

在ToDoItem中接收：
```react
class ToDoItem extends Component{
    constructor(props) {
        super(props);
        //通过bind去修改this指向，保证在handleClick可以使用this.props,并且在组件创建的时候
        //通过第一个执行的方法constructor去改变handleClick的this指向
        this.handleClick = this.handleClick.bind(this);
    }
    render() {
        return <li
            onClick={this.handleClick}
        >{this.props.content}</li>
    }

    handleClick() {
        this.props.deleteItem(this.props.index)
    }
}
```

单向数据流：
**子组件不能修改父组件传递的数据值，如果要修改只能通过父组件向子组件传递方法来修改**

#### 代码优化
TodoList

```react
class TodoList extends Component{
    constructor(props) {
        super(props);
        this.state = {
            inputValue: "",
            list: [],
        };
        this.handleInputChange = this.handleInputChange.bind(this);
        this.handleButtonClick = this.handleButtonClick.bind(this);
        this.handleDelete = this.handleDelete.bind(this);
    }
    render() {
        return (
            <Fragment>
                <div>
                    <label htmlFor="insertArea">输入内容</label>
                    <input
                        id="insertArea"
                        value={this.state.inputValue}
                        onChange={this.handleInputChange}
                    />
                    <button onClick={this.handleButtonClick}>提交</button>
                </div>
                <ul>
                    {this.getToDoItem()}
                </ul>
            </Fragment>
        )
    }
    handleInputChange(e) {
        const value = e.target.value;
        this.setState(() => ({
            inputValue: value,
        }));
    }
    handleButtonClick(index) {
        this.setState((prevState) => ({
            list: [...prevState.list, prevState.inputValue],
            inputValue: "",
        }));
    }
    handleDelete (index) {
        this.setState((prevState) => {
            const list = [...this.state.list];
            list.splice(index, 1);
            return {list}   //等价于{list: list}
        });
    }
    getToDoItem () {
        return this.state.list.map((item, index) => {
            return (
                <ToDoItem key={index}
                    content={item}
                    index={index}
                    deleteItem={this.handleDelete}
                />
            );
        })
    }
}
```
TodoItem
```react
class ToDoItem extends Component{
    constructor(props) {
        super(props);
        this.handleClick = this.handleClick.bind(this);
    }
    render() {
        const { content } = this.props;
        return <li
            onClick={this.handleClick}
        >{content}</li>
    }

    handleClick() {
        const {deleteItem, index} = this.props;
        deleteItem(index)
    }
}
```

## 总结：
react特点：

* 声明式开发：减少DOM操作
* 可以与其它框架并存：flex，redux框架来解决复杂组件通信
* 组件化
* 单向数据流
* 视图层框架：在嵌套包含多个组件的项目中只做视图渲染开发
* 函数式编程：方便自动化测试

