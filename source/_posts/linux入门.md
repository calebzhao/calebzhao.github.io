---
title: linux入门
date: 2020-02-25 16:52:40
tags:
- linux
categories: linux
---

# 1、基础命令

```
man [command]    查看命令帮助文档
ip address 		检查网卡地址配置
ping       		测试网络连通性
nmtui      		图形界面修改网卡地址
exit       		注销
clear      		清屏操作
shutdown   		1分钟后关机
shutdown -h 5  	指定5分钟后关机
shutdown -h now 立即关机 
shutdown -r 5   5分钟后重启
shutdown -r now 立即重启
shutdown -c     取消关机
halt		   直接关机
pwoeroff	   直接关机
reboot		   直接重启
echo "hello world"		   输出回显内容
id <username>	查看用户是否存在
whoami			查看当前登陆用户名


```

# 2、文件目录相关命令

```
ls <xx>         		列出目录或文件(list)
ls -d <xx>      		只列出目录
ls -l <xx>      		以详细方式查看
ls -a <fileName>  		查看所有文件（包括查看隐藏文件）
ll <xx>         		以详细方式查看
ll -h <xx>				以可读的方式显示文件大小
mkdir <xxx>    			创建目录，不能创建多级目录(make directory)
mkdir -p <directory> 	创建多级目录
pwd					   显示当前所在工作目录(print work directory)
cd ~				   进入用户主目录
cd -				   快速在2个目录之间来回切换
touch <fileName>  		创建文件
cat  <fileName>  		查看文件内容
cp  <source> <dest>  	不递归复制文件或目录，
cp -r <source] <dest> 	递归复制目录
mv <source> <dest> 		移动文件目录或重命名文件
rm <fileName> 			删除单个文件，有提示
rm -r <direction 		删除目录, 有提示
rm -rf <direction> 		删除目录，无提示
```



# 3、mount挂载点

- 查看光驱设备

    ```
    [calebzhao@oldboyedu a]$ ls -l /dev/cdrom 
    lrwxrwxrwx. 1 root root 3 Feb 25 14:31 /dev/cdrom -> sr0
    ```

- 查看挂载点

  ```
  [calebzhao@oldboyedu a]$ ls -d /mnt/
  /mnt/
  ```

- 挂载设备（centos7 必须root）
	
	```  
	[root@oldboyedu mnt]# mount /dev/cdrom /mnt/
	mount: /dev/sr0 is write-protected, mounting read-only
	```
	
- 查看挂载后内容
	```
	[root@oldboyedu /]# ll /mnt/
	total 694
	-rw-rw-r--. 1 root root     14 Sep 10 03:06 CentOS_BuildTag
	drwxr-xr-x. 3 root root   2048 Sep  6 19:48 EFI
	-rw-rw-r--. 1 root root    227 Aug 30  2017 EULA
	-rw-rw-r--. 1 root root  18009 Dec 10  2015 GPL
	drwxr-xr-x. 3 root root   2048 Sep 10 02:07 images
	drwxr-xr-x. 2 root root   2048 Sep 10 02:07 isolinux
	drwxr-xr-x. 2 root root   2048 Sep  6 19:48 LiveOS
	drwxrwxr-x. 2 root root 671744 Sep 12 02:41 Packages
	drwxrwxr-x. 2 root root   4096 Sep 12 02:48 repodata
	-rw-rw-r--. 1 root root   1690 Dec 10  2015 RPM-GPG-KEY-CentOS-7
	-rw-rw-r--. 1 root root   1690 Dec 10  2015 RPM-GPG-KEY-CentOS-Testing-7
	-r--r--r--. 1 root root   2883 Sep 12 02:50 TRANS.TBL
	
	```
	
- 卸载挂载点

    ```
    unmount /mnt
    ```

- `df [-h]` 查看挂载情况/查看磁盘使用情况

    ```bash
    [root@oldboyedu /]# df
    Filesystem     1K-blocks    Used Available Use% Mounted on
    devtmpfs          487140       0    487140   0% /dev
    tmpfs             497872       0    497872   0% /dev/shm
    tmpfs             497872    7792    490080   2% /run
    tmpfs             497872       0    497872   0% /sys/fs/cgroup
    /dev/sda3       29140072 1221564  27918508   5% /
    /dev/sda1         201380  110708     90672  55% /boot
    tmpfs              99576       0     99576   0% /run/user/1000
    /dev/sr0         4554702 4554702         0 100% /mnt
    s
    ```

- 磁盘挂载文件
  
   `/ect/fstab`实现存储设备开机自动挂载配置       
  
  ```
  [root@oldboyedu /]# cat /etc/fstab 
  
  #
  # /etc/fstab
  # Created by anaconda on Tue Feb 25 14:25:14 2020
  #
  # Accessible filesystems, by reference, are maintained under '/dev/disk'
  # See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
  #
  UUID=941614ac-5626-4be7-a7b5-78d3db57eeb6 /                       xfs     defaults        0 0
  UUID=0e345aa0-e4a3-480e-b96d-f710170b46c0 /boot                   xfs     defaults        0 0
  UUID=67b779f8-9336-4d3b-8fae-643a5d65ef33 swap                    swap    defaults        0 0
  #定义存储文件信息 						 挂载点                 文件系统
  /dev/cdrom							   /nmt
  ```
  
  


# 4、系统特殊符号

```
~ 	用户主目录
.. 	上一级目录
- 	快速切换目录，返回上一次所在目录
>  	标准输出重定向， 会覆盖文件原有内容(echo "hello world" > a.txt)
>> 	标准输出重定向， 会在文件原有内容末尾追加新内容(echo "hello world" >> a.txt)
&& 	代表前一个命令执行成功后再执行后面的命令
#  	代表将配置文件该行注释
$  	sell脚本中表示引用变量 
   	vim编辑器中表示移动光标到行尾
   	在命令提示符中表示普通用户
```



# 5、vim使用

- 命令：

    ```
    vi 编辑文件, 注意centos7默认没有vim
    i  进入编辑模式，从光标当前所在位置进入编辑状态
    I   将光标移动到一行的行首，再进入编辑模式
    esc 退出编辑模式
    :wq 保存并退出
    :w   保存
    :q  退出
    :wq! 强制保存并退出
    :q! 强制退出
    ```

- 快捷方式：

   ```
   将一行内容删除   					deletedelete=dd
   将多行内容删除   					3dd
   操作错误如何还原  				    小写字母u  undo
   将光标快速切换到文件首部   			  小写字母gg
   将光标快速切换到文件尾部   			  大写字母G
   移动到指定的行						13gg （表示移动到第13行）
   移动到当前行行首					0或者^
   移动到当前行行尾					$
   ```

- vim的模式
   1. 命令模式
   2. 编辑模式
   3. 低行模式

- 命令模式 ----> 插入模式

    ````
    o   在光标所在行的下面，新起一行进行编辑
    O   在光标所在行的上面，新起一行进行编辑
    
    C	删除光标所在位置到行尾内容进入编辑状态
    
    i  	进入编辑模式，从光标当前所在位置进入编辑状态
    I   将光标移动到一行的行首，再进入编辑模式
    A	快速切换光标所在位置到行尾进入编辑状态
    cc	清空当前行的所有内容并进入编辑状态
    ````

## 5.1、进入vim
![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200309230448.png)  

## 5.2、vim配置
![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200309230556.png) 

## 5.3、移动光标
![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200309230650.png)

## 5.4、屏幕滚动
![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200309230715.png)

## 5.5、插入文本类
![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200309230739.png)

## 5.6、删除命令
![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200309230805.png)

## 5.7、复制粘贴
![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200309231047.png)

## 5.8、撤销
![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200309231127.png)

## 5.9、搜索及替换
![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200309231303.png)

## 5.10、书签
![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200309231450.png)

## 5.11、visual模式
![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200309231515.png)

## 5.12、行方式命令
![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200309231542.png)
若不指定n1，n2，则表示将整个文件内容作为command的输入 

## 5.13、宏
![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200309231616.png)

## 5.14、窗口操作
![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200309231639.png)

## 5.15、文件及其他
![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200309231703.png)



# 6、快捷方式

```
tab				   路径/命令补全
ctrl + c       		中断命令执行
ctrl + l       		清屏操作
ctrl + a       		快速将光标移到行首
ctrl + e       		快速将光标移到行尾
ctrl + 左由方向键  	按照英文单词左右移动
ctrl + w			将空格分隔的一个字符串整体进行删除（剪切）
ctrl + u			将光标所在位置到行首内容进行删除(剪切)
ctrl + u			将光标所在位置到行尾内容进行删除(剪切)
ctrl + y			粘贴剪贴板的内容
ctrl + s			xshell进入到锁定状态，无法敲任何内容
ctrl + q			解除锁定状态， quit退出锁定状态
```


# 7、系统目录结构

![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200225205027.png)



文件系统的是用来组织和排列文件存取的，所以它是可见的，在Linux中，我们可以通过ls等工具来查看其结构，在Linux系统中，我们见到的都是树形结构；比如操作系统安装在一个文件系统中，它表现为由/ 起始的树形结构。linux文件系统的最顶端是/，我们称/为Linux的root，也就是 Linux操作系统的文件系统。Linux的文件系统的入口就是/，所有的目录、文件、设备都在/之下，/就是Linux文件系统的组织者，也是最上级的领导者。

　　由于linux是开放源代码，各大公司和团体根据linux的核心代码做各自的操作，编程。这样就造成在根下的目录的不同。这样就造成个人不能使用他人的linux系统的PC。因为你根本不知道一些基本的配置，文件在哪里。。。这就造成了混乱。这就是FHS（Filesystem Hierarchy Standard ）机构诞生的原因。该机构是linux爱好者自发的组成的一个团体，主要是是对linux做一些基本的要求，不至于是操作者换一台主机就成了linux的‘文盲’。

 　根据FHS(http://www.pathname.com/fhs/)的官方文件指出， 他们的主要目的是希望让使用者可以了解到已安装软件通常放置于那个目录下， 所以他们希望独立的软件开发商、操作系统制作者、以及想要维护系统的用户，都能够遵循FHS的标准。 也就是说，FHS的重点在于规范每个特定的目录下应该要放置什么样子的数据而已。 这样做好处非常多，因为Linux操作系统就能够在既有的面貌下(目录架构不变)发展出开发者想要的独特风格。

　　事实上，FHS是根据过去的经验一直再持续的改版的，FHS依据文件系统使用的频繁与否与是否允许使用者随意更动， 而将目录定义成为四种交互作用的形态，用表格来说有点像底下这样：

|                          | 可分享的(shareable)        | 不可分享的(unshareable) |
| ------------------------ | -------------------------- | ----------------------- |
| 不变的(static)           | /usr (软件放置处)          | /etc (配置文件)         |
| /opt (第三方协力软件)    | /boot (开机与核心档)       |                         |
| 可变动的(variable)       | /var/mail (使用者邮件信箱) | /var/run (程序相关)     |
| /var/spool/news (新闻组) | /var/lock (程序相关)       |                         |

 

**四中类型:**

**1.可分享的：**

　　可以分享给其他系统挂载使用的目录，所以包括执行文件与用户的邮件等数据， 是能够分享给网络上其他主机挂载用的目录；

**2.不可分享的：**

　自己机器上面运作的装置文件或者是与程序有关的socket文件等， 由于仅与自身机器有关，所以当然就不适合分享给其他主机了。

**3.不变的：**

　有些数据是不会经常变动的，跟随着distribution而不变动。 例如函式库、文件说明文件、系统管理员所管理的主机服务配置文件等等；

**4.可变动的：**

　经常改变的数据，例如登录文件、一般用户可自行收受的新闻组等。

事实上，FHS针对目录树架构仅定义出三层目录底下应该放置什么数据而已，分别是底下这三个目录的定义：

```
/ (root, 根目录)：与开机系统有关；

/usr (unix software resource)：与软件安装/执行有关；

/var (variable)：与系统运作过程有关。
```

## 7.1、根目录 (/) 的意义与内容：

　　根目录是整个系统最重要的一个目录，因为不但所有的目录都是由根目录衍生出来的， 同时根目录也与开机/还原/系统修复等动作有关。 由于系统开机时需要特定的开机软件、核心文件、开机所需程序、 函式库等等文件数据，若系统出现错误时，根目录也必须要包含有能够修复文件系统的程序才行。 因为根目录是这么的重要，所以在FHS的要求方面，他希望根目录不要放在非常大的分区， 因为越大的分区内你会放入越多的数据，如此一来根目录所在分区就可能会有较多发生错误的机会。

因此FHS标准建议：根目录(/)所在分区应该越小越好， 且应用程序所安装的软件最好不要与根目录放在同一个分区内，保持根目录越小越好。 如此不但效能较佳，根目录所在的文件系统也较不容易发生问题。说白了，就是根目录和Windows的C盘一个样。

**根据以上原因，FHS认为根目录(/)下应该包含如下子目录：**

http://www.pathname.com/fhs/pub/fhs-2.3.pdf

| 目录   | 应放置档案内容                                               |
| ------ | ------------------------------------------------------------ |
| /bin   | 系统有很多放置执行档的目录，但/bin比较特殊。因为/bin放置的是在单人维护模式下还能够被操作的指令。在/bin底下的指令可以被root与一般帐号所使用，主要有：cat,chmod(修改权限), chown, date, mv, mkdir, cp, bash等等常用的指令。 |
| /boot  | 主要放置开机会使用到的档案，包括Linux核心档案以及开机选单与开机所需设定档等等。Linux kernel常用的档名为：vmlinuz ，如果使用的是grub这个开机管理程式，则还会存在/boot/grub/这个目录。 |
| /dev   | 在Linux系统上，任何装置与周边设备都是以档案的型态存在于这个目录当中。 只要通过存取这个目录下的某个档案，就等于存取某个装置。比要重要的档案有/dev/null, /dev/zero, /dev/tty , /dev/lp*, / dev/hd*, /dev/sd*等等 |
| /etc   | 系统主要的设定档几乎都放置在这个目录内，例如人员的帐号密码档、各种服务的启始档等等。 一般来说，这个目录下的各档案属性是可以让一般使用者查阅的，但是只有root有权力修改。 FHS建议不要放置可执行档(binary)在这个目录中。 比较重要的档案有：/etc/inittab, /etc/init.d/, /etc/modprobe.conf, /etc/X11/, /etc/fstab, /etc/sysconfig/等等。 另外，其下重要的目录有：/etc/init.d/ ：所有服务的预设启动script都是放在这里的，例如要启动或者关闭iptables的话： /etc/init.d/iptables start、/etc/init.d/ iptables stop/etc/xinetd.d/ ：这就是所谓的super daemon管理的各项服务的设定档目录。 /etc/X11/ ：与X Window有关的各种设定档都在这里，尤其是xorg.conf或XF86Config这两个X Server的设定档。 |
| /home  | 这是系统预设的使用者家目录(home directory)。 在你新增一个一般使用者帐号时，预设的使用者家目录都会规范到这里来。比较重要的是，家目录有两种代号： ~ ：代表当前使用者的家目录，而 ~guest：则代表用户名为guest的家目录。 |
| /lib   | 系统的函式库非常的多，而/lib放置的则是在开机时会用到的函式库，以及在/bin或/sbin底下的指令会呼叫的函式库而已 。 什么是函式库呢？妳可以将他想成是外挂，某些指令必须要有这些外挂才能够顺利完成程式的执行之意。 尤其重要的是/lib/modules/这个目录，因为该目录会放置核心相关的模组(驱动程式)。 |
| /media | media是媒体的英文，顾名思义，这个/media底下放置的就是可移除的装置。 包括软碟、光碟、DVD等等装置都暂时挂载于此。 常见的档名有：/media/floppy, /media/cdrom等等。 |
| /mnt   | 如果妳想要暂时挂载某些额外的装置，一般建议妳可以放置到这个目录中。在古早时候，这个目录的用途与/media相同啦。 只是有了/media之后，这个目录就用来暂时挂载用了。 |
| /opt   | 这个是给第三方协力软体放置的目录 。 什么是第三方协力软体啊？举例来说，KDE这个桌面管理系统是一个独立的计画，不过他可以安装到Linux系统中，因此KDE的软体就建议放置到此目录下了。 另外，如果妳想要自行安装额外的软体(非原本的distribution提供的)，那么也能够将你的软体安装到这里来。 不过，以前的Linux系统中，我们还是习惯放置在/usr/local目录下。 |
| /root  | 系统管理员(root)的家目录。 之所以放在这里，是因为如果进入单人维护模式而仅挂载根目录时，该目录就能够拥有root的家目录，所以我们会希望root的家目录与根目录放置在同一个分区中。 |
| /sbin  | Linux有非常多指令是用来设定系统环境的，这些指令只有root才能够利用来设定系统，其他使用者最多只能用来查询而已。放在/sbin底下的为开机过程中所需要的，里面包括了开机、修复、还原系统所需要的指令。至于某些伺服器软体程式，一般则放置到/usr/sbin/当中。至于本机自行安装的软体所产生的系统执行档(system binary)，则放置到/usr/local/sbin/当中了。常见的指令包括：fdisk, fsck, ifconfig, init, mkfs等等。 |
| /srv   | srv可以视为service的缩写，是一些网路服务启动之后，这些服务所需要取用的资料目录。 常见的服务例如WWW, FTP等等。 举例来说，WWW伺服器需要的网页资料就可以放置在/srv/www/里面。呵呵，看来平时我们编写的代码应该放到这里了。 |
| /tmp   | 这是让一般使用者或者是正在执行的程序暂时放置档案的地方。这个目录是任何人都能够存取的，所以你需要定期的清理一下。当然，重要资料不可放置在此目录啊。 因为FHS甚至建议在开机时，应该要将/tmp下的资料都删除。 |

事实上FHS针对根目录所定义的标准就仅限于上表，不过仍旧有些目录也需要我们了解一下，具体如下：

| 目录        | 应放置文件内容                                               |
| ----------- | ------------------------------------------------------------ |
| /lost+found | 这个目录是使用标准的ext2/ext3档案系统格式才会产生的一个目录，目的在于当档案系统发生错误时，将一些遗失的片段放置到这个目录下。 这个目录通常会在分割槽的最顶层存在，例如你加装一个硬盘于/disk中，那在这个系统下就会自动产生一个这样的目录/disk/lost+found |
| /proc       | 这个目录本身是一个虚拟文件系统(virtual filesystem)喔。 他放置的资料都是在内存当中，例如系统核心、行程资讯(process)（是进程吗?）、周边装置的状态及网络状态等等。因为这个目录下的资料都是在记忆体（内存）当中，所以本身不占任何硬盘空间。比较重要的档案（目录）例如： /proc/cpuinfo, /proc/dma, /proc/interrupts, /proc/ioports, /proc/net/*等等。呵呵，是虚拟内存吗[guest]？ |
| /sys        | 这个目录其实跟/proc非常类似，也是一个虚拟的档案系统，主要也是记录与核心相关的资讯。 包括目前已载入的核心模组与核心侦测到的硬体装置资讯等等。 这个目录同样不占硬盘容量。 |


　　除了这些目录的内容之外，另外要注意的是，因为根目录与开机有关，开机过程中仅有根目录会被挂载， 其他分区则是在开机完成之后才会持续的进行挂载的行为。就是因为如此，因此根目录下与开机过程有关的目录， 就不能够与根目录放到不同的分区去。

那哪些目录不可与根目录分开呢？有底下这些：

**/etc：**配置文件

**/bin：**重要执行档

**/dev：**所需要的设备文件

**/lib：**执行档所需的函式库与核心所需的模块

**/sbin：**重要的系统执行文件

这五个目录不可与根目录分开在不同的分区。

## 7.2、/usr 的意义与内容

　　依据FHS的基本定义，/usr里面放置的数据属于可分享的与不可变动的(shareable, static)， 如果你知道如何透过网络进行分区的挂载(例如在服务器篇会谈到的NFS服务器)，那么/usr确实可以分享给局域网络内的其他主机来使用喔。

　　/usr不是user的缩写，其实usr是Unix Software Resource的缩写， 也就是Unix操作系统软件资源所放置的目录，而不是用户的数据啦。这点要注意。 FHS建议所有软件开发者，应该将他们的数据合理的分别放置到这个目录下的次目录，而不要自行建立该软件自己独立的目录。

　　因为是所有系统默认的软件(distribution发布者提供的软件)都会放置到/usr底下，因此这个目录有点类似Windows 系统的C:\Windows\ + C:\Program files\这两个目录的综合体，系统刚安装完毕时，这个目录会占用最多的硬盘容量。 一般来说，/usr的次目录建议有底下这些：

| 目录          | 应放置文件内容                                               |
| ------------- | ------------------------------------------------------------ |
| /usr/X11R6/   | 为X Window System重要数据所放置的目录，之所以取名为X11R6是因为最后的X版本为第11版，且该版的第6次释出之意。 |
| /usr/bin/     | 绝大部分的用户可使用指令都放在这里。请注意到他与/bin的不同之处。(是否与开机过程有关) |
| /usr/include/ | c/c++等程序语言的档头(header)与包含档(include)放置处，当我们以tarball方式 (*.tar.gz 的方式安装软件)安装某些数据时，会使用到里头的许多包含档。 |
| /usr/lib/     | 包含各应用软件的函式库、目标文件(object file)，以及不被一般使用者惯用的执行档或脚本(script)。 某些软件会提供一些特殊的指令来进行服务器的设定，这些指令也不会经常被系统管理员操作， 那就会被摆放到这个目录下啦。要注意的是，如果你使用的是X86_64的Linux系统， 那可能会有/usr/lib64/目录产生 |
| /usr/local/   | 统管理员在本机自行安装自己下载的软件(非distribution默认提供者)，建议安装到此目录， 这样会比较便于管理。举例来说，你的distribution提供的软件较旧，你想安装较新的软件但又不想移除旧版， 此时你可以将新版软件安装于/usr/local/目录下，可与原先的旧版软件有分别啦。 你可以自行到/usr/local去看看，该目录下也是具有bin, etc, include, lib...的次目录 |
| /usr/sbin/    | 非系统正常运作所需要的系统指令。最常见的就是某些网络服务器软件的服务指令(daemon) |
| /usr/share/   | 放置共享文件的地方，在这个目录下放置的数据几乎是不分硬件架构均可读取的数据， 因为几乎都是文本文件嘛。在此目录下常见的还有这些次目录：/usr/share/man：联机帮助文件/usr/share/doc：软件杂项的文件说明/usr/share/zoneinfo：与时区有关的时区文件 |
| /usr/src/     | 一般原始码建议放置到这里，src有source的意思。至于核心原始码则建议放置到/usr/src/linux/目录下。 |

 

## 7.3、/var 的意义与内容

　　如果/usr是安装时会占用较大硬盘容量的目录，那么/var就是在系统运作后才会渐渐占用硬盘容量的目录。 因为/var目录主要针对常态性变动的文件，包括缓存(cache)、登录档(log file)以及某些软件运作所产生的文件， 包括程序文件(lock file, run file)，或者例如MySQL数据库的文件等等。常见的次目录有：

| 目录        | 应放置文件内容                                               |
| ----------- | ------------------------------------------------------------ |
| /var/cache/ | 应用程序本身运作过程中会产生的一些暂存档                     |
| /var/lib/   | 程序本身执行的过程中，需要使用到的数据文件放置的目录。在此目录下各自的软件应该要有各自的目录。 举例来说，MySQL的数据库放置到/var/lib/mysql/而rpm的数据库则放到/var/lib/rpm去 |
| /var/lock/  | 某些装置或者是文件资源一次只能被一个应用程序所使用，如果同时有两个程序使用该装置时， 就可能产生一些错误的状况，因此就得要将该装置上锁(lock)，以确保该装置只会给单一软件所使用。 举例来说，刻录机正在刻录一块光盘，你想一下，会不会有两个人同时在使用一个刻录机烧片？ 如果两个人同时刻录，那片子写入的是谁的数据？所以当第一个人在刻录时该刻录机就会被上锁， 第二个人就得要该装置被解除锁定(就是前一个人用完了)才能够继续使用 |
| /var/log/   | 非常重要。这是登录文件放置的目录。里面比较重要的文件如/var/log/messages, /var/log/wtmp(记录登入者的信息)等。 |
| /var/mail/  | 放置个人电子邮件信箱的目录，不过这个目录也被放置到/var/spool/mail/目录中，通常这两个目录是互为链接文件。 |
| /var/run/   | 某些程序或者是服务启动后，会将他们的PID放置在这个目录下      |
| /var/spool/ | 这个目录通常放置一些队列数据，所谓的“队列”就是排队等待其他程序使用的数据。 这些数据被使用后通常都会被删除。举例来说，系统收到新信会放置到/var/spool/mail/中， 但使用者收下该信件后该封信原则上就会被删除。信件如果暂时寄不出去会被放到/var/spool/mqueue/中， 等到被送出后就被删除。如果是工作排程数据(crontab)，就会被放置到/var/spool/cron/目录中。 |

　　由于FHS仅是定义出最上层(/)及次层(/usr, /var)的目录内容应该要放置的文件或目录数据， 因此，在其他次目录层级内，就可以随开发者自行来配置了。

 

## 7. 4、目录树(directory tree)

　　在Linux底下，所有的文件与目录都是由根目录开始的。那是所有目录与文件的源头, 然后再一个一个的分支下来，因此，我们也称这种目录配置方式为：目录树(directory tree), 这个目录树的主要特性有：

- 目录树的启始点为根目录 (/, root)；
- 每一个目录不止能使用本地端的 partition 的文件系统，也可以使用网络上的 filesystem 。举例来说， 可以利用 Network File System (NFS) 服务器挂载某特定目录等。
- 每一个文件在此目录树中的文件名(包含完整路径)都是独一无二的。

如果我们将整个目录树以图的方法来显示，并且将较为重要的文件数据列出来的话，那么目录树架构就如下图所示： 

![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200225211035.png)

## 7.5、绝对路径与相对路径

　　除了需要特别注意的FHS目录配置外，在文件名部分我们也要特别注意。因为根据档名写法的不同，也可将所谓的路径(path)定义为绝对路径(absolute)与相对路径(relative)。 这两种文件名/路径的写法依据是这样的：

**绝对路径：**

　　由根目录(/)开始写起的文件名或目录名称， 例如 /home/dmtsai/.bashrc；

**相对路径：**

　　相对于目前路径的文件名写法。 例如 ./home/dmtsai 或 http://www.cnblogs.com/home/dmtsai/ 等等。反正开头不是 / 就属于相对路径的写法

而你必须要了解，相对路径是以你当前所在路径的相对位置来表示的。举例来说，你目前在 /home 这个目录下， 如果想要进入 /var/log 这个目录时，可以怎么写呢？

cd /var/log  (absolute)

cd ../var/log (relative)

因为你在 /home 底下，所以要回到上一层 (../) 之后，才能继续往 /var 来移动的，特别注意这两个特殊的目录：

.  ：代表当前的目录，也可以使用 ./ 来表示；

.. ：代表上一层目录，也可以 ../ 来代表。

这个 . 与 .. 目录概念是很重要的，你常常会看到 cd .. 或 ./command 之类的指令下达方式， 就是代表上一层与目前所在目录的工作状态。

**实例1：如何先进入/var/spool/mail/目录，再进入到/var/spool/cron/目录内？**

　　**命令：**

　　cd /var/spool/mail

　　cd ../cron

　　**说明：**

　　由于/var/spool/mail与/var/spool/cron是同样在/var/spool/目录中。如此就不需要在由根目录开始写起了。这个相对路径是非常有帮助的，尤其对于某些软件开发商来说。 一般来说，软件开发商会将数据放置到/usr/local/里面的各相对目录。 但如果用户想要安装到不同目录呢？就得要使用相对路径。

**实例2：网络文件常常提到类似./run.sh之类的数据，这个指令的意义为何？**

　　**说明：**

　　由于指令的执行需要变量的支持，若你的执行文件放置在本目录，并且本目录并非正规的执行文件目录(/bin, /usr/bin等为正规)，此时要执行指令就得要严格指定该执行档。./代表本目录的意思，所以./run.sh代表执行本目录下， 名为run.sh的文件



# 8、网络配置

## 8.1、网卡配置文件

```properties
[root@oldboyedu /]# cat /etc/sysconfig/network-scripts/ifcfg-ens33 
# 指定网络类型，以太网Ethernet、 电话10M、 军用（帧中继）、金融公司(FastEthernet)
TYPE=Ethernet

PROXY_METHOD=none
BROWSER_ONLY=no
# 网络启动协议，如何让主机得到ip地址
# a)、自己手动配置none、static(静态地址)
# b)、自动获取地址dhcp
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
# 主机网卡的逻辑名称
NAME=ens33
# 虚拟主机会给每个硬件一个标识
UUID=1817bc5a-7026-4fef-be2f-5e3384d7c013
# 主机网卡的名称，设备的的物理名称
DEVICE=ens33
# 设置网卡是否处于开启状态（激活状态）
ONBOOT=yes
# 配置静态IP地址
IPADDR=192.168.42.200
# 子网掩码，定义网络中可以有多少主机
PREFIX=24
# 网关
GATEWAY=192.168.42.2
# dns（域名解析相关）
DNS1=223.5.5.5
IPV6_PRIVACY=no
```

配置文件修改之后重启网卡服务

- 方式一(针对所有网卡执行重启)

  ```shell
  systemctl start network
  systemctl stop network
  # 一定只能使用这条命令
  systemctl restart network
  systemctl status network
  ```

- 方式二(只针对指定网卡执行重启，**推荐**)

  ```
  # 一次性执行多条命令,一定只能使用这条命令
  ifdown ens33 && ifup ens33 
  
  # 停止指定ens33网卡
  ifdown ens33
  # 启动ens33网卡
  ifup ens33 
  ```

异常问题：网卡配置文件正常，无法重启网络服务

```shell
systemctl stop NetworkManager 		网络管理服务关闭
systemctl disable NetworkManager 	禁用网络管理服务，不开机自启
```



## 8.2、dns配置文件
- 方式1：网卡设置dns
```shell
	[root@oldboyedu /]# cat /etc/sysconfig/network-scripts/ifcfg-ens33 
	...省略部分
	 # dns（域名解析相关）
	DNS1=223.5.5.5
	...省略部分
```


​	
- 方式2：全局设置dns配置
    ```
    cat /etc/resolv.conf
    ```
    
      **需要注意：** 网卡配置文件中的dns地址的优先级高于`/etc/resolv.conf`配置文件

## 8.3、主机名配置

- 临时设置hostname
    ```
    hostname xxxx
    ```

- **centos7永久设置**hostname

  ```
  [root@oldboyedu /]# vi /etc/hostname 
  oldboyedu.com
  ```

- **centos6永久设置**hostname

  ```
  vi /etc/sysconfig/network
  ```

 - centos7同时临时设置并永久设置

    ```
    hostnamectl set-hostname [xxx]
    ```

## 8.4、hosts文件配置

```
[calebzhao@caleb ~]$ cat /etc/hosts
```



## 8.5、查看网卡地址信息

```
ip address
```

## 8.6、查看端口占用

命令：`netstat -anp|grep 端口`

```
[root@caleb /home/calebzhao]#netstat -anp|grep 8080
tcp6       0      0 :::8080                 :::*                    LISTEN      1320/java 
```



# 9、查看操作系统信息

## 9.1、查看操作系统版本

- `cat /etc/redhat-release`
```
[root@oldboyedu /]# `cat /etc/redhat-release`
CentOS Linux release 7.7.1908 (Core)
```

- `uname -a`
```
[calebzhao@caleb ~]$ uname -a
Linux caleb 3.10.0-1062.12.1.el7.x86_64 #1 SMP Tue Feb 4 23:02:59 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
```

## 9.2、查看CPU信息
虚拟机配置：
![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200309211042.png)

```
总核数 = 物理CPU个数 X 每颗物理CPU的核数
总逻辑CPU数 = 物理CPU个数 X 每颗物理CPU的核数 X 超线程数

# 查看物理CPU个数
cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l

# 查看每个物理CPU中core的个数(即核数)
cat /proc/cpuinfo| grep "cpu cores"| uniq

# 查看逻辑CPU的个数
cat /proc/cpuinfo| grep "processor"| wc -l
```

-  `cat /proc/cpuinfo`查看cpu信息 

```shell
[calebzhao@caleb ~]$ cat /proc/cpuinfo 
# 系统中逻辑处理核的编号。对于单核处理器，则课认为是其CPU编号，对于多核处理器则可以是物理核、或者使用超线程技术虚拟的逻辑核
processor	: 0
# CPU制造商
vendor_id	: GenuineIntel
# CPU产品系列代号
cpu family	: 6
# CPU属于其系列中的哪一代的代号
model		: 158
# cpu型号
model name	: Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz
# CPU属于制作更新版本
stepping	: 10
microcode	: 0x96
# CPU的实际使用主频
cpu MHz		: 2208.004
# CPU二级缓存大小
cache size	: 9216 KB
# 从0开始，每个物理封装的唯一标识符，即物理处理器（物理核）编号，
# 拥有相同 physical id 的所有逻辑处理器共享同一个物理插座（socket）。
# 每个 physical id 代表一个唯一的物理封装。
physical id	: 0
# 位于相同物理封装中的逻辑处理器的数量，物理核包含的逻辑核数。
# Hyper-Threading creates logical CPUs (refered to as sibling CPUs by the kernel)，
# 也就是说SIBLING是内核认为的单个物理处理器所有的超线程个数。如果SIBLING小于等于实际物理核数的话，
# 就说明没有启动超线程，反之启用超线程。有时不确定是否超线程时，siblings:12可以表述为“每个CPU有12个逻辑物理核”。
siblings	: 3
# 每个内核的唯一标识符，即在当前物理核中它的编号，每个 core id 均代表一个唯一的处理器内核，
# 如果有n个processor（逻辑核）具有相同的"core id”和physical id ，那么超线程是打开的，且为n线程。
core id		: 0
# 该逻辑核所处CPU的物理核数
cpu cores	: 3
# 用来区分不同逻辑核的编号，系统中每个逻辑核的此编号必然不同，此编号不一定连续
apicid		: 0
initial apicid	: 0
# 是否具有浮点运算单元（Floating Point Unit）
fpu		: yes
# 是否支持浮点计算异常
fpu_exception	: yes
# 执行cpuid指令前，eax寄存器中的值，根据不同的值cpuid指令会返回不同的内容
cpuid level	: 22
# 表明当前CPU是否在内核态支持对用户空间的写保护（Write Protection）
wp		: yes
# 当前CPU支持的功能
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon nopl xtopology tsc_reliable nonstop_tsc eagerfpu pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch invpcid_single ssbd ibrs ibpb stibp fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid mpx rdseed adx smap clflushopt xsaveopt xsavec arat spec_ctrl intel_stibp flush_l1d arch_capabilities
# 在系统内核启动时粗略测算的CPU速度（Million Instructions Per Second）
bogomips	: 4416.00
# 每次刷新缓存的大小单位
clflush size	: 64
# 缓存地址对齐单位
cache_alignment	: 64
# 可访问地址空间位数
address sizes	: 43 bits physical, 48 bits virtual
# 对能源管理的支持，有以下几个可选支持功能：
power management:

processor	: 1
vendor_id	: GenuineIntel
cpu family	: 6
model		: 158
model name	: Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz
stepping	: 10
microcode	: 0x96
cpu MHz		: 2208.004
cache size	: 9216 KB
physical id	: 0
siblings	: 3
core id		: 1
cpu cores	: 3
apicid		: 1
initial apicid	: 1
fpu		: yes
fpu_exception	: yes
cpuid level	: 22
wp		: yes
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon nopl xtopology tsc_reliable nonstop_tsc eagerfpu pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch invpcid_single ssbd ibrs ibpb stibp fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid mpx rdseed adx smap clflushopt xsaveopt xsavec arat spec_ctrl intel_stibp flush_l1d arch_capabilities
bogomips	: 4416.00
clflush size	: 64
cache_alignment	: 64
address sizes	: 43 bits physical, 48 bits virtual
power management:

processor	: 2
vendor_id	: GenuineIntel
cpu family	: 6
model		: 158
model name	: Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz
stepping	: 10
microcode	: 0x96
cpu MHz		: 2208.004
cache size	: 9216 KB
physical id	: 0
siblings	: 3
core id		: 2
cpu cores	: 3
apicid		: 2
initial apicid	: 2
fpu		: yes
fpu_exception	: yes
cpuid level	: 22
wp		: yes
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon nopl xtopology tsc_reliable nonstop_tsc eagerfpu pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch invpcid_single ssbd ibrs ibpb stibp fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid mpx rdseed adx smap clflushopt xsaveopt xsavec arat spec_ctrl intel_stibp flush_l1d arch_capabilities
bogomips	: 4416.00
clflush size	: 64
cache_alignment	: 64
address sizes	: 43 bits physical, 48 bits virtual
power management:

processor	: 3
vendor_id	: GenuineIntel
cpu family	: 6
model		: 158
model name	: Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz
stepping	: 10
microcode	: 0x96
cpu MHz		: 2208.004
cache size	: 9216 KB
physical id	: 1
siblings	: 3
core id		: 0
cpu cores	: 3
apicid		: 4
initial apicid	: 4
fpu		: yes
fpu_exception	: yes
cpuid level	: 22
wp		: yes
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon nopl xtopology tsc_reliable nonstop_tsc eagerfpu pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch invpcid_single ssbd ibrs ibpb stibp fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid mpx rdseed adx smap clflushopt xsaveopt xsavec arat spec_ctrl intel_stibp flush_l1d arch_capabilities
bogomips	: 4416.00
clflush size	: 64
cache_alignment	: 64
address sizes	: 43 bits physical, 48 bits virtual
power management:

processor	: 4
vendor_id	: GenuineIntel
cpu family	: 6
model		: 158
model name	: Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz
stepping	: 10
microcode	: 0x96
cpu MHz		: 2208.004
cache size	: 9216 KB
physical id	: 1
siblings	: 3
core id		: 1
cpu cores	: 3
apicid		: 5
initial apicid	: 5
fpu		: yes
fpu_exception	: yes
cpuid level	: 22
wp		: yes
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon nopl xtopology tsc_reliable nonstop_tsc eagerfpu pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch invpcid_single ssbd ibrs ibpb stibp fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid mpx rdseed adx smap clflushopt xsaveopt xsavec arat spec_ctrl intel_stibp flush_l1d arch_capabilities
bogomips	: 4416.00
clflush size	: 64
cache_alignment	: 64
address sizes	: 43 bits physical, 48 bits virtual
power management:

processor	: 5
vendor_id	: GenuineIntel
cpu family	: 6
model		: 158
model name	: Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz
stepping	: 10
microcode	: 0x96
cpu MHz		: 2208.004
cache size	: 9216 KB
physical id	: 1
siblings	: 3
core id		: 2
cpu cores	: 3
apicid		: 6
initial apicid	: 6
fpu		: yes
fpu_exception	: yes
cpuid level	: 22
wp		: yes
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon nopl xtopology tsc_reliable nonstop_tsc eagerfpu pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch invpcid_single ssbd ibrs ibpb stibp fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid mpx rdseed adx smap clflushopt xsaveopt xsavec arat spec_ctrl intel_stibp flush_l1d arch_capabilities
bogomips	: 4416.00
clflush size	: 64
cache_alignment	: 64
address sizes	: 43 bits physical, 48 bits virtual
power management:

[calebzhao@caleb ~]$ 


```

- `lscpu`查看cpu信息

```
[calebzhao@caleb ~]$ lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                6
On-line CPU(s) list:   0-5
Thread(s) per core:    1
Core(s) per socket:    3
Socket(s):             2
NUMA node(s):          1
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 158
Model name:            Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz
Stepping:              10
CPU MHz:               2208.004
BogoMIPS:              4416.00
Hypervisor vendor:     VMware
Virtualization type:   full
L1d cache:             32K
L1i cache:             32K
L2 cache:              256K
L3 cache:              9216K
NUMA node0 CPU(s):     0-5
Flags:                 fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon nopl xtopology tsc_reliable nonstop_tsc eagerfpu pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch invpcid_single ssbd ibrs ibpb stibp fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid mpx rdseed adx smap clflushopt xsaveopt xsavec arat spec_ctrl intel_stibp flush_l1d arch_capabilities

```



## 9.3、查看内存信息

### 9.3.1、 `cat /proc/meminfo`

```
[calebzhao@caleb ~]$ cat /proc/meminfo 
MemTotal:         995760 kB
MemFree:          709928 kB
MemAvailable:     689948 kB
Buffers:            2076 kB
Cached:            86840 kB
SwapCached:            0 kB
Active:            85324 kB
Inactive:          69276 kB
Active(anon):      66100 kB
Inactive(anon):     7344 kB
Active(file):      19224 kB
Inactive(file):    61932 kB
Unevictable:           0 kB
Mlocked:               0 kB
SwapTotal:       2097148 kB
SwapFree:        2097148 kB
Dirty:                 0 kB
Writeback:             0 kB
AnonPages:         65688 kB
Mapped:            25284 kB
Shmem:              7756 kB
Slab:              63816 kB
SReclaimable:      21664 kB
SUnreclaim:        42152 kB
KernelStack:        4352 kB
PageTables:         4340 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     2595028 kB
Committed_AS:     271304 kB
VmallocTotal:   34359738367 kB
VmallocUsed:      180580 kB
VmallocChunk:   34359310332 kB
HardwareCorrupted:     0 kB
AnonHugePages:      6144 kB
CmaTotal:              0 kB
CmaFree:               0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:       89984 kB
DirectMap2M:      958464 kB
DirectMap1G:           0 kB
[calebzhao@caleb ~]$ 

```

### 9.3.2、free [-h]

```
# 显示字节数
[calebzhao@caleb ~]$ free
              total        used        free      shared  buff/cache   available
Mem:         995760      175336      709844        7756      110580      689864
Swap:       2097148           0     2097148

-h参数表示以人类可读的方式查看
[calebzhao@caleb ~]$ free -h
              total        used        free      shared  buff/cache   available
Mem:           972M        171M        693M        7.6M        108M        673M
Swap:          2.0G          0B        2.0G
[calebzhao@caleb ~]$ 
```

## 9.4、查看负载信息
- `w`命令查看负载信息/查看系统用户登陆信息

```
[calebzhao@caleb ~]$ w
 21:47:13 up 39 min,  1 user,  load average: 0.00, 0.01, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
calebzha pts/0    192.168.42.1     21:08    1.00s  0.03s  0.01s w
```



# 10、开机自启动

## 10.1、rc.local

编辑`/etc/rc.local `文件，每一行写一条开机自启动的命令

例如：

```
[calebzhao@caleb ~]$ vi /etc/rc.local 
#!/bin/bash
# THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
#
# It is highly advisable to create own systemd services or udev rules
# to run scripts during boot instead of using this file.
#
# In contrast to previous versions due to parallel execution during boot
# this script will NOT be run after all other services.
#
# Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure
# that this script will be executed during boot.

touch /var/lock/subsys/local
sytemctl start sshd
```

**注意**：

像上面这样直接保存了如果重启了还是是没效果，首先看`/etc/rc.local `文件的权限是否有x执行权限，命令如下：

```
[calebzhao@caleb ~]$ ll /etc/rc.local 
lrwxrwxrwx. 1 root root 13 Mar  5 22:10 /etc/rc.local -> rc.d/rc.local
```

如果`/etc/rc.local `文件有执行权限，但是`rc.local`自启动还是无效，从上面命令输出结果可以看到`rc.local`只是一个软链接，实际文件是`/etc/rc.d/rc.local`，这个时候看一下`/etc/rc.d/rc.local`文件的权限，查看命令如下：

```
[calebzhao@caleb ~]$ ll /etc/rc.d/rc.local 
-rw-r--r--. 1 root root 574 Mar  6 22:58 /etc/rc.d/rc.local
```

可以看到没有执行的权限，rc.local文件中已经说明了必须执行`chmod +x /etc/rc.d/rc.local`命令才会起效果

```
chmod +x /etc/rc.d/rc.local 
```

再重启就会有效果了。

总结：rc.local文件的作用

1. 文件中的内容信息，会在系统启动之后自动加载
2. 文件中的编写内容，一定是命令信息



练习：实现开机自动创建/oldgirl/oldgirl.txt文件，并且文件中有“oldgirl.com”信息内容

实现命令如下：

```
[root@caleb calebzhao]# vi /etc/rc.local 
#!/bin/bash
# THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
#
# It is highly advisable to create own systemd services or udev rules
# to run scripts during boot instead of using this file.
#
# In contrast to previous versions due to parallel execution during boot
# this script will NOT be run after all other services.
#
# Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure
# that this script will be executed during boot.

touch /var/lock/subsys/local
systemctl start sshd

mkdir /oldgirl
cd /oldgirl
touch oldgirl.txt
echo "oldgirl.com" >> oldgirl.txt
```

wq保存后，reboot重启查看/oldgirl/oldgirl.txt文件是存在



## 10.2、systemctl方式

```
cat > /usr/lib/systemd/system/redis.service <<-EOF
[Unit]
Description=Redis 6379
After=syslog.target network.target
[Service]
Type=forking
PrivateTmp=yes
Restart=always
ExecStart=/usr/local/redis/bin/redis-server /usr/local/redis/redis.conf
ExecStop=/usr/local/redis/bin/redis-cli -h 127.0.0.1 -p 6379 -a jcon shutdown
User=root
Group=root
LimitCORE=infinity
LimitNOFILE=100000
LimitNPROC=100000
[Install]
WantedBy=multi-user.target
EOF
```

```
#启用
systemctl enable redis
systemctl disable redis
systemctl start redis
systemctl restart redis
systemctl 
```



# 11、系统运行级别

## 11.1、查询运行级别
- centos6
  
    ```
    [calebzhao@caleb ~]$ runlevel
    N 3
    ```
    
    文件信息：`cat /etc/inittab`


- centos7

    ```
    systemctl get-default
    ```

    文件信息：`/usr/lib/systemd/system/runlevel*target`
    
    ![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200308222411.png)

## 11.2、切换运行级别

- 临时调整：`init <数字级别>`

- 永久调整：
  - centos6
  
    ```
    vi /etc/inittab
    ```
  
  - centos7
  
    ```shell
    # <Target.target>的值见上述图片，ll /usr/lib/systemd/system/runlevel*target命令可以看到
    systemctl set-default <Target.target>
    ```

临时调整运行级别示例：

```
[calebzhao@caleb ~]$ init 1
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ===
Authentication is required to manage system services or units.
Authenticating as: root
Password: 
==== AUTHENTICATION COMPLETE ===

Broadcast message from calebzhao@caleb on pts/0 (Fri 2020-03-06 23:22:57 CST):

The system is going down to rescue mode NOW!
```

centos7执行后，查看虚拟机会发现如下提示：

![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200306233149.png)

## 11.3、各个级别的含义

- centos6:

```
0 	系统的关机级别 init 0 进入到关机状态
1	系统的单用户模式，用于修复系统或重置密码信息，没有网络
2	系统的多用户模式，没有网络
3	系统的多用户模式 正常系统运行级别，有网络
4	预留级别
5	图形化界面级别
6	系统的重启级别
```

- centos7:

```
0	系统的关机级别 开机后就立马关机
1	系统的维护模式，用于修复系统或重置密码信息，没有网络
2	系统的多用户模式，没有网络
3	系统的多用户模式 正常系统运行级别，有网络
4	预留级别
5	图形化界面级别
6	系统的重启级别

lrwxrwxrwx. 1 root root 15 Mar  5 22:10 /usr/lib/systemd/system/runlevel0.target -> poweroff.target
lrwxrwxrwx. 1 root root 13 Mar  5 22:10 /usr/lib/systemd/system/runlevel1.target -> rescue.target
lrwxrwxrwx. 1 root root 17 Mar  5 22:10 /usr/lib/systemd/system/runlevel2.target -> multi-user.target
lrwxrwxrwx. 1 root root 17 Mar  5 22:10 /usr/lib/systemd/system/runlevel3.target -> multi-user.target
lrwxrwxrwx. 1 root root 17 Mar  5 22:10 /usr/lib/systemd/system/runlevel4.target -> multi-user.target
lrwxrwxrwx. 1 root root 16 Mar  5 22:10 /usr/lib/systemd/system/runlevel5.target -> graphical.target
lrwxrwxrwx. 1 root root 13 Mar  5 22:10 /usr/lib/systemd/system/runlevel6.target -> reboot.target
```

# 12、安装软件 && 开源软件镜像站
## 12.1、概念

和程序软件安装相关的目录是: `/usr/local`

`usr`不是user的缩写，而是`unix software resource`

**系统中如何安装软件（以吃饭为例）**

- 1、订餐点外面 （做好的饭、筷子）      yum安装软件   简单快捷(比如360软件管家、一键安装)
- 2、买半成品      （速冻饺子  再加工）  rpm安装软件    需要有软件安装包（自己下载好，再双击安装）
- 3、自己做饭     （食材           做饭）     编译安装软件   可以灵活调整



## 12.2、yum安装软件

- 查看源文件

  ```
  [calebzhao@caleb ~]$ ll /etc/yum.repos.d/
  total 48
  -rw-r--r--. 1 root root 1664 Sep  5  2019 CentOS-Base.repo
  -rw-r--r--. 1 root root 1309 Sep  5  2019 CentOS-CR.repo
  -rw-r--r--. 1 root root  649 Sep  5  2019 CentOS-Debuginfo.repo
  -rw-r--r--. 1 root root  314 Sep  5  2019 CentOS-fasttrack.repo
  -rw-r--r--. 1 root root  630 Sep  5  2019 CentOS-Media.repo
  -rw-r--r--. 1 root root 1331 Sep  5  2019 CentOS-Sources.repo
  -rw-r--r--. 1 root root 6639 Sep  5  2019 CentOS-Vault.repo
  -rw-r--r--. 1 root root  948 Mar  5 22:09 epel.repo
  -rw-r--r--. 1 root root 1050 Sep 18 07:25 epel.repo.rpmnew
  -rw-r--r--. 1 root root 1047 Mar  5 22:09 epel-testing.repo
  -rw-r--r--. 1 root root 1149 Sep 18 07:25 epel-testing.repo.rpmnewyum
  ```
- 安装软件
	```
	# -y参数自动确认，无交互性询问是否安装
	yum install <softwareName> [-y]
	
	# bash-completeion可以补全centos7的部分命令参数
	yum install -y vim tree waget net-tools nmap 
	
	```

```
1.用YUM安装软件包命令：yum install xxxx
2.用YUM删除软件包命令：yum remove xxxx

1.使用YUM查找软件包
命令：yum search ~
2.列出所有可安装的软件包
命令：yum list
3.列出所有可更新的软件包
命令：yum list updates
4.列出所有已安装的软件包
命令：yum list installed
5.列出所有已安装但不在Yum Repository 內的软件包
命令：yum list extras
6.列出所指定软件包
命令：yum list ～
7.使用YUM获取软件包信息
命令：yum info ～
8.列出所有软件包的信息
命令：yum info
9.列出所有可更新的软件包信息
命令：yum info updates
10.列出所有已安裝的软件包信息
命令：yum info installed
11.列出所有已安裝但不在Yum Repository 內的软件包信息
命令：yum info extras
12.列出软件包提供哪些文件
命令：yum provides~
三、清除YUM缓存
yum 会把下载的软件包和header存储在cache中，而不会自动删除。如果我们觉得它们占用了磁盘空间，可以使用yum clean指令进行清除，更精确的用法是yum clean headers清除header，yum clean packages清除下载的rpm包，yum clean all 清除所有。
1.清除缓存目录(/var/cache/yum)下的软件包
命令：yum clean packages
2.清除缓存目录(/var/cache/yum)下的 headers
命令：yum clean headers
3.清除缓存目录(/var/cache/yum)下旧的 headers
命令：yum clean oldheaders
4.清除缓存目录(/var/cache/yum)下的软件包及旧的headers
命令：yum clean, yum clean all (= yum clean packages; yum clean oldheaders)
四、yum命令工具使用举例
yum update 升级系统
yum install ～ 安装指定软件包
yum update ～ 升级指定软件包
yum remove ～ 卸载指定软件
yum grouplist 查看系统中已经安装的和可用的软件组，可用的可以安装
yum grooupinstall ～安装上一个命令显示的可用的软件组中的一个
yum grooupupdate ～更新指定软件组的软件包
yum grooupremove ～ 卸载指定软件组中的软件包
yum deplist ～ 查询指定软件包的依赖关系
yum list yum* 列出所有以yum开头的软件包
yum localinstall ～ 从硬盘安装rpm包并使用yum解决依赖
```



## 12.3、rpm安装软件

``` 
rpm -Uvh *.rpm --nodeps --force
```



## 12.4 、卸载软件

```
# 搜索已安装的软件
rpm -qa | grep firefox

# 卸载软件
rpm -e  xxx
```



## 12.4、源码安装软件



## 12.5、yum源

### 12.5.1、阿里云镜像

阿里云官网：https://developer.aliyun.com/mirror/

**使用阿里云软件源：**

1. 备份

    ```
    mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
    ```

2. 下载新的 CentOS-Base.repo 到 /etc/yum.repos.d/ 
    ```
    curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
    ```
3. 运行 yum makecache 生成缓存
4. 其他
   非阿里云ECS用户会出现 Couldn't resolve host 'mirrors.cloud.aliyuncs.com' 信息，不影响使用。用户也可自行修改相关配置: eg:
   
    ```
   sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo
    ```

### 12.5.2、清华大学镜像站

镜像站官网：https://mirrors.tuna.tsinghua.edu.cn/help/epel/ 


# 13、环境变量

## 13.1、临时变量

```
[calebzhao@caleb ~]$ x=3
[calebzhao@caleb ~]$ echo $x
3
```

## 13.2、系统环境变量

- export环境境变量

    ```
    [root@caleb calebzhao]# vi /etc/profile
    ...省略
    y=4
    
    # export暴露环境变量到全局
    export y
    ```

wq保存退出，执行`source /etc/profile`使环境变量生效（若没有这一步则虽然改了配置文件，但是不会生效）

- 查看PATH

  ```
  [root@caleb calebzhao]# echo $PATH
  /usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/home/calebzhao/.local/bin:/home/calebzhao/bin
  ```

  可以看到多个路径之间以英文`:`分隔

## 13.3、用户环境变量

设置的环境变量只对当前用户生效，其他用户无效
```
[root@caleb calebzhao]# vi ~/.bashrc
```

wq保存退出，执行`source /etc/profile`使环境变量生效（若没有这一步则虽然改了配置文件，但是不会生效）

# 14、which查找命令位置

```
which <command>
```

# 15、alias命令别名

## 15. 1、查看系统默认别名设置

```
[root@caleb calebzhao]# alias
alias cp='cp -i'
alias egrep='egrep --color=auto'
alias fgrep='fgrep --color=auto'
alias grep='grep --color=auto'
alias l.='ls -d .* --color=auto'
alias ll='ls -l --color=auto'
alias ls='ls --color=auto'
alias mv='mv -i'
alias rm='rm -i'
alias which='alias | /usr/bin/which --tty-only --read-alias --show-dot --show-tilde'
```

## 15.2、设置别名

语法：`alias 别名名称='命令'`

示例：

```
[root@caleb calebzhao]# alias catnet='cat /etc/sysconfig/network-scripts/ifcfg-ens33' 
[root@caleb calebzhao]# catnet
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=1817bc5a-7026-4fef-be2f-5e3384d7c013
DEVICE=ens33
ONBOOT=yes
IPADDR=192.168.42.200
PREFIX=24
GATEWAY=192.168.42.2
DNS1=223.5.5.5
IPV6_PRIVACY=no
[root@caleb calebzhao]# 
```

注意：部分命令由于/root/.bashrc中也有别名，所以会出现通过alias设置后，断开连接后再重新连接刚刚设置的别名就失效了。

![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200309133422.png)

从图片可以看到`/root/.bashrc`中也有alias命令，所以对于`rm cp mv`命令需要在这里修改才能真正生效

## 15.2、使别名功能失效

- unalias取消别名

    ```
    unalias <aliasName>
    ```

- 利用撬棍`\`执行命令

  比如将rm这种危险的操作设置了别名，但是有时我又是真正的想删除文件

  ```
  alias rm='echo 这是一个危险的命令，如果确定需要执行，请使用\rm xxx'
  \rm <fileName>
  ```


- 绝对路径方式执行命令

  ```
  /usr/bin/rm -rf /testdir
  ```

  

## 15.3、内置与外置命令

### 15.3.1、系统中将命令分为两大类

1. 外置命令    需要进行安装
2. 内置命令     所有系统都内置的命令

### 15.3.2、查看内置或外置命令的方法

```
[root@caleb calebzhao]# type cd
cd is a shell builtin
[root@caleb calebzhao]# type ls
ls is aliased to `ls --color=auto'
[root@caleb calebzhao]# type mkdir
mkdir is /usr/bin/mkdir

```

# 16、登陆提示

## 16.1、登陆前提示文件

```
# 文件中的内容会在用户登陆系统之前进行显示
/etc/issuse.net
```



## 16.2、登陆后提示文件

```
# 文件中的内容，会在用户登陆系统之后进行显示
/etc/motd
```

# 17、日志文件/var/log

## 17.1、查看重要日志文件

```
[root@caleb calebzhao]# ll /var/log/
total 1492
drwxr-xr-x. 2 root   root      232 Feb 25 14:29 anaconda
drwx------. 2 root   root       23 Feb 25 14:31 audit
-rw-------. 1 root   root        0 Mar  9 03:24 boot.log
-rw-------. 1 root   root     8333 Feb 26 03:48 boot.log-20200226
-rw-------. 1 root   root     7676 Mar  6 03:26 boot.log-20200306
-rw-------. 1 root   root    41445 Mar  9 03:24 boot.log-20200309
-rw-------. 1 root   utmp      384 Mar  8 23:06 btmp
-rw-------. 1 root   utmp        0 Feb 25 14:26 btmp-20200306
drwxr-xr-x. 2 chrony chrony      6 Aug  8  2019 chrony
-rw-------. 1 root   root     9327 Mar  9 20:01 cron
-rw-------. 1 root   root    34639 Mar  6 03:26 cron-20200306
-rw-r--r--. 1 root   root   123287 Mar  8 22:22 dmesg
-rw-r--r--. 1 root   root   130659 Mar  8 11:52 dmesg.old
-rw-r-----. 1 root   root        0 Feb 25 14:31 firewalld
-rw-------. 1 root   root     1301 Mar  5 22:10 grubby
-rw-r--r--. 1 root   root      193 Feb 25 14:25 grubby_prune_debug
-rw-r--r--. 1 root   root   292292 Mar  9 20:09 lastlog
-rw-------. 1 root   root     1867 Mar  8 22:22 maillog
-rw-------. 1 root   root      388 Mar  5 21:46 maillog-20200306
-rw-------. 1 root   root   659874 Mar  9 20:23 messages
-rw-------. 1 root   root   328279 Mar  6 03:26 messages-20200306
drwxr-xr-x. 2 root   root        6 Feb 25 14:29 rhsm
-rw-------. 1 root   root    13089 Mar  9 20:23 secure
-rw-------. 1 root   root     3857 Mar  5 22:10 secure-20200306
-rw-------. 1 root   root        0 Mar  6 03:26 spooler
-rw-------. 1 root   root        0 Feb 25 14:26 spooler-20200306
-rw-------. 1 root   root    64000 Feb 25 14:26 tallylog
drwxr-xr-x. 2 root   root       23 Sep 14 02:16 tuned
-rw-------. 1 root   root      715 Mar  5 21:46 vmware-network.1.log
-rw-------. 1 root   root     5161 Mar  5 18:04 vmware-network.2.log
-rw-------. 1 root   root      719 Feb 25 14:31 vmware-network.3.log
-rw-------. 1 root   root      715 Mar  6 20:16 vmware-network.log
-rw-------. 1 root   root    15434 Mar  8 22:22 vmware-vgauthsvc.log.0
-rw-r--r--. 1 root   root    18597 Mar  9 18:53 vmware-vmsvc.log
-rw-rw-r--. 1 root   utmp    27648 Mar  9 20:23 wtmp
-rw-------. 1 root   root     6959 Mar  5 22:10 yum.log
```

有2个重要的系统日志文件

- messages   记录系统或服务程序运行的状态信息

- secure       用户登陆信息，作用，可以监控文件信息，分析是否有过多的登陆失败信息来预警是否遭到暴力破解入侵系统

  ```
  [root@caleb calebzhao]# cat /var/log/secure
  Mar  6 20:16:57 caleb polkitd[564]: Loading rules from directory /etc/polkit-1/rules.d
  Mar  6 20:16:57 caleb polkitd[564]: Loading rules from directory /usr/share/polkit-1/rules.d
  Mar  6 20:16:57 caleb polkitd[564]: Finished loading, compiling and executing 2 rules
  Mar  6 20:16:57 caleb polkitd[564]: Acquired the name org.freedesktop.PolicyKit1 on the system bus
  Mar  6 20:17:04 caleb sshd[959]: Server listening on 0.0.0.0 port 22.
  Mar  6 20:17:04 caleb sshd[959]: Server listening on :: port 22.
  Mar  6 21:30:42 caleb login: pam_unix(login:session): session opened for user root by LOGIN(uid=0)
  Mar  6 21:30:42 caleb login: ROOT LOGIN ON tty1
  Mar  6 21:31:04 caleb sshd[1266]: Accepted password for calebzhao from 192.168.42.1 port 60195 ssh2
  Mar  6 21:31:04 caleb sshd[1266]: pam_unix(sshd:session): session opened for user calebzhao by (uid=0)
  Mar  6 22:47:55 caleb unix_chkpwd[1324]: password check failed for user (calebzhao)
  Mar  6 22:47:55 caleb sudo: pam_unix(sudo:auth): authentication failure; logname=calebzhao uid=1000 euid=0 tty=/dev/pts/0 ruser=calebzhao rhost=  user=calebzhao
  Mar  6 22:48:02 caleb unix_chkpwd[1326]: password check failed for user (calebzhao)
  Mar  6 22:48:08 caleb sudo: calebzhao : user NOT in sudoers ; TTY=pts/0 ; PWD=/home/calebzhao ; USER=root ; COMMAND=/bin/vi /etc/rc.local
  Mar  6 22:48:16 caleb su: pam_unix(su:session): session opened for user root by calebzhao(uid=1000)
  Mar  6 22:49:48 caleb polkitd[564]: Registered Authentication Agent for unix-process:1354:918477 (system bus name :1.27 [/usr/bin/pkttyagent --notify-fd 5 --fallback], object path /org/freedesktop/PolicyKit1/AuthenticationAgent, locale en_US.UTF-8)
  Mar  6 22:49:48 caleb polkitd[564]: Unregistered Authentication Agent for unix-process:1354:918477 (system bus name :1.27, object path /org/freedesktop/PolicyKit1/AuthenticationAgent, locale en_US.UTF-8) (disconnected from bus)
  ...省略中间部分
  Mar  8 23:06:24 caleb su: pam_succeed_if(su:auth): requirement "uid >= 1000" not met by user "root"
  Mar  8 23:06:30 caleb su: pam_unix(su:session): session opened for user root by calebzhao(uid=1000)
  Mar  9 13:50:28 caleb sshd[1679]: Accepted password for calebzhao from 192.168.42.1 port 52993 ssh2
  Mar  9 13:50:28 caleb sshd[1679]: pam_unix(sshd:session): session opened for user calebzhao by (uid=0)
  Mar  9 13:50:50 caleb sshd[1683]: error: Received disconnect from 192.168.42.1 port 52993:0:
  Mar  9 13:50:50 caleb sshd[1683]: Disconnected from 192.168.42.1 port 52993
  Mar  9 13:50:50 caleb sshd[1679]: pam_unix(sshd:session): session closed for user calebzhao
  Mar  9 13:50:59 caleb login: pam_unix(login:session): session closed for user calebzhao
  Mar  9 19:58:52 caleb sshd[1820]: Accepted password for calebzhao from 192.168.42.1 port 56835 ssh2
  Mar  9 19:58:52 caleb sshd[1820]: pam_unix(sshd:session): session opened for user calebzhao by (uid=0)
  Mar  9 20:09:01 caleb su: pam_unix(su:session): session opened for user root by calebzhao(uid=1000)
  Mar  9 20:23:35 caleb sshd[1206]: pam_unix(sshd:session): session closed for user calebzhao
  Mar  9 20:23:35 caleb su: pam_unix(su:session): session closed for user root
  
  ```
  下面分析以上日志中重要的登陆信息的含义：

  ```
  Mar  9 13:50:28 caleb sshd[1679]: Accepted password for calebzhao from 192.168.42.1 port 52993 ssh2
  Mar  9 13:50:28 caleb sshd[1679]: pam_unix(sshd:session): session opened for user calebzhao by (uid=0)
  01              02    03                        04
  01：用户是什么时间登陆
  02：登陆的主机名称
  03：用户使用什么访问进行远程登陆的
  04：登陆情况说明
      a  正确登陆情况说明
      b  错误登陆情况说明
  ```
## 17.2、如何查看日志信息

  ```
  # 查看前面5行的信息
  head -5 /var/log/secure
  
  # 查看倒数几行的信息，下面代码表示查看倒数6行的信息
  tail -6 /var/log/secure
  
  # 查看日志文件方法, -f表示监视文件内容的变化，-n 表示查看倒数几行的内容
  tail -f [-n 8] /var/log/secure
  ```

  

# 18、优化命令提示符

- 查看当前命令提示符格式

```
[calebzhao@caleb ~]$ echo $PS1
[\u@\h \W]\$
```

- 格式语法

![image-20200311011708068](C:/Users/calebzhao/AppData/Roaming/Typora/typora-user-images/image-20200311011708068.png) 

- 修改命令提示符方法

  修改`/etc/profile`文件，加入`export PS1='[\u@\H \w]\$'`

  ![](https://cdn.jsdelivr.net/gh/calebzhao/cdn/img/20200311012309.png) 

# 19、vsftpd

FTP，即：文件传输协议（File Transfer Protocol），基于客户端/服务器模式，默认使用20、21端口号，其中端口20（数据端口）用于进行数据传输，端口21（命令端口）用于接受客户端发出的相关FTP命令与参数。FTP服务器普遍部署于局域网中，具有容易搭建、方便管理的特点。而且有些FTP客户端工具还可以支持文件的多点下载以及断点续传技术，因此FTP服务得到了广大用户的青睐。
**FTP协议有以下两种工作模式：**

- 主动模式（PORT）：FTP服务器主动向客户端发起连接请求。
- 被动模式（PASV）：FTP服务器等待客户端发起连接请求（FTP的默认工作模式）。

vsftpd是一款运行在Linux操作系统上的FTP服务程序，具有很高的安全性和传输速度。
**vsftpd有以下三种认证模式：**

- 匿名开放模式：是一种最不安全的认证模式，任何人都可以无需密码验证而直接登陆。
- 本地用户模式：是通过Linux系统本地的账户密码信息进行认证的模式，相较于匿名开放模式更安全，而且配置起来简单。
- 虚拟用户模式：是这三种模式中最安全的一种认证模式，它需要为FTP服务单独建立用户数据库文件，虚拟出用来进行口令验证的账户信息，而这些账户信息在服务器系统中实际上是不存在的，仅供FTP服务程序进行认证使用。

**表1：vsftpd服务常用的参数以及作用**
![Centos 7使用vsftpd搭建FTP服务器](https://s4.51cto.com/images/blog/201809/01/a494619d4854f119174ae0103e681bef.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

### 19.1、全局操作

**1、安装vsftpd服务**
`yum -y install vsftpd`
**2、去掉配置文件里的注释行**

```
mv /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.bak
grep -v "#" /etc/vsftpd/vsftpd.conf.bak > /etc/vsftpd/vsftpd.conf
```

![Centos 7使用vsftpd搭建FTP服务器](https://s4.51cto.com/images/blog/201809/01/f83cf8e6cab6539f0bd06c08a4df2a3b.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)
**3、配置firewalld防火墙开放2231和45000-49000端口**

```
firewall-cmd --permanent --add-port=2231/tcp
firewall-cmd --permanent --add-port=45000-49000/tcp
firewall-cmd --reload
```

**4、配置selinux允许FTP服务**
*注：没有selinux相关命令的话，需要安装policycoreutils-python包*

```
yum -y install policycoreutils-python.x86_64
setsebool -P ftpd_full_access=on
```

### 19.2、 匿名开放模式

**1、修改配置文件，带注释的是需要修改和新增的配置**
`vim /etc/vsftpd/vsftpd.conf`

```
anonymous_enable=YES   #启用匿名访问模式
anon_umask=022   #匿名用户上传文件的umask值
anon_upload_enable=YES   #允许匿名用户上传文件
anon_mkdir_write_enable=YES   #允许匿名用户创建目录
anon_other_write_enable=YES   #允许匿名用户重命名、删除等操作
anon_root=/data/anon   #匿名用户的FTP根目录
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
listen_port=2231   #vsftpd服务监听的端口号
listen=NO
listen_ipv6=YES
pam_service_name=vsftpd
userlist_enable=YES
tcp_wrappers=YES
pasv_min_port=45000   #PASV模式最小端口号
pasv_max_port=49000   #PASV模式最大端口号
```

**2、创建并授权匿名用户FTP根目录**

```
mkdir -p /data/anon/pub
chown -R ftp /data/anon/pub/
```

**3、启动vsftpd服务，并加入开机启动**

```
systemctl start vsftpd
systemctl enable vsftpd
```

**4、测试**
![Centos 7使用vsftpd搭建FTP服务器](https://s4.51cto.com/images/blog/201809/01/af125becb259e0f91f6d2e2812857d3f.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

### 19.4、本地用户模式

**1、修改配置文件，删除之前的匿名模式配置内容，带注释的是需要修改和新增的配置**
`vim /etc/vsftpd/vsftpd.conf`

```
anonymous_enable=NO   #关闭匿名访问模式
local_enable=YES
write_enable=YES
local_umask=022
local_root=/data/user   #指定本地用户的FTP根目录
chroot_local_user=YES   #将用户权限禁锢在FTP目录
allow_writeable_chroot=YES   #允许对FTP根目录执行写入操作
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
listen_port=2231
listen=NO
listen_ipv6=YES
pam_service_name=vsftpd
userlist_enable=YES
tcp_wrappers=YES
pasv_min_port=45000
pasv_max_port=49000
```

**2、创建本地用户，并指定家目录**

```
useradd -d /data/user -s /sbin/nologin user
echo "123456" | passwd --stdin user
```

**3、重启vsftpd服务**
`systemctl restart vsftpd`
**4、测试**
![Centos 7使用vsftpd搭建FTP服务器](https://s4.51cto.com/images/blog/201809/01/6e269f5ecd7e0f39218d53974cd97e7b.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

### 19.3、四、虚拟用户模式

**1、创建用于FTP认证的用户数据库文件**
`vim /etc/vsftpd/vuser.txt`
*注：第一行用户名，第二行密码，依此类推*

```
xuad
123456
limin
123456
```

明文信息不安全，需要使用db_load命令用哈希（hash）算法将明文信息转换成数据文件，然后将明文信息文件删除。

```
db_load -T -t hash -f /etc/vsftpd/vuser.txt /etc/vsftpd/vuser.db
chmod 600 /etc/vsftpd/vuser.db
rm -f /etc/vsftpd/vuser.txt
```

![Centos 7使用vsftpd搭建FTP服务器](https://s4.51cto.com/images/blog/201809/01/ec4e7bed35ec9bb95bebcb0bf3344d08.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)
**2、创建虚拟用户映射的系统本地用户和FTP根目录**

```
useradd -d /data/ftproot -s /sbin/nologin virtual
chmod -Rf 755 /data/ftproot/
chmod o+w /data/ftproot
```

![Centos 7使用vsftpd搭建FTP服务器](https://s4.51cto.com/images/blog/201809/01/a66160f5de1e0425c026d58d7119c2ad.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)
**3、建立用于支持虚拟用户的PAM文件**
PAM（可插拔认证模块）是一种认证机制，通过一些动态链接库和统一的API把系统提供的服务与认证方式分开，使得系统管理员可以根据需求灵活调整服务程序的不同认证方式。PAM采用了分层设计（应用程序层、应用接口层、鉴别模块层）的思想，其结构如下图所示。
![Centos 7使用vsftpd搭建FTP服务器](https://s4.51cto.com/images/blog/201809/01/1a853fe6ff3e719e3b172c089adb3dad.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)
新建一个用于虚拟用户认证的PAM文件vsftpd.vu，其中PAM文件内的“db=”参数为使用db_load命令生成的账户密码数据文件的路径，但不用写数据文件的后缀。
`vim /etc/pam.d/vsftpd.vu`

```
auth     required     pam_userdb.so  db=/etc/vsftpd/vuser
account  required     pam_userdb.so  db=/etc/vsftpd/vuser
```

**4、为两个虚拟用户设置不同的权限，xuad拥有所有权限，而limin只有读取权限。**

```
mkdir /etc/vsftpd/vusers_dir
touch /etc/vsftpd/vusers_dir/limin
vim /etc/vsftpd/vusers_dir/xuad
anon_upload_enable=YES
anon_mkdir_write_enable=YES
anon_other_write_enable=YES
```

**5、修改配置文件，删除之前的匿名模式配置内容，带注释的是需要修改和新增的配置**
`vim /etc/vsftpd/vsftpd.conf`

```
anonymous_enable=NO
anon_umask=022
local_enable=YES
guest_enable=YES   #开启虚拟用户模式
guest_username=virtual   #指定虚拟用户对应的系统用户
allow_writeable_chroot=YES   #允许对FTP根目录执行写入操作
write_enable=YES
local_umask=022
local_root=/data/ftproot
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
listen_port=2231
listen=NO
listen_ipv6=YES
pam_service_name=vsftpd.vu   #指定PAM文件
userlist_enable=YES
tcp_wrappers=YES
user_config_dir=/etc/vsftpd/vusers_dir   #指定虚拟用户配置文件目录
pasv_min_port=45000
pasv_max_port=49000
```

**6、重启vsftpd服务**
`systemctl restart vsftpd`
**7、测试**
![Centos 7使用vsftpd搭建FTP服务器](https://s4.51cto.com/images/blog/201809/01/0c989e5eafc67b157954230e33c19735.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

![Centos 7使用vsftpd搭建FTP服务器](https://s4.51cto.com/images/blog/201809/01/4d4f4ecbfc456a6b0134cfcd2f2e2a10.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=) 

### 19.5、阿里云虚拟用户模式vsftpd.conf配置

```properties
# Example config file /etc/vsftpd/vsftpd.conf
#
# The default compiled in settings are fairly paranoid. This sample file
# loosens things up a bit, to make the ftp daemon more usable.
# Please see vsftpd.conf.5 for all compiled in defaults.
#
# READ THIS: This example file is NOT an exhaustive list of vsftpd options.
# Please read the vsftpd.conf.5 manual page to get a full idea of vsftpd's
# capabilities.
#
# Allow anonymous FTP? (Beware - allowed by default if you comment this out).
# 禁用匿名用户登录
anonymous_enable=NO
anon_upload_enable=YES
anon_mkdir_write_enable=YES
# 指定上传权限掩码
anon_umask=022

## 开启虚拟用户模式
guest_enable=YES

## 开启虚拟用户模式
guest_username=virtual_ftp

##允许对ftp根目录执行写入操作
allow_writeable_chroot=YES
#（自建配置）将所有本地用户限制在家目录中
chroot_local_user=YES

##ftp根目录
local_root=/data/ftp

##启用本地用户登陆
local_enable=YES

##允许执行写操作
write_enable=YES


#
# Uncomment this to allow the anonymous FTP user to upload files. This only
# has an effect if the above global write enable is activated. Also, you will
# obviously need to create a directory writable by the FTP user.
# When SELinux is enforcing check for SE bool allow_ftpd_anon_write, allow_ftpd_full_access
#anon_upload_enable=YES
#
# Uncomment this if you want the anonymous FTP user to be able to create
# new directories.
#anon_mkdir_write_enable=YES
#
# Activate directory messages - messages given to remote users when they
# go into a certain directory.
dirmessage_enable=YES
#
# Activate logging of uploads/downloads.
xferlog_enable=YES
#
# Make sure PORT transfer connections originate from port 20 (ftp-data).
connect_from_port_20=YES
#
# If you want, you can arrange for uploaded anonymous files to be owned by
# a different user. Note! Using "root" for uploaded files is not
# recommended!
#chown_uploads=YES
#chown_username=whoever
#
# You may override where the log file goes if you like. The default is shown
# below.
#xferlog_file=/var/log/xferlog
#
# If you want, you can have your log file in standard ftpd xferlog format.
# Note that the default log file location is /var/log/xferlog in this case.
xferlog_std_format=YES
#
# You may change the default value for timing out an idle session.
#idle_session_timeout=600
#
# You may change the default value for timing out a data connection.
#data_connection_timeout=120
#
# It is recommended that you define on your system a unique user which the
# ftp server can use as a totally isolated and unprivileged user.
#nopriv_user=ftpsecure
#
# Enable this and the server will recognise asynchronous ABOR requests. Not
# recommended for security (the code is non-trivial). Not enabling it,
# however, may confuse older FTP clients.
#async_abor_enable=YES
#
# By default the server will pretend to allow ASCII mode but in fact ignore
# the request. Turn on the below options to have the server actually do ASCII
# mangling on files when in ASCII mode. The vsftpd.conf(5) man page explains
# the behaviour when these options are disabled.
# Beware that on some FTP servers, ASCII support allows a denial of service
# attack (DoS) via the command "SIZE /big/file" in ASCII mode. vsftpd
# predicted this attack and has always been safe, reporting the size of the
# raw file.
# ASCII mangling is a horrible feature of the protocol.
#ascii_upload_enable=YES
#ascii_download_enable=YES
#
# You may fully customise the login banner string:
#ftpd_banner=Welcome to blah FTP service.
#
# You may specify a file of disallowed anonymous e-mail addresses. Apparently
# useful for combatting certain DoS attacks.
#deny_email_enable=YES
# (default follows)
#banned_email_file=/etc/vsftpd/banned_emails
#
# You may specify an explicit list of local users to chroot() to their home
# directory. If chroot_local_user is YES, then this list becomes a list of
# users to NOT chroot().
# (Warning! chroot'ing can be very dangerous. If using chroot, make sure that
# the user does not have write access to the top level directory within the
# chroot)
#chroot_local_user=YES
#chroot_list_enable=YES
# (default follows)
#chroot_list_file=/etc/vsftpd/chroot_list
#
# You may activate the "-R" option to the builtin ls. This is disabled by
# default to avoid remote users being able to cause excessive I/O on large
# sites. However, some broken FTP clients such as "ncftp" and "mirror" assume
# the presence of the "-R" option, so there is a strong case for enabling it.
#ls_recurse_enable=YES
#
# When "listen" directive is enabled, vsftpd runs in standalone mode and
# listens on IPv4 sockets. This directive cannot be used in conjunction
# with the listen_ipv6 directive.
##是否以独立方式运行监听服务
listen=YES
#listen_ipv6=YES
##指定虚拟用户模式的pam文件
pam_service_name=vsftpd.vu
userlist_enable=YES
tcp_wrappers=YES
## 指定虚拟用户配置文件目录
user_config_dir=/etc/vsftpd/vusers_dir
# 启动被动模式
pasv_enable=YES
# 指定ecs服务器的公网访问地址
pasv_address=47.114.60.139
## 指定ftp被动模式的最小端口号
pasv_min_port=45000
##指定ftp被动模式的最大端口号
pasv_max_port=49000
```

# 20、后台启动应用

## 20.1、nohup后台启动输出日志

```
nohup java -jar xxx.jar &
```

## 20.2、nohup后台启动，不输出日志

```
nohup java -Dfile.encoding=utf-8 -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m -Xms1024m -Xmx1024m -Xmn512m -Xss256k -XX:SurvivorRatio=8 -XX:+UseConcMarkSweepGC -jar xxx.jar > /dev/null 2>&1 &
```

>&只是将该命令放到后台运行，若想终端关闭，程序也想运行，需要使用nohup 命令

# 21、dcoker

## 21.1、安装docker

参见：https://developer.aliyun.com/mirror/docker-ce

```
# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# Step 2: 添加软件源信息
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# Step 3: 更新并安装Docker-CE
sudo yum makecache fast
sudo yum -y install docker-ce
# Step 4: 开启Docker服务
sudo service docker start

# 注意：
# 官方软件源默认启用了最新的软件，您可以通过编辑软件源的方式获取各个版本的软件包。例如官方并没有将测试版本的软件源置为可用，您可以通过以下方式开启。同理可以开启各种测试版本等。
# vim /etc/yum.repos.d/docker-ee.repo
#   将[docker-ce-test]下方的enabled=0修改为enabled=1
#
# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# yum list docker-ce.x86_64 --showduplicates | sort -r
#   Loading mirror speeds from cached hostfile
#   Loaded plugins: branch, fastestmirror, langpacks
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            @docker-ce-stable
#   docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable
#   Available Packages
# Step2: 安装指定版本的Docker-CE: (VERSION例如上面的17.03.0.ce.1-1.el7.centos)
# sudo yum -y install docker-ce-[VERSION]
```

**安装校验**

```
root@iZbp12adskpuoxodbkqzjfZ:$ docker version
Client:
 Version:      17.03.0-ce
 API version:  1.26
 Go version:   go1.7.5
 Git commit:   3a232c8
 Built:        Tue Feb 28 07:52:04 2017
 OS/Arch:      linux/amd64

Server:
 Version:      17.03.0-ce
 API version:  1.26 (minimum version 1.12)
 Go version:   go1.7.5
 Git commit:   3a232c8
 Built:        Tue Feb 28 07:52:04 2017
 OS/Arch:      linux/amd64
 Experimental: false
```

## 2.2、镜像操作命令

### 2.2.1、镜像列表

**命令：`docker image ls`**

```
[root@caleb /var/lib/docker]#docker image ls
REPOSITORY                TAG                 IMAGE ID            CREATED             SIZE
ubuntu                    latest              74435f89ab78        10 days ago         73.9MB
rancher/server            stable              98d8bb571885        7 weeks ago         1.08GB
rancher/scheduler         v0.8.6              fbedeaddc3e9        17 months ago       248MB
rancher/agent             v1.2.11             1cc7591af4f5        23 months ago       243MB
rancher/net               v0.13.17            f170c38e3763        23 months ago       311MB
rancher/dns               v0.17.4             678bde0de4d2        23 months ago       249MB
rancher/healthcheck       v0.3.8              ce78cf69cc0b        23 months ago       391MB
rancher/metadata          v0.10.4             02104eb6e270        23 months ago       251MB
rancher/network-manager   v0.7.22             13381626c510        23 months ago       256MB
rancher/net               holder              665d9f6e8cc1        3 years ago         267MB

```

标识镜像唯一性的方式：

1. REPOSITORY:TAG

   `rancher/server: stable`

2. IMAGE ID (sha256:64位的号码，默认只截取12位)

   `98d8bb571885`

3. 查看完整的不截取的命令

   ```
   [root@caleb /var/lib/docker]#docker image ls --no-trunc
   REPOSITORY                TAG                 IMAGE ID                                                                  CREATED             SIZE
   ubuntu                    latest              sha256:74435f89ab7825e19cf8c92c7b5c5ebd73ae2d0a2be16f49b3fb81c9062ab303   10 days ago         73.9MB
   rancher/server            stable              sha256:98d8bb571885cf663f2057cf03505eeccfc7f374647283c1793acf06c81f5b73   7 weeks ago         1.08GB
   rancher/scheduler         v0.8.6              sha256:fbedeaddc3e938001c098ece0848778aebd25d44673aa64574e945aa4072ef1c   17 months ago       248MB
   rancher/agent             v1.2.11             sha256:1cc7591af4f5461cf56a9b9ff39c791b0ecf1c8fa39595148c863a2687edd091   23 months ago       243MB
   rancher/net               v0.13.17            sha256:f170c38e3763dde9eb4e756cf5691b75cb56e6252391e86bc75632abf38dda18   23 months ago       311MB
   rancher/dns               v0.17.4             sha256:678bde0de4d2f70752d6f073adfad33bd83e3f5e086b54445d4dde3480c43b7e   23 months ago       249MB
   rancher/healthcheck       v0.3.8              sha256:ce78cf69cc0b6a80b51ea39c1b2953bb29d2916d62e08156063ba578da430433   23 months ago       391MB
   rancher/metadata          v0.10.4             sha256:02104eb6e2706609a633352f446d3771fcc2301147cad4390c7759a4d36d7f6b   23 months ago       251MB
   rancher/network-manager   v0.7.22             sha256:13381626c510deef04cb9dfb7e6aa953ef313a02f7f63686d2d14e460e1e8c41   23 months ago       256MB
   rancher/net               holder              sha256:665d9f6e8cc1ec3fa1181398d0661715cf71134db65666072998c54dbeeb7072   3 years ago         267MB
   
   ```

### 2.2.2、镜像详细信息查看

**命令：`docker image inspect <IMAGE ID>`**

```
[root@caleb /var/lib/docker]#docker image inspect ubuntu:latest
[
    {
        "Id": "sha256:74435f89ab7825e19cf8c92c7b5c5ebd73ae2d0a2be16f49b3fb81c9062ab303",
        "RepoTags": [
            "ubuntu:latest"
        ],
        "RepoDigests": [
            "ubuntu@sha256:35c4a2c15539c6c1e4e5fa4e554dac323ad0107d8eb5c582d6ff386b383b7dce"
        ],
        "Parent": "",
        "Comment": "",
        "Created": "2020-06-17T01:20:56.323568224Z",
        "Container": "9477bb1d88721b63e0c6a359eb48a081741a0992480bf34bbcebf4e5734fcf67",
        "ContainerConfig": {
            "Hostname": "9477bb1d8872",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            ],
            "Cmd": [
                "/bin/sh",
                "-c",
                "#(nop) ",
                "CMD [\"/bin/bash\"]"
            ],
            "ArgsEscaped": true,
            "Image": "sha256:5412cfe47ff528df823761788f99a11568e2d25f48a4245a3d67cdc3e564c839",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": {}
        },
        "DockerVersion": "18.09.7",
        "Author": "",
        "Config": {
            "Hostname": "",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            ],
            "Cmd": [
                "/bin/bash"
            ],
            "ArgsEscaped": true,
            "Image": "sha256:5412cfe47ff528df823761788f99a11568e2d25f48a4245a3d67cdc3e564c839",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": null
        },
        "Architecture": "amd64",
        "Os": "linux",
        "Size": 73856440,
        "VirtualSize": 73856440,
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/7d06ba07416021b68c8659584940fb067caabfd68e15e908187d0f238b19a1e5/diff:/var/lib/docker/overlay2/35ef01336839c7256c0a4c9a00c539b3c24ac00115d3fdd0ad6af94f95e224f4/diff:/var/lib/docker/overlay2/c6d1698946ba12934795b49bfa2433902009c358055821ca7d01459bbbdd4813/diff",
                "MergedDir": "/var/lib/docker/overlay2/e8470a0ec67796a2946cd47189e4cfe92e0e2ca62da07bb77edcc090927d5c77/merged",
                "UpperDir": "/var/lib/docker/overlay2/e8470a0ec67796a2946cd47189e4cfe92e0e2ca62da07bb77edcc090927d5c77/diff",
                "WorkDir": "/var/lib/docker/overlay2/e8470a0ec67796a2946cd47189e4cfe92e0e2ca62da07bb77edcc090927d5c77/work"
            },
            "Name": "overlay2"
        },
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:e1c75a5e0bfa094c407e411eb6cc8a159ee8b060cbd0398f1693978b4af9af10",
                "sha256:9e97312b63ff63ad98bb1f3f688fdff0721ce5111e7475b02ab652f10a4ff97d",
                "sha256:ec1817c93e7c08d27bfee063f0f1349185a558b87b2d806768af0a8fbbf5bc11",
                "sha256:05f3b67ed530c5b55f6140dfcdfb9746cdae7b76600de13275197d009086bb3d"
            ]
        },
        "Metadata": {
            "LastTagTime": "0001-01-01T00:00:00Z"
        }
    }
]

```

### 2.2.3、镜像的导入导出

-  导出

  命令：`docker image save <镜像ID> > <目标文件>`

  ```
  [root@caleb /var/lib/docker]#docker image save 74435f89ab78 > /tmp/ubuntu.tar
  [root@caleb /var/lib/docker]#ll -h /tmp/
  total 73M
  drwxr-xr-x. 4 root root 102 Jun 26 16:12 0_tidb
  drwxr-xr-x. 2 root root 144 Jun 26 16:21 0_TIKV_LOCK_FILES
  drwx------. 2 root root   6 Jun 26 16:13 dashboard-logs261183115
  drwx------. 2 root root   6 Jun 26 16:21 dashboard-logs496482015
  drwxr-xr-x. 2 root root  18 Jun 22 13:21 hsperfdata_root
  drwx------. 3 root root  17 Jun 22 11:20 systemd-private-38947d2b57d44462aae2eba49fabd273-chronyd.service-lDOIqY
  -rw-r--r--. 1 root root 73M Jun 27 16:08 ubuntu.tar
  drwx------. 2 root root   6 Jun 22 11:20 vmware-root_607-3988556120
  ```

- 导入

  命令：`docker image load -i <导出的镜像文件>`

  ```
  [root@caleb /var/lib/docker]#docker image load -i /tmp/ubuntu.tar 
  e1c75a5e0bfa: Loading layer [==================================================>]  75.22MB/75.22MB
  9e97312b63ff: Loading layer [==================================================>]  1.011MB/1.011MB
  ec1817c93e7c: Loading layer [==================================================>]  15.36kB/15.36kB
  05f3b67ed530: Loading layer [==================================================>]  3.072kB/3.072kB
  Loaded image ID: sha256:74435f89ab7825e19cf8c92c7b5c5ebd73ae2d0a2be16f49b3fb81c9062ab303
  ```

  验证导入成功：

  ```
  [root@caleb /var/lib/docker]#docker image ls
  REPOSITORY                TAG                 IMAGE ID            CREATED             SIZE
  <none>                    <none>              74435f89ab78        10 days ago         73.9MB
  rancher/server            stable              98d8bb571885        7 weeks ago         1.08GB
  hello-world               latest              bf756fb1ae65        5 months ago        13.3kB
  rancher/scheduler         v0.8.6              fbedeaddc3e9        17 months ago       248MB
  rancher/agent             v1.2.11             1cc7591af4f5        23 months ago       243MB
  rancher/net               v0.13.17            f170c38e3763        23 months ago       311MB
  rancher/dns               v0.17.4             678bde0de4d2        23 months ago       249MB
  rancher/healthcheck       v0.3.8              ce78cf69cc0b        23 months ago       391MB
  rancher/metadata          v0.10.4             02104eb6e270        23 months ago       251MB
  rancher/network-manager   v0.7.22             13381626c510        23 months ago       256MB
  rancher/net               holder              665d9f6e8cc1        3 years ago         267MB
  ```

  会发现刚刚导入镜像没有REPOSITORY及TAG

### 2.2.4、添加TAG

命令：`docker image tag 镜像ID REPOSITORY:TAG`

**注意：如果该镜像已有TAG则执行这条命令是添加新的TAG**

  	 [root@caleb /var/lib/docker]#docker image tag 74435f89ab78 ubuntu:latest
  	  [root@caleb /var/lib/docker]#docker image ls
  	  REPOSITORY                TAG                 IMAGE ID            CREATED             SIZE
  	  ubuntu                    latest              74435f89ab78        10 days ago         73.9MB
  	  rancher/server            stable              98d8bb571885        7 weeks ago         1.08GB
  	  hello-world               latest              bf756fb1ae65        5 months ago        13.3kB
  	  rancher/scheduler         v0.8.6              fbedeaddc3e9        17 months ago       248MB
  	  rancher/agent             v1.2.11             1cc7591af4f5        23 months ago       243MB
  	  rancher/net               v0.13.17            f170c38e3763        23 months ago       311MB
  	  rancher/dns               v0.17.4             678bde0de4d2        23 months ago       249MB
  	  rancher/healthcheck       v0.3.8              ce78cf69cc0b        23 months ago       391MB
  	  rancher/metadata          v0.10.4             02104eb6e270        23 months ago       251MB
  	  rancher/network-manager   v0.7.22             13381626c510        23 months ago       256MB
  	  rancher/net               holder              665d9f6e8cc1        3 years ago         267MB
  	  ```
  	
  	  **测试已有TAG情况下执行该命令**
  	
  	  ```
  	  [root@caleb /var/lib/docker]#docker image ls
  	  REPOSITORY                TAG                 IMAGE ID            CREATED             SIZE
  	  ubuntu                    latest              74435f89ab78        10 days ago         73.9MB
  	  ...省略
  	
  	  [root@caleb /var/lib/docker]#docker image tag 74435f89ab78 calebzhao/ubuntu:v1
  	
  	  [root@caleb /var/lib/docker]#docker image ls
  	  REPOSITORY                TAG                 IMAGE ID            CREATED             SIZE
  	  calebzhao/ubuntu          v1                  74435f89ab78        10 days ago         73.9MB
  	  ubuntu                    latest              74435f89ab78        10 days ago         73.9MB
  	  ...省略
  	  ```

  






















### 2.2.5、删除镜像

​	命令: `docker image rm <镜像ID>`

1. 先下载hello-world测试镜像

   ```
   [root@caleb /var/lib/docker]#docker image pull hello-world
   Using default tag: latest
   latest: Pulling from library/hello-world
   0e03bdcc26d7: Pull complete 
   Digest: sha256:d58e752213a51785838f9eed2b7a498ffa1cb3aa7f946dda11af39286c3db9a9
   Status: Downloaded newer image for hello-world:latest
   docker.io/library/hello-world:latest
   ```

2. 查看镜像列表

   ```
   [root@caleb /var/lib/docker]#docker image ls
   REPOSITORY                TAG                 IMAGE ID            CREATED             SIZE
   ubuntu                    latest              74435f89ab78        10 days ago         73.9MB
   rancher/server            stable              98d8bb571885        7 weeks ago         1.08GB
   hello-world               latest              bf756fb1ae65        5 months ago        13.3kB
   rancher/scheduler         v0.8.6              fbedeaddc3e9        17 months ago       248MB
   rancher/agent             v1.2.11             1cc7591af4f5        23 months ago       243MB
   rancher/net               v0.13.17            f170c38e3763        23 months ago       311MB
   rancher/dns               v0.17.4             678bde0de4d2        23 months ago       249MB
   rancher/healthcheck       v0.3.8              ce78cf69cc0b        23 months ago       391MB
   rancher/metadata          v0.10.4             02104eb6e270        23 months ago       251MB
   rancher/network-manager   v0.7.22             13381626c510        23 months ago       256MB
   rancher/net               holder              665d9f6e8cc1        3 years ago         267MB
   ```

3. 删除镜像

   ```
   [root@caleb /var/lib/docker]#docker image rm bf756fb1ae65
   Untagged: hello-world:latest
   Untagged: hello-world@sha256:d58e752213a51785838f9eed2b7a498ffa1cb3aa7f946dda11af39286c3db9a9
   Deleted: sha256:bf756fb1ae65adf866bd8c456593cd24beb6a0a061dedf42b26a993176745f6b
   Deleted: sha256:9c27e219663c25e0f28493790cc0b88bc973ba3b1686355f221c38a36978ac63
   ```

### 2.2.6、删除全部镜像

```
docker image rm -f `docker image ls -q`
```

### 2.2.7、只查看镜像ID

命令: `docker image ls -q`

```
[root@caleb /var/lib/docker]#docker image ls -q
74435f89ab78
74435f89ab78
98d8bb571885
bf756fb1ae65
fbedeaddc3e9
1cc7591af4f5
f170c38e3763
678bde0de4d2
ce78cf69cc0b
02104eb6e270
13381626c510
665d9f6e8cc1

```



## 2.3、容器管理

### 2.3.1、创建启动第一个容器

命令: `docker container run -it 镜像ID`

`i`参数表示以交互式方式启动

`t`参数表示启动一个tty终端

```
[root@caleb /var/lib/docker]#docker container run -it 831691599b88
[root@32a38ffcbaee /]# cat /etc/redhat-release 
CentOS Linux release 8.2.2004 (Core) 
```

### 2.3.2、查看容器列表

命令: `docker container ls` 

```
[root@caleb /home/calebzhao]#docker container ls
CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS              PORTS               NAMES
32a38ffcbaee        831691599b88        "/bin/bash"         About a minute ago   Up About a minute                       funny_rhodes

```

- CONTAINER ID：容器唯一ID（自动生成的）

- NAMES：容器名称，如果不指定会自动生成，可以手动指定

  **手动指定容器名称**：

  ```
  [root@caleb /var/lib/docker]#docker container run -it --name "claebzhao_centos" 831691599b88
  ```

  查看容器列表, 会发现NAMES已变成我们指定的容器名称

  ```
  [root@caleb /home/calebzhao]#docker container ls
  CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
  cbaf9b7dd4a6        831691599b88        "/bin/bash"         3 seconds ago       Up 2 seconds                            claebzhao_centos
  ```

  

- STATUS：容器的运行状态

  **`docker container ls -a` 可以查看所有容器（无论是否已启动）**

### 2.3.3、以守护式方式启动

命令: `docker container run -d --name “容器名称” 镜像ID`

```
[root@caleb /home/calebzhao]#docker container run -d --name "caleb_nginx" nginx:1.19.0
Unable to find image 'nginx:1.19.0' locally
1.19.0: Pulling from library/nginx
8559a31e96f4: Pull complete 
8d69e59170f7: Pull complete 
3f9f1ec1d262: Pull complete 
d1f5ff4f210d: Pull complete 
1e22bfa8652e: Pull complete 
Digest: sha256:21f32f6c08406306d822a0e6e8b7dc81f53f336570e852e25fbe1e3e3d0d0133
Status: Downloaded newer image for nginx:1.19.0
44157226ab7c8271ce34cd1f5c9721f4225a329aa4c8276ddda1b93840ccef94
[root@caleb /home/calebzhao]#docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
44157226ab7c        nginx:1.19.0        "/docker-entrypoint.…"   29 seconds ago      Up 28 seconds       80/tcp              caleb_nginx
cbaf9b7dd4a6        831691599b88        "/bin/bash"              17 minutes ago      Up 17 minutes   
```

### 2.2.4、查看容器详细信息

命令：`docker container inspect 容器ID或NAMES`

```
[root@caleb /home/calebzhao]#docker container inspect caleb_nginx
[
    {
        "Id": "44157226ab7c8271ce34cd1f5c9721f4225a329aa4c8276ddda1b93840ccef94",
        "Created": "2020-06-27T09:12:11.849621832Z",
        "Path": "/docker-entrypoint.sh",
        "Args": [
            "nginx",
            "-g",
            "daemon off;"
        ],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 13305,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2020-06-27T09:12:12.224153471Z",
            "FinishedAt": "0001-01-01T00:00:00Z"
        },
        "Image": "sha256:2622e6cca7ebbb6e310743abce3fc47335393e79171b9d76ba9d4f446ce7b163",
        "ResolvConfPath": "/var/lib/docker/containers/44157226ab7c8271ce34cd1f5c9721f4225a329aa4c8276ddda1b93840ccef94/resolv.conf",
        "HostnamePath": "/var/lib/docker/containers/44157226ab7c8271ce34cd1f5c9721f4225a329aa4c8276ddda1b93840ccef94/hostname",
        "HostsPath": "/var/lib/docker/containers/44157226ab7c8271ce34cd1f5c9721f4225a329aa4c8276ddda1b93840ccef94/hosts",
        "LogPath": "/var/lib/docker/containers/44157226ab7c8271ce34cd1f5c9721f4225a329aa4c8276ddda1b93840ccef94/44157226ab7c8271ce34cd1f5c9721f4225a329aa4c8276ddda1b93840ccef94-json.log",
        "Name": "/openrestry",
        "RestartCount": 0,
        "Driver": "overlay2",
        "Platform": "linux",
        "MountLabel": "",
        "ProcessLabel": "",
        "AppArmorProfile": "",
        "ExecIDs": null,
        "HostConfig": {
            "Binds": null,
            "ContainerIDFile": "",
            "LogConfig": {
                "Type": "json-file",
                "Config": {}
            },
            "NetworkMode": "default",
            "PortBindings": {},
            "RestartPolicy": {
                "Name": "no",
                "MaximumRetryCount": 0
            },
            "AutoRemove": false,
            "VolumeDriver": "",
            "VolumesFrom": null,
            "CapAdd": null,
            "CapDrop": null,
            "Capabilities": null,
            "Dns": [],
            "DnsOptions": [],
            "DnsSearch": [],
            "ExtraHosts": null,
            "GroupAdd": null,
            "IpcMode": "private",
            "Cgroup": "",
            "Links": null,
            "OomScoreAdj": 0,
            "PidMode": "",
            "Privileged": false,
            "PublishAllPorts": false,
            "ReadonlyRootfs": false,
            "SecurityOpt": null,
            "UTSMode": "",
            "UsernsMode": "",
            "ShmSize": 67108864,
            "Runtime": "runc",
            "ConsoleSize": [
                0,
                0
            ],
            "Isolation": "",
            "CpuShares": 0,
            "Memory": 0,
            "NanoCpus": 0,
            "CgroupParent": "",
            "BlkioWeight": 0,
            "BlkioWeightDevice": [],
            "BlkioDeviceReadBps": null,
            "BlkioDeviceWriteBps": null,
            "BlkioDeviceReadIOps": null,
            "BlkioDeviceWriteIOps": null,
            "CpuPeriod": 0,
            "CpuQuota": 0,
            "CpuRealtimePeriod": 0,
            "CpuRealtimeRuntime": 0,
            "CpusetCpus": "",
            "CpusetMems": "",
            "Devices": [],
            "DeviceCgroupRules": null,
            "DeviceRequests": null,
            "KernelMemory": 0,
            "KernelMemoryTCP": 0,
            "MemoryReservation": 0,
            "MemorySwap": 0,
            "MemorySwappiness": null,
            "OomKillDisable": false,
            "PidsLimit": null,
            "Ulimits": null,
            "CpuCount": 0,
            "CpuPercent": 0,
            "IOMaximumIOps": 0,
            "IOMaximumBandwidth": 0,
            "MaskedPaths": [
                "/proc/asound",
                "/proc/acpi",
                "/proc/kcore",
                "/proc/keys",
                "/proc/latency_stats",
                "/proc/timer_list",
                "/proc/timer_stats",
                "/proc/sched_debug",
                "/proc/scsi",
                "/sys/firmware"
            ],
            "ReadonlyPaths": [
                "/proc/bus",
                "/proc/fs",
                "/proc/irq",
                "/proc/sys",
                "/proc/sysrq-trigger"
            ]
        },
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/9bfabbc20dfdb3c950f62f09aa40351f1f47e9af8116e01819123e26fb37dcf1-init/diff:/var/lib/docker/overlay2/8016e84ae93b0108ced6291fb289cc3c6892b3463a6f5e18971b515f4689bc58/diff:/var/lib/docker/overlay2/0f5cf1ea25e6f46e46d713101c07c1dac703d4188a60d6801db9022ec863a533/diff:/var/lib/docker/overlay2/2bad2ffb3d277697006aa77a10c92ae0fa095150c8e9811d985e2bf8b3898403/diff:/var/lib/docker/overlay2/7c3e51ca01efb9bee9538db6a9db7753c1aa27b3d436533e098917bac894526b/diff:/var/lib/docker/overlay2/328a933deddaa7ff169afbe5743e64ef38267a3114a9b47eccc53af2a71be82c/diff",
                "MergedDir": "/var/lib/docker/overlay2/9bfabbc20dfdb3c950f62f09aa40351f1f47e9af8116e01819123e26fb37dcf1/merged",
                "UpperDir": "/var/lib/docker/overlay2/9bfabbc20dfdb3c950f62f09aa40351f1f47e9af8116e01819123e26fb37dcf1/diff",
                "WorkDir": "/var/lib/docker/overlay2/9bfabbc20dfdb3c950f62f09aa40351f1f47e9af8116e01819123e26fb37dcf1/work"
            },
            "Name": "overlay2"
        },
        "Mounts": [],
        "Config": {
            "Hostname": "44157226ab7c",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "ExposedPorts": {
                "80/tcp": {}
            },
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "NGINX_VERSION=1.19.0",
                "NJS_VERSION=0.4.1",
                "PKG_RELEASE=1~buster"
            ],
            "Cmd": [
                "nginx",
                "-g",
                "daemon off;"
            ],
            "Image": "nginx:1.19.0",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": [
                "/docker-entrypoint.sh"
            ],
            "OnBuild": null,
            "Labels": {
                "maintainer": "NGINX Docker Maintainers <docker-maint@nginx.com>"
            },
            "StopSignal": "SIGTERM"
        },
        "NetworkSettings": {
            "Bridge": "",
            "SandboxID": "7ca70bf7715f8ade7534f455f7e942246c6790e6a63f3c4d2201e6e772104e17",
            "HairpinMode": false,
            "LinkLocalIPv6Address": "",
            "LinkLocalIPv6PrefixLen": 0,
            "Ports": {
                "80/tcp": null
            },
            "SandboxKey": "/var/run/docker/netns/7ca70bf7715f",
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "EndpointID": "95d3211d582f213348cf1f989424ae0e0b7f9775921cc624bb57feba61d28458",
            "Gateway": "172.17.0.1",
            "GlobalIPv6Address": "",
            "GlobalIPv6PrefixLen": 0,
            "IPAddress": "172.17.0.3",
            "IPPrefixLen": 16,
            "IPv6Gateway": "",
            "MacAddress": "02:42:ac:11:00:03",
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "47e20989d54c55152c1b564861716c666ffd501e39e1c187feb5d04f3434c45e",
                    "EndpointID": "95d3211d582f213348cf1f989424ae0e0b7f9775921cc624bb57feba61d28458",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.3",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:03",
                    "DriverOpts": null
                }
            }
        }
    }
]

```

访问刚刚启动的nginx

```
[root@caleb /home/calebzhao]#curl 172.17.0.3
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```



### 2.2.5、交互式方式启动，用完即销毁

应用场景：工具类、开发、测试、临时性的任务，用完 就不需要了

命令: `docker contaienr run -it --name "名称" --rm 镜像ID`

```
docker container run -it --name "claebzhao_centos" --rm 831691599b88
```

### 2.2.6、暴露docker端口到外部

命令: `docker container run -d -p 宿主机端口:容器内服务端口 镜像ID`

```
[root@caleb /home/calebzhao]#docker container run -d -p 9000:80 --name "mynginx" nginx:1.19.0
558da0df05649888b39d7db291b09febbd4c735949acdb28da9e07eed3a498be

[root@caleb /home/calebzhao]#docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
558da0df0564        nginx:1.19.0        "/docker-entrypoint.…"   7 minutes ago       Up 2 minutes        0.0.0.0:9000->80/tcp   mynginx

```

### 2.2.7、启动已有容器

命令：`docker container start 容器ID或名称`

启动的时候不用带-d -p --name等参数了，使用run命令启动时已经固定到容器中了，后面直接start启动即可

### 2.2.8、停止运行中的容器

命令: `docker container stop 容器ID或名称`



### 2.2.9、容器的连接

- attach方式连接

命令：`docker container attach 容器ID或名称`

**注意这种方式连接到容器后，如果exit退出，则容器也退出了**

```
# 启动容器
[root@caleb /home/calebzhao]#docker container start claebzhao_centos
claebzhao_centos

# 查看容器列表，发现centos已经启动
[root@caleb /home/calebzhao]#docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
558da0df0564        nginx:1.19.0        "/docker-entrypoint.…"   13 minutes ago      Up 2 minutes        0.0.0.0:9000->80/tcp   mynginx
cbaf9b7dd4a6        831691599b88        "/bin/bash"              About an hour ago   Up 12 seconds                              claebzhao_centos

# 连接到已经启动的容器
[root@caleb /home/calebzhao]#docker container attach claebzhao_centos
# 已经进入到centos了, 然后退出
[root@cbaf9b7dd4a6 /]# exit
exit

# 再次查看启动的容器列表，会发现启动了centos容器不在了，说明attach方式连接容器时容器内退出，那么容器也停止运行
[root@caleb /home/calebzhao]#docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
558da0df0564        nginx:1.19.0        "/docker-entrypoint.…"   21 minutes ago      Up 9 minutes        0.0.0.0:9000->80/tcp   mynginx
```

- exec -it方式连接 （**推荐**）

  命令: `docker container exec -it 容器名称或ID /bin/bash`

  **这种方式连接容器会以一个子进程的方式登陆到容器，退出也是退出的子进程，所以容器并不会退出**

  ```
  [root@caleb /home/calebzhao]#docker container exec -it claebzhao_centos /bin/bash
  [root@cbaf9b7dd4a6 /]# 
  [root@cbaf9b7dd4a6 /]# exit
  exit
  [root@caleb /home/calebzhao]#docker container ls
  CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
  558da0df0564        nginx:1.19.0        "/docker-entrypoint.…"   21 minutes ago      Up 9 minutes        0.0.0.0:9000->80/tcp   mynginx
  cbaf9b7dd4a6        831691599b88        "/bin/bash"              About an hour ago   Up About a minute 
  ```

  ### 2.2.10、删除单个容器

  命令：`docker container rm 容器ID或名称`

  ### 2.2.11、批量删除所有容器

  ```
  docker container rm -f `docker container ls -a -q`
  ```

  ### 2.2.12、交互式方式启动的容器后台运行

  注意：默认情况下`docker container run -it --name "centos" centos`这种以交互式方式启动的容器就是执行命令`/bin/bash`,  所以当退出bash时那么这个这个进程就退出了，那么容器也就退出了，所以为了让退出命令行窗口时不杀死该docker容器，那么就要让`/bin/bash`这个命令一直在运行，有3种方式：

- Ctrl + P,  Ctrl + Q

- 让/bin/bash一直运行一个死循环

  ```
  docker container run -it --name "centos" centos sleep 100000000
  ```

- 让程序一直在前台运行（hung在前台）

  制作守护容器时，常用的方法，比如制作nginx容器时，让nginx不以守护方式运行，而是在前台运行

### 2.2.13、查看容器内运行的进程

命令: `docker container top 容器ID或名称`

```
[root@caleb /home/calebzhao]#docker container top mynginx
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                14427               14409               0                   18:00               ?                   00:00:00            nginx: master process nginx -g daemon off;
101                 14471               14427               0                   18:01               ?                   00:00:00            nginx: worker process

```

### 2.2.14、查看容器日志

命令：`docker container logs -tf 容器ID或名称`

`-t`参数表示查看的时候每一行携带时间

`-f`同`tail -f`命令， 滚动输出日志

```
[root@caleb /home/calebzhao]#docker container logs mynginx
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
192.168.42.1 - - [27/Jun/2020:09:49:51 +0000] "GET / HTTP/1.1" 200 612 "-" "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/78.0.3904.108 Safari/537.36" "-"
192.168.42.1 - - [27/Jun/2020:09:49:52 +0000] "GET /favicon.ico HTTP/1.1" 404 555 "http://192.168.42.200:9000/" "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/78.0.3904.108 Safari/537.36" "-"
2020/06/27 09:49:52 [error] 28#28: *2 open() "/usr/share/nginx/html/favicon.ico" failed (2: No such file or directory), client: 192.168.42.1, server: localhost, request: "GET /favicon.ico HTTP/1.1", host: "192.168.42.200:9000", referrer: "http://192.168.42.200:9000/"
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: IPv6 listen already enabled, exiting
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
192.168.42.1 - - [27/Jun/2020:09:54:57 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/78.0.3904.108 Safari/537.36" "-"
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: IPv6 listen already enabled, exiting
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: IPv6 listen already enabled, exiting
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
172.17.0.1 - - [27/Jun/2020:11:16:26 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"

```



### 2.2.15、数据卷

- 手工方式

  1. 从宿主机复制到容器

     `docker container cp /data/a.txt n1:/opt/`

  2. 从容器复制到宿主机
  	
  	`docker container cp n1:/opt/a.txt /tmp`
  
- 数据卷方式

  `docker container run -d -p 8080:80 --name "nginx1" -v /data/nginx1/www:/usr/share/nginx/www -v /etc/nginx/nginx/conf:/etc/nginx/nginx.conf -v /var/logs/nginx:/var/logs/nginx nginx`

- 数据卷容器

  1. 宿主机模拟数据目录

     ```
     mkdir -p /opt/volume/nginx/conf
     mkdir -p /opt/volume/nginx/www
     mkdir -p /opt/volume/nginx/logs
     ```

  2. 启动数据卷容器

     ```
     docker container run -it --name "nginx_volumes" -v /opt/volume/nginx/conf:/etc/nginx/conf -v /opt/volume/nginx/www:/usr/share/nginx/html -v /opt/volume/nginx/logs:/var/log/nginx centos /bin/bash
     
     ctrl + p,  ctrl + q
     ```

  3. 使用数据卷容器

     ```
     docker container run -d -p 9001:80 --volumes-from nginx_volumes --name "nginx1" nginx
     docker container run -d -p 9002:80 --volumes-from nginx_volumes --name "nginx2" nginx
     ```

     作用：在集中管理集群时，大批量的容器都需要挂载相同的多个数据卷时，可以采用数据卷容器进行统一管理

## 2.4、制作镜像

  1. 下载基础镜像并启动

     ```
     docker image pull centos:centos7
     docker container run -it --name "centos7" centos:centos7
     ```

  2. 进入启动的docker容器中安装基础软件，这里以ssh服务为例

     ```
     yum install -y openssh-server
     mkdir /var/run/sshd
     echo 'UseDNS no' >> /etc/ssh/sshd_config
     sed -i -e '/pam_loginuid.so/d' /etc/pam.d/sshd
     echo 'root:123456' |chpasswd
     /usr/bin/ssh-keygen -A
     ```

     然后查找sshd命令的位置: 

     ```
     [root@ad123df56cf4 /]# find / -name sshd
     /etc/pam.d/sshd
     /etc/sysconfig/sshd
     /usr/sbin/sshd
     /var/emptysshd
     ```

     安装网络工具：

     ```
     yum install -y net-tools
     
     [root@0d3f313d447d /]# ifconfig
     eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
             inet 172.17.0.3  netmask 255.255.0.0  broadcast 172.17.255.255
             ether 02:42:ac:11:00:03  txqueuelen 0  (Ethernet)
             RX packets 5028  bytes 11413068 (10.8 MiB)
             RX errors 0  dropped 0  overruns 0  frame 0
             TX packets 3129  bytes 173376 (169.3 KiB)
             TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
     
     lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
             inet 127.0.0.1  netmask 255.0.0.0
             loop  txqueuelen 1000  (Local Loopback)
             RX packets 0  bytes 0 (0.0 B)
             RX errors 0  dropped 0  overruns 0  frame 0
             TX packets 0  bytes 0 (0.0 B)
             TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
     
     ```

     此时可以从宿主机通过ssh连接docker了

     ```
     [calebzhao@caleb ~]$ssh root@172.17.0.3
     The authenticity of host '172.17.0.3 (172.17.0.3)' can't be established.
     DSA key fingerprint is SHA256:n69xPI+Gc6J9xyFWdaaGFw+QpgI1XuGc+oJMQpqcJ0U.
     DSA key fingerprint is MD5:3b:3e:ab:b3:79:98:66:2e:a4:12:85:ee:88:6e:b1:ed.
     Are you sure you want to continue connecting (yes/no)? yes
     Warning: Permanently added '172.17.0.3' (DSA) to the list of known hosts.
     root@172.17.0.3's password: 
     [root@0d3f313d447d ~]# 
     ```

     

  3. 制作镜像

     命令：`docker commit 源镜像ID或名称 目标镜像REPOSITOR:TAG`

     ```
     [root@caleb /home/calebzhao]#docker commit centos7 centos7_ssh:v1
     sha256:042b5e29e700b9d4ac5928b9a925e417e5894f7b21f02bd6977dd2d1d0279e17
     [root@caleb /home/calebzhao]#docker image ls
     REPOSITORY                TAG                 IMAGE ID            CREATED             SIZE
     centos7_ssh               v1                  042b5e29e700        7 seconds ago       288MB
     ...省略其他image
     
     ```

  4. 启动制作的镜像，并启动ssh服务

     ```
     ddocker container run -d --name=sshd_2222 -p 2222:22 cnetos7_ssh:v2 /usr/sbin/sshd -D
     ```

  5. 验证容器正常启动

     ```
     [root@caleb /var/lib/docker]#docker container ls
     CONTAINER ID        IMAGE               COMMAND               CREATED             STATUS              PORTS               NAMES
     a4b8f3b31924        centos7_ssh:v1      "/root/run-sshd.sh"   7 seconds ago       Up 5 seconds                            centos7_ssh_v1
     ```

  6. 验证可以正常远程连接

     首先查看容器地址

     ```
     [root@caleb /var/lib/docker]#docker container inspect centos7_ssh_v1
     [
         {
             "Id": "a4b8f3b31924b9f294b647c8cb91c8c057aeb0061b1fed7afd3cc3137ed6ace0",
             "Created": "2020-06-27T15:27:24.68785408Z",
             "Path": "/root/run-sshd.sh",
             "Args": [],
             "State": {
                 "Status": "running",
                 "Running": true,
                 "Paused": false,
                 "Restarting": false,
                 "OOMKilled": false,
                 "Dead": false,
                 "Pid": 19037,
                 "ExitCode": 0,
                 "Error": "",
                 "StartedAt": "2020-06-27T15:27:25.554543189Z",
                 "FinishedAt": "0001-01-01T00:00:00Z"
             },
             "Image": "sha256:c8f81bfa86fefe989b732340eda03407a835c3caad91ed50bd57d95d46b0a300",
             "ResolvConfPath": "/var/lib/docker/containers/a4b8f3b31924b9f294b647c8cb91c8c057aeb0061b1fed7afd3cc3137ed6ace0/resolv.conf",
             "HostnamePath": "/var/lib/docker/containers/a4b8f3b31924b9f294b647c8cb91c8c057aeb0061b1fed7afd3cc3137ed6ace0/hostname",
             "HostsPath": "/var/lib/docker/containers/a4b8f3b31924b9f294b647c8cb91c8c057aeb0061b1fed7afd3cc3137ed6ace0/hosts",
             "LogPath": "/var/lib/docker/containers/a4b8f3b31924b9f294b647c8cb91c8c057aeb0061b1fed7afd3cc3137ed6ace0/a4b8f3b31924b9f294b647c8cb91c8c057aeb0061b1fed7afd3cc3137ed6ace0-json.log",
             "Name": "/centos7_ssh_v1",
             "RestartCount": 0,
             "Driver": "overlay2",
             "Platform": "linux",
             "MountLabel": "",
             "ProcessLabel": "",
             "AppArmorProfile": "",
             "ExecIDs": null,
             "HostConfig": {
                 "Binds": null,
                 "ContainerIDFile": "",
                 "LogConfig": {
                     "Type": "json-file",
                     "Config": {}
                 },
                 "NetworkMode": "default",
                 "PortBindings": {},
                 "RestartPolicy": {
                     "Name": "no",
                     "MaximumRetryCount": 0
                 },
                 "AutoRemove": false,
                 "VolumeDriver": "",
                 "VolumesFrom": null,
                 "CapAdd": null,
                 "CapDrop": null,
                 "Capabilities": null,
                 "Dns": [],
                 "DnsOptions": [],
                 "DnsSearch": [],
                 "ExtraHosts": null,
                 "GroupAdd": null,
                 "IpcMode": "private",
                 "Cgroup": "",
                 "Links": null,
                 "OomScoreAdj": 0,
                 "PidMode": "",
                 "Privileged": false,
                 "PublishAllPorts": false,
                 "ReadonlyRootfs": false,
                 "SecurityOpt": null,
                 "UTSMode": "",
                 "UsernsMode": "",
                 "ShmSize": 67108864,
                 "Runtime": "runc",
                 "ConsoleSize": [
                     0,
                     0
                 ],
                 "Isolation": "",
                 "CpuShares": 0,
                 "Memory": 0,
                 "NanoCpus": 0,
                 "CgroupParent": "",
                 "BlkioWeight": 0,
                 "BlkioWeightDevice": [],
                 "BlkioDeviceReadBps": null,
                 "BlkioDeviceWriteBps": null,
                 "BlkioDeviceReadIOps": null,
                 "BlkioDeviceWriteIOps": null,
                 "CpuPeriod": 0,
                 "CpuQuota": 0,
                 "CpuRealtimePeriod": 0,
                 "CpuRealtimeRuntime": 0,
                 "CpusetCpus": "",
                 "CpusetMems": "",
                 "Devices": [],
                 "DeviceCgroupRules": null,
                 "DeviceRequests": null,
                 "KernelMemory": 0,
                 "KernelMemoryTCP": 0,
                 "MemoryReservation": 0,
                 "MemorySwap": 0,
                 "MemorySwappiness": null,
                 "OomKillDisable": false,
                 "PidsLimit": null,
                 "Ulimits": null,
                 "CpuCount": 0,
                 "CpuPercent": 0,
                 "IOMaximumIOps": 0,
                 "IOMaximumBandwidth": 0,
                 "MaskedPaths": [
                     "/proc/asound",
                     "/proc/acpi",
                     "/proc/kcore",
                     "/proc/keys",
                     "/proc/latency_stats",
                     "/proc/timer_list",
                     "/proc/timer_stats",
                     "/proc/sched_debug",
                     "/proc/scsi",
                     "/sys/firmware"
                 ],
                 "ReadonlyPaths": [
                     "/proc/bus",
                     "/proc/fs",
                     "/proc/irq",
                     "/proc/sys",
                     "/proc/sysrq-trigger"
                 ]
             },
             "GraphDriver": {
                 "Data": {
                     "LowerDir": "/var/lib/docker/overlay2/cfddefdd90a72c53cfb6deb142580782e8a38aadfb6752400bee0ed3ce2c149b-init/diff:/var/lib/docker/overlay2/e14468cd09751b5959257e2adcc47b31be6745bf9d18e69076a01a1f11e6322e/diff:/var/lib/docker/overlay2/ce65549ad843e97d7499eabfc27792e67627932c6dfa7176b79dbdf048b1d7aa/diff",
                     "MergedDir": "/var/lib/docker/overlay2/cfddefdd90a72c53cfb6deb142580782e8a38aadfb6752400bee0ed3ce2c149b/merged",
                     "UpperDir": "/var/lib/docker/overlay2/cfddefdd90a72c53cfb6deb142580782e8a38aadfb6752400bee0ed3ce2c149b/diff",
                     "WorkDir": "/var/lib/docker/overlay2/cfddefdd90a72c53cfb6deb142580782e8a38aadfb6752400bee0ed3ce2c149b/work"
                 },
                 "Name": "overlay2"
             },
             "Mounts": [],
             "Config": {
                 "Hostname": "a4b8f3b31924",
                 "Domainname": "",
                 "User": "",
                 "AttachStdin": false,
                 "AttachStdout": false,
                 "AttachStderr": false,
                 "Tty": false,
                 "OpenStdin": false,
                 "StdinOnce": false,
                 "Env": [
                     "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
                 ],
                 "Cmd": [
                     "/root/run-sshd.sh"
                 ],
                 "Image": "centos7_ssh:v1",
                 "Volumes": null,
                 "WorkingDir": "",
                 "Entrypoint": null,
                 "OnBuild": null,
                 "Labels": {
                     "org.label-schema.build-date": "20200504",
                     "org.label-schema.license": "GPLv2",
                     "org.label-schema.name": "CentOS Base Image",
                     "org.label-schema.schema-version": "1.0",
                     "org.label-schema.vendor": "CentOS",
                     "org.opencontainers.image.created": "2020-05-04 00:00:00+01:00",
                     "org.opencontainers.image.licenses": "GPL-2.0-only",
                     "org.opencontainers.image.title": "CentOS Base Image",
                     "org.opencontainers.image.vendor": "CentOS"
                 }
             },
             "NetworkSettings": {
                 "Bridge": "",
                 "SandboxID": "c2e821f204513529a7053c7b9a7982833e7e79c9e3e6df477c56e891dad8c2ad",
                 "HairpinMode": false,
                 "LinkLocalIPv6Address": "",
                 "LinkLocalIPv6PrefixLen": 0,
                 "Ports": {},
                 "SandboxKey": "/var/run/docker/netns/c2e821f20451",
                 "SecondaryIPAddresses": null,
                 "SecondaryIPv6Addresses": null,
                 "EndpointID": "9cc00e4e99872b931f48b7f163b69932e39db2cf86c0547477b859febdc959e8",
                 "Gateway": "172.17.0.1",
                 "GlobalIPv6Address": "",
                 "GlobalIPv6PrefixLen": 0,
                 "IPAddress": "172.17.0.2",
                 "IPPrefixLen": 16,
                 "IPv6Gateway": "",
                 "MacAddress": "02:42:ac:11:00:02",
                 "Networks": {
                     "bridge": {
                         "IPAMConfig": null,
                         "Links": null,
                         "Aliases": null,
                         "NetworkID": "47e20989d54c55152c1b564861716c666ffd501e39e1c187feb5d04f3434c45e",
                         "EndpointID": "9cc00e4e99872b931f48b7f163b69932e39db2cf86c0547477b859febdc959e8",
                         "Gateway": "172.17.0.1",
                         "IPAddress": "172.17.0.2",
                         "IPPrefixLen": 16,
                         "IPv6Gateway": "",
                         "GlobalIPv6Address": "",
                         "GlobalIPv6PrefixLen": 0,
                         "MacAddress": "02:42:ac:11:00:02",
                         "DriverOpts": null
                     }
                 }
             }
         }
     ]
     
     ```

     可以看到` "IPAddress": "172.17.0.2" ，宿主机上用ssh连接` 

     ```
     [root@caleb /var/lib/docker]#ssh root@172.17.0.2
     root@172.17.0.2's password: 
     Last login: Sat Jun 27 15:30:03 2020 from gateway
     ```

     

# 22、用户管理

```
#将用户mysql添加到test用户组中
usermod -G test mysql
```

# 23、防火墙

**查看已开放的端口**

```
firewall-cmd --list-ports
firewall-cmd --list-all
```

**开放端口（开放后需要要重启防火墙才生效）**

```
firewall-cmd --zone=public --add-port=3338/tcp --permanent

/sbin/iptables -I INPUT -p tcp --dport 8000 -j ACCEPT #开启8000端口

/etc/rc.d/init.d/iptables save #保存配置

/etc/rc.d/init.d/iptables restart #重启服务

/etc/init.d/iptables status # 查看端口是否已经开放 
```

**关闭端口（关闭后需要要重启防火墙才生效）**

```
firewall-cmd --zone=public --remove-port=3338/tcp --permanent
```

**重启防火墙**

```
firewall-cmd --reload
```

**开机启动防火墙**

```
systemctl enable firewalld
```

**开启防火墙**

```
systemctl start firewalld
```

**禁止防火墙开机启动**

```
systemctl disable firewalld
```

 **显示规则列表**

``` 
firewall-cmd --list-rich-rules 
```

**添加规则**

```
firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="10.90.35.0/24" port protocol="tcp" port="38080" accept"

firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="10.90.35.0/24" port protocol="tcp" port="38080" reject"
```

**删除规则**

```
firewall-cmd --permanent --remove-rich-rule '规则列表'
```

删除后需要reload，然后再`firewall-cmd --list-rich-rules `才会看到删除了

**停止防火墙**

```
systemctl stop firewalld
```

 

# 24、ffmpeg安装

1, 到http://www.videolan.org/developers/x264.html下载264编码库last_x264.tar.bz2

```
tar -xjf last_x264.tar.bz2 
cd x264-snapshot-20180520-2245/
./configure --prefix=/usr/local/x264 --enable-shared --enable-static --disable-asm
make
make install
```


2, 到http://ffmpeg.org/download.html下载 ffmpeg 最新版本ffmpeg-4.0.tar.bz2

```
tar xjf ffmpeg-4.0.tar.bz2
cd ffmpeg-4.0/
./configure --prefix=/usr/local/ffmpeg --enable-shared --enable-yasm --enable-libx264 --enable-gpl --enable-pthreads --extra-cflags=-I/usr/local/x264/include --extra-ldflags=-L/usr/local/x264/lib//-j7 7个核编译, 加快编译速度
make -j7
make install
```

3, 测试

```
ffmpeg -version
```

如果无异常则安装成功；
异常
错误信息：
ffmpeg: error while loading shared libraries: libavdevice.so.58: cannot open shared object file: No such file or directory
先　find / -name libavdevice.so.58 得到该文件的目录地址，我找到的是在ffmpeg安装目录的lib目录下；
/usr/local/ffmpeg/lib/



错误信息：
找不到libx264.so.155

```
find / -name libx264.so.155

vim /etc/ld.so.conf.d/x264.conf
```

添加 /usr/local/x264/lib/

4、 修改配置文件

```
vim /etc/ld.so.conf
```

在文件末尾加上`/usr/local/ffmpeg/lib/`保存退出，然后更新环境变量：`sudo ldconfig`

在文件末尾加上`/usr/local/x264/lib/`保存退出，然后更新环境变量：`sudo ldconfig`

5、 配置环境变量

```
vim /etc/profile
```
使用 vim /etc/profile命令打开profile文件，在文件末添加环境变量：
```
PATH=$PATH:/usr/local/ffmpeg/bin
 
export PATH
```

随后执行` source /etc/profile`使配置生效

6、 查看是否成功

```
ffmpeg -version
```



# 25、systemctl

```
systemctl list-unit-files  服务列表
systemctl start xx
systemctl stop xx
systemctl restart xx
systemctl status xx
systemctl daemon-reload
```

# 26、firefox安装

## 1、下载firefox

官网下载： https://download-ssl.firefox.com.cn/releases/firefox/89.0/zh-CN/Firefox-latest-x86_64.tar.bz2

2、安装

将`Firefox-latest-x86_64.tar.bz2`上传到`/usr/local`目录下

```

tar xvf Firefox-latest-x86_64.tar.bz2
```

建议软连接

```
sudo ln -sf /usr/local/firefox /usr/bin/firefox
```

3、启动出错解放方法

```
XPCOMGlueLoad error for file /usr/local/firefox/libmozgtk.so:
libgtk-3.so.0: cannot open shared object file: No such file or directory
Couldn't load XPCOM.
```

执行`yum install gtk3 libXt `



# 27、升级sshd

## 27.1、执行yum update openssh先升级下

（这里准备统一openssh版本为7.4p1之后再统一编译安装升级到openssh8.0p1）

```
[root@linux-node3 ~]# yum update openssh -y
 
[root@linux-node3 ~]# ssh -V
OpenSSH_7.4p1, OpenSSL 1.0.2k-fips  26 Jan 2017
[root@linux-node3 ~]#
```

## 27.2、安装telnet-server以及xinetd

```
[root@linux-node3 ~]# yum install xinetd telnet-server -y
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.163.com
 * epel: mirrors.aliyun.com
 * extras: mirrors.cn99.com
 * updates: mirrors.cn99.com
Package 2:xinetd-2.3.15-13.el7.x86_64 already installed and latest version
Package 1:telnet-server-0.17-64.el7.x86_64 already installed and latest version
Nothing to do
[root@linux-node3 ~]#
```

## 27.3、安装telnet-server以及xinetd

```
[root@linux-node3 ~]``# yum install xinetd telnet-server -y
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 ``* base: mirrors.163.com
 ``* epel: mirrors.aliyun.com
 ``* extras: mirrors.cn99.com
 ``* updates: mirrors.cn99.com
Package 2:xinetd-2.3.15-13.el7.x86_64 already installed and latest version
Package 1:telnet-server-0.17-64.el7.x86_64 already installed and latest version
Nothing to ``do
[root@linux-node3 ~]``#
```

**配置telnet**

现在很多centos7版本安装telnet-server以及xinetd之后没有一个叫telnet的配置文件了。

如果下面telnet文件不存在的话，可以跳过这部分的更改

```
[root@linux-node3 ~]# ll /etc/xinetd.d/telnet
ls: cannot access /etc/xinetd.d/telnet: No such file or directory
```

如果下面文件存在，请更改配置telnet可以root登录，把disable = no改成disable = yes

``` 
[root@linux-node3 ~]# vim /etc/xinetd.d/telnet
service telnet
{
    disable = yes
    flags       = REUSE
    socket_type = stream       
    wait        = no
    user        = root
    server      = /usr/sbin/in.telnetd
    log_on_failure  += USERID
}
```

配置telnet登录的终端类型，在/etc/securetty文件末尾增加一些pts终端，如下

```
[root@linux-node3 ~]# vim /etc/securetty

pts/0
pts/1
pts/2
pts/3
```

启动telnet服务，并设置开机自动启动

```
[root@linux-node3 ~]# systemctl enable xinetd
[root@linux-node3 ~]# systemctl enable telnet.socket
Created symlink from /etc/systemd/system/sockets.target.wants/telnet.socket to /usr/lib/systemd/system/telnet.socket.
[root@linux-node3 ~]# systemctl start telnet.socket
[root@linux-node3 ~]# systemctl start xinetd
[root@linux-node3 ~]# netstat -lntp|grep 23
tcp6       0      0 :::23                   :::*                    LISTEN      1/systemd          

#开启防火墙23端口
[root@linux-node3 ~]# firewall-cmd --permanent --add-port=23/tcp --zone=public
[root@linux-node3 ~]# firewall-cmd --reload

```

切换到telnet方式登录，以后的操作都在telnet终端下操作，防止ssh连接意外中断造成升级失败



## 27.4、安装依赖包

升级需要几个组件，有些是和编译相关的等

```
[root@linux-node3 ~]``# yum install -y gcc gcc-c++ glibc make autoconf openssl openssl-devel pcre-devel pam-devel
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 ``* base: mirrors.163.com
 ``* epel: mirrors.aliyun.com
 ``* extras: mirrors.cn99.com
 ``* updates: mirrors.cn99.com
Package ``gcc``-4.8.5-36.el7_6.1.x86_64 already installed and latest version
Package ``gcc``-c++-4.8.5-36.el7_6.1.x86_64 already installed and latest version
Package glibc-2.17-260.el7_6.4.x86_64 already installed and latest version
Package 1:``make``-3.82-23.el7.x86_64 already installed and latest version
Package autoconf-2.69-11.el7.noarch already installed and latest version
Package 1:openssl-1.0.2k-16.el7_6.1.x86_64 already installed and latest version
Package 1:openssl-devel-1.0.2k-16.el7_6.1.x86_64 already installed and latest version
Package pcre-devel-8.32-17.el7.x86_64 already installed and latest version
Package pam-devel-1.1.8-22.el7.x86_64 already installed and latest version
Nothing to ``do
[root@linux-node3 ~]``#
```

安装pam和zlib等（后面的升级操作可能没用到pam，安装上也没啥影响，如果不想安装pam请自行测试）

```
[root@linux-node3 ~]``# yum install -y pam* zlib*
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 ``* base: mirrors.163.com
 ``* epel: mirrors.aliyun.com
 ``* extras: mirrors.cn99.com
 ``* updates: mirrors.cn99.com
Package pam_yubico-2.26-1.el7.x86_64 already installed and latest version
Package pam_script-1.1.8-1.el7.x86_64 already installed and latest version
Package pam_oath-2.4.1-9.el7.x86_64 already installed and latest version
Package pam_snapper-0.2.8-4.el7.x86_64 already installed and latest version
Package pam_ssh_agent_auth-0.10.3-2.16.el7.x86_64 already installed and latest version
Package pam_2fa-1.0-1.el7.x86_64 already installed and latest version
Package pam_mapi-0.3.4-1.el7.x86_64 already installed and latest version
Package pam_ssh_user_auth-1.0-1.el7.x86_64 already installed and latest version
Package pam_mount-2.16-5.el7.x86_64 already installed and latest version
Package pam_radius-1.4.0-3.el7.x86_64 already installed and latest version
Package pamtester-0.1.2-4.el7.x86_64 already installed and latest version
Package pam_afs_session-2.6-5.el7.x86_64 already installed and latest version
Package pam_pkcs11-0.6.2-30.el7.x86_64 already installed and latest version
Package pam-1.1.8-22.el7.x86_64 already installed and latest version
Package pam_ssh-2.3-1.el7.x86_64 already installed and latest version
Package 1:pam_url-0.3.3-4.el7.x86_64 already installed and latest version
Package pam_wrapper-1.0.7-2.el7.x86_64 already installed and latest version
Package pam-kwallet-5.5.2-1.el7.x86_64 already installed and latest version
Package pam-devel-1.1.8-22.el7.x86_64 already installed and latest version
Package pam_krb5-2.4.8-6.el7.x86_64 already installed and latest version
Package zlib-devel-1.2.7-18.el7.x86_64 already installed and latest version
Package zlib-static-1.2.7-18.el7.x86_64 already installed and latest version
Package zlib-1.2.7-18.el7.x86_64 already installed and latest version
Package zlib-ada-1.4-0.5.20120830CVS.el7.x86_64 already installed and latest version
Package zlib-ada-devel-1.4-0.5.20120830CVS.el7.x86_64 already installed and latest version
Nothing to ``do
[root@linux-node3 ~]``#
```

**下载openssh包和openssl的包**

 我们都下载最新版本，下载箭头指的包

https://openbsd.hk/pub/OpenBSD/OpenSSH/portable/

**开始安装openssl**

```
[root@linux-node3 ~]# wget https://github.com/openssl/openssl/archive/refs/tags/OpenSSL_1_1_1l.tar.gz

解压文件
[root@linux-node3 /data/tools]# tar xfz OpenSSL_1_1_1l.tar.gz
[root@linux-node3 /data/tools]# ll
total 5228
drwxr-xr-x 20 root root    4096 Apr 27 12:20 openssl-OpenSSL_1_1_1l
-rw-r--r--  1 root root 5348369 Apr 27 12:19 OpenSSL_1_1_1l
[root@linux-node3 /data/tools]# cd
[root@linux-node3 ~]#
 
 
现在是系统默认的版本，等会升级完毕对比下
[root@linux-node3 ~]# openssl version
OpenSSL 1.0.2k-fips  26 Jan 2017
[root@linux-node3 ~]#
```

 备份下面2个文件或目录（如果存在的话就执行）

```
[root@linux-node3 ~]# ll /usr/bin/openssl
-rwxr-xr-x 1 root root 555248 Mar 12 18:12 /usr/bin/openssl
[root@linux-node3 ~]# mv /usr/bin/openssl /usr/bin/openssl_bak
 
[root@linux-node3 ~]# ll /usr/include/openssl
total 1864
-rw-r--r-- 1 root root   6146 Mar 12 18:12 aes.h
-rw-r--r-- 1 root root  63204 Mar 12 18:12 asn1.h
-rw-r--r-- 1 root root  24435 Mar 12 18:12 asn1_mac.h
-rw-r--r-- 1 root root  34475 Mar 12 18:12 asn1t.h
-rw-r--r-- 1 root root  38742 Mar 12 18:12 bio.h
-rw-r--r-- 1 root root   5351 Mar 12 18:12 blowfish.h
......
 
[root@linux-node3 ~]# mv /usr/include/openssl /usr/include/openssl_bak
[root@linux-node3 ~]#
```

**编译安装新版本的openssl**

配置、编译、安装3个命令一起执行

&&符号表示前面的执行成功才会执行后面的

```
[root@linux-node3 ~]# cd /data/tools/openssl-1.0.2r/
 
[root@linux-node3 /data/tools/openssl-1.0.2r]# ./config shared --prefix=/usr/local/openssl && make && make install
```

以上命令执行完毕，echo $?查看下最后的make install是否有报错，0表示没有问题





下面2个文件或者目录做软链接 （刚才前面的步骤mv备份过原来的）

```
[root@linux-node3 ~]# ln -s /usr/local/openssl/bin/openssl /usr/bin/openssl
[root@linux-node3 ~]# ln -s /usr/local/openssl/include /usr/include/openssl
[root@linux-node3 ~]# ll /usr/bin/openssl
lrwxrwxrwx 1 root root 26 Apr 27 12:31 /usr/bin/openssl -> /usr/local/bin/openssl
[root@linux-node3 ~]# ll /usr/include/openssl -ld
lrwxrwxrwx 1 root root 30 Apr 27 12:31 /usr/include/openssl -> /usr/local/include/openssl
[root@linux-node3 ~]#
```

命令行执行下面2个命令加载新配置

```
echo "/usr/local/openssl/lib6/" >> /etc/ld.so.conf
/sbin/ldconfig
```

  查看确认版本。没问题

```
[root@testssh ~]# openssl version
OpenSSL 1.0.2r 26 Feb 2019
```

## 27.5、 **安装openssh** 

上传openssh的tar包并解压

```
[root@testssh ~]# cd /data/tools/
[root@testssh tools]# ll
total 7628
-rw-r--r-- 1 root root 1597697 Apr 18 07:02 openssh-8.0p1.``tar``.gz
drwxr-xr-x 20 root root  4096 Apr 23 23:12 openssl-1.0.2r
-rw-r--r-- 1 root root 5348369 Feb 26 22:34 openssl-1.0.2r.``tar``.gz
-rwxr-xr-x 1 root root 853040 Apr 11 2018 sshd
[root@testssh tools]# tar xfz openssh-8.0p1.tar.gz
[root@testssh tools]# cd openssh-8.0p1
可能文件默认显示uid和gid数组都是1000，这里重新授权下。不授权可能也不影响安装（请自行测试）
[root@testssh tools]# chown -R root.root /data/tools/openssh-8.0p1
```

 

 

命令行删除原先ssh的配置文件和目录

然后配置、编译、安装

注意下面编译安装的命令是一行，请把第一行末尾的 \ 去掉，然后在文本里弄成一行之后放命令行执行

```
[root@testssh tools]# rm -rf /etc/ssh/*
[root@testssh tools]# ./configure --prefix=/usr/local/openssh --sysconfdir=/etc/ssh --with-pam --with-ssl-dir=/usr/local/openssl --with-md5-passwords --mandir=/usr/share/man --with-zlib=/usr/local/zlib --without-hardening
[root@testssh tools]# make && make install
```

### 修改配置文件

```
vim /etc/ssh/sshd_config
查找#PermitRootLogin prohibit-password 改成 PermitRootLogin yes 并取消注释
同时如果端口非22端口，需要更改为对应端口

#关闭selinux
sed -i.bak 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
setenforce 0
```

### 重启OpenSSH

```
#添加到自启动
systemctl enable sshd

nohup systemctl restart sshd
```

测试没问题后可以把telnet服务关闭了

```
[root@linux-node3 ~]# systemctl disable xinetd.service
Removed symlink /etc/systemd/system/multi-user.target.wants/xinetd.service.
[root@linux-node3 ~]# systemctl stop xinetd.service
[root@linux-node3 ~]# systemctl disable telnet.socket
[root@linux-node3 ~]# systemctl stop telnet.socket
[root@linux-node3 ~]# netstat -lntp
```

