---
title: 使用Docker快速配置和编译OpenJDK
tags:
    - OpenJDK
    - Docker
---

## 查找快速编译环境的Dockers镜像

使用`docker search bolingcavalry`查找镜像：

```bash
root:~$ docker search bolingcavalry
NAME                                        DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
bolingcavalry/disconf_mysql                 Element of baidu disconf enviroment, mysql i…   2
bolingcavalry/kafka                                                                         2
bolingcavalry/centos6.7-jdk1.8-ssh                                                          1
bolingcavalry/ubuntu16-openresty                                                            1
bolingcavalry/bolingcavalryopenjdk                                                          1
bolingcavalry/k8stomcatdemo                                                                 0
bolingcavalry/centos7-hbase126-standalone                                                   0
bolingcavalry/disconf_nginx                 Element of baidu disconf enviroment, nginx i…   0
bolingcavalry/elkdemo                                                                       0
bolingcavalry/elasticsearch-with-ik         集成了ik分词器的elasticsearch镜像，tag即elas…              0                                                                  
bolingcavalry/ssh-kafka292081-zk346                                                         0
bolingcavalry/nginx-with-tomcat-host                                                        0
bolingcavalry/online_deploy_tomcat                                                          0
bolingcavalry/disconf_tomcat                Element of baidu disconf enviroment, with di…   0
bolingcavalry/dubbo_admin_tomcat                                                            0
bolingcavalry/rabbitmq-server                                                               0
bolingcavalry/disconf_standalone_demo                                                       0
bolingcavalry/elasticsearch-head            Tag is the version number of elasticsearch，…    0
bolingcavalry/dubbo_provider_tomcat                                                         0
bolingcavalry/k8spvdemo                                                                     0
bolingcavalry/openjdksrc11                                                                  0
bolingcavalry/bolingcavalrytomcat                                                           0
bolingcavalry/nacosserver                                                                   0
bolingcavalry/ubuntu16-mongodb349                                                           0
bolingcavalry/rabbitmqproducer                                                              0
```

## 创建并运行容器

在上一步中选择需要的镜像，如bolingcavalry/bolingcavalryopenjdk，使用命令`docker run --name=<自定义容器名> -idt <镜像名>`创建并运行容器：

```bash
docker run --name=compilejdk -idt bolingcavalry/bolingcavalryopenjdk
```

然后使用`docker exec -it <自定义容器名> /bin/bash`进入容器：

```bash
docker exec -it compilejdk /bin/bash
```

## 开始编译

进入源码目录：

```bash
cd /usr/local/openjdk
```

进行配置：

```bash
./configure
```

开始编译：

```bash
make all DISABLE_HOTSPOT_OS_VERSION_CHECK=OK CONF=linux-x86_64-normal-server-release
```

## 查看成果

然后等待编译完成。当编译完成后，进入输出目录：

```bash
cd /usr/local/openjdk/build/linux-x86_64-normal-server-release/jdk
ls -l
```

可以使用java命令验证：

```bash
cd /usr/local/openjdk/build/linux-x86_64-normal-server-release/jdk/bin
./java -version
```
