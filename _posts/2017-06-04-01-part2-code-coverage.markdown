---
title: 代码覆盖率
layout: post
date: 2017-06-04 23:59:57
permalink: part-two-code-coverage
share: true
---

本节，我们将通过[Coverage.py](http://coverage.readthedocs.io/en/coverage-4.4.1/)给*flask-microservices-users*项目增加代码覆盖率……

---

导航到*flask-microservices-users*项目目录，创建并且激活虚拟环境，然后安装Coverage.py：

```sh
$ python3.6 -m venv env
$ source env/bin/activate
(env)$ pip install coverage==4.4.1
(env)$ pip freeze > requirements.txt
```

接下来，我们需要在*mamage.py*配置覆盖率报表，在导入包后我们进行如下配置：

```python
COV = coverage.coverage(
    branch=True,
    include='project/*',
    omit=[
        'project/tests/*'
    ]
)
COV.start()
```

添加一个新的命令：

```python
@manager.command
def cov():
    """Runs the unit tests with coverage."""
    tests = unittest.TestLoader().discover('project/tests')
    result = unittest.TextTestRunner(verbosity=2).run(tests)
    if result.wasSuccessful():
        COV.stop()
        COV.save()
        print('Coverage Summary:')
        COV.report()
        COV.html_report()
        COV.erase()
        return 0
    return 1
```

别忘记导入包

```python
import coverage
```

文件应该像如下所示：

```python
# manage.py


import unittest
import coverage

from flask_script import Manager

from project import create_app, db
from project.api.models import User


COV = coverage.coverage(
    branch=True,
    include='project/*',
    omit=[
        'project/tests/*',
        'project/server/config.py',
        'project/server/*/__init__.py'
    ]
)
COV.start()


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
def cov():
    """Runs the unit tests with coverage."""
    tests = unittest.TestLoader().discover('project/tests')
    result = unittest.TextTestRunner(verbosity=2).run(tests)
    if result.wasSuccessful():
        COV.stop()
        COV.save()
        print('Coverage Summary:')
        COV.report()
        COV.html_report()
        COV.erase()
        return 0
    return 1


@manager.command
def recreate_db():
    """Recreates a database."""
    db.drop_all()
    db.create_all()
    db.session.commit()


@manager.command
def seed_db():
    """Seeds the database."""
    db.session.add(User(username='michael', email="michael@realpython.com"))
    db.session.add(User(username='michaelherman', email="michael@mherman.org"))
    db.session.commit()


if __name__ == '__main__':
    manager.run()
```

#### 检查

最后，我们确保可以不依赖Docker Compose单独测试该项目。

首先配置Postgres然后初始化数据库 - `users_dev`和`users_test`，然后添加环境变量：

```sh
(env)$ export APP_SETTINGS=project.config.DevelopmentConfig
(env)$ export DATABASE_URL=postgres://postgres:postgres@localhost:5432/users_dev
(env)$ export DATABASE_TEST_URL=postgres://postgres:postgres@localhost:5432/users_test
```

> 依赖于你本地环境，你可能需要更改的你用户名和密码.

创建和初始化数据库，然后执行测试（不带测试覆盖率）：

```sh
(env)$ python manage.py recreate_db
(env)$ python manage.py seed_db
(env)$ python manage.py test
```

你应该可以看到如下错误：

```sh
======================================================================
FAIL: test_app_is_development (test_config.TestDevelopmentConfig)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "flask-microservices-users/project/tests/test_config.py", line 25, in test_app_is_development
    'postgres://postgres:postgres@users-db:5432/users_dev'
AssertionError: False is not true

======================================================================
FAIL: test_app_is_testing (test_config.TestTestingConfig)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "flask-microservices-users/project/tests/test_config.py", line 41, in test_app_is_testing
    'postgres://postgres:postgres@users-db:5432/users_test'
AssertionError: False is not true

----------------------------------------------------------------------
Ran 15 tests in 0.330s
```

打开*test_config.py*。当前我们是硬编码数据URI，让我们更改为成环境变量获取。这样配置就能更[更干净一点](https://12factor.net/config)。

`test_app_is_development()`:

```python
def test_app_is_development(self):
    self.assertTrue(app.config['SECRET_KEY'] == 'my_precious')
    self.assertTrue(app.config['DEBUG'] is True)
    self.assertFalse(current_app is None)
    self.assertTrue(
        app.config['SQLALCHEMY_DATABASE_URI'] ==
        os.environ.get('DATABASE_URL')
    )
```

`test_app_is_testing()`:

```python
def test_app_is_testing(self):
    self.assertTrue(app.config['SECRET_KEY'] == 'my_precious')
    self.assertTrue(app.config['DEBUG'])
    self.assertTrue(app.config['TESTING'])
    self.assertFalse(app.config['PRESERVE_CONTEXT_ON_EXCEPTION'])
    self.assertTrue(
        app.config['SQLALCHEMY_DATABASE_URI'] ==
        os.environ.get('DATABASE_TEST_URL')
    )
```

增加导入包：

```python
import os
```

现在测试应该可以通过。尝试执行代码覆盖率：

```sh
(env)$ python manage.py cov
```

你应该可以看到人如下输出

```sh
Coverage Summary:
Name                    Stmts   Miss Branch BrPart  Cover
---------------------------------------------------------
project/__init__.py        12      5      0      0    58%
project/api/models.py      13     10      0      0    23%
project/api/views.py       53      0     10      0   100%
project/config.py          16      0      0      0   100%
---------------------------------------------------------
TOTAL                      94     15     10      0    86%
```

可以在新建的"htmlcov"目录下查看web版本。现在你可以快速查看哪些代码被测试覆盖了。将该目录增加到*.gitignore**文件，提交代码到GitHub.

让我们快速尝试在Docker Compose验证。

退出当前*flask-microservices-users*虚拟环境，切换到*flask-microservices-main*目录中，确保当前激活dev主机（通过`docker-machine ls`)，然后更新容器：

```sh
$ docker-compose up -d --build
```

使用代码覆盖率执行测试

```sh
$ docker-compose run users-service python manage.py cov
```

最终，我们切换到`aws`机器，更新容器，然后执行代码覆盖率测试：

```sh
$ docker-machine env aws
$ eval $(docker-machine env aws)
$ docker-compose -f docker-compose-prod.yml up -d --build
$ docker-compose -f docker-compose-prod.yml run users-service python manage.py cov
```
