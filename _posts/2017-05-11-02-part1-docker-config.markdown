---
title: Docker配置
layout: post
date: 2017-05-11 23:59:59
permalink: part-one-docker-config
share: true
---

让我们将Flask app放入容器里

---

我们先确保机器已经安装了Docker，Docker Compose和Docker Machine：

```sh
$ docker -v
Docker version 17.03.1-ce, build c6d412e
$ docker-compose -v
docker-compose version 1.11.2, build dfed245
$ docker-machine -v
docker-machine version 0.10.0, build 76ed2a6
```

接下来，我们需要通过[Docker Machine](https://docs.docker.com/machine/)创建Docker主机，并且将Docker client指向它：

```sh
$ docker-machine create -d virtualbox dev
$ eval "$(docker-machine env dev)"
```

>想了解更多上面命令查看 [这里](https://stackoverflow.com/questions/40038572/eval-docker-machine-env-default/40040077#40040077).

添加*Dockerfile*到跟目录，确保代码如下所示：

```
FROM python:3.6.1

# set working directory
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

# add requirements (to leverage Docker cache)
ADD ./requirements.txt /usr/src/app/requirements.txt

# install requirements
RUN pip install -r requirements.txt

# add app
ADD . /usr/src/app

# run server
CMD python manage.py runserver -h 0.0.0.0
```

然后添加 *docker-compose.yml* 文件到根目录下：

```
version: '2.1'

services:

  users-service:
    container_name: users-service
    build: .
    volumes:
      - '.:/usr/src/app'
    ports:
      - 5001:5000 # expose ports - HOST:CONTAINER
```

该配置用来通过Dockerfile创建一个`users-service`容器。

> *docker-compose.yml* 文件使用的是相对路径。

`volume`用来将代码挂载到容器里。这个用来在开发环境中更新代码所用。没有这个，你就每次更改代码都要重新构建镜像。

注意到在[Docker compose文件版本(version)](https://docs.docker.com/compose/compose-file/) 使用了`2.1`，记住了它*不是**指明Docker Compose安装的版本，仅仅用来指明使用哪种格式。

构建镜像：

```sh
$ docker-compose build
```

首次构建时需要花费一段时间。后面再构建的话就会很快，Docker会在第一次构建后会缓存构建结果。完成后，启动容器：

```sh
$ docker-compose up -d
```

>`-d`参数用来指明容器运行在后端。

获取机器IP地址：

```sh
$ docker-machine ip dev
```

访问[http://YOUR-IP:5001/ping](http://YOUR-IP:5001/ping)。确保看到了会之前一样的JSON结果输出。接下来，配置*docker-compose.yml*加载开发环境配置：

```
version: '2.1'

services:

  users-service:
    container_name: users-service
    build: .
    volumes:
      - '.:/usr/src/app'
    ports:
      - 5001:5000 # expose ports - HOST:CONTAINER
    environment:
      - APP_SETTINGS=project.config.DevelopmentConfig
```

然后更新 *project/\_\_init\_\_.py* 通过环境变量来加载配置：

```python
# project/__init__.py


import os
from flask import Flask, jsonify


# instantiate the app
app = Flask(__name__)

# set config
app_settings = os.getenv('APP_SETTINGS')
app.config.from_object(app_settings)


@app.route('/ping', methods=['GET'])
def ping_pong():
    return jsonify({
        'status': 'success',
        'message': 'pong!'
    })
```

更新容器：

```sh
$ docker-compose up -d
```

如果向测试的话，在 *\_\_init\_\_.py*路由处理之前增加`print`语句，查看app的config来确保工作：

```python
print(app.config)
```
然后看看日志输出：

```sh
$ docker-compose logs -f users-service
```

你应该会看到如下输出：

```
<Config {
  'DEBUG': True, 'TESTING': False, 'PROPAGATE_EXCEPTIONS': None,
  'PRESERVE_CONTEXT_ON_EXCEPTION': None, 'SECRET_KEY': None,
  'PERMANENT_SESSION_LIFETIME': datetime.timedelta(31), 'USE_X_SENDFILE':
  False, 'LOGGER_NAME': 'project', 'LOGGER_HANDLER_POLICY': 'always',
  'SERVER_NAME': None, 'APPLICATION_ROOT': None, 'SESSION_COOKIE_NAME':
  'session', 'SESSION_COOKIE_DOMAIN': None, 'SESSION_COOKIE_PATH': None,
  'SESSION_COOKIE_HTTPONLY': True, 'SESSION_COOKIE_SECURE': False,
  'SESSION_REFRESH_EACH_REQUEST': True, 'MAX_CONTENT_LENGTH': None,
  'SEND_FILE_MAX_AGE_DEFAULT': datetime.timedelta(0, 43200),
  'TRAP_BAD_REQUEST_ERRORS': False, 'TRAP_HTTP_EXCEPTIONS': False,
  'EXPLAIN_TEMPLATE_LOADING': False, 'PREFERRED_URL_SCHEME': 'http',
  'JSON_AS_ASCII': True, 'JSON_SORT_KEYS': True,
  'JSONIFY_PRETTYPRINT_REGULAR': True, 'JSONIFY_MIMETYPE':
  'application/json', 'TEMPLATES_AUTO_RELOAD': None}
>
```

在继续之前确保移除`print`语句。
