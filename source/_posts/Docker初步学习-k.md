---
title: Docker初步学习-$k
date: 2018-01-31 09:53:17
tags: docker
---

## Docker简介
Docker是用来做什么的，举个例子，有一批货物需要运走，此时需要一种服务--运输服务，假设目前没有交通工具，但是拥有组装汽车的所有原材料（轮胎，发动机等等），想要实现运输就需要自己手动组装，并且要了解各个部件的工作原理，非常的麻烦。而Docker则会帮我们组装好一辆货车，不需要你进行麻烦且繁琐的操作，当你需要其他服务也是一样，直接通过Docker可以直接拿来使用。现在想想开发php时需要哪些服务，nginx、php-fpm、mysql、redis等，有了Docker你不需要一个一个安装、编译、配置，使用docker pull 命令可以轻松拿到这些，你要做的只是让他们之间彼此关联。 

## 基本概念
一个完整的Docker有以下几个部分组成：
1、Docker Daemon守护进程（Docker 服务，类似于mysql-server）
2、Docker Client客户端（使用 ' docker [command] '完成一些操作,类似于mysql-client）
3、Image镜像，镜像是可被Docker运行的文件，如同例子中交通工具的原材料+如何组装成货车的配置信息。
4、Container容器，镜像运行时的实体，在例子中Docker将原材料载入，之后加工成货车（容器），同样的原材料+同样的货车配置信息，就可以生产出多辆货车，容器也是一样。Docker可以从一个镜像运行出多个同样的容器，所以一个容器出现故障，重新启动一个就ok。容器可以被创建、启动、停止、删除、暂停。
当然你也可以给货车增加个导弹系统，只要你有原材料和组装导弹的配置信息，那么新产出的火车可以提供运输和作战两种服务，同理一个容器可以运行php+nginx+mysql等不同的服务，一切来源于你的镜像。
容器的实质是进程，但与直接在宿主执行的进程不同，容器进程运行于属于自己的独立的命名空间。因此容器可以拥有自己的root文件系统、自己的网络配置、自己的进程空间，甚至自己的用户 ID 空间。容器内的进程是运行在一个隔离的环境里，虽然容器内是虚拟环境，使用起来，就好像是在一个独立于宿主的系统下操作一样。

## Docker比传统虚拟机更快

![图一:  虚拟机 vs Docker](http://upload-images.jianshu.io/upload_images/5306603-00a830355296979e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Docker也是一种虚拟化技术。与传统虚拟机对比，虚拟机运行在虚拟硬件上，应用运行在虚拟机内核上。而 Docker daemon 是宿主机上的一个进程, 应用只是 Docker daemon 的一个子进程, 换句话说， 应用直接运行在宿主机内核上；虚拟机需要特殊硬件虚拟化技术支持，因而只能运行在物理机上。Docker 没有硬件虚拟化，因而可以运行在物理机、虚拟机，甚至 Docker 容器内(嵌套运行)；因为没有硬件虚拟化及多运行一个 Linux 内核的开销，应用运行在 Docker 上比虚拟机上更轻、更快。
## 运行一个容器
了解相关概念之后开始练习使用。
1、首先在你的电脑上完成Docker的[安装](https://docs.docker.com/engine/installation/)，
2、寻找自己想要使用的镜像文件
去[Docker Hub](https://hub.docker.com/)（是一个集中存储镜像的服务，称为**Docker Registry**）上检索自己想要使用的服务，检索出的每条内容被称为一个仓库（Repository），每个仓库中有多个标签（tags，如php：5.5，其中php是仓库名，5.5是标签），一个标签对应一个镜像，拉取镜像到本地的命令是 ‘docker pull 镜像名:仓库名’， 如
`docker pull php:5.5`

当标签是 latest 时，可省略：
`docker pull php 等同于 docker pull php:latest`

![图二：docker hub 检索结果](http://upload-images.jianshu.io/upload_images/5306603-8b49137fefa0037e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![图三：含有多个标签的 php 仓库](http://upload-images.jianshu.io/upload_images/5306603-7e82e54f90a6a23c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

也可以通过命令行搜索，如搜索php相关镜像：
`docker search php`

![图四： docker search 结果](http://upload-images.jianshu.io/upload_images/5306603-a5dce1f3a56d0f8e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

请尝试下载一个nginx 镜像到本地，要指定标签。
3、查看本地镜像列表
`docker images`

4、创建容器
`docker run -d --name nginx-test -v /var/www:/var/www -p 8080:80  nginx`

-d,让容器在后台执行；
--name，给容器分配名字
-v，挂载数据卷。容器像正常的Linux系统一样可以存储数据，但是当运行的容器出现故障不能正常启动时，我们就要通过同样的镜像运行一个新的容器实例，容器运行的服务是相同的，但是彼此隔离，不能共用数据，所以要通过数据卷的挂载实现数据的持久化。-v相当于把主机的/var/www目录同步到容器中的/var/www目录，如果容器不存在这个目录，则自动创建，挂载后两个文件夹实现了数据双向同步，每个容器的数据得以保存。新启用的nginx容器实例通过同样的挂载方式可以直接使用之前的数据。想要修改容器中nginx服务的配置，就要使用到数据卷，因为这是可变数据。数据卷是一个可供一个或多个容器使用的特殊目录，有如下特点，数据卷可以在容器之间共享和重用；对数据卷的修改会立马生效；对数据卷的更新，不会影响镜像；数据卷默认会一直存在，即使容器被删除
-p，端口映射，主机的8080端口映射到容器的80端口，此时访问主机8080会转到容器的80端口来获取数据
最后的nginx就是仓库名，会寻找nginx:latest的镜像启动，这里也可用本地镜像的id，镜像id通过 docker images 命令 可以得到
## 容器的管理
容器启动之后，查看本地正在运行的容器
`docker ps`

查看本地所有容器，包括运行中和暂停运行的
`docker ps -a`

![图五 : docker ps -a 结果](http://upload-images.jianshu.io/upload_images/5306603-65669cc976657a78.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

将暂停的容器启动
`docker start 容器名称/容器id（容器id可用 docker ps 查看）`

暂停一个运行的容器
`docker stop 容器名称/容器id`

删除一个容器,删除正在运行的容器要增加 -f 参数
`docker rm 容器名称/容器id`

## 进入容器
进入容器的方式有多种，这里只介绍和推荐使用Docker自带的docker exec方式，Docker1.3增加新的exec命令行工具，进入container更加方便：
`docker exec -i -t  容器名称/id  bash`

bash是容器系统的shell，如容器是centos系统，shell可以是bash、sh，进入容器后，尝试运行 ' ls / ' 命令
## 结语
尝试运行一个容器实例，这会让你对Docker的有进一步理解。
## 参考链接
[http://jm.taobao.org/2016/05/12/introduction-to-docker/](http://jm.taobao.org/2016/05/12/introduction-to-docker/)
[https://yeasy.gitbooks.io/docker_practice/content/basic_concept/container.html](https://yeasy.gitbooks.io/docker_practice/content/basic_concept/container.html)
