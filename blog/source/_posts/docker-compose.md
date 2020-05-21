---
title: docker compose
date: 2020-05-19 13:42:24
index_img: https://www.ruanyifeng.com/blogimg/asset/2018/bg2018021311.jpg
categories:
    - 容器化相关
tags:
    - docker-compose
---

# docker compose初识
#### 简介   
- 管理多个docker容器
- 简化部署和对多个容器的操作
- 单机情况下适用（即容器都在同一个主机）

#### [安装指南](https://docs.docker.com/compose/install/)

```
## 1.获取官方镜像
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
## 2.给予权限
sudo chmod +x /usr/local/bin/docker-compose
## 3.添加快捷方式
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
## 4.尝试
docker-compose --version
```

#### 常用命令
```
docker-compose ps  --查看在运行容器列表
docker-compose kill --停止容器
docker-compose stop --停止容器 service
docker-compose start --启动容器 service
docker-compose up  --通过docker-compose.yml启动
docker-compose images --查看镜像列表
docker-compose build --no-cache xxxx --无缓存式构建镜像

```

#### [docker-compose.yml编写](https://blog.csdn.net/Aria_Miazzy/article/details/89326829)
- 以yy_docker的docker-compose.yml为例
```
version: '2'
#serviece服务列表，以下为各个服务
services:
  mysql:
    ## 镜像名称
    image: dcloud_mysql
    ## 容器名称
    container_name: db_mysql
    privileged: true
    ## 构建相关
    build:
      ## 指定相对路径
      context: ./mysql/
      ## 指定dockerfile
      dockerfile: ./Dockerfile
    ## 设置意外退出后一致重启  
    restart: always
    ## 对外映射端口号
    ports:
      - 0.0.0.0:61903:3306
    ## 特殊的环境变量参数  
    environment:
        - MYSQL_ROOT_PASSWORD=admin123!@#qwe
    ## 文件夹映射    
    volumes:
      - ./data/mysql:/var/lib/mysql
  redis:
    image: dcloud_redis
    container_name: db_redis
    build:
      context: ./redis/
      dockerfile: ./Dockerfile
    restart: always
    ports:
      - 0.0.0.0:61904:17328
  
  dcloud:
    image: dcloud_api
    container_name: dcapi
    privileged: true
    restart: always
    build:
      context: ./dcloud/
      dockerfile: ./Dockerfile
    ports:
      - 0.0.0.0:61901:60060
      - 0.0.0.0:61902:60065
    ## 与其他服务相连接，通过容器名可ping通  
    links:
      - mysql
      - redis
    ## 依赖关系，后于mysql启动  
    depends_on:
      - mysql
  nginx:
    image: dcloud_nginx
    container_name: dc_nginx
    privileged: true
    restart: always
    build:
      context: ./nginx/
      dockerfile: ./Dockerfile
    ports:
      - 0.0.0.0:61905:62052
    links:
      - mysql
      - redis
    depends_on:
      - dcloud
    
```