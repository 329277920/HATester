# 高可用与负载均衡 #
>[一、keepalived](1#)<br>
>&nbsp;&nbsp;[1、概述](1.1#)<br>
>&nbsp;&nbsp;[2、安装](1.2#)<br>
>[二、nginx](2#)<br>
>&nbsp;&nbsp;[2.1、概述](2.1#)<br>
>&nbsp;&nbsp;[2.2、安装](2.1#)<br>
>[附录](10#)<br>
>&nbsp;&nbsp;[1、VRRP](10.1#)<br>

<h2 id="1">一、keepalived</h2>
<h3 id="1.1">1、概述</h3>
keepalived是一个由c语言写的免费的路由软件，它基于linux平台的提供高可用和负载均衡设施。它工作在4层（传输层）。基于 [RAAP](https://baike.baidu.com/item/%E8%99%9A%E6%8B%9F%E8%B7%AF%E7%94%B1%E5%99%A8%E5%86%97%E4%BD%99%E5%8D%8F%E8%AE%AE/2991482?fr=aladdin&fromid=932628&fromtitle=VRRP) 协议实现高可用。

<h3 id="1.2">2、安装</h3>
发布地址: [http://www.keepalived.org/download.html](http://www.keepalived.org/download.html)<br>
kp版本: [v2.0.7](http://www.keepalived.org/software/keepalived-2.0.7.tar.gz)<br>
环境: centos7<br>
ip: 192.168.56.100（虚IP），192.168.56.110（节点1主），192.168.56.120（节点2从）<br>

**步骤一**<br>
在110、120节点上分别安装 keepalived v2.0.7

    
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
配置vrrp，对外统一用192.168.56.100访问。keepalived安装后，自动安装成服务，服务启动时，默认从/etc/keepalived/keepalived.conf加载配置。所以在110与120节点上，分别创建/etc/keepalived/keepalived.conf。内容如下：<br>

**110配置:**

	! Configuration File for keepalived
	vrrp_instance vrrp_instance_1 {
    	state MASTER   # 主节点
    	interface enp0s8 # 绑定网卡，keepalived将在该网卡下绑定VIP
    	virtual_router_id 100 # 路由ID，与从节点一致
    	priority 100 # 权重，主节点高于从节点
    	advert_int 1 # VRRP广播时间
    	authentication {
    	    auth_type PASS # VRRP访问模式，密码模式
    	    auth_pass nanfei # 所有节点的密码需配置一致
    	}
    	virtual_ipaddress {
    	    192.168.56.100/24 # VIP
    	}
	}	

**110启动:**

	systemctl start keepalived # 启动
    systemctl status keepalived # 查看启动状态

![](https://i.imgur.com/ZvhJ69l.png)
	
**110验证:**，正常应能ping通192.168.56.100，且在enp0s8中可以看到绑定的100ip。

	ip addr
	
![](https://i.imgur.com/5CmoBwR.png)


**120配置:**

	! Configuration File for keepalived
	vrrp_instance vrrp_instance_1 {
    	state BACKUP   # 从节点
    	interface enp0s8 # 绑定网卡，keepalived将在该网卡下绑定VIP
    	virtual_router_id 100 # 路由ID，与从节点一致
    	priority 50 # 权重，主节点高于从节点
    	advert_int 1 # VRRP广播时间
    	authentication {
        	auth_type PASS # VRRP访问模式，密码模式
        	auth_pass nanfei # 所有节点的密码需配置一致
    	}
    	virtual_ipaddress {
        	192.168.56.100/24 # VIP
    	}
	}

**120启动:**

	systemctl start keepalived # 启动
    systemctl status keepalived # 查看启动状态

![](https://i.imgur.com/HIdIvgZ.png)

**120验证:**，正常应能ping通192.168.56.100，并enp0s8中应不存在100ip的绑定，如果存在，则代表110的对外IP无法访问。 

	ip addr

![](https://i.imgur.com/gOowhXl.png)

**vip验证**，将110的keepalived服务关闭，正常能看到120的enp0s8网卡会自动绑定上100的ip，且100的ip应能正常访问 。 <br>
在110上执行:
	
	systemctl stop keepalived

在120上执行:

	ip addr

![](https://i.imgur.com/kVlUHDl.png)

至此，keepalived搭建成功。

<h2 id="2">二、nginx</h2>
参考:[http://www.nginx.cn/doc/](http://www.nginx.cn/doc/)

<h3 id="2.1">1、概述</h3>
Nginx是一个高性能的 Web 和反向代理服务器，官方数据最大支持50000个并发连接。

<h3 id="2.2">2、安装</h3>
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

