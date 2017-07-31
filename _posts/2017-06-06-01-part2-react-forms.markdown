---
title: React 表单
layout: post
date: 2017-06-06 23:59:58
permalink: part-two-react-forms
share: true
---

在本节，我们将创建一个函数式组件用来创建新的用户……

---

在项目 *flask-microservices-client*, 添加一个新的 *AddUser.jsx* 文件到 "components" 目录里:

```javascript
import React from 'react';

const AddUser = (props) => {
  return (
    <form>
      <div className="form-group">
        <input
          name="username"
          className="form-control input-lg"
          type="text"
          placeholder="Enter a username"
          required
        />
      </div>
      <div className="form-group">
        <input
          name="email"
          className="form-control input-lg"
          type="email"
          placeholder="Enter an email address"
          required
        />
      </div>
      <input
        type="submit"
        className="btn btn-primary btn-lg btn-block"
        value="Submit"
      />
    </form>
  )
}

export default AddUser;
```

在 *index.js* 导入组件:

```javascript
import AddUser from './components/AddUser';
```

然后更新 `render` 方法：

```javascript
render() {
  return (
    <div className="container">
      <div className="row">
        <div className="col-md-6">
          <br/>
          <h1>All Users</h1>
          <hr/><br/>
          <AddUser/>
          <br/>
          <UsersList users={this.state.users}/>
        </div>
      </div>
    </div>
  )
}
```

确保`dev`机器开启，设置`REACT_APP_USERS_SERVICE_URL`环境变量设置为实际`dev`机器关联的IP地址。执行`npm start`进行测试。如果正常的话，你应该可以看到紧随这用户列表多了一个表单。

现在，由于这是一个单页面应用，我们想移除掉表单提交后刷新页面的默认行为：

*Steps*:

1. 处理表单提交事件
1. 获取用户输入
1. 发送AJAX请求
1. 更新页面

#### 处理表单事件

要处理表单事件，更新*AddUser.jsx*里的`form`元素：

```javascript
<form onSubmit={(event) => event.preventDefault()}>
```

尝试提交表单，应该没有任何事情发生，正是我们想要的 —— 组织浏览器默认行为。

接下来，添加如下方法到 `App`组件中：

```javascript
addUser(event) {
  event.preventDefault();
  console.log('sanity check!')
}
```

由于 `AddUser` 是函数式组件，我们需要通过props传递该方法到下游。更新 `AddUser` 元素如下：

```javascript
<AddUser addUser={this.addUser.bind(this)}/>
```

这里，我们使用`bind()`手动绑定上下文`this`。如果没有这样的话，方法内部就不用正确引用正确的上下文。想验证的话，简单在`addUser()`加入`console.log(this)`，提交表单验证下。上下文是什么？移除`bind()`再次测试看看上下文是什么？

> 更多关于该信息查看官方文档 [事情处理](https://facebook.github.io/react/docs/handling-events.html).

Update the `form` element again:

```javascript
<form onSubmit={(event) => props.addUser(event)}>
```

在浏览器中测试，你应该可以在JavaScript控制台中看到`sanity check!`输出。

#### 获取用户输入

我们将使用 [受控组件](https://facebook.github.io/react/docs/forms.html#controlled-components)来获取用户提交。

添加两个新属性到state对象：

```javascript
this.state = {
  users: [],
  username: '',
  email: ''
}
```

然后传递到组件：

```javascript
<AddUser
  username={this.state.username}
  email={this.state.email}
  addUser={this.addUser.bind(this)}
/>
```

现在这个可以通过`props`访问，可以用来获取到当前的输入值：

```javascript
<div className="form-group">
  <input
    name="username"
    className="form-control input-lg"
    type="text"
    placeholder="Enter a username"
    required
    value={props.username}
  />
</div>
<div className="form-group">
  <input
    name="email"
    className="form-control input-lg"
    type="email"
    placeholder="Enter an email address"
    required
    value={props.email}
  />
</div>
```

所以，这里通过父空间来定义输入值。测试表单，发生了什么？你应该在输入框不能键入任何内容，因为这些值都是由父空间推入的。

> 如果初始值该为`test`又会是什么情况呢？试试看。

那么我们如何在父组件来更新state？

首先，添加一个`handleChange`方法到`App`组件中：

```javascript
handleChange(event) {
  const obj = {};
  obj[event.target.name] = event.target.value;
  this.setState(obj);
}
```

传递到控件:

```javascript
<AddUser
  username={this.state.username}
  email={this.state.email}
  handleChange={this.handleChange.bind(this)}
  addUser={this.addUser.bind(this)}
/>
```

添加到form输入框中：

```javascript
<div className="form-group">
  <input
    name="username"
    className="form-control input-lg"
    type="text"
    placeholder="Enter a username"
    required
    value={props.username}
    onChange={props.handleChange}
  />
</div>
<div className="form-group">
  <input
    name="email"
    className="form-control input-lg"
    type="email"
    placeholder="Enter an email address"
    required
    value={props.email}
    onChange={props.handleChange}
  />
</div>
```

现在测试下表单，现在应该正常工作了,你可以如下一样加入日志输出到`addUser`在控制台查看state的值：

```javascript
addUser(event) {
  event.preventDefault();
  console.log(this.state);
}
```

现在我们已经取到值了，让我们通过AJAX发送输出保存到数据库，并且更新DOM……

#### 发送AJAX请求

回到 *flask-microservices-users*. 我们需要什么JSON结构来添加用户- username 和 email, 对吗? 我们使用 Axios 来发送POST请求。

```javascript
addUser(event) {
  event.preventDefault();
  const data = {
    username: this.state.username,
    email: this.state.email
  }
  axios.post(`${process.env.REACT_APP_USERS_SERVICE_URL}/users`, data)
  .then((res) => { console.log(res); })
  .catch((err) => { console.log(err); })
}
```

测试，如果邮箱唯一的话应该都是正常工作的。
Test it out. It should work as long as the email address is unique.

> 如果你遇到问题了，分析一下开发者工具里"network"选项里的response对象，你也可以在Docker外启动*flask-microservices-users*，使用Flask debugger或者`print`语句诊断。

#### 更新页面

最后，让我们在成功提交后更新用户列表并清理表单。

```javascript
addUser(event) {
  event.preventDefault();
  const data = {
    username: this.state.username,
    email: this.state.email
  }
  axios.post(`${process.env.REACT_APP_USERS_SERVICE_URL}/users`, data)
  .then((res) => {
    this.getUsers();
    this.setState({ username: '', email: '' });
  })
  .catch((err) => { console.log(err); })
}
```

搞定，测试一下。审查并提交你的代码。
