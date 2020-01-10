---
title: power designer日常使用
date: 2020-01-09 09:24:11
tags:
- power designer
- mysql
categories: mysql
---



# 1、Check Model错误解决

## 1.1、Datatype attributes错误、PowerDesigner生成mysql脚本后多了一个national关键字

![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200109092614.png)

解决方案：

选择Database->Edit Current DBMS菜单，如下图：

![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200109092846.png)

选择National,勾选Computed，如下图所示：

![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200109092943.png)