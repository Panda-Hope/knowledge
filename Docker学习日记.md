## 开始整活

滴滴~，同学都2022年了，不是吧，你居然还不知道Docker是什么？，哈哈，其实我也不知道o(╯□╰)o，所以开始整活搞起，  
Docker之前都是公司的运维同学搞得，自己以前也只是零零散散的学习过一些命令和介绍，由于现在自己要弄点私人服务器，以前那点小知识不够用了，
当然就得开始有点系统的学习了。

## 什么是Docker？

简单的说，Docker是一个用于开发，交付和运行应用程序的开放平台，对我而言，理解Docker的强大和使用之后，才开始真正懂得容器化是真的香啊~。
为了更好的理解Docker存在的必要性，我们简单梳理一下运维部署的历史：

- 传统部署时代：  
  早期，各机构是在物理服务器上运行应用程序。由于无法限制在物理服务器中运行的应用程序资源使用，因此会导致各种资源分配问题，服务的运行、统筹都是一大痛点。
- 虚拟化部署时代：  
  自从虚拟化技术被引入了，虚拟化技术允许你在单个物理服务器的 CPU 上运行多台虚拟机（VM）。 虚拟化能使应用程序在不同 VM 之间被彼此隔离，且能提供一定程度的安全性。
  虚拟化技术能够更好地利用物理服务器的资源，并且可轻松地添加或更新应用程序。
- Docker容器部署时代：  
  容器类似于 VM，但是更宽松的隔离特性，使容器之间可以共享操作系统（OS），容器具备持续开发、集成和部署、敏捷应用程序的创建和部署、松散耦合、分布式、弹性、解放的微服务等等优点。
  
总的来说，Docker改变了以往服务在各个操作系统的隔离与独立性等等不方便管理的各种问题，Docker将各个应用比如服务器、数据库、软件程序等都封装成一个独立的容器，
并使其支持跨平台快速开发、部署。极大的改变了服务开发部署的方式。  

Docker采用C/S架构，采用镜像、容器、卷、网络等来管理和开发各个后台的服务：  

<img src="https://github.com/Panda-Hope/panda-hope.github.io/blob/master/static/img/architecture.svg" width="800" />

## 初识Docker

首先我们先了解下Docker最基本的三个要素：镜像、容器以及卷，他们是Docker运行中最常使用的功能，理解它们的基本概念是非常重要的。

### 镜像
什么是镜像？，简要概之：镜像即是计算机磁盘上面的文件，比如/file、/usr这种，镜像就像一个模板，用于构建一个运行时容器执行命令所需的内容。
在Docker中镜像是可以继承的，比如我们可以官方`Nginx`的镜像中添加自己常用的Nginx的配置，从而构建一个属于自身的Nginx镜像，以方便后续的部署和维护。  

下面我们来些镜像的常用命令：
```
docker image pull // 拉取镜像
docker image ls // 列出本地镜像
docker image rmi // 删除镜像
docker image inspect // 查看镜像详情
docker image build // 从Dockerfile构建一个新的镜像
```

### 容器
容器是一个可运行的软件程序，它执行镜像文件中命令，开始运行程序，容器可移植到不同平台如：Linux、Window中运行，
同时各个容器之间相互独立，互不影响。Docker奉行一个容器执行一件事情的原则，即每个容器当且仅当做好自己的事情，

下面我们来些容器的常用命令：

```
docker run // 从镜像中运行一个容器
docker restart // 重新启动一个容器
docker ps // 查看运行中的容器列表
docker stop // 停止运行一个容器
docker rm // 删除一个容器
```

### 卷
如果说镜像与容器，一个是软件的代码、一个是运行时的软件程序，那么卷又是什么呢？，在软件运行中必然会产生一些运行时的数据，
比如MySQL储存的数据，卷则是储存这些运行时数据而存在的。由于Docker运行在虚拟机中，卷的使用方式也是通过文件映射的方式来使用的。  

卷具有两种类型：命名卷与数据卷
- 命名卷：数据存在Docker虚拟机中，储存位置有Docker决定
- 数据卷：数据存在服务器本地，储存位置由开发人员自己决定  

相比较于数据卷，命令卷的功能更加丰富，并且支持其他的卷驱动软件，



### Docker常用命令
除了镜像、容器以及卷以外，我们还有一些常用的Docker命令，用于我们一些日常的操作。

```
docker exec -it `container_name` bash // 进入容器内执行shell会话
docker network create // 创建docker网络
docker network connect  // 将运行中的容器添加到网络
docker inspect // 检查容器的详细信息
docker log // 查看容器运行日志
docker compose up // 开始执行compose文件，创建并运行容器
docker compose start // 开启服务
docker compose stop //  停止服务
```

## 牛刀小试

在理解了Docker基本的运行原理之后，我们认识到Docker的本质即是一个一个独立的虚拟机容器，由于Docker奉行`ach container should do one thing and do it well`即一个容器当且应当
做一件事情的原则，我们开始为web服务的各个模块构建自身的docker容器。

### Nginx
首先我们开始构建服务器的Docker容器如下：

```
FROM nginx
COPY dist/ /usr/share/nginx/html/
COPY default.conf /etc/nginx/conf.d/default.conf
```
这段代码指示Docker构建一个Nginx镜像，由于我们的前端采用的是SPA框架，我们直接将打包完成的dist目录替换掉Nginx的静态文件目录，并替换掉Nginx的默认配置，以下是我们的Nginx配置：  

```
server {
    listen       3000;
    root /usr/share/nginx/html;

    gzip on;
    gzip_comp_level 6;
    gzip_types text/plain application/x-javascript text/css text/javascript;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```
这段配置代码指示Nginx服务将接收到的全部请求均转发到index.html文件，这是SPA框架比如React、Vue常见的配置。

### Server
在配置完成前端项目之后，我们开始配置自己的服务：
```
FROM node:18

WORKDIR /app

ENV NODE_OPTIONS="--openssl-legacy-provider"

ENV EGG_SERVER_ENV="prod"

# 将代码&依赖拷贝至镜像中
COPY ["package.json", "package-lock.json*", "./"]

RUN npm install

# 复制运行代码
COPY [".", "."]

# 编译Typescript代码
RUN npm run tsc

# node 服务启动的端口
EXPOSE 4000

# node 服务启动命令
CMD [ "npm", "run", "start" ]
```
这段代码的指示如下：
1. 从Node复制镜像
2. 将当前工作目录切换到/app
3. 设置Node环境变量
4. 将package.json复制到当前目录
5. 安装node_modules依赖包
6. 将程序代码复制进入工作目录
7. 编译Typescript代码
8. 将程序运行端口指定为4000
9. 开始运行服务程序  

由此，我们得到了一个在4000端口运行的Node服务端程序

### Mysql
最后，我们开始配置数据库，我们便可以得到一个由Nginx服务器、前端SPA页面、Node服务端程序与MySQL数据库共同组成的现代web架构的雏形。

```
FROM mysql:5.7.4

EXPOSE 3306

ENV MYSQL_ROOT_PASSWORD=password
```

现在我们将以往在web架构上各个互相独立的模块，通过docker进行了统一的管理，但是显然仅仅能够使得程序运行时不足以满足当代web服务的需求的，因此我们将在接下来的环节继续完善
我们的应用，以使得其能够满足我们的需求。

## 使用docker-compose

刚刚，我们已经通过Docker来构建好了一个web服务架构所需的各个应用，但是目前各个应用之间的Docker构建任然是相互独立的，
这个时候如果我们想一键部署或者迁移整个服务当怎么处理呢？。docker-compose便是用来解决此问题而来的。  

Compose是一个用来配置多个Docker容器的应用，你可以通过一个`yaml`配置文件，来一键配置你的服务。接下来，我们开始创建自己的compose配置文件。

```
version: "3.9"

services:
  # Nginx服务器
  nginx:
    image: nginx:latest
    networks:
      - net
    volumes:
      - nginx-log:/var/log/nginx/
    ports:
      - "80:80"
  # 前端SPA页面
  web:
    build: ./webSpa
    networks:
      - net
    environment:
      NODE_ENV: production
    ports:
      - "3000:3000"
  # 服务端
  server:
    build: ./server
    networks:
      - net
    environment:
      NODE_ENV: production
    volumes:
      - server-log:logs
    ports:
      - "4000:4000"
  # mysql数据库
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
    ports:
      - "3306:3006"
    volumes:
      - mysql-data:var/lib/mysql

volumes:
  nginx-log:
    driver: local
  server-log:
    driver: local
  mysql-data:
    driver: local

networks:
  net:
    driver: bridge
```


### compose配置

### 配置开发环境

## 服务部署

### CI/CD

### 服务监测

### 日志接入





