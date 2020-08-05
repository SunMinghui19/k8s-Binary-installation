# CorDNS
！！！！！！！！！！！！！注意，这种情况只限于没有启用service且deployment创建了一个副本的情境下
假设有这样几个pod  
![image](https://github.com/SunMinghui19/k8s-Binary-installation/blob/master/images/%E7%90%86%E6%83%B3%E7%8A%B6%E6%80%81%E4%B8%89%E5%B1%82%E6%9C%8D%E5%8A%A1.JPG)   
三个pod之间相互通信可以通过ip进行通信，nginx在和php通信的时候要明确php的ip地址，php和mysql通信的时候php要明确知道mysql的ip地址  
！！！！关键问题：pod是临时的  
![image](https://github.com/SunMinghui19/k8s-Binary-installation/blob/master/images/pod%E6%AD%BB%E6%8E%89%E4%BA%86.JPG)   
如果pod挂掉了，那么就要创建一个新的mysql。这个时候创建的mysql的ip地址可能跟刚刚的ip地址不一样。如果mysql一直变那么php就要一直改，这样的话我们程序的耦合度就太高了。  

我们应该如何实现后端发生改变前段不用修改，这个时候需要一个名称解析，有了DNS两个pod通信不再通过ip通信。而是通过名称通信，dns中记录每一个pod的 服务名和地址的对应关系  
![image](https://github.com/SunMinghui19/k8s-Binary-installation/blob/master/images/cordns.JPG)   

其实就是php通过直接访问mysql的名字，这个时候cordns就会将mysql对应的地址发送给php，这样就可以顺利建立连接。同时，如果mysql的地址变了，也会将更改后的信息告诉coreDNS，将改变后的信息记录起来。  

# Service
由于pod的地址会发生改变，通过Service可以为pod提供一个统一的访问入口  
![image](https://github.com/SunMinghui19/k8s-Binary-installation/blob/master/images/service1.JPG)  
买水果的人不知道在哪，你不可能去果园里直接买。这个时候就需要一个中间层--菜市场，卖水果的人在菜市场租一个摊位，那么买水果的直接到菜市场就能买到水果了。菜市场就相当于一个访问入口，我不知道苹果、香蕉、西红柿在哪。我只要去菜市场就能找到了  
  
同类，Service也是类似的概念。 
![image](https://github.com/SunMinghui19/k8s-Binary-installation/blob/master/images/service2.JPG)  




# 发布服务（将pod发布出来）
用service来实现发布服务


问题：1、ip不是固定的
     2、即使ip固定，外网也无法访问的到
     
