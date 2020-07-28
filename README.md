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

### 2.1、关闭防火墙
```
	[所有主节点都执行]
# systemctl stop firewalld
# systemctl disable firewalld
```
### 2.2、关闭selinux
[所有主节点都执行]
setenforce 0  #临时关闭
# setenforce 0

# vim /etc/selinux/config
将config中的SELINUX=enforcing后面的修改为disabled

### 2.3、配置主机名
[所有主节点都执行]
hostnamectl set-hostname 主机名
使用hostname检验是否更改成功
 ### 2.4、配置名称解析（更改host文件）
[所有主节点都执行]
vim /etc/hosts
添加如下几行
192.168.133.180 k8s-master1
192.168.133.181 k8s-node1
192.168.133.182 k8s-node2
### 2.5、配置时间同步
	选择一个节点作为服务端，剩下的作为客户端
	master1为时间服务器的服务端
	其他的为时间服务器的客户端
1）	配置k8s-master1
#yum install chrony –y
#vim /etc/chrony.conf
修改三项
	server 127.127.1.0 iburst
	allow 192.168.133.0/24
	local stratum 10
	# systemctl start chronyd
# systemctl enable chronyd
# ss -unl | grep 123
UNCONN     0      0            *:123                      *:*
2）	配置k8s-node1和k8s-node2
#yum install chrony –y
#vim /etc/chrony.conf
		server 192.168.133.180 iburst
# systemctl start chronyd
# systemctl enable chronyd
# chronyc sources
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^* k8s-master1                  10   6    17    60    -44us[  -56us] +/-  192us
### 2.6、关闭交换分区
[所有主节点都执行]
我们在使用k8s的时候如果开着交换分区可能导致服务起不来
[root@localhost sunminghui]# swapoff –a  #暂时关闭
[root@localhost sunminghui]# vim /etc/fstab #永久关闭
删除一行：swap
[root@localhost sunminghui]# free –m #检验是否成功
                total        used        free      shared  buff/cache   available
Mem:           2818         107        2565           8         144        2539
Swap:             0           0           0








