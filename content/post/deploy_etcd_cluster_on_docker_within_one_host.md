## 把玩etcd：基于docker在单机拉起集群



在单机上相对隔离的拉起etcd集群，docker是首选方案。首先要确保etcd的docker节点处于同一个网段，网络互相联通；然后根据官方文档，在etcd节点启动时做好配置，做到各节点互相发现；之后就可以使用etcdctl客户端来操作集群了。

#### 创建子网

>  sudo docker network create --subnet 172.16.1.0/24 etcds

docker创建子网非常简单，可以指定网段，指定子网名

#### 启动节点

###### 准备工作

1. 采用etcd源码附送的Dockerfile-release文件创建自己的docker镜像，这里是bg2bkk/etcd:latest；实际上采用官方的Dockerfile build出来也是可以的
2. 各个实例节点采用指定ip，即拉起docker时需要已知节点ip；各节点指定hostname用于节点名，指定容器名name用于容器间的dns路由。以节点1为例，参数为 ***--ip 172.16.0.5 --hostname etcd1 --name etcd1***
3. 启动前在宿主机创建***/work/etcd***目录，用于各容器的snap和wal的存储
4. 指定集群配置  --initial-cluster-token  和 --initial-cluster

```bash

docker run -d -ti --ip 172.16.0.6 --volume=/work/etcd/etcd2:/etcd-data --hostname etcd2 --name etcd2 --network etcds bg2bkk/etcd:latest /usr/local/bin/etcd --data-dir=/etcd-data --name etcd2 --initial-advertise-peer-urls http://172.16.0.6:2380 --listen-peer-urls http://172.16.0.6:2380 --advertise-client-urls http://172.16.0.6:2379 --listen-client-urls http://172.16.0.6:2379 --initial-cluster-token etcd-cluster --initial-cluster etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380 --initial-cluster-state new



docker run -d -ti --ip 172.16.0.5 --volume=/work/etcd/etcd1:/etcd-data --hostname etcd1 --name etcd1 --network etcds bg2bkk/etcd:latest /usr/local/bin/etcd --data-dir=/etcd-data --name etcd1 --initial-advertise-peer-urls http://172.16.0.5:2380 --listen-peer-urls http://172.16.0.5:2380 --advertise-client-urls http://172.16.0.5:2379 --listen-client-urls http://172.16.0.5:2379 --initial-cluster-token etcd-cluster --initial-cluster etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380 --initial-cluster-state new



docker run -d -ti --ip 172.16.0.7 --volume=/work/etcd/etcd3:/etcd-data --hostname etcd3 --name etcd3 --network etcds bg2bkk/etcd:latest /usr/local/bin/etcd --data-dir=/etcd-data --name etcd3 --initial-advertise-peer-urls http://172.16.0.7:2380 --listen-peer-urls http://172.16.0.7:2380 --advertise-client-urls http://172.16.0.7:2379 --listen-client-urls http://172.16.0.7:2379 --initial-cluster-token etcd-cluster --initial-cluster etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380 --initial-cluster-state new

```

#### 使用集群

* 启动集群后，可以通过docker logs -f etcd1等方式查看log

* etcdctl工具需要指定endpoints，如下图所示；endpoints可以指定为容器名+端口，比如etcd1:2379

```bash
$etcdctl --endpoints=172.16.0.5:2379 member list

ade526d28b1f92f7, started, etcd1, http://etcd1:2380, http://172.16.0.5:2379
bd388e7810915853, started, etcd3, http://etcd3:2380, http://172.16.0.7:2379
d282ac2ce600c1ce, started, etcd2, http://etcd2:2380, http://172.16.0.6:2379

```

* 如果用于跨机调试，例如将etcd集群作为远程配置中心，那么可以在etcd1容器增加配置 -p 2379:2379 -p 2380:2380，即可跨主机从远程指定--endpoints=host:2379来访问集群，非常方便。

##### 附送脚本

```bash

DIR=/work/etcd
CLUSTER=etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380

NAME=etcd2
IP=172.16.0.6
echo docker run -d -ti --ip $IP  --volume=$DIR/$NAME:/etcd-data --hostname $NAME --name $NAME --network etcds bg2bkk/etcd:latest /usr/local/bin/etcd --data-dir=/etcd-data --name $NAME --initial-advertise-peer-urls http://$IP:2380 --listen-peer-urls http://$IP:2380 --advertise-client-urls http://$IP:2379 --listen-client-urls http://$IP:2379  --initial-cluster-token etcd-cluster  --initial-cluster $CLUSTER --initial-cluster-state new


NAME=etcd1
IP=172.16.0.5
echo docker run -d -ti --ip $IP  --volume=$DIR/$NAME:/etcd-data --hostname $NAME --name $NAME --network etcds bg2bkk/etcd:latest /usr/local/bin/etcd --data-dir=/etcd-data --name $NAME --initial-advertise-peer-urls http://$IP:2380 --listen-peer-urls http://$IP:2380 --advertise-client-urls http://$IP:2379 --listen-client-urls http://$IP:2379  --initial-cluster-token etcd-cluster  --initial-cluster $CLUSTER --initial-cluster-state new



NAME=etcd3
IP=172.16.0.7
echo docker run -d -ti --ip $IP  --volume=$DIR/$NAME:/etcd-data --hostname $NAME --name $NAME --network etcds bg2bkk/etcd:latest /usr/local/bin/etcd --data-dir=/etcd-data --name $NAME --initial-advertise-peer-urls http://$IP:2380 --listen-peer-urls http://$IP:2380 --advertise-client-urls http://$IP:2379 --listen-client-urls http://$IP:2379  --initial-cluster-token etcd-cluster  --initial-cluster $CLUSTER --initial-cluster-state new


```





