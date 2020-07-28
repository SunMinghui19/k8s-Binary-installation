# k8s-Binary-installation
system version :centos 7   k8s version:1.16 



# 搭建一个完整的Kubernetes集群

```
 1、生产环境K8S平台规划  
 2、初始化服务器  
 3、为Etcd和APIServer自签SSL证书  
 4、Etcd数据库集群部署  
 5、部署Master组件  
 6、部署Node组件  
 7、部署K8s集群网络   
 8、部署集群内部DNS解析服务（CoreDNS）  
```

## 1、生产环境K8S平台规划
```
master: 
 主机名：k8s-master1  
 IP：192.168.133.180  
worker1:  
 主机名：k8s-node1  
 IP：192.168.133.181  
worker2： 
 主机名：k8s-node2  
 IP：192.168.133.182  

K8S版本：1.16  
安装方式：离线-二进制  
操作系统：centos 7  
```
## 2、初始化服务器

### 2.1 关闭防火墙
```
[所有主节点都执行]
# systemctl stop firewalld
# systemctl disable firewalld
```
### 2.2 关闭selinux
```
[所有节点都执行] 先配置临时关闭，然后修改文件成永久性关闭
#临时关闭
# setenforce 0
```
```
#永久性关闭
# vim /etc/selinux/config 
将config中的SELINUX=enforcing后面的修改为disabled
```
### 2.3 配置主机名
```
[所有主节点都执行]
hostnamectl set-hostname 主机名
使用hostname检验是否更改成功

#此处根据自己的集群配置设置主机名
```
 ### 2.4 配置名称解析（更改host文件）
 ```
[所有主节点都执行]
vim /etc/hosts
添加如下几行
192.168.133.180 k8s-master1
192.168.133.181 k8s-node1
192.168.133.182 k8s-node2
```
### 2.5 配置时间同步
```
选择一个节点作为服务端，剩下的作为客户端master1为时间服务器的服务端其他的为时间服务器的客户端
```
#### 2.5.1 配置k8s-master1的chrony
```
#yum install chrony –y
#vim /etc/chrony.conf
修改chrony.conf中以下三项
	server 127.127.1.0 iburst #127.127.1.0是本地时间服务器
	allow 192.168.133.0/24 #允许192.168.133网段下的ip访问
	local stratum 10
# systemctl start chronyd
# systemctl enable chronyd

通过如下命令验证是否安装设置成功
# ss -unl | grep 123
UNCONN     0      0            *:123                      *:*
```
#### 2.5.2 配置k8s-node1和k8s-node2的chrony
```
#yum install chrony –y
#vim /etc/chrony.conf
修改chrony.conf对应项成为如下内容
		server 192.168.133.180 iburst
# systemctl start chronyd
# systemctl enable chronyd

#worker节点的时间源为^* k8s-master1项即为正常
# chronyc sources 
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^* k8s-master1                  10   6    17    60    -44us[  -56us] +/-  192us
```
## 2.6、关闭交换分区
```
[所有主节点都执行]
我们在使用k8s的时候如果开着交换分区可能导致服务起不来
# swapoff –a  #暂时关闭
# vim /etc/fstab #永久关闭
	注释一行：swap
[root@localhost sunminghui]# free –m #检验是否成功
	       total        used        free      shared  buff/cache   available
Mem:           2818         107        2565           8         144        2539
Swap:             0           0           0
```
# 三、给etcd来颁发证书并部署etcd
```
给etcd颁发证书主要包括下面几步：
1）创建证书颁发机构
2）填写表单——写明etcd所在节点的IP
3）向证书颁发机构申请证书

因为需要上传文件，如果不使用xftp的话可以使用如下指令安装lrzsz
yum install -y lrzsz
```
## 3.1 给etcd颁发证书
```
第一步：上传TLS安装包
	传到/root下
第二部：
	#tar xvf /root/ TLS.tar.gz
	#cd /root/TLS
	# ./cfssl.sh（将三个文件复制到/usr/local/bin下）
	
	# cd etcd/
	#vim server-csr.json（申请证书之前需要填的表）
		！！！！！修改host中的IP地址，这里的IP是etcd所在节点的IP地址!!!!!

		{
		    "CN": "etcd",
		    "hosts": [
			"192.168.133.180",
			"192.168.133.181",
			"192.168.133.182"
			],
		    "key": {
			"algo": "rsa",
			"size": 2048
		    },
		    "names": [
			{
			    "C": "CN",
			    "L": "BeiJing",
			    "ST": "BeiJing"
			}
		    ]
		}
# ./generate_etcd_cert.sh

#如果执行如下命令，出现4个pem文件即为成功生成了证书颁发机构和etcd证书
# ls *pem
ca-key.pem  ca.pem  server-key.pem  server.pem
```
## 3.2 部署etcd集群
```
etcd需要三台虚拟机
在master、node1、node2上分别安装一个etcd,构建etcd集群
```
### 3.2.1 master上安装etcd
```
# tar xvf etcd.tar.gz
# mv etcd.service  /usr/lib/systemd/system
#mv etcd /opt/
# vim /opt/etcd/cfg/etcd.conf
如果布etcd集群那么ETCD_INITIAL_CLUSTER里面就要写全etcd集群的ip，如果只有一个etcd，那么填写一个ip即可。其余需要填写ip地址的地方填写各自节点的ip即可
	#[Member]
	ETCD_NAME="etcd-1"
	ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
	ETCD_LISTEN_PEER_URLS="https://192.168.133.180:2380"
	ETCD_LISTEN_CLIENT_URLS="https://192.168.133.180:2379"

	#[Clustering]
	ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.133.180:2380"
	ETCD_ADVERTISE_CLIENT_URLS="https://192.168.133.180:2379"
	ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.133.180:2380,etcd-2=https://192.168.133.181:2380,etcd-3=https://192.168.133.182:2380"
	ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
	ETCD_INITIAL_CLUSTER_STATE="new"
	
# rm -rf /opt/etcd/ssl/*
# \cp -fv /root/TLS/etcd/{ca,server,server-key}.pem /opt/etcd/ssl/
```
### 3.2.2 worker节点上安装etcd
```
将etcd管理程序和程序目录发送到node1和node2
# scp /usr/lib/systemd/system/etcd.service root@k8s-node1:/usr/lib/systemd/system/
# scp /usr/lib/systemd/system/etcd.service root@k8s-node2:/usr/lib/systemd/system/
# scp -r /opt/etcd/  root@k8s-node1:/opt/
# scp -r /opt/etcd/  root@k8s-node2:/opt/
		
在node1上修改配置文件
# vim /opt/etcd/cfg/etcd.conf
#配置原理同master
	#[Member]
	ETCD_NAME="etcd-2"
	ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
	ETCD_LISTEN_PEER_URLS="https://192.168.133.181:2380"
	ETCD_LISTEN_CLIENT_URLS="https://192.168.133.181:2379"

	#[Clustering]
	ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.133.181:2380"
	ETCD_ADVERTISE_CLIENT_URLS="https://192.168.133.181:2379"
	ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.133.180:2380,etcd-2=https://192.168.133.181:2380,etcd-3=https://192.168.133.182:2380"
	ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
	ETCD_INITIAL_CLUSTER_STATE="new"

在node2上修改配置文件
# vim /opt/etcd/cfg/etcd.conf
配置原理同master
	#[Member]
	ETCD_NAME="etcd-3"
	ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
	ETCD_LISTEN_PEER_URLS="https://192.168.133.182:2380"
	ETCD_LISTEN_CLIENT_URLS="https://192.168.133.182:2379"

	#[Clustering]
	ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.133.182:2380"
	ETCD_ADVERTISE_CLIENT_URLS="https://192.168.133.182:2379"
	ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.133.180:2380,etcd-2=https://192.168.133.181:2380,etcd-3=https://192.168.133.182:2380"
	ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
	ETCD_INITIAL_CLUSTER_STATE="new"
```
### 3.2.3 启动etcd并检查集群的健康性
```
在三个节点依次启动etcd
# systemctl start etcd
# systemctl enable etcd

检查是否启动成功
此处ip地址需要根据自己的etcd集群ip具体填写
# /opt/etcd/bin/etcdctl --ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.133.180:2379,https://192.168.133.181:2379,https://192.168.133.182:2379" cluster-health
	member 4370315ad93240ec is healthy: got healthy result from https://192.168.133.180:2379
	member 847a67ae71e9b632 is healthy: got healthy result from https://192.168.133.181:2379
	member a7aa314db8c5a2dc is healthy: got healthy result from https://192.168.133.182:2379
	cluster is healthy
```







