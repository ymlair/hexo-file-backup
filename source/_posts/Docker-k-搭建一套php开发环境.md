---
title: Docker($k)搭建一套php开发环境
date: 2018-01-31 09:52:01
tags: docker
---

## 导语
下面内容将介绍如何把容器当作一个命令来使用以及搭建一套php+nginx的 web 服务，这里需要两个镜像，用两个镜像的主要目的是学习如何让 Docker 容器之间相互通信。阅读完下面的内容就可以搭建自己的 Docker 服务了。
## 把 php 容器当作命令行使用
镜像下载：
`docker pull php:7.0-fpm-alpine php`
这里的镜像是基于 [alpine](https://alpinelinux.org/) 系统的，因为基于alpine系统的镜像文件会比较小，下载速度更快。由于国内下载镜像文件较慢，推荐使用镜像加速器[DaoCloud](http://www.daocloud.io/mirror#accelerator-doc)。
![图一：docker images | grep php](http://upload-images.jianshu.io/upload_images/5306603-4834550be10bce02.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下载镜像是为了搭建一个 web 服务，如果只想简单的使用 php 命令行，怎么办？我们知道从镜像启动的容器中肯定是可以使用命令行，如果每次使用 php 命令行都进入容器，显得特别麻烦，其实 Docker 可以这样用：
`docker run -it --rm  php:7.0-fpm-alpine php --version`
![图二：把php容器当作命令行使用](http://upload-images.jianshu.io/upload_images/5306603-d76bb6d80410473b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
命令比较长，给它设置个别名就好多了。下面介绍下相关参数：
-i：以交互模式运行容器，通常与 -t 同时使用；
-t： 为容器重新分配一个伪输入终端，通常与 -i 同时使用；
--rm：容器退出时自动删除，如果不加这个参数，当你执行完上面的命令，php容器会退出，变为一个暂停状态的容器，通过 `docker ps -a` 可以查询到；
php --version：在容器名后面的字符会被当作容器的shell命令来处理；
*注：关于参数 -i -t ，这里上面的命令可以不加，因为没有交互操作，在使用node容器的命令行时会有交互，需要加上，两个参数同时使用就好：
`docker run -i -t --rm node:alpine node`

![图三：node容器命令行的使用](http://upload-images.jianshu.io/upload_images/5306603-d96f8843e4eb70c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

想用容器同时执行多个命令，不能直接在后面加 `&&`，需要使用 `sh -c `来实现，：
`docker run  --rm  php:7.0-fpm-alpine sh -c ' echo "123" && echo "456" ' `

## 启动php服务
执行命令：
```
docker run \
	-d \
	--name php \
	-v /root/docker/etc/php/php.ini:/usr/local/etc/php/conf.d/php.ini:ro \
	-v /root/docker/html:/var/www/html \
	php:7.0-fpm-alpine
```
-d：后台运行容器，并且返回容器 ID；
--name：给容器命名，容器名是唯一的，操作容器时可以使用名称代替容器 ID；
:ro：表示挂载的文件或者文件夹为只读模式；
从命令可以知道容器是后台运行，名字是 php，它挂载了主机的一个文件 php.ini 和一个目录 /root/docker/html，并且 php.ini 是只读的，所以在容器内不可以对这个文件做修改。/usr/local/etc/php/conf.d 这个目录是容器中的 php 读取用户自定义配置文件的目录，正常情况下都可以在 Docker Hub 上有说明，如果没有可以自己运行 `phpinfo();` 来查看。之前介绍过，只要挂载，那么本地主机目录就会和容器内的目录同步。需要修改容器的 php 配置时，只要在主机本地编辑保存这个 php.ini 文件，然后执行：
`docker restart php （php是容器名字）`

## php 容器和 nginx 容器通信
首先下载nginx镜像：
`docker pull nginx:stable-alpine`
启动 nginx 服务：
```
docker run \
	-d \
	--link php \
	--name nginx \
	-v /root/docker/etc/nginx/conf.d/:/etc/nginx/conf.d/ \
	-v /root/docker/html:/var/www/html \
        -v /var/log/nginx:/var/log/nginx \
	-p 8088:80 \
	nginx:stable-alpine
```
--link：确保 nginx 可以与 php 之间通信，在 nginx 容器中直接  `ping php` 是可以通的，实际上加上这个参数后，会在 nginx 容器增加 host 解析，如图：

![图四：nginx与php之间通信](http://upload-images.jianshu.io/upload_images/5306603-04260da4a4a546ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
增加 nginx 虚拟主机配置，放到主机目录  /root/docker/etc/nginx/conf.d 下：
```
server {
	listen 80;
	root /var/www/html;
	# Add index.php to the list if you are using PHP
	index index.php index.html index.htm index.nginx-debian.html;

	location ~ \.php$ {
		fastcgi_pass php:9000;
                fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
		include        fastcgi_params;
	}
	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log  debug;
}
```
fastcgi_pass 后面使用的 php 就是 --link 参数增加的host解析，直接用别名代替ip地址，更加方便。然后重启 nginx 服务：
`docker restart nginx`
在 主机本地的 /root/docker/html 目录新建 index.php：
`echo "<?php\nphpinfo();" | tee /root/docker/html/index.php`
现在一个 web 服务搭建好了，如图：

![图五：web服务](http://upload-images.jianshu.io/upload_images/5306603-9e68506a92c18fbc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 查看 nginx 日志
容器内 nginx 的日志会写入容器内的 /var/log/nginx 目录下，由于这个目录和主机的 /var/log/nginx 目录是同步的，所以，想看容器内 nginx 的日志，查看主机的文件 /var/log/nginx/access.log 就可以：
 `tail -f /var/log/nginx/access.log`

![图六：本地查看nginx容器访问日志](http://upload-images.jianshu.io/upload_images/5306603-c2d664e8bcf92ab7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 结语
基本的启动配置服务的命令上面都有介绍，自己可以尝试给这个 web 服务增加个 mysql
 存储功能。


