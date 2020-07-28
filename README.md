# k8s-Binary-installation
system version :centos 7   k8s version:1.16 

**此教程适合初学者使用，大佬请飘过**


# 搭建一个完整的Kubernetes集群

```
 1、生产环境K8S平台规划  
 2、服务器硬件配置推荐  
 3、官方提供三种部署方式  
 4、为Etcd和APIServer自签SSL证书  
 5、Etcd数据库集群部署  
 6、部署Master组件  
 7、部署Node组件  
 8、部署K8s集群网络  
 9、部署Web UI（Dashboard  
 10、部署集群内部DNS解析服务（CoreDNS）  
```

## 1、生产环境K8S平台规划
```
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
操作系统：centos 7  
```






