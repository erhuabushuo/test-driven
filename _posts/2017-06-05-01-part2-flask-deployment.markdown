---
title: Flask 部署
layout: post
date: 2017-06-05 23:59:59
permalink: part-two-flask-deployment
share: true
---

让我们更新`users-service`容器，本地还有生产服务器，然后用本地的容器外执行React app进行测试……

---

#### 开发

要更新Docker，先提交上传*flask-microserves-users*代码到GitHub（如果需要的话）。确保在Travis CI测试通过，然后切换到*flask-microservices-main*。

设置`dev`为活动服务器，然后更新容器：

```sh
$ docker-machine env dev
$ eval $(docker-machine env dev)
$ docker-compose up -d --build
```

确保测试通过：

```sh
$ docker-compose run users-service python manage.py test
```
现在，我们可以使用React应用针对Docker容器运行的Flask进行测试：

1. 通过`docker-machine ip dev`获取到`dev`机器的IP。
1. 切换到*flask-microservices-main*然后更新IP环境变量 - `export REACT_APP_USERS_SERVICE_URL=DOCKER_MACHINE_IP`。
1. 使用`npm start`启动app，然后确保依然工作。

#### 生产

现在让我们更新并测试生产服务器。

在*flask-microservices-main*项目下，设置`aws`为活跃机器然后更新容器：

```sh
$ docker-machine env aws
$ eval $(docker-machine env aws)
$ docker-compose -f docker-compose-prod.yml up -d --build
```

就和以前一样，使用React应用来针对Flask应用测试：

1. 获取`aws`机器的IP - `docker-machine ip aws`。
1. 切换到*flask-microservices-main*更新针对IP设置的环境变量 - `export REACT_APP_USERS_SERVICE_URL=DOCKER_MACHINE_IP`。
1. 启动应用- `npm start` - 然后确保应用正常工作。
