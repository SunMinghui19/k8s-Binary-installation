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

# 四、为api server签发证书
```
# cd /root/TLS/k8s/
# vim server-csr.json
（下面填写了我自己主机的ip地址，具体填写内容要根据你集群的ip来。将所有的ip都填写进去）
	{
	    "CN": "kubernetes",
	    "hosts": [
	      "10.0.0.1",
	      "127.0.0.1",
	      "kubernetes",
	      "kubernetes.default",
	      "kubernetes.default.svc",
	      "kubernetes.default.svc.cluster",
	      "kubernetes.default.svc.cluster.local",
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
		    "ST": "BeiJing",
		    "O": "k8s",
		    "OU": "System"
# ./generate_k8s_cert.sh
```

# 五、部署master服务器
```
# tar xvf k8s-master.tar.gz 
# mv kube-apiserver.service kube-controller-manager.service kube-scheduler.service /usr/lib/systemd/system/
# mv kubernetes /opt/
# cp /root/TLS/k8s/{ca*pem,server.pem,server-key.pem} /opt/kubernetes/ssl/ -rvf

修改apiserver的配置文件
# vim /opt/kubernetes/cfg/kube-apiserver.conf
etcd-servers处填写etcd的IP，其余部分为当前主机IP即master的IP
	KUBE_APISERVER_OPTS="--logtostderr=false \
	--v=2 \
	--log-dir=/opt/kubernetes/logs \
	--etcd-servers=https://192.168.133.180:2379,https://192.168.133.181:2379,https://192.168.133.182:2379 \
	--bind-address=192.168.133.180 \
	--secure-port=6443 \
	--advertise-address=192.168.133.180 \
	--allow-privileged=true \
	--service-cluster-ip-range=10.0.0.0/24 \
	--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \
	--authorization-mode=RBAC,Node \
	--enable-bootstrap-token-auth=true \
	--token-auth-file=/opt/kubernetes/cfg/token.csv \
	--service-node-port-range=30000-32767 \
	--kubelet-client-certificate=/opt/kubernetes/ssl/server.pem \
	--kubelet-client-key=/opt/kubernetes/ssl/server-key.pem \
	--tls-cert-file=/opt/kubernetes/ssl/server.pem  \
	--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \
	--client-ca-file=/opt/kubernetes/ssl/ca.pem \
	--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \
	--etcd-cafile=/opt/etcd/ssl/ca.pem \
	--etcd-certfile=/opt/etcd/ssl/server.pem \
	--etcd-keyfile=/opt/etcd/ssl/server-key.pem \
	--audit-log-maxage=30 \
	--audit-log-maxbackup=3 \
	--audit-log-maxsize=100 \
	--audit-log-path=/opt/kubernetes/logs/k8s-audit.log"

启动master节点
# systemctl start kube-apiserver
# systemctl enable kube-apiserver
# systemctl start kube-scheduler
# systemctl enable kube-scheduler
# systemctl start kube-controller-manager
# systemctl enable kube-controller-manager

配置可以直接使用kubelet
# cp /opt/kubernetes/bin/kubectl   /bin/


检查启动结果
# ps aux |grep kube
可以看到进程中有kube-apiserver、kube-schedule、kube-controller-manager的字样
		

配置TLS基于bootstrap自动颁发证书（自动的为访问kubelet的服务颁发证书，此处为授权操作）
# kubectl create clusterrolebinding kubelet-bootstrap \
--clusterrole=system:node-bootstrapper \
--user=kubelet-bootstrap
```

# 六、部署worker服务器
```
 node节点需要安装如下软件：
	docker：启动容器
	kubelet：接收apiserver的指令，控制docker容器
	kube-proxy：为worker上的容器配置网络功能
```
##  6.1安装配置docker
```
# tar xvf k8s-node.tar.gz 
# mv docker.service /usr/lib/systemd/system
# mkdir /etc/docker
# cp daemon.json /etc/docker
# tar xf docker-18.09.6.tgz 
# mv docker/* /bin/
# systemctl start docker
# systemctl enable docker
 # docker info


执行docker info出现如下警告
WARNING: bridge-nf-call-iptables is disabled
WARNING: bridge-nf-call-ip6tables is disabled

解决办法：
vi /etc/sysctl.conf

添加以下内容
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

最后再执行
sysctl -p

此时docker info就看不到此报错了
```

## 6.2安装kubelet和kube-proxy
```
!!!!!注意:给个node节点都要做如下操作

1）生成程序目录和管理脚本
# tar xvf k8s-node.tar.gz 
# mv kubelet.service  kube-proxy.service  /usr/lib/systemd/system/
# mv kubernetes /opt/


2)修改配置文件（4个）
# vim /opt/kubernetes/cfg/kube-proxy.kubeconfig
	修改一行:server: https://192.168.133.180:6443
	这里指定的是master的IP地址

# vim /opt/kubernetes/cfg/kube-proxy-config.yml
	修改一行：hostnameOverride: k8s-node1
	这里是指定当前主机的主机名

# vim /opt/kubernetes/cfg/kubelet.conf
	修改一行：--hostname-override=k8s-node1 \
	这里是指定当前主机的主机名

# vim /opt/kubernetes/cfg/bootstrap.kubeconfig
	修改一行：server: https://192.168.133.180:6443
	这里指定的是master的IP地址

3)从master节点复制证书到worker节点
[root@k8s-master1 ~]# cd /root/TLS/k8s/
[root@k8s-master1 k8s]# scp ca.pem kube-proxy.pem  kube-proxy-key.pem root@k8s-node1:/opt/kubernetes/ssl/

4）启动kubelet和kube-proxy服务
[root@k8s-node1 ~]# systemctl start kube-proxy
[root@k8s-node1 ~]# systemctl enable kube-proxy
[root@k8s-node1 ~]# systemctl start kubelet
[root@k8s-node1 ~]# systemctl enable kubelet

[root@k8s-node1 ~]# tail -f /opt/kubernetes/logs/kubelet.INFO 
如果看到最后一行信息是如下内容，就表示启动服务正常：
I0709 15:42:29.150959   10657 bootstrap.go:150] No valid private key and/or certificate found, reusing existing private key or creating a new one

5）在master节点为worker节点颁发证书
[root@k8s-master1 k8s]# kubectl get csr
NAME           AGE     REQUESTOR           CONDITION
node-csr-zPG-VxeXMszsgMz5pkdEnWeCnnBdYL7r666ECwbg5pc   5m25s   kubelet-bootstrap   Pending
 [root@k8s-master1 k8s]# kubectl certificate approve node-csr-zPG-VxeXMszsgMz5pkdEnWeCnnBdYL7r666ECwbg5pc	
！！！注意：名称（NAME）必须用自己的名称。


6）给worker节点颁发证书之后，就可以在master上看到worker节点了
[root@k8s-master1 k8s]# kubectl get nodes
NAME        STATUS     ROLES    AGE     VERSION
k8s-node1   NotReady   <none>   3m53s   v1.16.0
```

# 7、安装网络插件flannel
```
！！！！【两个node都要做】
1）确认启用CNI插件
[root@k8s-node1 ~]# grep "cni" /opt/kubernetes/cfg/kubelet.conf 
--network-plugin=cni \

2）安装CNI
[root@k8s-node1 ~]# mkdir -pv /opt/cni/bin  /etc/cni/net.d
[root@k8s-node1 ~]# tar xf k8s-node.tar.gz 
[root@k8s-node1 ~]# tar zxvf cni-plugins-linux-amd64-v0.8.2.tgz  -C  /opt/cni/bin


3）在master上执行yaml脚本，实现在worker节点安装启动网络插件功能
[root@k8s-master1 YAML]# kubectl apply -f kube-flannel.yaml 
	podsecuritypolicy.policy/psp.flannel.unprivileged created
	clusterrole.rbac.authorization.k8s.io/flannel created
	clusterrolebinding.rbac.authorization.k8s.io/flannel created
	serviceaccount/flannel created
	configmap/kube-flannel-cfg created
	daemonset.apps/kube-flannel-ds-amd64 created
注意：这个操作受限于网络，可能需要5-10分钟才能执行成功，如果网速过慢，会提示超时。如果实在下载不下来，可以更换docker的镜像源

tip：更换docker镜像源
1、修改配置文件
vim /etc/docker/daemon.json
删除所有内容替换为如下内容：
{
"registry-mirrors": [
"https://kfwkfulq.mirror.aliyuncs.com",
"https://2lqq34jg.mirror.aliyuncs.com",
"https://pee6w651.mirror.aliyuncs.com",
"https://registry.docker-cn.com",
"http://hub-mirror.c.163.com",
],
"dns": ["8.8.8.8","8.8.4.4"]
}
2、重启服务
# systemctl daemon-reload
# systemctl restart docker


```
# 8、安装网络插件flannel



