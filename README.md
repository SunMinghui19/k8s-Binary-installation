# k8s-Binary-installation
system version :centos 7   k8s version:1.16 

**此教程适合初学者使用，大佬请飘过**


# 搭建一个完整的Kubernetes集群

1、	生产环境K8S平台规划
2、	服务器硬件配置推荐
3、	官方提供三种部署方式
4、	为Etcd和APIServer自签SSL证书
5、	Etcd数据库集群部署
6、	部署Master组件
7、	部署Node组件
8、	部署K8s集群网络
9、	部署Web UI（Dashboard）
10、	部署集群内部DNS解析服务（CoreDNS）




centos7固定ip  (参考https://www.cnblogs.com/lfhappy/p/10798400.html)

第一步：修改VM和本机的网络配置
打开VM->编辑->虚拟网络编辑器->VMnet8
取消勾选使用本地DHCP（并在那个页面查看虚拟的网段和子网掩码）
选择NAT设置可以查看自己的网关
本机电脑（windows）->网络设置->更改适配器选项->右键选中VMnet8的属性->找到internet协议版本4选择属性->填写ip地址和子网掩码

第二步：配置固定ip
# vim /etc/sysconfig/network-scripts/ifcfg-ens33
	修改BOOTPROT0=static
	修改ONBOOT=yes
	添加DNS1=114.114.114.114
	添加IPADDR=自己根据虚拟网段自己选择
	添加NETMASK=虚拟网络编辑器获取
	添加GATEWAY=虚拟网络编辑器获取


部署单master集群
一、集群规划
	master
		主机名：k8s-master1
		IP：192.168.133.180
	worker1
		主机名：k8s-node1
		IP：192.168.133.181
	worker2：
		主机名：k8s-node2
		IP：192.168.133.182

K8S版本：1.16
安装方式：离线-二进制
操作系统：centos 7.3


二、初始化服务器
 1、关闭防火墙
	[所有主节点都执行]
[root@localhost /]# systemctl stop firewalld
[root@localhost /]# systemctl disable firewalld

2、关闭selinux
[所有主节点都执行]
setenforce 0  #临时关闭
[root@localhost sunminghui]# setenforce 0
[root@localhost sunminghui]# vim /etc/selinux/config
将SELINUX=enforcing后面的修改为disabled

3、配置主机名
[所有主节点都执行]
hostnamectl set-hostname 主机名
使用hostname检验是否更改成功
 4、配置名称解析（更改host文件）
[所有主节点都执行]
vim /etc/hosts
添加如下几行（此处需要根据自己的k8s集群填写相应IP）
192.168.133.180 k8s-master1
192.168.133.181 k8s-node1
192.168.133.182 k8s-node2
 5、配置时间同步
	选择一个节点作为服务端，剩下的作为客户端
	master1为时间服务器的服务端
	其他的为时间服务器的客户端
1）	配置k8s-master1
#yum install chrony –y
#vim /etc/chrony.conf
修改三项
	server 127.127.1.0 iburst（此处127.127.1.0是设置为本机时间）
	allow 192.168.133.0/24（修改为你自己的集群网段）
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
 6、关闭交换分区
[所有主节点都执行]
我们在使用k8s的时候如果开着交换分区可能导致服务起不来
[root@localhost sunminghui]# swapoff –a  #暂时关闭
[root@localhost sunminghui]# vim /etc/fstab #永久关闭
删除一行：swap
[root@localhost sunminghui]# free –m #检验是否成功
                total        used        free      shared  buff/cache   available
Mem:           2818         107        2565           8         144        2539
Swap:             0           0           0


