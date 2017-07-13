---
title: Jinja 模板
layout: post
date: 2017-05-13 23:59:58
permalink: part-one-jinja-templates
share: true
---

目前我们返回结果是JSON，我们来给它弄成模板……

---

添加一个新的路由处理器*project/api/views.py*: 

```python
@users_blueprint.route('/', methods=['GET'])
def index():
    return render_template('index.html')
```

更新Blueprint配置：

```python
users_blueprint = Blueprint('users', __name__, template_folder='./templates')
```
然后添加"templates"目录到"project/api"，在该目录创建*index.html*文件：

{% raw %}
```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Flask on Docker</title>
    <!-- meta -->
    <meta name="description" content="">
    <meta name="author" content="">
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <!-- styles -->
    <link href="//maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css" rel="stylesheet">
    {% block css %}{% endblock %}
  </head>
  <body>
    <div class="container">
      <div class="row">
        <div class="col-md-4">
          <br>
          <h1>All Users</h1>
          <hr><br>
          <form action="/" method="POST">
            <div class="form-group">
              <input name="username" class="form-control input-lg" type="text" placeholder="Enter a username" required>
            </div>
            <div class="form-group">
              <input name="email" class="form-control input-lg" type="email" placeholder="Enter an email address" required>
            </div>
            <input type="submit" class="btn btn-primary btn-lg btn-block" value="Submit">
          </form>
          <br>
          <hr>
          <div>
            {% if users %}
              {% for user in users %}
                <h4 class="well"><strong>{{user.username}}</strong> - <em>{{user.created_at.strftime('%Y-%m-%d')}}</em></h4>
              {% endfor %}
            {% else %}
              <p>No users!</p>
            {% endif %}
          </div>
        </div>
      </div>
    </div>
    <!-- scripts -->
    <script src="https://code.jquery.com/jquery-2.2.4.min.js"
      integrity="sha256-BbhdlvQf/xTY9gja0Dq3HiwQF8LaCRTXxZKRutelT44="
      crossorigin="anonymous"></script>
    {% block js %}{% endblock %}
  </body>
</html>
```
{% endraw %}

准备好测试了码？打开你浏览器并访问`dev`机器绑定的IP地址。

那么测试怎么写？

```python
def test_main_no_users(self):
    """Ensure the main route behaves correctly when no users have been
    added to the database."""
    response = self.client.get('/')
    self.assertEqual(response.status_code, 200)
    self.assertIn(b'<h1>All Users</h1>', response.data)
    self.assertIn(b'<p>No users!</p>', response.data)
```

看看测试是否通过？

```sh
$ docker-compose run users-service python manage.py test
```

让我们来更新路由处理，从数据库获取所有用户并且在模板中显示出来，先从编写测试开始：

```python
def test_main_with_users(self):
    """Ensure the main route behaves correctly when users have been
    added to the database."""
    add_user('michael', 'michael@realpython.com')
    add_user('fletcher', 'fletcher@realpython.com')
    response = self.client.get('/')
    self.assertEqual(response.status_code, 200)
    self.assertIn(b'<h1>All Users</h1>', response.data)
    self.assertNotIn(b'<p>No users!</p>', response.data)
    self.assertIn(b'<strong>michael</strong>', response.data)
    self.assertIn(b'<strong>fletcher</strong>', response.data)
```

确保执行测试失败，然后更新视图：

```python
@users_blueprint.route('/', methods=['GET'])
def index():
    users = User.query.all()
    return render_template('index.html', users=users)
```

现在应该是测试通过了！

那表单怎么搞？用户应该可以通过提交表来来添加一个新用户，该用户就会保存到数据库去，再写测试：

```python
def test_main_add_user(self):
    """Ensure a new user can be added to the database."""
    with self.client:
        response = self.client.post(
            '/',
            data=dict(username='michael', email='michael@realpython.com'),
            follow_redirects=True
        )
        self.assertEqual(response.status_code, 200)
        self.assertIn(b'<h1>All Users</h1>', response.data)
        self.assertNotIn(b'<p>No users!</p>', response.data)
        self.assertIn(b'<strong>michael</strong>', response.data)
```

运行测试确保测试不通过，更新视图：

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

最后我们将代码更新到AWS中。

1. `eval $(docker-machine env aws)`
1. `docker-compose -f docker-compose-prod.yml up -d`
