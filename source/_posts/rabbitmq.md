# 1、安装

## 1.1、windows安装

### 1.1.1、安装erlang

下载地址：https://www.erlang-solutions.com/resources/download.html

### 1.1.1、windows安装rabbitmq

下载地址页面

```
https://www.rabbitmq.com/install-windows.html#installer
```

安装包地址

```
https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.8.2/rabbitmq-server-3.8.2.exe
```



## 1.2、centos安装

### 1.2.1、安装erlang

下载地址：https://www.erlang-solutions.com/resources/download.html

![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200307140319.png)

### 1.2.2、按照rabbitmq

https://www.rabbitmq.com/install-rpm.html#install-erlang-from-epel-repository

![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200307140218.png)



## 1.3、安装web管理插件

```
rabbitmq-plugins enable rabbitmq_management
```

## 1.4、启动RabbitMQ

### 启动RabbitMQ，浏览器访问：[http://192.168.64.128:15672/](http://192.168.64.128:15672/#/)，出现以下界面说明安装完成！

