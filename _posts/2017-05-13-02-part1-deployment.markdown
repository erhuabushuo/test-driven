---
title: 部署
layout: post
date: 2017-05-13 23:59:57
permalink: part-one-aws-deployment
share: true
---

路由完成和测试后，让我们来部署我们的应用！

---

根据 [这里](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/get-set-up-for-amazon-ec2.html )提供的操作指令来注册AWS (有必要的乎) 然后创建一个IAM用户（有必要的话），确保将证书放置在*~/.aws/credentials* 文件，然后新建一个新的主机：

```sh
$ docker-machine create --driver amazonec2 aws
```

> 要了解更多查看Docker [Amazon Web Services (AWS) EC2 示例](https://docs.docker.com/machine/examples/aws/)。

一旦完成后，设置为当前活跃host并且将Docker client指向它：

```sh
$ docker-machine env aws
$ eval $(docker-machine env aws)
```

运行如下指令查看当前执行机器：

```sh
$ docker-machine ls
```

创建一个新的compose文件命名为*docker-compose-prod.yml* ，将除了`volumes`的其他参数都拷贝过来。

> 如果你保留该参数会发生什么？

启动容器，创建数据库，基础数据以及运行测试：

```sh
$ docker-compose -f docker-compose-prod.yml up -d --build
$ docker-compose -f docker-compose-prod.yml run users-service python manage.py recreate_db
$ docker-compose -f docker-compose-prod.yml run users-service python manage.py seed_db
$ docker-compose -f docker-compose-prod.yml run users-service python manage.py test
```

将5001端口加入到 [安全组](http://stackoverflow.com/questions/26338301/ec2-how-to-add-port-8080-in-security-group).

拿到IP并确保在浏览器中能正常访问。

#### 配置

那app的配置和环境变量怎么弄？这些配置都正常码？我们是否用的是生产环境配置？要检查的话，运行：

```sh
$ docker-compose -f docker-compose-prod.yml run users-service env
```

你应该看到了`APP_SETTINGS`变量赋予值为`project.config.DevelopmentConfig`。

要更新该项，通过*docker-compose-prod.yml*更改环境变量：

```
environment:
  - APP_SETTINGS=project.config.ProductionConfig
  - DATABASE_URL=postgres://postgres:postgres@users-db:5432/users_prod
  - DATABASE_TEST_URL=postgres://postgres:postgres@users-db:5432/users_test
```

更新：

```sh
$ docker-compose -f docker-compose-prod.yml up -d
```

重新创建db和初始数据：

```sh
$ docker-compose -f docker-compose-prod.yml run users-service python manage.py recreate_db
$ docker-compose -f docker-compose-prod.yml run users-service python manage.py seed_db
```

确保app仍然运行，再次检查环境变量：

#### Gunicorn

要使用Gunicorn, 首先增加用来到*requirements.txt* 文件：

```
gunicorn==19.7.1
```

然后更新*docker-compose-prod.yml*加入`command`键到`users-service`：

```
command: gunicorn -b 0.0.0.0:5000 manage:app
```

它将覆盖*services/users/Dockerfile*文件里的`CMD`指令，`python manage.py runserver -h 0.0.0.0`。

更新：

```sh
$ docker-compose -f docker-compose-prod.yml up -d --build
```

> `--build`参数是必须的，因为我们需要重新安装依赖。

#### Nginx

接下来，我们将使用Nginx作为反向代理Web服务器。创建一个新的文件夹名为"nginx"在project目录下，在创建*Dockerfile**文件，敲如如下内容：

```
FROM nginx:1.13.0

RUN rm /etc/nginx/conf.d/default.conf
ADD /flask.conf /etc/nginx/conf.d
```

再新建一个*flask.conf*的配置文件到"nginx"文件夹：

```
server {

    listen 80;

    location / {
        proxy_pass http://users-service:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

}
```

添加一个`nginx`服务到*docker-compose-prod.yml*：

```
nginx:
  container_name: nginx
  build: ./nginx/
  restart: always
  ports:
    - 80:80
  depends_on:
    users-service:
      condition: service_started
  links:
    - users-service
```

移除`ports`暴露到宿主端口，仅暴露到其他容器：

```
expose:
  - '5000'
```

构建镜像并执行容器：

```sh
$ docker-compose -f docker-compose-prod.yml up -d --build nginx
```

将80端口添加到AWS安全组。在浏览器测试网站。

让我们将本地也进行更新下:

首先也给 *docker-compose.yml* 文件添加nginx服务:

```sh
nginx:
  container_name: nginx
  build: ./nginx/
  restart: always
  ports:
    - 80:80
  depends_on:
    users-service:
      condition: service_started
  links:
    - users-service
```

接下来，我们需要更新当前活跃host，通过如下命令确认：

```sh
$ docker-machine active
aws
```

切换到`dev`：

```sh
$ eval "$(docker-machine env dev)"
```

执行nginx容器：

```sh
$ docker-compose up -d --build nginx
```

查看IP并且进行测试

> 注意到你访问本地站点时需要或者不需要端口吗？ - [http://YOUR-IP/users](http://YOUR-IP/users) 或者 [http://YOUR-IP:5001/users](http://YOUR-IP:5001/users). 为什么？
