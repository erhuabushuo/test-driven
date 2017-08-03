---
title: Flask 重构
layout: post
date: 2017-06-07 23:59:59
permalink: part-two-flask-refactor
share: true
---

在本节课，我们将重构我们的用户服务……

---

切换到*flask-microservices-users*，激活虚拟环境，添加环境变量，然后执行测试：

```sh
$ source env/bin/activate
(env)$ export APP_SETTINGS=project.config.DevelopmentConfig
(env)$ export DATABASE_URL=postgres://postgres:postgres@localhost:5432/users_dev
(env)$ export DATABASE_TEST_URL=postgres://postgres:postgres@localhost:5432/users_test
(env)$ python manage.py test
```

> 你需要根据你本地Postgres配置修改用户名和密码。

#### 移除主路由

由于我们使用RESTful API实现用户服务，我们移除主路由：

```python
@users_blueprint.route('/', methods=['GET', 'POST'])
def index():
    if request.method == 'POST':
        username = request.form['username']
        email = request.form['email']
        db.session.add(User(username=username, email=email))
        db.session.commit()
    users = User.query.order_by(User.created_at.desc()).all()
    return render_template('index.html', users=users)
```

再次执行测试，这下应该有三个失败的：

```sh
FAIL: test_main_add_user (test_users.TestUserService)
FAIL: test_main_no_users (test_users.TestUserService)
FAIL: test_main_with_users (test_users.TestUserService)
```

将它们从*flask-microservices-users/project/tests/test_users.py*移除，然后再次测试。应该都正常通过，把"templates"目录也移除。

#### 按时间排序用户

接下来，让我们更新`/users`路由通过`created_at`降序。

我们将还是先从测试开始，但是我们先要更改测试里头的`add_user()`辅助函数，让它接收可选的`created_at`日期参数。为什么？这样我们就可以简单处理之前初始数据库代码。

```python
def add_user(username, email, created_at=datetime.datetime.utcnow()):
    user = User(username=username, email=email, created_at=created_at)
    db.session.add(user)
    db.session.commit()
    return user
```

然后更新*flask-microservices-users/project/api/models.py*里的 `__init__` 方法接收一个可选参数：

```python
def __init__(self, username, email, created_at=datetime.datetime.utcnow()):
    self.username = username
    self.email = email
    self.created_at = created_at
```

执行测试，确保我们没有打断之前的代码。现在我们就可以更新`test_all_users()`测试代码：

```python
def test_all_users(self):
    """Ensure get all users behaves correctly."""
    created = datetime.datetime.utcnow() + datetime.timedelta(-30)
    add_user('michael', 'michael@realpython.com', created)
    add_user('fletcher', 'fletcher@realpython.com')
    with self.client:
        response = self.client.get('/users')
        data = json.loads(response.data.decode())
        self.assertEqual(response.status_code, 200)
        self.assertEqual(len(data['data']['users']), 2)
        self.assertTrue('created_at' in data['data']['users'][0])
        self.assertTrue('created_at' in data['data']['users'][1])
        self.assertIn('michael', data['data']['users'][1]['username'])
        self.assertIn(
            'michael@realpython.com', data['data']['users'][1]['email'])
        self.assertIn('fletcher', data['data']['users'][0]['username'])
        self.assertIn(
            'fletcher@realpython.com', data['data']['users'][0]['email'])
        self.assertIn('success', data['status'])
```

有什么不一样？

1. 我们定义了日期`created`，然后创建`michael`时赋予了该值。
1. 由于`michael`含有`created_at`日期,而且是一个区间较前的一个时间段，我们断言它应该出现在列表第二位。

测试应该失败。要让测试通过，我们需要更新视图，SQL查询加上根据日期降序：

```python
@users_blueprint.route('/users', methods=['GET'])
def get_all_users():
    """Get all users"""
    users = User.query.order_by(User.created_at.desc()).all()
    users_list = []
    for user in users:
        user_object = {
            'id': user.id,
            'username': user.username,
            'email': user.email,
            'created_at': user.created_at
        }
        users_list.append(user_object)
    response_object = {
        'status': 'success',
        'data': {
          'users': users_list
        }
    }
    return make_response(jsonify(response_object)), 200
```

执行测试！离开虚拟环境，提交代码，确保在Travis CI测试通过?

#### Update Docker

切换`dev`为激活机器，然后更新容器：

```sh
$ docker-machine env dev
$ eval $(docker-machine env dev)
$ export REACT_APP_USERS_SERVICE_URL=http://DOCKER_MACHINE_DEV_IP
$ docker-compose up -d --build
```

切换到 *flask-microservices-main*, 确保测试通过：

```sh
$ docker-compose run users-service python manage.py test
```

在浏览器再次测试.

在生成服务器上做同样的事情：设置`aws`或当前活跃机器，更新容器：

```sh
$ docker-machine env aws
$ eval $(docker-machine env aws)
$ export REACT_APP_USERS_SERVICE_URL=http://DOCKER_MACHINE_AWS_IP
$ docker-compose -f docker-compose-prod.yml up -d --build
```

测试！

#### 接下来

现在暂停一下，审读下代码，写更多的单元或者集成测试。自己去实现来验证你的理解程度。

> 想反馈你的代码。发Github链接地址到`michael@realpython.com`
