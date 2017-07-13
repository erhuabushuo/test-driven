---
title: 工作流
layout: post
date: 2017-05-13 23:59:59
permalink: part-one-workflow
share: true
---

#### 别名

我们先为`docker-compose`和`docker-machine`创建别名`dc`和`dm`。

简单添加如下指令到*.bashrc*文件：

```
alias dc='docker-compose'
alias dm='docker-machine'
```

保存文件，并执行：

```sh
$ source ~/.bashrc
```

测试我们的别名！

> 使用 Windows? 你需要先创建 [PowerShell Profile](https://msdn.microsoft.com/en-us/powershell/scripting/core-powershell/ise/how-to-use-profiles-in-windows-powershell-ise) (如果你还没有的话), 然后你可以通过 [Set-Alias](https://msdn.microsoft.com/en-us/powershell/reference/5.1/microsoft.powershell.utility/set-alias)来创建 - 如, `Set-Alias dc docker-compose`.

#### "Saved" 状态

是否VM卡在了"Saved"状态？

```sh
$ docker-machine ls

NAME   ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
aws    *        amazonec2    Running   tcp://34.207.173.181:2376           v17.05.0-ce
dev    -        virtualbox   Saved                                         Unknown
```

你只能停掉VM：

1. 启动 virtualbox - `virtualbox`
1. 选择VM并按 "启动"
1. 退出VM并选择 "关闭电源"
1. 退出virtualbox

T现在VM应该成"Stopped"状态：

```sh
$ docker-machine ls

NAME   ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
aws    *        amazonec2    Running   tcp://34.207.173.181:2376           v17.05.0-ce
dev    -        virtualbox   Stopped                                       Unknown
```

现在你可以启动机器了：

```sh
$ docker-machine start dev
```

现在应该是"Running"：

```sh
$ docker-machine ls

NAME   ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
aws    *        amazonec2    Running   tcp://34.207.173.181:2376           v17.05.0-ce
dev    -        virtualbox   Running   tcp://192.168.99.100:2376           v17.05.0-ce
```

#### 常用命令

构建镜像：

```sh
$ docker-compose build
```

启动容器:

```sh
$ docker-compose up -d
```

创建数据库：

```sh
$ docker-compose run users-service python manage.py recreate_db
```

初始化数据：

```sh
$ docker-compose run users-service python manage.py seed_db
```

执行测试：

```sh
$ docker-compose run users-service python manage.py test
```

#### 其他命令

停止容器：

```sh
$ docker-compose stop
```

停止并移除容器：

```sh
$ docker-compose down
```

想要强制构建？

```sh
$ docker-compose build --no-cache
```

移除镜像：

```sh
$ docker rmi $(docker images -q)
```
