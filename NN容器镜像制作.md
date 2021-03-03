使用docker拉取linux镜像
```# docker pull centos```

将该镜像实例化，并进入镜像内容部
```# docker run -i -t centos:latest  bash```

此时进入到容器系统内
更新软件
```# yum update```

安装 git 和 wget
```# yum -y install git wget```

安装python相关
```# yum -y install python3 python3-pip```

安装相关机器学习包
```
# pip3 install numpy
# pip3 install ipython
# pip3 install scikit-learn
# pip3 install scikit-image
# pip3 install jinja2
# pip3 install imageio
# pip3 install h5py
# pip3 install pyyaml
# pip3 install pandas
```
安装机器学习框架和画图软件
```
# pip3 install keras==2.1.0
# pip3 install matplotlib==2.1.1
# pip3 install tensorflow==1.12.0
```




