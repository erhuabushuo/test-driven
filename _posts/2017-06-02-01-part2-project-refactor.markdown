---
title: 项目重构
layout: post
date: 2017-06-02 23:59:58
permalink: part-two-project-refactor
share: true
---

在本课，我们将项目拆成多个项目，通过服务方式来划分。

---

在我们拆开单体应用前。你可以选择将微服务架构存放在单个项目（通过单个git仓库）。你可以通过每个独立的"services"目录来区分它们，每种方式都有利有弊，多做一些关于mono repo和mutiple repo调研。

我们现在创建两个新项目：

1. *flask-microservices-main*
1. *flask-microservices-client*

针对每个项目创建git仓库添加*.gitignore*文件。

*main* 项目将存储Docker compose文件，Nginx配置文件和任何管理脚本。大体上，你将从该项目来管理所有的服务，同时我们将会在*client*端增加React。

*步骤:*

1. 测试当前的结构
1. 重构
1. 测试新的
1. 更新生产服务

#### 测试当前结构

执行测试：

```sh
$ docker-compose run users-service python manage.py test
```

应该是所有测试通过。通过已有的测试，我们可以更加有把握进行重构。

#### 重构

停掉容器并移除镜像：

```sh
$ docker-compose down
```

然后移除容器

```sh
$ docker-compose rm
```

在生产服务器上做相同操作。更改活动的主机并且指向Docker client，然后停掉容器，移除相关镜像：

```sh
$ docker-machine env aws
$ eval $(docker-machine env aws)
$ docker-compose -f docker-compose-prod.yml down
$ docker-compose -f docker-compose-prod.yml rm
```

切换回活动主机到dev:

```sh
$ docker-machine env dev
$ eval $(docker-machine env dev)
```
将*flask-microservices-users*以下文件和目录移到*flask-microservices-main*项目：

```sh
docker-compose-prod.yml
docker-compose.yml
project/nginx/Dockerfile
project/nginx/flask.conf
```

回到*flask-microservices-users* 项目，提交变更的代码。

然后更新 `users-db`和`users-service`里的`build`指令指向[git 仓库](https://docs.docker.com/engine/reference/commandline/build/#git-repositories)：

1. `users-db`

    ```
    build: https://github.com/realpython/flask-microservices-users.git#master:project/db
    ```

    > `master:project/db` 表示在**master**分支里的"project/db"目录下找*Dockerfile*。

1. `users-service`:

    ```
    build: https://github.com/realpython/flask-microservices-users.git
    ```

由于我们将“构建上下文”从本地机器到了git仓库，所以我们需要从`users-service*移除volume。一旦移除，我们就可以构建镜像，并且启动容器：

```sh
$ docker-compose up -d --build
```

> 在它启动的同时请思考以下我们为什么要移除volume，什么是构建上下文，去Google查询一些帮助吧。

#### 测试新的结构

一旦启动后，创建和初始化db，然后执行测试：

```sh
$ docker-compose run users-service python manage.py recreate_db
$ docker-compose run users-service python manage.py seed_db
$ docker-compose run users-service python manage.py test
```

通过`docker-machine ip dev`获取到IP，确保在应用在浏览器中工作。

#### 更新生产服务器

首先，切换活动主机，并将Docker client指向它：

```sh
$ docker-machine env aws
$ eval $(docker-machine env aws)
```

启动容器：

```sh
$ docker-compose -f docker-compose-prod.yml up -d --build
```

跟之前一样，创建和初始化数据库，然后执行测试：

```sh
$ docker-compose -f docker-compose-prod.yml run users-service python manage.py recreate_db
$ docker-compose -f docker-compose-prod.yml run users-service python manage.py seed_db
$ docker-compose -f docker-compose-prod.yml run users-service python manage.py test
```

在浏览器访问确认，然后提交代码到GitHub。
