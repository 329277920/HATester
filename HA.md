# 高可用与负载均衡 #
>[一、概述](1#)<br>
>[二、keepalived](2#)<br>
>&nbsp;&nbsp;[1、概述](2.1#)<br>
>&nbsp;&nbsp;[2、安装](2.2#)<br>
>[三、nginx](2#)<br>
>&nbsp;&nbsp;[1、概述](3.1#)<br> 
>&nbsp;&nbsp;[2、安装](3.2#)<br>
>&nbsp;&nbsp;[3、vrrp_script](3.3#)<br>
>[附录](10#)<br>
>&nbsp;&nbsp;[1、VRRP](10.1#)<br>

<h2 id="1">一、概述</h2>
<h3 id="1.1">1、高可用</h3>


<h2 id="2">二、keepalived</h2>
<h3 id="2.1">1、概述</h3>
keepalived是一个由c语言写的免费的路由软件，它基于linux平台的提供高可用和负载均衡设施。它工作在4层（传输层）。基于 [RAAP](https://baike.baidu.com/item/%E8%99%9A%E6%8B%9F%E8%B7%AF%E7%94%B1%E5%99%A8%E5%86%97%E4%BD%99%E5%8D%8F%E8%AE%AE/2991482?fr=aladdin&fromid=932628&fromtitle=VRRP) 协议实现高可用。

<h3 id="2.2">2、安装</h3>
发布地址: [http://www.keepalived.org/download.html](http://www.keepalived.org/download.html)<br>
kp版本: [v2.0.7](http://www.keepalived.org/software/keepalived-2.0.7.tar.gz)<br>
安装依赖:[https://github.com/acassen/keepalived/blob/master/INSTALL](https://github.com/acassen/keepalived/blob/master/INSTALL)<br>
环境: centos7<br>
ip: 192.168.100.100（虚IP），192.168.100.128（节点1主），192.168.100.129（节点2从）<br>
ip: 192.168.100.101（虚IP），192.168.100.129（节点1主），192.168.100.128（节点2从）<br>

**上述使用了2个虚拟IP，实现keepalived的双主模式，客户端可以依据规则连接上任意一个虚拟IP。**

**步骤一**<br>
在128、129节点上分别安装 keepalived v2.0.7

    
	yum install -y gcc   --安装c编译器
    yum -y install openssl-devel

	mkdir -p /apps/keepalived
	cd /apps/keepalived
	wget http://www.keepalived.org/software/keepalived-2.0.7.tar.gz
	tar zxvf keepalived-2.0.7.tar.gz 

    cd keepalived-2.0.7
	./configure -- 执行安装检查

    make && make install -- 编译并安装


**步骤二**<br>
配置vrrp，对外统一用192.168.56.100访问。keepalived安装后，自动安装成服务，服务启动时，默认从/etc/keepalived/keepalived.conf加载配置。所以在110与120节点上，分别创建**/etc/keepalived/keepalived.conf**，内容如下：<br>

*备注* : keepalived默认将日志输出到:"/var/log/messages"，修改方法参考:[https://blog.csdn.net/u013256816/article/details/49356689
](https://blog.csdn.net/u013256816/article/details/49356689),文章中提到的"/etc/sysconfig/keepalived"我安装后并没有发现，查看下keepalived的服务启动文件("")。

	cat /usr/lib/systemd/system/keepalived.service #查看命令

![](https://i.imgur.com/2RPIVaM.png)

上图可以看到keepalived从"/usr/local/etc/sysconfig/keepalived"加载启动参数，$KEEPALIVED_OPTIONS,修改该文件，如下：

	KEEPALIVED_OPTIONS="-D -S 0"

然后修改"/etc/rsyslog.conf"，在末尾加入如下内容:

	local0.*       /apps/keepalived/keepalived-2.0.7/logs/keepalived.log #日志目录


**128配置:**
 
    ! Configuration File for keepalived

    # 路由1
	vrrp_instance vrrp_instance_1 {
    	state MASTER   # 主节点
    	interface ens33  # 绑定网卡，keepalived将在该网卡下绑定VIP
    	virtual_router_id 100 # 路由ID，与从节点一致
    	priority 100 # 权重，主节点高于从节点
    	advert_int 1 # VRRP广播时间（秒）
    	authentication {
    	    auth_type PASS  # VRRP访问模式，密码模式
    	    auth_pass nanfei # 所有节点的密码需配置一致
    	}
    	virtual_ipaddress {
    	    192.168.100.100/24 # 虚拟IP地址
    	}
    }	

	# 路由2
    vrrp_instance vrrp_instance_2 {
    	state BACKUP   
    	interface ens33 
    	virtual_router_id 101 
    	priority 50
    	advert_int 1 
    	authentication {
    	    auth_type PASS  
    	    auth_pass nanfei 
    	}
    	virtual_ipaddress {
    	    192.168.100.101/24 
    	}
    }	

**128启动:**

	systemctl start keepalived # 启动
    systemctl status keepalived # 查看启动状态
	
**128验证:**，正常应能ping通192.168.100.100与192.168.100.101，且在ens33中可以看到绑定的100、101ip。

	ip addr
	
![](https://i.imgur.com/dpDrEKd.png)

**vrrp监控**：有兴趣的可以使用wireshark等抓包工具，分析vrrp协议。下图可以看到128服务器每个1秒，会向224.0.0.18（vrrp广播地址）发送vrrp包。

![](https://i.imgur.com/CE2wOXg.png)


**129配置:**

    ! Configuration File for keepalived

    # 路由1
	vrrp_instance vrrp_instance_1 {
    	state BACKUP   # 从节点
    	interface ens33  # 绑定网卡，keepalived将在该网卡下绑定VIP
    	virtual_router_id 100 # 路由ID，与从节点一致
    	priority 50 # 权重，主节点高于从节点
    	advert_int 1 # VRRP广播时间
    	authentication {
    	    auth_type PASS  # VRRP访问模式，密码模式
    	    auth_pass nanfei # 所有节点的密码需配置一致
    	}
    	virtual_ipaddress {
    	    192.168.100.100/24 # 虚拟IP地址
    	}
    }	

	# 路由2
    vrrp_instance vrrp_instance_2 {
    	state MASTER   
    	interface ens33 
    	virtual_router_id 101 
    	priority 100
    	advert_int 1 
    	authentication {
    	    auth_type PASS  
    	    auth_pass nanfei 
    	}
    	virtual_ipaddress {
    	    192.168.100.101/24 
    	}
    }	


**129启动:**

	systemctl start keepalived # 启动
    systemctl status keepalived # 查看启动状态

![](https://i.imgur.com/HIdIvgZ.png)

**129验证:**，正常情况129会接手101虚拟IP的访问，即在网卡上绑定了101的IP地址，128则删除101的绑定。。 

	ip addr #128

![](https://i.imgur.com/uJFodVU.png)

	ip addr #129

![](https://i.imgur.com/PmVwCjG.png)

**vip验证**，随意关闭128或129的keepalived服务，检查各自端口的变化 。客户端测试对ip100、101的访问。 <br>

	
	systemctl stop keepalived # 停止服务


至此，keepalived搭建成功。

<h3 id="3.1">3、vrrp_script</h3>

上面的测试是手动停止keepalived，工作在ARRP协议下的故障转移，更多的时候需要检测某个服务（如nginx）的状态，来进行故障转移。毕竟，如果用keepalived+nginx来实现HA，仅仅keepalived工作正常是不够的，下面用一个实例来测试keepalived的健康检查。<br>
[http://www.cnblogs.com/arjenlee/p/9258188.html](http://www.cnblogs.com/arjenlee/p/9258188.html)

前置条件：在128和129部署一个http服务器，开放80端口，建立一个可以正常访问的页面。步骤（略）。<br>

分别停止128与129的keepalived

	systemctl stop keepalived

编写脚本，事例如下:

	#!/bin/bash
	code=1 #定义返回码:非0为告知keepalived检测失败
	for c in `ps -ef |grep HATestProject |grep dotnet`;do
    	code=0 # 这里是判断有没有 dotnet 运行的一个站点
	done
	echo $code >> /apps/keepalived/keepalived-2.0.7/scripts/log # 将每次的检测结果写入自定义日志文件（非必须）
	exit $code # 返回码

配置vrrp_script

	vi /etc/keepalived/keepalived.conf

顶部增加内容如下:(注：需要添加到 vrrp_instance 的上面) 

	 
    vrrp_script checkDotnet # 名称
    {
         script "/apps/keepalived/keepalived-2.0.7/scripts/check.sh" #脚本文件路径
         interval 1 #检测间隔（秒）         
    }

修改各个 vrrp_instance ，指定校验规则

	 track_script{
            checkDotnet # 规则名称
        }

校验方法：

>1、启动 keepalived，查看IP，可以看到未绑定虚拟IP
>2、启动被检测的服务，上述实例为一个 asp.netcore的站点
>3、查看网卡配置，已经成功绑定了虚拟IP。
>4、停止被检测服务，查看IP状态。
>5、如果检测失败，则可以用/var/log/messages 里查看日志，排查。

<h2 id="2">三、nginx</h2>
参考:[http://www.nginx.cn/doc/](http://www.nginx.cn/doc/)

<h3 id="3.1">1、概述</h3>
Nginx是一个高性能的 Web 和反向代理服务器，官方数据最大支持50000个并发连接。

<h3 id="3.2">2、安装</h3>
参考: [http://www.nginx.cn/install](http://www.nginx.cn/install)<br>
发布地址: [http://nginx.org/download/](http://nginx.org/download/)<br>
版本: [v1.9.9](http://nginx.org/download/nginx-1.9.9.tar.gz)<br>
环境: centos7<br>


**步骤一**<br>
在110、120节点上分别安装 nginx v1.9.9。
    
    yum install -y gcc automake autoconf libtool make # 编译环境
   
    yum install -y gcc gcc-c++

    mkdir -p /apps/nginx # 创建源码保存目录
    cd /apps/nginx/
    wget http://nginx.org/download/nginx-1.9.9.tar.gz
    tar -zxvf nginx-1.9.9.tar.gz 
    cd nginx-1.9.9

    ./configure --sbin-path=/usr/local/nginx/nginx \
		--conf-path=/usr/local/nginx/nginx.conf \
		--pid-path=/usr/local/nginx/nginx.pid 		 
	 
    make && make install # 编译并安装

    /usr/local/nginx/nginx # 启动 

验证：分别访问 110和120的80端口， http://ip:80，出现如下图:

![](https://i.imgur.com/G9GZ0hV.png)

