---
title: 测试配置
layout: post
date: 2017-05-12 23:59:58
permalink: part-one-test-setup
share: true
---

让我们来针对我们的endpoint构建测试……

---

添加"tests"目录到"project"目录下，然后在新创建的目录再创建如下文件：

```sh
$ touch __init__.py base.py test_config.py test_users.py
```

更新每个文件……

*\_\_init\_\_.py*:

```python
# project/tests/__init__.py
```

*base.py*:

```python
# project/tests/base.py


from flask_testing import TestCase

from project import app, db


class BaseTestCase(TestCase):
    def create_app(self):
        app.config.from_object('project.config.TestingConfig')
        return app

    def setUp(self):
        db.create_all()
        db.session.commit()

    def tearDown(self):
        db.session.remove()
        db.drop_all()
```

*test_config.py*:

```python
# project/tests/test_config.py


import unittest

from flask import current_app
from flask_testing import TestCase

from project import app


class TestDevelopmentConfig(TestCase):
    def create_app(self):
        app.config.from_object('project.config.DevelopmentConfig')
        return app

    def test_app_is_development(self):
        self.assertTrue(app.config['SECRET_KEY'] == 'my_precious')
        self.assertTrue(app.config['DEBUG'] is True)
        self.assertFalse(current_app is None)
        self.assertTrue(
            app.config['SQLALCHEMY_DATABASE_URI'] ==
            'postgres://postgres:postgres@users-db:5432/users_dev'
        )


class TestTestingConfig(TestCase):
    def create_app(self):
        app.config.from_object('project.config.TestingConfig')
        return app

    def test_app_is_testing(self):
        self.assertTrue(app.config['SECRET_KEY'] == 'my_precious')
        self.assertTrue(app.config['DEBUG'])
        self.assertTrue(app.config['TESTING'])
        self.assertFalse(app.config['PRESERVE_CONTEXT_ON_EXCEPTION'])
        self.assertTrue(
            app.config['SQLALCHEMY_DATABASE_URI'] ==
            'postgres://postgres:postgres@users-db:5432/users_test'
        )


class TestProductionConfig(TestCase):
    def create_app(self):
        app.config.from_object('project.config.ProductionConfig')
        return app

    def test_app_is_production(self):
        self.assertTrue(app.config['SECRET_KEY'] == 'my_precious')
        self.assertFalse(app.config['DEBUG'])
        self.assertFalse(app.config['TESTING'])


if __name__ == '__main__':
    unittest.main()
```

*test_users.py*:

```python
# project/tests/test_users.py


import json

from project.tests.base import BaseTestCase


class TestUserService(BaseTestCase):
    """Tests for the Users Service."""

    def test_users(self):
        """Ensure the /ping route behaves correctly."""
        response = self.client.get('/ping')
        data = json.loads(response.data.decode())
        self.assertEqual(response.status_code, 200)
        self.assertIn('pong!', data['message'])
        self.assertIn('success', data['status'])
```

增加 [Flask-Testing](https://pythonhosted.org/Flask-Testing/) 到 requirements 文件:

```
Flask-Testing==0.6.2
```

添加一个新的命令到*manage.py*，用来发现和执行tests：

```python
@manager.command
def test():
    """Runs the tests without code coverage."""
    tests = unittest.TestLoader().discover('project/tests', pattern='test*.py')
    result = unittest.TextTestRunner(verbosity=2).run(tests)
    if result.wasSuccessful():
        return 0
    return 1
```

别忘了导入 `unittest`:

```python
import unittest
```

我们需要重新构建镜像，因为requirements是在构建时间安装而非运行时间：

```sh
$ docker-compose up -d --build
```

容器跑起来后，执行tests：

```sh
$ docker-compose run users-service python manage.py test
```

你应该会得到如下错误：

```sh
self.assertTrue(app.config['SECRET_KEY'] == 'my_precious')
```

更新 BaseConfig:

```python
class BaseConfig:
    """Base configuration"""
    DEBUG = False
    TESTING = False
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    SECRET_KEY = 'my_precious'
```

然后重新测试！

```sh
----------------------------------------------------------------------
Ran 4 tests in 0.118s

OK
```
