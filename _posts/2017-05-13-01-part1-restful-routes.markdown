---
title: RESTful 路由
layout: post
date: 2017-05-13 23:59:57
permalink: part-one-restful-routes
share: true
---

接下来，我们来设置三个新路由，遵循使用TDD进行RESTful最佳实践：

| Endpoint    | HTTP Method | CRUD Method | Result          |
|-------------|-------------|-------------|-----------------|
| /users      | GET         | READ        | get all users   |
| /users/:id  | GET         | READ        | get single user |
| /users      | POST        | CREATE      | add a user      |

针对每一项，我们将

1. 写一个测试
1. 执行一个测试，并看着它失败 (**red**)
1. 简单编写让代码通过测试 (**green**)
1. **重构** (如果需要的话)

让我们从POST路由开始……

#### POST

增加一个测试到 *project/tests/test_users.py*：

```python
def test_add_user(self):
    """Ensure a new user can be added to the database."""
    with self.client:
        response = self.client.post(
            '/users',
            data=json.dumps(dict(
                username='michael',
                email='michael@realpython.com'
            )),
            content_type='application/json',
        )
        data = json.loads(response.data.decode())
        self.assertEqual(response.status_code, 201)
        self.assertIn('michael@realpython.com was added!', data['message'])
        self.assertIn('success', data['status'])
```

执行测试，并确定该测试失败：

```sh
$ docker-compose run users-service python manage.py test
```

然后添加路由处理器到*project/api/views.py*

```python
@users_blueprint.route('/users', methods=['POST'])
def add_user():
    post_data = request.get_json()
    username = post_data.get('username')
    email = post_data.get('email')
    db.session.add(User(username=username, email=email))
    db.session.commit()
    response_object = {
        'status': 'success',
        'message': f'{email} was added!'
    }
    return make_response(jsonify(response_object)), 201
```

更新导入包:

```python
from flask import Blueprint, jsonify, request, make_response

from project.api.models import User
from project import db
```

执行测试，应该是所有都是通过的:

```sh
Ran 5 tests in 0.201s

OK
```

那错误和异常处理呢？例如：

1. 没有发送消息内容
1. 发送内容无效 - 例如，JSON对象为空，或者包含错误的键。
1. 数据库已经存在该用户

再增加一些测试：

```python
def test_add_user_invalid_json(self):
    """Ensure error is thrown if the JSON object is empty."""
    with self.client:
        response = self.client.post(
            '/users',
            data=json.dumps(dict()),
            content_type='application/json',
        )
        data = json.loads(response.data.decode())
        self.assertEqual(response.status_code, 400)
        self.assertIn('Invalid payload.', data['message'])
        self.assertIn('fail', data['status'])

def test_add_user_invalid_json_keys(self):
    """Ensure error is thrown if the JSON object does not have a username key."""
    with self.client:
        response = self.client.post(
            '/users',
            data=json.dumps(dict(email='michael@realpython.com')),
            content_type='application/json',
        )
        data = json.loads(response.data.decode())
        self.assertEqual(response.status_code, 400)
        self.assertIn('Invalid payload.', data['message'])
        self.assertIn('fail', data['status'])

def test_add_user_duplicate_user(self):
    """Ensure error is thrown if the email already exists."""
    with self.client:
        self.client.post(
            '/users',
            data=json.dumps(dict(
                username='michael',
                email='michael@realpython.com'
            )),
            content_type='application/json',
        )
        response = self.client.post(
            '/users',
            data=json.dumps(dict(
                username='michael',
                email='michael@realpython.com'
            )),
            content_type='application/json',
        )
        data = json.loads(response.data.decode())
        self.assertEqual(response.status_code, 400)
        self.assertIn(
            'Sorry. That email already exists.', data['message'])
        self.assertIn('fail', data['status'])
```

确保测试失败，然后我们更新路由处理器：

```python
@users_blueprint.route('/users', methods=['POST'])
def add_user():
    post_data = request.get_json()
    if not post_data:
        response_object = {
            'status': 'fail',
            'message': 'Invalid payload.'
        }
        return make_response(jsonify(response_object)), 400
    username = post_data.get('username')
    email = post_data.get('email')
    try:
        user = User.query.filter_by(email=email).first()
        if not user:
            db.session.add(User(username=username, email=email))
            db.session.commit()
            response_object = {
                'status': 'success',
                'message': f'{email} was added!'
            }
            return make_response(jsonify(response_object)), 201
        else:
            response_object = {
                'status': 'fail',
                'message': 'Sorry. That email already exists.'
            }
            return make_response(jsonify(response_object)), 400
    except exc.IntegrityError as e:
        db.session().rollback()
        response_object = {
            'status': 'fail',
            'message': 'Invalid payload.'
        }
        return make_response(jsonify(response_object)), 400
```

增加导入:

```python
from sqlalchemy import exc
```

Ensure the tests pass, and then move on to the next route...

#### GET single user

Start with a test:

```python
def test_single_user(self):
    """Ensure get single user behaves correctly."""
    user = User(username='michael', email='michael@realpython.com')
    db.session.add(user)
    db.session.commit()
    with self.client:
        response = self.client.get(f'/users/{user.id}')
        data = json.loads(response.data.decode())
        self.assertEqual(response.status_code, 200)
        self.assertTrue('created_at' in data['data'])
        self.assertIn('michael', data['data']['username'])
        self.assertIn('michael@realpython.com', data['data']['email'])
        self.assertIn('success', data['status'])
```

Add the following imports:

```python
from project import db
from project.api.models import User
```

Ensure the test breaks before writing the view:

```python
@users_blueprint.route('/users/<user_id>', methods=['GET'])
def get_single_user(user_id):
    """Get single user details"""
    user = User.query.filter_by(id=user_id).first()
    response_object = {
        'status': 'success',
        'data': {
          'username': user.username,
          'email': user.email,
          'created_at': user.created_at
        }
    }
    return make_response(jsonify(response_object)), 200
```

The tests should pass. Now, what about error handling?

1. An id is not provided
1. The id does not exist

Tests:

```python
def test_single_user_no_id(self):
    """Ensure error is thrown if an id is not provided."""
    with self.client:
        response = self.client.get('/users/blah')
        data = json.loads(response.data.decode())
        self.assertEqual(response.status_code, 404)
        self.assertIn('User does not exist', data['message'])
        self.assertIn('fail', data['status'])

def test_single_user_incorrect_id(self):
    """Ensure error is thrown if the id does not exist."""
    with self.client:
        response = self.client.get('/users/999')
        data = json.loads(response.data.decode())
        self.assertEqual(response.status_code, 404)
        self.assertIn('User does not exist', data['message'])
        self.assertIn('fail', data['status'])
```

Updated view:

```python
@users_blueprint.route('/users/<user_id>', methods=['GET'])
def get_single_user(user_id):
    """Get single user details"""
    response_object = {
        'status': 'fail',
        'message': 'User does not exist'
    }
    try:
        user = User.query.filter_by(id=int(user_id)).first()
        if not user:
            return make_response(jsonify(response_object)), 404
        else:
            response_object = {
                'status': 'success',
                'data': {
                  'username': user.username,
                  'email': user.email,
                  'created_at': user.created_at
                }
            }
            return make_response(jsonify(response_object)), 200
    except ValueError:
        return make_response(jsonify(response_object)), 404
```

#### GET all users

Again, let's start with a test. Since we'll have to add a few users first, let's add a quick helper function to the top of the *project/tests/test_user.py* file, just above the `TestUserService()` class.

```python
def add_user(username, email):
    user = User(username=username, email=email)
    db.session.add(user)
    db.session.commit()
    return user
```

Now, refactor the *test_single_user()* test, like so:

```python
def test_single_user(self):
    """Ensure get single user behaves correctly."""
    user = add_user('michael', 'michael@realpython.com')
    with self.client:
        response = self.client.get(f'/users/{user.id}')
        data = json.loads(response.data.decode())
        self.assertEqual(response.status_code, 200)
        self.assertTrue('created_at' in data['data'])
        self.assertIn('michael', data['data']['username'])
        self.assertIn('michael@realpython.com', data['data']['email'])
        self.assertIn('success', data['status'])
```

With that, let's add the new test:

```python
def test_all_users(self):
    """Ensure get all users behaves correctly."""
    add_user('michael', 'michael@realpython.com')
    add_user('fletcher', 'fletcher@realpython.com')
    with self.client:
        response = self.client.get('/users')
        data = json.loads(response.data.decode())
        self.assertEqual(response.status_code, 200)
        self.assertEqual(len(data['data']['users']), 2)
        self.assertTrue('created_at' in data['data']['users'][0])
        self.assertTrue('created_at' in data['data']['users'][1])
        self.assertIn('michael', data['data']['users'][0]['username'])
        self.assertIn(
            'michael@realpython.com', data['data']['users'][0]['email'])
        self.assertIn('fletcher', data['data']['users'][1]['username'])
        self.assertIn(
            'fletcher@realpython.com', data['data']['users'][1]['email'])
        self.assertIn('success', data['status'])
```

Make sure it fails. Then add the view:

```python
@users_blueprint.route('/users', methods=['GET'])
def get_all_users():
    """Get all users"""
    users = User.query.all()
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

Does the test past?

Before moving on, let's test the route in the browser - [http://YOUR-IP:5000/users](http://YOUR-IP:5000/users). You should see:

```json
{
  "data": {
    "users": [ ]
  },
  "status": "success"
}
```

Add a seed command to the *manage.py* file to populate the database with some initial data:

```python
@manager.command
def seed_db():
    """Seeds the database."""
    db.session.add(User(username='michael', email="michael@realpython.com"))
    db.session.add(User(username='michaelherman', email="michael@mherman.org"))
    db.session.commit()
```

Try it out:

```sh
$ docker-compose run users-service python manage.py seed_db
```

Make sure you can view the users in the JSON response [http://YOUR-IP:5000/users](http://YOUR-IP:5000/users).
