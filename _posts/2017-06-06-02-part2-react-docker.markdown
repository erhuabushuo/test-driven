---
title: React和Docker
layout: post
date: 2017-06-06 23:59:58
permalink: part-two-react-docker
share: true
---

让我们来容器话React应用……

---

切换*flask-microservices-client*目录，在根目录下增加*Dockerfile*文件，确保审查了代码注释：

```
FROM node:latest

# set working directory
RUN mkdir /usr/src/app
WORKDIR /usr/src/app

# add `/usr/src/app/node_modules/.bin` to $PATH
ENV PATH /usr/src/app/node_modules/.bin:$PATH

# install and cache app dependencies
ADD package.json /usr/src/app/package.json
RUN npm install --silent
RUN npm install react-scripts@0.9.5 -g --silent

# add app
ADD . /usr/src/app

# start app
CMD ["npm", "start"]
```

提交代码，然后推送到GitHub，在*flask-microservices-main*项目中，添加一个新的服务到*docker-compose.yml*文件：

```
web-service:
  container_name: web-service
  build: https://github.com/realpython/flask-microservices-client.git
  ports:
    - '3007:3000' # expose ports - HOST:CONTAINER
  environment:
    - NODE_ENV=development
    - REACT_APP_USERS_SERVICE_URL=${REACT_APP_USERS_SERVICE_URL}
  depends_on:
    users-service:
      condition: service_started
  links:
    - users-service
```

打开终端，确保`dev`是当前激活机器，然后添加有效IP到*flask-microservices-main*:

```sh
$ export REACT_APP_USERS_SERVICE_URL=DOCKER_MACHINE_IP
```

构建镜像并启动容器：

```sh
$ docker-compose up --build -d web-service
```

在浏览器访问 [http://DOCKER_MACHINE_IP:3007/](http://DOCKER_MACHINE_IP:3007/) 来测试应用。

在切换到主路由时发生了什么？由于我们仍然路由到Flask应用（通过Nginx)，你看到的还是旧的应用。我们需要更新Nginx配置路由到React应用。在操作前，让我们创建一个[build](https://github.com/facebookincubator/create-react-app/blob/master/packages/react-scripts/template/README.md#deployment)本地React应用，在Docker之外，我们将生成静态文件.

#### 创建React应用Build

确保`REACT_APP_USERS_SERVICE_URL`环境变量值设置正确:

```sh
$ export REACT_APP_USERS_SERVICE_URL=DOCKER_MACHINE_IP
```

> 所有的环境变量都是[embedded](https://github.com/facebookincubator/create-react-app/blob/master/packages/react-scripts/template/README.md#adding-custom-environment-variables
) 在构建时嵌入到应用的. 这个需要注意.

然后运行在*flask-microservices-client*下运行`build`命令： 

```sh
$ npm run build
```

你应该会看到一个"build"目录，里面存放了静态文件。我们需要启动一个基本的web服务器来运行它。我们使用标注库[HTTP server](https://docs.python.org/3/library/http.server.html#module-http.server)。切换到"build"目录，然后运行服务器：

```sh
$ python3 -m http.server
```

服务将会监听在 [http://localhost:8000/](http://localhost:8000/). 在浏览器进行测试并确保工作。一旦完成，结束服务器，切换回项目根目录。

#### Dockerfile

Update the *Dockerfile*

```
FROM node:latest

# set working directory
RUN mkdir /usr/src/app
WORKDIR /usr/src/app

# add `/usr/src/app/node_modules/.bin` to $PATH
ENV PATH /usr/src/app/node_modules/.bin:$PATH

# add environment variables
ARG REACT_APP_USERS_SERVICE_URL
ARG NODE_ENV
ENV NODE_ENV $NODE_ENV
ENV REACT_APP_USERS_SERVICE_URL $REACT_APP_USERS_SERVICE_URL

# install and cache app dependencies
ADD package.json /usr/src/app/package.json
RUN npm install --silent
RUN npm install pushstate-server -g --silent

# add app
ADD . /usr/src/app

# build react app
RUN npm run build

# start app
CMD ["pushstate-server", "build"]
```

When the image is built, we can pass arguments to the *Dockerfile*, via the [ARG](https://docs.docker.com/engine/reference/builder/#arg) instruction, which can then be used as environment variables. `npm run build` will generate static files that are served up on port 9000 via the  [pushstate-server](https://www.npmjs.com/package/pushstate-server).

Let's test it without Docker Compose.

First, build the image, making sure to use the `--build-arg` flag to pass in the appropriate arguments:

```sh
$ docker build -t "test" ./ --build-arg NODE_ENV=development --build-arg REACT_APP_USERS_SERVICE_URL=http://DOCKER_MACHINE_IP
```

This uses the *Dockerfile* found in the project root, `./`, to build a new image called `test` with the required build arguments.

> You can view all images by running `docker image`.

Spin up the container from the `test` image, mapping port 9000 in the container to port 9000 outside the container.

```sh
$ docker run -d -p 9000:9000 test
```

Navigate to [http://localhost:9000/](http://localhost:9000/) in your browser to test.

Grab the container ID by running `docker ps`, and then view the container's environment variables:

```sh
$ docker exec CONTAINER_ID bash -c 'env'
```

Once done, stop and remove the container:

```sh
$ docker stop CONTAINER_ID
$ docker rm CONTAINER_ID
```

Finally, remove the image:

```sh
$ docker rmi test
```

Commit and push your code.

#### Docker Compose

With the *Dockerfile* set up and tested, update the `web-service` in *docker-compose.yml*:

```
web-service:
  container_name: web-service
  build:
    context: https://github.com/realpython/flask-microservices-client.git
    args:
      - NODE_ENV=development
      - REACT_APP_USERS_SERVICE_URL=${REACT_APP_USERS_SERVICE_URL}
  ports:
    - '9000:9000' # expose ports - HOST:CONTAINER
  depends_on:
    users-service:
      condition: service_started
  links:
    - users-service
```

So, instead of passing `NODE_ENV` and `REACT_APP_USERS_SERVICE_URL` as environment variables, which happens at runtime, we defined them as build arguments.

Update the containers:

```sh
$ docker-compose up -d --build
```

Test it out again at [http://DOCKER_MACHINE_IP:9000/](http://DOCKER_MACHINE_IP:9000/).

> Curious about the environment variables? Run `docker-compose run web-service env` to view them.

With that, let's update Nginx...

#### Nginx

Make the following updates to *flask.conf* in "flask-microservices-main":

```
server {

    listen 80;

    location / {
        proxy_pass http://web-service:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /users {
        proxy_pass http://users-service:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

}
```

What's happening?

1. The `location` blocks define the [reverse proxies](https://www.nginx.com/resources/glossary/reverse-proxy-server/).
1. When a requested URI matches the URI in a location block, Nginx passes the request either to the pushstate-server (serving the React app) or to the WSGI/Guicorn server (serving up the Flask app).

While we're at it, let's update the name from *flask.conf* to *nginx.conf* so it's more relevant. Make sure to update the *Dockerfile* in "flask-microservices-main/nginx" as well:

```
FROM nginx:1.13.0

RUN rm /etc/nginx/conf.d/default.conf
ADD /nginx.conf /etc/nginx/conf.d
```

Update `nginx` in the *docker-compose.yml* file, so that it is linked to the `web-service`:

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
    web-service:
      condition: service_started
  links:
    - users-service
    - web-service
```

Update the containers (via `docker-compose up -d --build`) and then test it out in the browser:

1. [http://DOCKER_MACHINE_IP/](http://DOCKER_MACHINE_IP/)
1. [http://DOCKER_MACHINE_IP/users](http://DOCKER_MACHINE_IP/users)

Run the tests again (just for fun!):

```sh
$ docker-compose run users-service python manage.py test
```

Now, let's update production...

#### Update Production

Add the `web-service` to *docker-compose-prod.yml*:

```
web-service:
  container_name: web-service
  build:
    context: https://github.com/realpython/flask-microservices-client.git
    args:
      - NODE_ENV=production
      - REACT_APP_USERS_SERVICE_URL=${REACT_APP_USERS_SERVICE_URL}
  ports:
    - '9000:9000' # expose ports - HOST:CONTAINER
  depends_on:
    users-service:
      condition: service_started
  links:
    - users-service
```

Update `nginx` as well:

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
    web-service:
      condition: service_started
  links:
    - users-service
    - web-service
```

Set the `aws` machine as the active machine, change the environment variable to the IP associated with the `aws` machine, and update the containers:

```sh
$ docker-machine env aws
$ eval $(docker-machine env aws)
$ export REACT_APP_USERS_SERVICE_URL=http://DOCKER_MACHINE_AWS_IP
$ docker-compose -f docker-compose-prod.yml up -d --build
```

> Remember: Since the environment variables are added at build time, if you update the variables, you *will* have to rebuild the Docker image.

Test it again, and then commit and push your code.
