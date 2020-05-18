---
title: docker
date: 2020-05-18 16:08:24
index_img: https://www.runoob.com/wp-content/uploads/2016/04/docker01.png
---


# docker初识
#### 简介   
- Docker 是一个开源的应用容器引擎，基于 Go 语言 并遵从 Apache2.0 协议开源
- 与虚拟机类似，但是更加轻量级
- 生成docker镜像（image）之后，方便移植到其他机器
- 同一个镜像，可以快捷方便启动多次


#### 相关概念
- image：Docker 镜像（Image），就相当于是一个 root 文件系统，可以本地构建
- container:容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等
- DockerHub:镜像中心仓库，自己构建的镜像基本会依赖于基础镜像，本地没有的则会从dockerhub拉取;同时也可以将自己的image发布，供其他人使用

#### [安装指南](https://docs.docker.com/engine/install/centos/)

```
## 以centos为例（社区版）
#1. 设置docker存储库
sudo yum install -y yum-utils
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
#2.安装docker引擎
sudo yum install docker-ce docker-ce-cli containerd.io
#3.启动docker服务
sudo systemctl start docker
```

#### 常用命令
```
docker pull xxx  --拉取镜像
docker build -t xxx  . --创建镜像
docker images  --查看镜像列表
docker rmi xxx --删除镜像
docker run -d -p 1234:1234  --name myName  imageName  xxx --启动容器  -p 端口 映射 -d 后台运行
docker ps -a  --查询正在运行的容器
docker stop containerId  --停止某个容器
docker rm xxx  -- 移除某个容器
docker exec -it xxx bash  --进入相关容器
exit -- 退出相关容器
docker rm `docker ps -a -q` -- 删除所有容器
docker rmi `docker images -q` -- 删除所有镜像
docker rmi `docker images -q | awk '/^<none>/ { print $3 }'` -- 删除所有为none的容器
docker rmi --force `docker images | grep doss-api | awk '{print $3}'`    //其中doss-api为关键字
```

#### dockerHub使用（此处以mysql为例）
- 首先从dockerhub迁出mysql镜像,这里以5.7为例
   
```
docker pull mysql:5.7
```
- 部署容器
```
 docker run --name mysql57 -e MYSQL_ROOT_PASSWORD=123789 -d -p 3307:3306 mysql:tag
```
启动mysql镜像，docker run是启动容器的命令；i是交互式操作，t是一个终端，d指的是在后台运行，-P指在本地生成一个随机端口，用来映射mysql的3306端口，mysql指运行mysql镜像.

- 进入容器
```
docker exec -it mysql57 bash
mysql -u root -p
123456
```
如上操作即可访问mysql，如需外网访问需要开放root用户权限
```
mysql> use mysql;

mysql> update user set authentication_string = password('123456') where user = 'root';

mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;

mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'127.0.0.1' IDENTIFIED BY '123456' WITH GRANT OPTION;

flush privileges;
```
```
exit --退出容器
```
此时即可通过宿主机（docker容器所在服务器）的IP+映射端口访问部署的mysql数据库

