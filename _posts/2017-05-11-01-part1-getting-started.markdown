---
title: 开始
layout: post
date: 2017-05-11 23:59:58
permalink: part-one-getting-started
share: true
---

本节，我们将设置基本项目结构以及定义第一个服务……

---

创建一个新项目并且安装Flask:

```sh
$ mkdir flask-microservices-users && cd flask-microservices-users
$ mkdir project
$ python3.6 -m venv env
$ source env/bin/activate
(env)$ pip install flask==0.12.2
```
添加 *\_\_init\_\_.py* 文件到"project"目录并配置第一个路由：

```python
# project/__init__.py


from flask import Flask, jsonify


# 初始化应用
app = Flask(__name__)


@app.route('/ping', methods=['GET'])
def ping_pong():
    return jsonify({
        'status': 'success',
        'message': 'pong!'
    })
```

接下来安装 [Flask-Script](https://flask-script.readthedocs.io/en/latest/), 用来在命令行执行以及管理我们的应用：

```sh
(env)$ pip install flask-script==2.0.5
```

在项目根目录下增加 *manage.py* 文件:

```python
# manage.py


from flask_script import Manager

from project import app


manager = Manager(app)


if __name__ == '__main__':
    manager.run()
```

在这里，我们创建了`Manager`实例来处理命令行命令。

运行服务器:

```sh
(env)$ python manage.py runserver
```

浏览器访问 [http://localhost:5000/ping](http://localhost:5000/ping)，你将看到：

```json
{
  "message": "pong!",
  "status": "success"
}
```

结束掉服务器，然后增加*config.py*文件到"project"目录下：

```python
# project/config.py


class BaseConfig:
    """Base configuration"""
    DEBUG = False
    TESTING = False


class DevelopmentConfig(BaseConfig):
    """Development configuration"""
    DEBUG = True


class TestingConfig(BaseConfig):
    """Testing configuration"""
    DEBUG = True
    TESTING = True


class ProductionConfig(BaseConfig):
    """Production configuration"""
    DEBUG = False
```

更新*\_\_init\_\_.py*文件初始化时获取开发环境配置：

```python
# project/__init__.py


from flask import Flask, jsonify


# instantiate the app
app = Flask(__name__)

# set config
app.config.from_object('project.config.DevelopmentConfig')


@app.route('/ping', methods=['GET'])
def ping_pong():
    return jsonify({
        'status': 'success',
        'message': 'pong!'
    })
```

再次运行应用，这个时候[debug mode](http://flask.pocoo.org/docs/0.12/quickstart/#debug-mode) 应该打开了：

```sh
$ python manage.py runserver
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 107-952-069
```

现在当你修改代码，应用将自动重新加载。确认成功后，结束服务器并且退出虚拟环境，然后在根目录增加*requirements.txt*文件：


```
Flask==0.12.1
Flask-Script==2.0.5
```

最后, 同样将*.gitignore*文件增加到项目根目录：

```
__pycache__
env
```

初始化一个git仓库并提交你的代码。
