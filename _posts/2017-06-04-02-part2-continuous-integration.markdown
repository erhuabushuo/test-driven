---
title: 持续集成
layout: post
date: 2017-06-04 23:59:58
permalink: part-two-continuous-integration
share: true
---

本节，我们将增加持续集成(CI)，这里我们使用[Travis CI](travis-ci.org)。

---

跟着 [初学指南](https://docs.travis-ci.com/user/getting-started/) (第一步和第二步) 将*flask-microservices-users*项目在Travis中开启，要触发构建，需要在项目根目录增加一个*.travis.yml*文件：

```
language: python

python:
  - "3.6"

service:
  - postgresql

install:
  - pip install -r requirements.txt

before_script:
  - export APP_SETTINGS="project.config.TestingConfig"
  - export DATABASE_TEST_URL=postgresql://postgres:@localhost/users_test
  - psql -c 'create database users_test;' -U postgres
  - python manage.py recreate_db

script:
  - python manage.py test
```

提交修改到GitHub。这将会触发一个新的构建并且测试通过。到现在，项目结构还是比较简单，我们遵循如下工作流：

1. 本地添加新功能
1. 提交代码
1. 确保Travis中测试通过

好了，来看看*flask-microservices-main*，我们同样需要配置该项目走CI。这里我们将在Docker中测试我们的所有服务。

开启 Travis, 添加*.travis.yml*文件：

```
sudo: required

services:
  - docker

env:
  global:
    - DOCKER_COMPOSE_VERSION=1.11.2

before_install:
  - sudo rm /usr/local/bin/docker-compose
  - curl -L https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-`uname -s`-`uname -m` > docker-compose
  - chmod +x docker-compose
  - sudo mv docker-compose /usr/local/bin

before_script:
  - docker-compose up --build -d

script:
  - docker-compose run users-service python manage.py test

after_script:
  - docker-compose down
```

提交代码到GitHub，确保在继续前该测试是通过的。
