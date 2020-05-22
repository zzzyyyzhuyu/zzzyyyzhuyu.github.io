---
title: docker-supervisor
date: 2020-05-22 17:49:15
index_img: 
categories:
     - 容器化相关
tags: 
     - docker
     - supervisor
---
# docker+supervisor顺序启动jar包
####  应用场景
- 一个容器内部署多个jar包，jar之间相互依赖的服务，必须有启动先后顺序
- 多个进程采用supervisor进行进程管理
####  思路
1. 基于centos/ubantu 配置jdk环境和supervisor环境
2. 编写supervisor配置文件
3. 编写dockerfile
4. 测试运行
#### 具体实现
- supervisor配置文件如下
```
# 配置文件包含目录和进程
# 第一段 supervsord 配置软件本身，使用 nodaemon 参数来运行。
# 第二段包含要控制的 多个服务。每一段包含一个服务的目录和启动这个服务的命令。
# priority为优先级 优先级低的先运行，但不保证执行顺序，可能后启动的先运行起来
# startsecs 运行时长xxx后 认为该服务启动成功
# startretries 启动失败时重新尝试的次数
# 启动说明 eureka ->config ->auth ->getway -> xxxx
# 当没有按照顺序启动起来时，会报错然后进行重新启动，在多次尝试之后会按依赖顺序启动
[supervisord]
nodaemon=true


[program:dcloud-eureka] 
command=java -jar -Xms256m -Xmx512m /usr/local/docker_files/exe/dcloud-eureka.jar >/dev/null 2>&1 &
priority= 2
startsecs=2

[program:dcloud-config] 
command=java -jar -Xms256m -Xmx256m /usr/local/docker_files/exe/dcloud-config.jar >/dev/null 2>&1 &
priority=20
startsecs=60

[program:dcloud-auth] 
command=java -jar -Xms512m -Xmx1024m /usr/local/docker_files/exe/dcloud-auth.jar  >/dev/null 2>&1 &
autorestart=true
priority=30
startsecs=150
startretries=100

[program:dcloud-gateway] 
command=java -jar -Xms512m -Xmx512m /usr/local/docker_files/exe/dcloud-gateway.jar >/dev/null 2>&1 &
autorestart=true
priority=40
startsecs=250
startretries=100

[program:dcloud-base] 
command=java -jar -Xms512m -Xmx1024m /usr/local/docker_files/exe/dcloud-base.jar >/dev/null 2>&1 &
autorestart=true
priority=80
startsecs=250
startretries=100

[program:dcloud-data] 
command=java -jar -Xms512m -Xmx1024m /usr/local/docker_files/exe/dcloud-data.jar >/dev/null 2>&1 &
autorestart=true
priority=100
startsecs=250
startretries=100

[program:dcloud-server] 
command=java -jar -Xms512m -Xmx1024m /usr/local/docker_files/exe/dcloud-server.jar >/dev/null 2>&1
autorestart=true
priority=200
startsecs=250
startretries=100

```
    
- Dockerfile如下（ubantu版本）
```
############################################
# version : birdben/tools:v1
# desc : 当前版本安装的ssh，wget，curl，supervisor 
############################################
# 设置继承自ubuntu官方镜像
FROM ubuntu:14.04


# 注意这里要更改系统的时区设置，因为在 web 应用中经常会用到时区这个系统变量，默认的 ubuntu 会让你的应用程序发生不可思议的效果哦
ENV DEBIAN_FRONTEND noninteractive

# 清空ubuntu更新包 替换源为阿里云的源
RUN sudo rm -rf /var/lib/apt/lists/*
ADD sources.list /etc/apt/sources.list
# 一次性安装vim，wget，curl，ssh server等必备软件
# RUN echo "deb http://archive.ubuntu.com/ubuntu precise main universe"> /etc/apt/sources.list

RUN sudo apt-get update
RUN sudo apt-get install -y vim wget curl openssh-server sudo

# make a new directory to store the jdk files
RUN mkdir /usr/local/java

# copy the jdk  archive to the image,and it will automaticlly unzip the tar file
ADD jdk-8u221-linux-x64.tar.gz /usr/local/java/

# make a symbol link
RUN ln -s /usr/local/java/jdk1.8.0_221 /usr/local/java/jdk

# 安装supervisor工具
RUN sudo apt-get install -y supervisor
RUN sudo mkdir -p /var/log/supervisor

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# jar包相关添加
COPY dcloud-eureka.jar /usr/local/docker_files/exe/dcloud-eureka.jar
COPY dcloud-auth.jar /usr/local/docker_files/exe/dcloud-auth.jar
COPY dcloud-data.jar /usr/local/docker_files/exe/dcloud-data.jar
COPY dcloud-server.jar /usr/local/docker_files/exe/dcloud-server.jar
COPY dcloud-gateway.jar /usr/local/docker_files/exe/dcloud-gateway.jar
COPY dcloud-base.jar /usr/local/docker_files/exe/dcloud-base.jar
COPY dcloud-config.jar /usr/local/docker_files/exe/dcloud-config.jar



# set environment variables
ENV JAVA_HOME /usr/local/java/jdk
ENV JRE_HOME ${JAVA_HOME}/jre
ENV CLASSPATH .:${JAVA_HOME}/lib:${JRE_HOME}/lib
ENV PATH ${JAVA_HOME}/bin:$PATH

# 容器需要开放SSH 22端口
EXPOSE 22 60065 60060


# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]



```
- centos版本dockerfile
```
############################################
# version : birdben/tools:v1
# desc : 当前版本安装的ssh，wget，curl，supervisor 
############################################
# 设置继承自centos官方镜像
FROM centos:7


# 注意这里要更改系统的时区设置，因为在 web 应用中经常会用到时区这个系统变量，默认的 ubuntu 会让你的应用程序发生不可思议的效果哦
ENV DEBIAN_FRONTEND noninteractive

#升级系统
RUN yum -y update
RUN yum  install -y vim wget curl
RUN yum  install -y  epel-release
RUN yum  install -y supervisor

# make a new directory to store the jdk files
RUN mkdir /usr/local/java

# copy the jdk  archive to the image,and it will automaticlly unzip the tar file
ADD jdk-8u221-linux-x64.tar.gz /usr/local/java/

# make a symbol link
RUN ln -s /usr/local/java/jdk1.8.0_221 /usr/local/java/jdk

RUN  mkdir -p /var/log/supervisor

# 添加 supervisord 的配置文件，并复制配置文件到对应目录下面。（supervisord.conf文件和Dockerfile文件在同一路径）
COPY supervisord.conf /etc/supervisord.conf

# jar包相关添加
COPY dcloud-eureka.jar /usr/local/docker_files/exe/dcloud-eureka.jar
COPY dcloud-auth.jar /usr/local/docker_files/exe/dcloud-auth.jar
COPY dcloud-data.jar /usr/local/docker_files/exe/dcloud-data.jar
COPY dcloud-server.jar /usr/local/docker_files/exe/dcloud-server.jar
COPY dcloud-gateway.jar /usr/local/docker_files/exe/dcloud-gateway.jar
COPY dcloud-base.jar /usr/local/docker_files/exe/dcloud-base.jar
COPY dcloud-config.jar /usr/local/docker_files/exe/dcloud-config.jar
COPY license.lic /usr/local/docker_files/exe/license.lic
COPY  license.lic   /license.lic 


# set environment variables
ENV JAVA_HOME /usr/local/java/jdk
ENV JRE_HOME ${JAVA_HOME}/jre
ENV CLASSPATH .:${JAVA_HOME}/lib:${JRE_HOME}/lib
ENV PATH ${JAVA_HOME}/bin:$PATH

# 容器需要开放SSH 22端口
EXPOSE 22 60065 60060


# 执行supervisord来同时执行多个命令，使用 supervisord 的可执行路径启动服务。
CMD ["/usr/bin/supervisord"]

```
- 换源的文件（防止api-get update失败）sources.list 

```
deb http://mirrors.aliyun.com/ubuntu/ trusty main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ trusty-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ trusty-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ trusty-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ trusty-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty-backports main restricted universe multiverse

```

- Tips
> ubuntu在api-get update时，部分电脑可能会出现 fetch failed,故将ubuntu源替换为阿里云的源






