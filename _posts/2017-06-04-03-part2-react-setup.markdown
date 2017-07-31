---
title: React 设置
layout: post
date: 2017-06-04 23:59:59
permalink: part-two-react-setup
share: true
---

让我们把焦点放到前端，添加[React](https://facebook.github.io/react/)……

---

React是声明式的，基于组件用来构建用户界面的JavaScript库。

如果你对于React还陌生的话，通过[快速开始](https://facebook.github.io/react/docs/hello-world.html)和精彩的 [我们为什么要构建React](https://facebook.github.io/react/blog/2013/06/05/why-react.html)博文。你也可能想通过[介绍React](https://github.com/mjhea0/react-intro) 教程深入了解相关Babel和Webpack。

在继续前确保你已经安装了 [Node](https://nodejs.org/en/) 和 [NPM](https://www.npmjs.com/)：

```sh
$ node -v
v7.10.0
$ npm -v
4.2.0
```

#### 项目配置

我将使用非常好用的[Create React App](https://github.com/facebookincubator/create-react-app)工具来帮我们生成骨架。

> 确保你已经了解了Webpack和babel。如果还没有你可以参考[Intro to React](https://github.com/mjhea0/react-intro)教程。

将Create React App安装到全局：

```sh
$ npm install create-react-app@1.3.0 --global
```

切换到*flask-microservices-client*目录，然后创建骨架：

```sh
$ create-react-app .
```

我们还需要将依赖安装，一旦安装完，我们就可以开启服务了：

```sh
$ npm start
```

现在我们可以开始构建我们第一个组件！

#### 第一个组件

首先，为了结构简单，从src目录移除 *App.css*, *App.js*, *App.test.js*, 和 *index.css*， 然后更新index.js：

```javascript
import React from 'react';
import ReactDOM from 'react-dom';

const App = () => {
  return (
    <div className="container">
      <div className="row">
        <div className="col-md-4">
          <br/>
          <h1>All Users</h1>
          <hr/><br/>
        </div>
      </div>
    </div>
  )
}

ReactDOM.render(
  <App />,
  document.getElementById('root')
);
```

发生了什么？

1. 在导入了`React`和`ReactDOM`类后，我们创建一个叫做Ap的p函数组件，它直接返回JSX。
1. 然后我们使用`ReactDOM`的`render()`方法将App挂载到HTML元素ID为`root`下。

    > Take note of `<div id="root"></div>` within the *index.html* file in the "public" folder.

添加Bootstrap到*index.html*的`head`元素里：

```html
<link
  href="//maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css"
  rel="stylesheet"
>
```

#### 基于类的组件

更新 *index.js*:

```javascript
import React, { Component } from 'react';
import ReactDOM from 'react-dom';

class App extends Component {
  constructor() {
    super()
  }
  render() {
    return (
      <div className="container">
        <div className="row">
          <div className="col-md-4">
            <br/>
            <h1>All Users</h1>
            <hr/><br/>
          </div>
        </div>
      </div>
    )
  }
}

ReactDOM.render(
  <App />,
  document.getElementById('root')
);
```

发生了什么？

1. 我们创建了基于类的组件，渲染时（内部）自动构建
1. 在运行时，`super()`将会调用`App`继承类`Component`的构造器

你可能注意到，根之前的显示没有变化

#### AJAX

要连接客户端到服务器，给`App`类添加一个`getUsers`方法：

```javascript
getUsers() {
  axios.get(`${process.env.REACT_APP_USERS_SERVICE_URL}/users`)
  .then((res) => { console.log(res); })
  .catch((err) => { console.log(err); })
}
```

我们将使用 [Axios](https://github.com/mzabriskie/axios)进行AJAX操作：

```sh
npm install axios@0.16.2 --save
```

添加导入：

```javascript
import axios from 'axios';
```

现在代码应该是这样子：

```javascript
import React, { Component } from 'react';
import ReactDOM from 'react-dom';
import axios from 'axios';

class App extends Component {
  constructor() {
    super()
  }
  getUsers() {
    axios.get(`${process.env.REACT_APP_USERS_SERVICE_URL}/users`)
    .then((res) => { console.log(res); })
    .catch((err) => { console.log(err); })
  }
  render() {
    return (
      <div className="container">
        <div className="row">
          <div className="col-md-4">
            <br/>
            <h1>All Users</h1>
            <hr/><br/>
          </div>
        </div>
      </div>
    )
  }
}

ReactDOM.render(
  <App />,
  document.getElementById('root')
);
```

要将它连接Flask，打开一个新的终端窗口，切换到*flask-microservices-users*，激活虚拟环境，配置环境变量：

```sh
$ source env/bin/activate
$ export APP_SETTINGS=project.config.DevelopmentConfig
$ export DATABASE_URL=postgres://postgres:postgres@localhost:5432/users_dev
$ export DATABASE_TEST_URL=postgres://postgres:postgres@localhost:5432/users_test
```

> 你可以要根据你本机Postgres更换环境变量里面配置的密码。

跑起你本机的Postgre服务，创建并初始化数据库：

```sh
$ python manage.py recreate_db
$ python manage.py seed_db
$ python manage.py runserver -p 5555
```

你的服务应该监听在[http://localhost:5555](http://localhost:5555)。在浏览器访问[http://localhost:5555/users](http://localhost:5555/users)来进行验证。

我们回到React，我们需要添加[环境变量](https://github.com/facebookincubator/create-react-app/blob/master/packages/react-scripts/template/README.md#adding-custom-environment-variables) `process.env.REACT_APP_USERS_SERVICE_URL`。停止Create React App服务，然后执行：

```sh
$ export REACT_APP_USERS_SERVICE_URL=http://localhost:5555
```

> 所有的自定义环境变量必须使用`REACT_APP_`开头。要了解更多，查看[官方文档](https://github.com/facebookincubator/create-react-app/blob/master/packages/react-scripts/template/README.md#adding-custom-environment-variables)。

我还需要调用`getUsers()`方法，可以在构造器`constructor()`执行：

```javascript
constructor() {
  super()
  this.getUsers()
}
```
通过`npm start`启动服务，打开[Chrome DevTools](https://developer.chrome.com/devtools)里的JavaScript控制台。你应该看到了如下错误：

```
XMLHttpRequest cannot load http://localhost:5555/users. No 'Access-Control-Allow-Origin' header is present on the requested resource. Origin 'http://localhost:3000' is therefore not allowed access.
```

简单来说，我们使用了[跨域](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing) AJAX请求(从 `http://localhost:3000` 到 `http://localhost:5555`), 违反了浏览器的“同源策略"。我们使用[Flask-CORS](https://flask-cors.readthedocs.io/en/latest/)扩展来解决该问题。

进入 *flask-microservices-users* 项目目录, 终止服务并安装 Flask-CORS：

```sh
(env)$ pip install flask-cors==3.0.2
(env)$ pip freeze > requirements.txt
```

为了简单，让我们运行所有路由允许跨域请求。简单的更新*flask-microservices-users/project/__init__.py文件里的`create_app()：

```python
def create_app():

    # instantiate the app
    app = Flask(__name__)

    # enable CORS
    CORS(app)

    # set config
    app_settings = os.getenv('APP_SETTINGS')
    app.config.from_object(app_settings)

    # set up extensions
    db.init_app(app)

    # register blueprints
    from project.api.views import users_blueprint
    app.register_blueprint(users_blueprint)

    return app
```
导入包：

```python
from flask_cors import CORS
```

要进行测试，重启动起两个服务，再次打开JavaScript控制台，这一次已将看到`console.log(res);`结果。现在我们来解析JSON对象：

```javascript
getUsers() {
  axios.get(`${process.env.REACT_APP_USERS_SERVICE_URL}/users`)
  .then((res) => { console.log(res.data.data); })
  .catch((err) => { console.log(err); })
}
```

现在你应该可以在JavaScript控制台看到包含两个对象的数组。

在我们继续前，我们需要快速重构下，还记得我们如何在构造器里调用`getUsers()`方法吗？

```javascript
constructor() {
  super()
  this.getUsers()
}
```

constructor()在component挂载到DOM前执行，但是如果AJAX请求时间超长，那么component就必须要等待？看看[竞态](https://en.wikipedia.org/wiki/Race_condition)。
不过，React通过生命周期方法解决了该问题。

#### 组件生命周期方法

基于类的组件含有几个函数，它将在组件生命中某个特定时间会执行。这些称为Lifecyle Methods（生命周期方法）。快速查看[官方文档](https://facebook.github.io/react/docs/react-component.html#the-component-lifecycle)学习每个方法。

AJAX调用 [应该在 `componentDidMount()` 方法执行](https://daveceddia.com/where-fetch-data-componentwillmount-vs-componentdidmount/):

```javascript
componentDidMount() {
  this.getUsers();
}
```

更新组件:

```javascript
class App extends Component {
  constructor() {
    super()
  }
  componentDidMount() {
    this.getUsers();
  }
  getUsers() {
    axios.get(`${process.env.REACT_APP_USERS_SERVICE_URL}/users`)
    .then((res) => { console.log(res.data.data.users); })
    .catch((err) => { console.log(err); })
  }
  render() {
    return (
      <div className="container">
        <div className="row">
          <div className="col-md-4">
            <br/>
            <h1>All Users</h1>
            <hr/><br/>
          </div>
        </div>
      </div>
    )
  }
}
```

确保一切依然正常工作。

#### state

我们来添加 [state](https://en.wikipedia.org/wiki/State_(computer_science)) - 例如，用户到组件，我们需要使用`setState()`，异步更新state.

更新 `getUsers()`:

```javascript
getUsers() {
  axios.get(`${process.env.REACT_APP_USERS_SERVICE_URL}/users`)
  .then((res) => { this.setState({ users: res.data.data.users }); })
  .catch((err) => { console.log(err); })
}
```

在构造器中设置state:

```javascript
constructor() {
  super()
  this.state = {
    users: []
  }
}
```

这里，我们使用``this.state`给类添加state`属性`，并且初始化`users`为空数组。

> 查看官方文档 [正确使用State](https://facebook.github.io/react/docs/state-and-lifecycle.html#using-state-correctly).

最终，更新`render()`方法来显示AJAX返回的用户。

```javascript
render() {
  return (
    <div className="container">
      <div className="row">
        <div className="col-md-6">
          <br/>
          <h1>All Users</h1>
          <hr/><br/>
          {
            this.state.users.map((user) => {
              return <h4 key={user.id} className="well"><strong>{ user.username }</strong> - <em>{user.created_at}</em></h4>
            })
          }
        </div>
      </div>
    </div>
  )
}
```

发生了什么？

1. 我们迭代用户（从AJAX请求返回）以及创建了一个新的H4元素。这也是为什么我们先要创建一个空数组，为了放置`map`报错。
1. `key`? - React追踪具体元素. 查看[官方文档](https://facebook.github.io/react/docs/lists-and-keys.html#keys)了解更多。

#### 函数 Component

让我们为用户列表构建一个新的组件。添加一个新的"components"目录到"src"，在该目录下建立一个*UsersList.jsx*文件：

```javascript
import React from 'react';

const UsersList = (props) => {
  return (
    <div>
      {
        props.users.map((user) => {
          return <h4 key={user.id} className="well"><strong>{user.username }</strong> - <em>{user.created_at}</em></h4>
        })
      }
    </div>
  )
}

export default UsersList;
```

> 为什么我们这用函数式组件而不是基于类的组件？

注意这里的组件我们使用`props`而不是`state`。大致上，组件都支持`prop`和`state`传递：

1. Props - 通过 `props`数据流下传 (从 `state` 到 `props`), 只读
1. State - 数据绑定给组件, 读写

    > 要了解更多，查看 [ReactJS: Props vs. State](http://lucybain.com/blog/2016/react-state-vs-pros/)

实践中尽量限制基于类（有状态）的组件来修改state，如果你仅仅是要呈现数据（如上），使用函数式（无状态）组件。

现在，我们需要通过上级传递state到子组件的`props`中，首先，导入到*index.js*:

```javascript
import UsersList from './components/UsersList';
```

然后更新 `render()` 方法:

```javascript
render() {
  return (
    <div className="container">
      <div className="row">
        <div className="col-md-6">
          <br/>
          <h1>All Users</h1>
          <hr/><br/>
            <UsersList users={this.state.users}/>
        </div>
      </div>
    </div>
  )
}
```

检查每个组件代码，然后提交你的代码。
