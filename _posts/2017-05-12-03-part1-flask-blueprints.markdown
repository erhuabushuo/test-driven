---
title: Flask 蓝图
layout: post
date: 2017-05-12 23:59:59
permalink: part-one-flask-blueprints
share: true
---

通过相应的测试，我们来重构应用，加入蓝图(Blueprints)……

---

> 不熟悉Blueprints? 查看 [Flask官方文档](http://flask.pocoo.org/docs/0.12/blueprints/). 基本上它是独立组件，用来包装代码，模板和静态文件。

创建一个新目录"api" 到"project"目录下，增加一个*\_\_init\_\_.py*文件以及*views.py*和*models.py*。然后修改*views.py*如下内容：

```python
# project/api/views.py


from flask import Blueprint, jsonify


users_blueprint = Blueprint('users', __name__)


@users_blueprint.route('/ping', methods=['GET'])
def ping_pong():
    return jsonify({
        'status': 'success',
        'message': 'pong!'
    })
```

在这里，我们创建了`Blueprint`类实例并且绑定了`ping_pong()`函数。

*models.py*:

```python
# project/api/models.py


import datetime

from project import db


class User(db.Model):
    __tablename__ = "users"
    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    username = db.Column(db.String(128), nullable=False)
    email = db.Column(db.String(128), nullable=False)
    active = db.Column(db.Boolean, default=False, nullable=False)
    created_at = db.Column(db.DateTime, nullable=False)

    def __init__(self, username, email):
        self.username = username
        self.email = email
        self.created_at = datetime.datetime.utcnow()
```

更新 *project/\_\_init\_\_.py*

```python
# project/__init__.py


import os
from flask import Flask, jsonify
from flask_sqlalchemy import SQLAlchemy


# instantiate the db
db = SQLAlchemy()


def create_app():

    # instantiate the app
    app = Flask(__name__)

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

更新 *manage.py*:

```python
# manage.py


import unittest

from flask_script import Manager

from project import create_app, db
from project.api.models import User


app = create_app()
manager = Manager(app)


@manager.command
def test():
    """Runs the unit tests without test coverage."""
    tests = unittest.TestLoader().discover('project/tests', pattern='test*.py')
    result = unittest.TextTestRunner(verbosity=2).run(tests)
    if result.wasSuccessful():
        return 0
    return 1


@manager.command
def recreate_db():
    """Recreates a database."""
    db.drop_all()
    db.create_all()
    db.session.commit()


if __name__ == '__main__':
    manager.run()
```

更新*project/tests/base.py* 和 *project/tests/test_config.py*里app导入：

```python
from project import create_app

app = create_app()
```

( `db` 在 *base.py* 导入依赖保留)

测试！

```sh
$ docker-compose up -d
$ docker-compose run users-service python manage.py recreate_db
$ docker-compose run users-service python manage.py test
```

修正相应错误然后我们继续前进……
