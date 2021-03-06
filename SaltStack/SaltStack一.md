---
title: sockstack
date: 2016-06-26 17:43:18
tags:
---
# 运维自动化之SaltStack

## 简介

SaltStack是用python语言写的，提供了API、支持多种操作系统（所有类Unix系统都默认安装Python），windows只能安装Minion端程序。



[官网](https://saltstack.com/)

[中国SaltStack用户组](http://www.saltstack.cn)

### SaltStack 三大功能：

* 远程执行
* 配置管理（状态、很难回滚）
* 云管理

> 运维三板斧：监控、执行、配置管理

类似软件：Puppet(ruby)、Ansible（python）

<!--more-->

### SaltStack 四种运行方式

* Local
* Master/Minion（传统C/S架构）
* Syndic（对应于zabbix的proxy)
* Salt SSH

> 由于C/S模式需要在每台客户端机器上安装Salt-Minion,很多人会觉得麻烦,但是最佳的体验就是装一个Minion。

### 典型案例

阿里大数据部门、360的远程执行

## QUICK START

### 安装

1. 通过epel源安装
2. 通过saltstack自己的仓库安装
<pre>
yum install https://repo.saltstack.com/yum/redhat/salt-repo-latest-1.el7.noarch.rpm -y
</pre>

####  Master端
<pre>
yum install salt-master salt-minion -y
</pre>

#### Minion端
<pre>
yum install salt-minion -y
</pre>

####  windows下Minion安装
<pre>
Salt-Minion-2016.3.1-AMD64-Setup.exe /S /master=yoursaltmaster /minion-name=yourminionname
</pre>

### 配置及启动

#### 启动salt-master
<pre>
systemctl start salt-master
</pre>

#### 配置并启动minion
<pre>
sed -i 's/# master: salt/master: 192.168.56.11/' /etc/salt/minion
systemctl start salt-minion
</pre>
另外一个重要的参数就是id,每个minion都有一个单独的ID，它也放在 	`/etc/salt` 目录下，如果不改的话，默认就是主机名
<pre>
[root@linux-node2 salt]# cat minion_id 
linux-node2.example.com
</pre>

> 这个ID不建议改，如果要改的话，要先把这个文件删除了。因为master会首先读这个文件，生产中可以用主机名，id可以不配，如果不配，它就会用主机名。

### 认证
SaltStack是通过AES来加密的，所以一般不需要关注安全性问题，一般Minion启动的时候就会在/etc/salt下建立一个pki的目录，生成密钥对并把里面的公钥发给Master端。
<pre>
[root@linux-node2 salt]# tree pki
pki
├── master
└── minion
    ├── minion.pem		私钥
    └── minion.pub		公钥
[root@linux-node1 salt]# tree pki/
pki/
├── master
│   ├── master.pem		Master私钥
│   ├── master.pub		Master公钥
│   ├── minions
│   ├── minions_autosign
│   ├── minions_denied
│   ├── minions_pre
│   │   ├── linux-node1.example.com		Minion发过来的两个私钥
│   │   └── linux-node2.example.com		默认会使用Minion_ID来做私钥的名称
│   └── minions_rejected
└── minion
    ├── minion.pem		Minion私钥
    └── minion.pub		Minion公钥
[root@linux-node1 salt]# md5sum pki/master/minions_pre/linux-node2.example.com 
eed44ea7c9d65c7aeddd56dedcebd3df  pki/master/minions_pre/linux-node2.example.com
[root@linux-node2 salt]# md5sum pki/minion/minion.pub 
eed44ea7c9d65c7aeddd56dedcebd3df  pki/minion/minion.pub
</pre>

#### 列出所有keys
<pre>
[root@linux-node1 salt]# salt-key 
Accepted Keys:
Denied Keys:
Unaccepted Keys:
linux-node1.example.com
linux-node2.example.com
Rejected Keys:
</pre>
#### Master端同意所有申请管理的Minion端keys
<pre>
salt-key -a linux-node1.example.com
salt-key -a linux-node*
salt-key -A
[root@linux-node1 salt]# tree pki/
pki/
├── master
│   ├── master.pem
│   ├── master.pub
│   ├── minions
│   │   ├── linux-node1.example.com		## Master同意
│   │   └── linux-node2.example.com
│   ├── minions_autosign
│   ├── minions_denied
│   ├── minions_pre
│   └── minions_rejected
└── minion
    ├── minion_master.pub		## 这个是Master的公钥（认证后Master端会把自己的公钥发给Minion端）
    ├── minion.pem
    └── minion.pub
</pre>

> 认证的过程就是公钥交换的过程，改完ID后所有的认证就得重新来做一遍

### ID的设置

ID设置有两种方式，一种是主机名，另外一种是通过IP地址，这时就需要看业务了，如果业务不确定就使用IP地址，如果业务确定就用主机名（idc01-bj-product-node1.shop.com）,DNS解析主机名不支持下滑线

### 远程执行
	salt '*' test.ping		## 引起来是因为 * 在shell下也是有意义的

> test是一个模块，ping是这个模块下面的一个方法，它是python的标准；这个ping不是ICMP的ping，它是saltmaster和minion之间的一个通信，用的也不是ICMP的协议

	salt "linux-node1.example.com" cmd.run 'w'		## 命令一般也要引起来，方便传参
操作实例：

	[root@linux-node1 salt]# salt '*' cmd.run 'mkdir hehe'
	linux-node2.example.com:
	linux-node1.example.com:
	[root@linux-node1 salt]# salt '*' cmd.run 'ls -l'
	linux-node2.example.com:
	    total 4
	    -rw-------. 1 root root 1175 May 20 06:37 anaconda-ks.cfg
	    drwxr-xr-x  2 root root    6 Jun 25 21:01 hehe
	linux-node1.example.com:
	    total 4
	    -rw-------. 1 root root 1175 May 20 06:37 anaconda-ks.cfg
	    drwxr-xr-x  2 root root    6 Jun 25 21:01 hehe

后面可以通过ACL来控制权限，它很危险，可以执行删除操作

### salt命令用法

查询 `status.all_status` 模块函数的使用方法

	[root@linux-node1 salt]# salt 'linux-node1.example.com' sys.doc status.all_status
	status.all_status:
	
	    Return a composite of all status data and info for this minion.
	    Warning: There is a LOT here!
	
	    CLI Example:
	
	        salt '*' status.all_status

查看test模块的用法

	salt '*' sys.doc test
	test.ping:
	
	    Used to make sure the minion is up and responding. Not an ICMP ping.
	
	    Returns ``True``.
	
	    CLI Example:
	
	        salt '*' test.ping

[其他用法](http://arlen.blog.51cto.com/7175583/1424216)


### 状态模块 

Salt通过 **状态模块** 来识别状态，所以需要写一个关于状态的文件，它是一个YAML格式的文件，并且文件名后缀以 `.sls` 结尾。

[SaltStack官方镜像文档](https://www.unixhot.com/docs/saltstack/index.html)

#### 理解YAML

> YAML是"YAML Ain't a Markup Language"（YAML不是一种置标语言）的递归缩写。它是类似于标准通用标记语言的子集XML的数据描述语言，语法比XML简单很多。因为它简单。
> 

##### 三个规则

1. 缩进（代表层级关系，2个空格，并且不能使用TAB键，整个saltstack里面都不能用TAB键）
2. 冒号（1. 跟缩进一起代理层级目录（以冒号结尾不用空格）；2. 分隔键值对[key: value],支持嵌套，冒号后面必须有一个空格）
3. 短横线（它是一个列表，`-` 后面必须有空格）

> saltstack配置文件也是YAML语法。

#### 写一个状态文件

编辑Master配置文件

`/etc/salt/master`

它是一个环境的定义，可以定义不同的路径，把不同业务放在不同的路径下，

	file_roots:			配置项
	  base:				配置base环境
	    - /srv/salt		可以写多个，它是个列表,base环境的根路径 

> salt使用一个写进ZeroMQ的轻量文件服务把文件分发到Minion端，这个文件服务直接集成到master的后端，不需要专门的端口；

> 文件服务工作在传递给Master端的各种环境上，每一个环境可以有多个根目录(root directories)，一个基础环境需要放置 `top` 文件 
> base环境默认必须有，并且不能修改名字

重启master

	systemctl restart salt-master

创建目录

	mkdir /srv/salt -p
	cd /srv/salt
	mkdir web
	cd web
	cat >> apache.sls <<EOF
	apache-install:		## 每一个ID就是一个配置项
	  pkg.installed:	## 这里面的模块可以是内置的状态模块，也可以是自定义的状态模块
	    - names:
	        - httpd
	        - httpd-devel
	
	apache-service:
	  service.running:
	    - name: httpd
	    - enable: True
	EOF
> 一个Salt状态可以使用单个的SLS文件（一个SLS文件就定义了一个状态），或者使用一个文件夹。后者更加灵活方便。

> apache-install是定义的ID，pkg是一个状态模块，模块分为执行模块和状态模块，installed是模块中定义的函数方法

> salt有很多的模块，它们都放在 `/usr/lib/python2.7/site-packages/salt/modules` 下，而 `pkg.py` 是放在 `/usr/lib/python2.7/site-packages/salt/states` 目录下的，所以说它是一个状态模块；

执行状态模块
<pre>
salt '*' state.sls web.apache
</pre>

> `state` 是一个模块（/usr/lib/python2.7/site-packages/salt/modules/state.py），sls是它的一个方法，它的作用就是 `Execute the states in one or more SLS files` ，这些状态文件是放在环境下的，并且这些状态文件里面会调用state的状态模块（/usr/lib/python2.7/site-packages/salt/states）。

> /var/cache/salt/minion是一个很重要的目录，一般master把文件发给minion并放在这个目录下,minion从上往下加载

#### 遇到的错误

<pre>
Salt request timed out. The master is not responding. If this error persists after verifying the master is up, worker_threads may need to be increased.
</pre>

#### 执行小技巧

> 已经有的是绿色，新完成的是浅绿色

#### 通过 `top.sls`来有选择地执行状态文件 

top文件也是sls结尾，也是YAML格式。它放在base环境下，也就是这里的/srv/salt下
<pre>
# The state system uses a "top" file to tell the minions what environment to
# use and what modules to use. The state_top file is defined relative to the
# root of the base environment as defined in "File Server settings" below.
# state_top: top.sls
</pre>

	cat > top.sls <<EOF
	base:
	  'linux-node1.example.com':
	    - web.apache
	  'linux-node2.example.com':
	    - web.apache
	EOF

##### 执行高级状态

	[root@linux-node1 salt]# salt '*' state.highstate
	linux-node2.example.com:
	----------
	          ID: apache-install
	    Function: pkg.installed
	        Name: httpd
	      Result: True
	     Comment: Package httpd is already installed
	     Started: 23:26:29.845158
	    Duration: 605.676 ms
	     Changes:   
	----------
	          ID: apache-install
	    Function: pkg.installed
	        Name: httpd-devel
	      Result: True
	     Comment: Package httpd-devel is already installed
	     Started: 23:26:30.450979
	    Duration: 0.433 ms
	     Changes:   
	----------
	          ID: apache-service
	    Function: service.running
	        Name: httpd
	      Result: True
	     Comment: The service httpd is already running
	     Started: 23:26:30.451937
	    Duration: 27.567 ms
	     Changes:   
	
	Summary for linux-node2.example.com
	------------
	Succeeded: 3
	Failed:    0
	------------
	Total states run:     3
	linux-node1.example.com:
	----------
	          ID: apache-install
	    Function: pkg.installed
	        Name: httpd
	      Result: True
	     Comment: Package httpd is already installed
	     Started: 23:26:29.999055
	    Duration: 608.856 ms
	     Changes:   
	----------
	          ID: apache-install
	    Function: pkg.installed
	        Name: httpd-devel
	      Result: True
	     Comment: Package httpd-devel is already installed
	     Started: 23:26:30.608080
	    Duration: 0.51 ms
	     Changes:   
	----------
	          ID: apache-service
	    Function: service.running
	        Name: httpd
	      Result: True
	     Comment: The service httpd is already running
	     Started: 23:26:30.609121
	    Duration: 35.008 ms
	     Changes:   
	
	Summary for linux-node1.example.com
	------------
	Succeeded: 3
	Failed:    0
	------------
	Total states run:     3

> 生产中不建议用* ,以后不能这样写命令，要先 `salt '*' state.highstate test=True` 


## SaltStack与ZeroMQ
 我们进行自动化运维大多数情况下，是我们的服务器数量已经远远超过人工SSH维护的范围，SaltStack可以支持数以千计，甚至更多的服务器。这些性能的提供主要来自于ZeroMQ，因为SaltStack底层是基于ZeroMQ进行高效的网络通信。ZMQ用于node与node间的通信，node可以是主机也可以是进程。
### ZeroMQ简介
  ZeroMQ（我们通常还会用ØMQ , 0MQ, zmq等来表示）是一个简单好用的传输层，像框架一样的一个套接字库，他使得Socket编程更加简单、简洁和性能更高。它还是一个消息处理队列库，可在多个线程、内核和主机盒之间弹性伸缩。
发布与订阅 ZeroMQ支持Publish/Subscribe，即发布与订阅模式，我们经常简称Pub/Sub。

![](https://www.unixhot.com/uploads/article/20151027/055f24981e25860e08942d7f0aa9d0ab.png)

#### 发布与订阅模式（Publish/Subscribe）简称Pub/Sub

所有的minion都会连到4505端口，而且是TCP的长连接

	salt '*' cmd.run 'w'

#### 请求与响应模式

默认监听4505端口，返回结果的时候通过4506端口，

#### 显示进程标题

<pre>
yum install -y python-setproctitle
systemctl restart salt-master
ps -ef|grep salt-mast
</pre>

[相关文章介绍](https://www.unixhot.com/article/15)

## SaltStack的数据系统

[SaltStack整体架构](http://www.cnblogs.com/alexyang8/p/3445333.html)

saltstack有两种数据系统：

* Grains（谷粒）
* Pillar（柱子）

### Grains

Grains是静态数据，它是在Minion启动的时候收集的Minion本地的相关信息，如：操作系统版本，内核版本，ＣＰＵ，内存，硬盘，设备型号，机器序列号。它可以做资产管理，只要不重启它，它就会只收集一次，当重启的时候才会再次收集，启动完后就不会变了,它是一个key/value的东西。

**作用**：

* 资产管理、信息查询
* 用于目标选择（不同于ID的另外目标定义方法，操作系统等）
* 配置管理中使用

#### 信息查询

查看所有信息
<pre>
salt 'linux-node1.example.com' grains.ls
salt 'linux-node1.example.com' grains.items
salt '*' grains.item os
salt '*' grains.item fqdn_ip4
</pre>

#### 目标选择

-G 参数

<pre>
salt -G 'os:CentOS' test.ping
salt -G 'os:CentOS' cmd.run 'echo hehe'
</pre>

通过写配置文件给某一个minion自定义一个grains

##### 写配置文件
<pre>
vim /etc/salt/minion
grains:
  roles: apache 
</pre>

	systemctl restart salt-minion

<pre> 
[root@linux-node1 salt]# salt '*' grains.item roles
linux-node2.example.com:
    ----------
    rolesode1.example.com:
    ----------
    roles:
[root@linux-node1 salt]# salt '*' grains.item roles
linux-node1.example.com:
    ----------
    roles:
linux-node2.example.com:
    ----------
    roles:
        apache:
</pre>

#### 重启所有apache

	salt -G 'roles:apache' cmd.run 'systemctl restart httpd'

> 生产不建议放在minion配置文件里面，写在 `/etc/salt/grains` 里面，minion会自动来这找；并且上面这条命令中的roles:后面是没有空格的

	vim /etc/salt/grains
	cloud: openstack

	salt '*' grains.item cloud

> 加完之后必须重启，因为它是静态的。但不重启也有一个刷新的命令 `salt '*' saltutil.sync_grains` 无论上面两种方法写在哪都可以成功。

### top file中使用grains案例
grains还可以用到top.sls文件里面

<pre>
base:
  'linux-node1.example.com':
    - web.apache
  'roles:apache': 
    - match: grain 
    - web.apache
</pre>


### 配置管理的案例

	vim /srv/salt/web/apache.sls


> 可以自己用python脚本来写一个grains，实现动态，这里说的动态是通过逻辑后产生的

#### 用Python开发一个grains

写一个python脚本返回一个字典就可以了。

* 放哪儿

<pre>
cd /srv/salt/
mkdir _grains
cd _grains
</pre>
* 写脚本

<pre>
vim my_grains.py
#!/usr/bin/env python
#-*- coding: utf-8 -*-

def my_grains():
    # 初始化一个grains字典
    grains = {}
    # 设置字典中的key/value
    grains['iaas'] = 'openstack'
    grains['edu'] = 'example'
    # 返回这个字典
    return grains
</pre>

* 把grains推送给minion
<pre>
salt '*' saltutil.sync_grains
</pre>

> 它会放在/var/cache/salt/minion/extmods/grains/my_grains.py

* 查看grains条目

<pre>
salt '*' grains.item iaas
</pre>

#### Grains优先级：

1. 系统自带
2. grains文件写的
3. minion配置文件写的
4. 自己写的



### Pillar

#### 简介

它也是数据系统，也是定义一个key/value，但是pillar数据是动态的，和minion启不启动没关系，它会给特定的minion指定特定的数据，跟top file很像。只有指定的minion自己能看到自己的数据。

#### 使用
* 查看pillar条目

<pre>
salt '*' pillar.items
</pre>

* 在master端开启pillar功能

修改配置文件/etc/salt/master并重启master服务 

	pillar_opts: True
	pillar_roots:
	  base:
	    - /srv/pillar
	systemctl restart salt-master

* 编辑相关配置文件

编辑pillar的SLS文件
<pre>
mkdir /srv/pillar
cd /srv/pillar
mkdir web
cd web
vim apache.sls
{% if grains['os'] == 'CentOS' %}
apache: httpd
{% elif grains['os'] == 'Debian' %}
apache: apache2
{% endif %}
</pre>

编辑topfile(pillar必须要写topfile,不像配置管理不用也可以)

<pre>
base:
  'linux-node2.oldboyedu.com':
    - web.apache
salt '*' saltutil.refresh_pillar
salt '*' pillar.items apache
</pre>
效果如下：
<pre>
[root@linux-node1 web]# salt '*' pillar.items apache
linux-node1.oldboyedu.com:
    ----------
    apache:
linux-node2.oldboyedu.com:
    ----------
    apache:
        httpd
linux-node2.oldboyedu.com:
    ----------
    apache:
</pre>

* 修改上面的文件并熟悉一下层级的概念
<pre>
vim apache.sls
hehe:
  {% if grains['os'] == 'CentOS' %}
  apache: httpd
  {% elif grains['os'] == 'Debian' %}
  apache: apache2
  {% endif %}
salt '*' saltutil.refresh_pillar
salt '*' pillar.items hehe

#### 个人理解

topfile.sls规定哪个客户端经过哪个pillar判断文件的扫描，而web.apache就是那个判断文件的具体逻辑实现。

#### 使用场景

目标选择

	salt -I 'apache:httpd' test.ping

### Grains VS Pillar

grians的items（roles）下可以是一个值，也可以是多个值，但是pillar的items下面还可以有多个items，它是一个嵌套的东西 ，它是一个真正的python字典的格式。

Grains | 类型 | 数据采集方式	|应用场景|定义位置|
---- |----| --- | --- | --- | 
Grains|	静态	|minion启动时收集	|数据查询、目标选择、配置管理|	minion端
Pillar| 动态 | master自定义 | 目标选择、配置管理、敏感数据 | 存储	master端

> 尽管看上去差不多，但是它们是有本质区别的，pillar数据是存储在master端，缓存在minion端的，而grains数据是存储在minion端，缓存在master端。

相关文章：

[区别](http://ohmystack.com/articles/salt-2-grains-and-pillar/)

[stackoverflow](http://stackoverflow.com/questions/13115700/salt-stack-grains-vs-pillars)

总体理解：

grains和pillar的目的都是为了给一个定义一个客户端，而grains会定义一些比较基础的属性，因为它是静态的，给每个客户端定义所有的属性没必要也不现实；而pillar可以根据需求来给每一个客户端定义不同的属性，而这个前提就是它首先能通过minion id和grians找到它要定义的那个客户端;所以pillar的设计就在于它要补充grains的不足性，其次才是关于静态、保密的那些常见区别。

| Differences                  | Grains                        | Pillars                             |
|------------------------------|-------------------------------|-------------------------------------|
| This is info which...        | ... Minion knows about itself | ... Minion asks Master about        |
|                              |                               |                                     |
| Distributed:                 | Yes (different per minion)    | No (single version per master)      |
| Centralized:                 | No                            | Yes                                 |
|                              |                               |                                     |
| Computed automatically:      | Yes (preset/computed value)   | No (only rendered from Jinja/YAML)  |
| Assigned manually:           | No (too elaborate)            | Yes (Jinja/YAML sources)            |
|                              |                               |                                     |
| Conceptually intrinsic to... | ... individual Minion node    | ... entire system managed by Master |
| Data under revision control: | No (computed values)          | Yes (Jinja/YAML sources)            |
|                              |                               |                                     |
| They define rather...        | _provided_ resources          | _required_ resources                |
|                              | (e.g. minion OS version)      | (e.g. packages to install)          |
|                              |                               |                                     |

## 远程执行

深入学习SaltStack远程执行

	salt '*' cmd.run 'w'

命令：salt

目标： '*'（好多种方式指定）

模块：cmd.run   自带150+模块。可以自己写模块

返回：可以写入数据库里面，执行后结果返回，通过Returnners组件

### 目标Targeting

你要指定哪个或者哪些minion来执行后面的东西 ，怎么来定位？

两种定位的方法：
1. 和minion id有关的
2. 和Minion_id无关的

#### 与minion_id有关的

* Minion id
	
		linux-node1.example.com

* 通配符
		
		*/linux-node*/linux-node[1|2].example.com/linux-node?.example.com)

* 列表
		 
		salt -L 'linux-node1.example.com,linux-node2.example.com' test.ping
* 正则表达式
		
		salt -E 'linux-(node1|node2)*' test.ping|salt -E 'linux-(node1|node2).example.com' test.ping）

> 所有匹配目标的方式都可以用在top file里面来指定目标 


#### 与minion_id无关的

主机名设置方案：

1. IP地址
2. 根据业务来进行设置

<pre>
redis-node1-redis03-idc03-soa.example.com
</pre>

redis-node1  redis第一个节点

redis04   集群

idc04   机房

soa	业务线

  
##### 子网和IP地址
<pre>
salt -S 192.168.56.11 test.ping
salt -S 192.168.56.0/24 test.ping
</pre>

##### NODE GROUPS

<pre>
vim /etc/salt/master
/nodegroup

nodegroups:
  web: 'L@linux-node1.example.com,linux-node2.example.com'
systemctl restart salt-master
salt -N web test.ping
</pre>

##### 混合匹配 

[官方文档](https://www.unixhot.com/docs/saltstack/topics/targeting/compound.html)

	salt -C '* and not web-dc1-srv' test.ping

##### 批处理

可以通过百分比来执行

### 模块

saltstack内置了丰富的模块来执行,每一个模块都是一个python文件，挑几个来学习

#### NETWORK

	salt '*' network.active_tcp
	salt '*' network.arp


#### SERVICE

	salt '*' service.available sshd
	salt '*' service.get_all

#### CP

	salt-cp '*' /etc/hosts /tmp/hehe

#### STATE

	salt '*' state.show_top
	salt '*' state.single pkg.installed name=lsof

### 返回程序

salt使用返回程序来实现把返回结果写到数据库里面(returnners)
#### SALT.RETURNERS.MYSQL

> Return data to mysql server

这个返回数据是minion直接返回的(直接返回数据到数据库，而不经过master端，所有的minion要装python的MySQL 库

这里我们可以使用salt来安装

<pre>
salt '*' state.single pkg.installed name=MySQL-python
yum install mariadb-server mariadb -y
systemctl start mariadb
</pre>

<pre>
mysql
CREATE DATABASE  `salt`
  DEFAULT CHARACTER SET utf8
  DEFAULT COLLATE utf8_general_ci;
USE `salt`;
CREATE TABLE `jids` (
  `jid` varchar(255) NOT NULL,
  `load` mediumtext NOT NULL,
  UNIQUE KEY `jid` (`jid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
CREATE INDEX jid ON jids(jid) USING BTREE;
CREATE TABLE `salt_returns` (
  `fun` varchar(50) NOT NULL,
  `jid` varchar(255) NOT NULL,
  `return` mediumtext NOT NULL,
  `id` varchar(255) NOT NULL,
  `success` varchar(10) NOT NULL,
  `full_ret` mediumtext NOT NULL,
  `alter_time` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  KEY `id` (`id`),
  KEY `jid` (`jid`),
  KEY `fun` (`fun`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
CREATE TABLE `salt_events` (
`id` BIGINT NOT NULL AUTO_INCREMENT,
`tag` varchar(255) NOT NULL,
`data` mediumtext NOT NULL,
`alter_time` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
`master_id` varchar(255) NOT NULL,
PRIMARY KEY (`id`),
KEY `tag` (`tag`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
grant all on salt.* to salt@'%' identified by 'salt@pw';
flush privileges;
mysql -h 192.168.56.11 -usalt -psalt@pw
</pre>

修改minion配置文件
<pre>
vim /etc/salt/minion
mysql.host: '192.168.56.11'
mysql.user: 'salt'
mysql.pass: 'salt@pw'
mysql.db: 'salt'
mysql.port: 3306
systemctl restart salt-minion
</pre>

执行结果：
<pre>
[root@linux-node1 salt]# salt '*' test.ping --return mysql
linux-node2.example.com:
    True
linux-node1.example.com:
    True
[root@linux-node1 salt]# mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 8
Server version: 5.5.47-MariaDB MariaDB Server

Copyright (c) 2000, 2015, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> use salt;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [salt]> show tables;
+----------------+
| Tables_in_salt |
+----------------+
| jids           |
| salt_events    |
| salt_returns   |
+----------------+
3 rows in set (0.00 sec)

MariaDB [salt]> select * from salt_returns;
+-----------+----------------------+--------+---------------------------+---------+-------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------+
| fun       | jid                  | return | id                        | success | full_ret                                                                                                                                              | alter_time          |
+-----------+----------------------+--------+---------------------------+---------+-------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------+
| test.ping | 20160626025439671711 | true   | linux-node2.example.com | 1       | {"fun_args": [], "jid": "20160626025439671711", "return": true, "retcode": 0, "success": true, "fun": "test.ping", "id": "linux-node2.example.com"} | 2016-06-26 02:54:39 |
| test.ping | 20160626025439671711 | true   | linux-node1.example.com | 1       | {"fun_args": [], "jid": "20160626025439671711", "return": true, "retcode": 0, "success": true, "fun": "test.ping", "id": "linux-node1.example.com"} | 2016-06-26 02:54:39 |
+-----------+----------------------+--------+---------------------------+---------+-------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------+
2 rows in set (0.00 sec)
</pre>

### 编写一个执行模块

	vim /usr/lib/python2.7/site-packages/salt/modules/service.py

编写模块：

1. 放哪（cd /srv/salt/_modules）
2. 命名（文件名就是模块名，如my_disk.py）

<pre>
[root@linux-node1 _modules]# cat my_disk.py 
#!/usr/bin/env python

def list():
  cmd = 'df -h'
  ret = __salt__['cmd.run'](cmd)
  return ret
</pre>

刷新

	salt '*' saltutil.sync_modules

<pre>
[root@linux-node2 ~]# tree /var/cache/salt/minion/
/var/cache/salt/minion/
├── accumulator
├── extmods
│   ├── grains
│   │   ├── my_grains.py
│   │   └── my_grains.pyc
│   └── modules
│       └── my_disk.py
├── files
│   └── base
│       ├── _grains
│       │   └── my_grains.py
│       ├── _modules
│       │   └── my_disk.py
│       ├── top.sls
│       └── web
│           └── apache.sls
├── highstate.cache.p
├── module_refresh
├── pkg_refresh
├── proc
└── sls.p

[root@linux-node1 _modules]# salt '*' my_disk.list
linux-node2.example.com:
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/sda3       196G  2.1G  194G   2% /
    devtmpfs        480M     0  480M   0% /dev
    tmpfs           489M   12K  489M   1% /dev/shm
    tmpfs           489M  6.7M  483M   2% /run
    tmpfs           489M     0  489M   0% /sys/fs/cgroup
    /dev/sda1       497M  128M  370M  26% /boot
    tmpfs            98M     0   98M   0% /run/user/0
linux-node1.example.com:
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/sda3       196G  2.3G  194G   2% /
    devtmpfs        480M     0  480M   0% /dev
    tmpfs           489M   40K  489M   1% /dev/shm
    tmpfs           489M  6.8M  483M   2% /run
    tmpfs           489M     0  489M   0% /sys/fs/cgroup
    /dev/sda1       497M  128M  370M  26% /boot
    tmpfs            98M     0   98M   0% /run/user/0
</pre>

<div style="box-shadw:0px 0px 3px #000;" markdown="1" >
		<img src="https://github.com/Aresona/edu-docs/blob/master/image/touxiang.jpg" />
</div>


作业
预习配置管理

https://github.com/unixhot