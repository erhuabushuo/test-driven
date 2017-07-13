---
title: Postgres 设置
layout: post
date: 2017-05-12 23:59:57
permalink: part-one-postgres-setup
share: true
---

本节，我们将配置Postgres，将它执行在另一个容器中，并且链接`users-service`容器……

---

添加 [Flask-SQLAlchemy](http://flask-sqlalchemy.pocoo.org/) 和 psycopg2 to 到 *requirements.txt* 文件:

```
Flask-SQLAlchemy==2.2
psycopg2==2.7.1
```

更新 *config.py*:

```python
# project/config.py


import os


class BaseConfig:
    """Base configuration"""
    DEBUG = False
    TESTING = False
    SQLALCHEMY_TRACK_MODIFICATIONS = False


class DevelopmentConfig(BaseConfig):
    """Development configuration"""
    DEBUG = True
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL')


class TestingConfig(BaseConfig):
    """Testing configuration"""
    DEBUG = True
    TESTING = True
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_TEST_URL')


class ProductionConfig(BaseConfig):
    """Production configuration"""
    DEBUG = False
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL')
```

更新 *\_\_init\_\_.py*, 创建SQLAchemy实例然后定义数据库模型：
```python
# project/__init__.py


import os
import datetime
from flask import Flask, jsonify
from flask_sqlalchemy import SQLAlchemy


# instantiate the app
app = Flask(__name__)

# set config
app_settings = os.getenv('APP_SETTINGS')
app.config.from_object(app_settings)

# instantiate the db
db = SQLAlchemy(app)

# model
class User(db.Model):
    __tablename__ = "users"
    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    username = db.Column(db.String(128), nullable=False)
    email = db.Column(db.String(128), nullable=False)
    active = db.Column(db.Boolean(), default=False, nullable=False)
    created_at = db.Column(db.DateTime, nullable=False)

    def __init__(self, username, email):
        self.username = username
        self.email = email
        self.created_at = datetime.datetime.utcnow()


# routes

@app.route('/ping', methods=['GET'])
def ping_pong():
    return jsonify({
        'status': 'success',
        'message': 'pong!'
    })
```

在"project"目录下新建"db"目录，创建*create.sql*文件到该目录：

```sql
CREATE DATABASE users_prod;
CREATE DATABASE users_dev;
CREATE DATABASE users_test;
```

接下来，在相同目录添加*Dockerfile*文件：

```
FROM postgres

# run create.sql on init
ADD create.sql /docker-entrypoint-initdb.d
```

这里，我们继承了官方Postgres镜像，并且将SQL文件放入到"docker-entrypoint-initdb.d"目录下，它将在初始化时执行。

更新 *docker-compose.yml*:

```
version: '2.1'

services:

  users-db:
    container_name: users-db
    build: ./project/db
    ports:
        - 5435:5432  # expose ports - HOST:CONTAINER
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    healthcheck:
      test: exit 0

  users-service:
    container_name: users-service
    build: ./
    volumes:
      - '.:/usr/src/app'
    ports:
      - 5001:5000 # expose ports - HOST:CONTAINER
    environment:
      - APP_SETTINGS=project.config.DevelopmentConfig
      - DATABASE_URL=postgres://postgres:postgres@users-db:5432/users_dev
      - DATABASE_TEST_URL=postgres://postgres:postgres@users-db:5432/users_test
    depends_on:
      users-db:
        condition: service_healthy
    links:
      - users-db
```

一旦运行，将会设置一些环境变量并且容器成功启动运行会发送0退出码。Postgres可以通过宿主机`5435`端口访问，其他容器则可以通过`5432`进行访问。

检查：

```sh
$ docker-compose up -d --build
```

更新 *manage.py*:

```python
# manage.py


from flask_script import Manager

from project import app, db


manager = Manager(app)


@manager.command
def recreate_db():
    """Recreates a database."""
    db.drop_all()
    db.create_all()
    db.session.commit()


if __name__ == '__main__':
    manager.run()
```

这里将注册一个`recreate_db`新命令，这样我们可以对开发数据库进行相应操作：
```
$ docker-compose run users-service python manage.py recreate_db
```

要检查是否工作，可以使用psql进行验证……

```sh
$ docker exec -ti $(docker ps -aqf "name=users-db") psql -U postgres

# \c users_dev
You are now connected to database "users_dev" as user "postgres".

# \dt
         List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | users | table | postgres
(1 row)

# \q
```
