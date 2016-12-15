+++
date = "2016-08-14T16:09:53+08:00"
draft = true
title = "docker操作指南"

+++

docker学习操作

* docker info
	* 查看docker是否正常工作
* systemctl status docker
	* 查看docker守护进程是否工作

* docker ps 
	* 查看正在运行的容器
	* docker ps -a 查看所有容器

* docker images
	* 查看镜像

* docker run -t -i ubuntu /bin/bash
	* 运行docker镜像
	* -t 开启terminal
	* -i 保证stdin开启
	* docker run --name huang_ubuntu -t -i ubuntu /bin/bash
		* 别名huang_ubuntu
	
* docker attach huang_ubuntu
	* 重新附着到运行容器中

* 运行守护式容器
	* docker run --name daemon_ubuntu -d ubuntu /bin/bash -c $CMD
	* docker logs -f daemon_ubuntu
		* 查看docker日志
	* docker logs -ft daemon_ubuntu
		* 加上时间戳
	
* docker stats huang_ubuntu
	* 容器运行信息

* docker exec -t -i huang_ubuntu /bin/bash
	* docker容器中执行程序，采用-t -i方式打开/bin/bash可以打开容器终端，能够附着在运行中容器里


* [docker build](http://docs.daocloud.io/allen-docker/docker-build-cache)
	* [mongoDB](http://www.heblug.org/chinese_docker/examples/mongodb.html)

