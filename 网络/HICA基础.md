# 网络基础

## 1. 网络理论

### 1.1 网络和互联网

* 网络： 多个设备和交换设备连接成一个网络
* 互联网： 多个路由器组成的多个网络

### 1.2 TCP/IP分层

如今互联网主流是tcp/ip分层，iso七层不做分析。

#### 1.2.1 协议分层

* 应用层： 应用层协议包括cs双方的程序，实现程序的功能
* 传输层： 主要记住TCP为应用层提供可靠传输，UDP为应用层提供报文转发
* 网络层：IP协议为数据包选址
* 数据链路层： 负责将网络层的数据包从链路上一端发到另一端上。
* 物理层：物理介质，光纤、双绞线等

#### 1.2.2 分层的优点

* 各层独立，每层只需要知道该层与层间接口所提供的服务
* 灵活性高，每层互不影响
* 标准化：每层对应的协议具备对应的功能
* 利于排障，故障分层，简化排障手段





![image-20240511095758189](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240511095758189.png)



## 2. VRP基础

VRP (Versatile Routing Platform)即通用路由平台，是华为公司具有完全自主知识产权，且可以运行在从低端到高端的全系列路由器、交换机等数据通信产品的通用网络操作系统。

VRP可以运行在多种硬件平台之上，包括路由器、局域网交换机、ATM交换机、拨号访问服务器、IP电话网关、电信级综合业务接入平台、智能业务选择网关，以及专用硬件防火墙等。

### 2.1 华为设备型号

S系列，是以太网交换机。从交换机的主要应用环境或用户定位来划分，企业园区网接入层主要应用的是S2700和S3700两大系列，汇聚层主要应用的是S5700系列，核心层主要应用的是S7700、S9300和S9700系列。同一系列交换机版本：精简版（LI）、标准版（ SI）、增强版（EI）、高级版（HI）。如：S2700-26TP-PWR-EI表示VRP设备软件版本类型为增强版。

AR系列，是访问路由器。路由器型号前面的AR是 Access Router（访问路由器）单词的首字母组合。AR系列企业路由器有多个型号，包括AR150、AR200、AR1200、AR2200、AR3200。它们是华为第三代路由器产品，提供路由、交换、无线、语音和安全等功能。AR路由器被部署在企业网络和公网之间，作为两个网络间传输数据的入口和出口。在AR路由器上部署多种业务能降低企业的网络建设成本和运维成本。根据一个企业的用户数和业务的复杂程度可以选择不同型号的AR路由器来部署到网络中。

### 2.2 用户和命令级别

VRP命令级别分为：0级（参观级）、1级（监控级）、2级（配置级）、3级（管理级）

登录网络设备的用户分0-15级，不同级别的用户能够执行不同级别的命令。

| 用户级别 | 命令级别   | 说明                                                         |
| -------- | ---------- | ------------------------------------------------------------ |
| 0        | 0          | 网络诊断类命令（ping、tracert)、从本设备访问其他设备的命令(telnet）等 |
| 1        | 0、1       | 系统维护命令，包括display等。但并不是所有的display命令都是监控级的，例如 display current-configuration和display saved-configuration都是管理级命令 |
| 2        | 0、1、2    | 业务配置命令，包括路由、各个网络层次的命令等                 |
| 3～15    | 0、1、2、3 | 涉及系统基本运行的命令，如文件系统、FTP下载、配置文件切换命令、用户管理命令、命令级别设置命令、系统内部参数设置命令等，还包括故障诊断的debugging命令 |



### 2.3 用户界面

用户在与设备进行信息交互的过程中，不同的用户拥有不同的用户界面

常用的是`CON`(0)和`VTY ` (一般0-4)

```
查看用户支持的类型
[AR1]display user-interface
  Idx  Type     Tx/Rx      Modem Privi ActualPrivi Auth  Int     
+ 0    CON 0    9600       -     15    15          P     -       
  129  VTY 0               -     0     -           N     -       
  130  VTY 1               -     0     -           N     -       
  131  VTY 2               -     0     -           N     -       
  132  VTY 3               -     0     -           N     -       
  133  VTY 4               -     0     -           N     -   
  

```

* 用户验证的三种方式
  1. Password：只需要密码验证
  2. AAA: 需要用户和密码
  3. None：直接登录无需验证

### 2.4 配置console口登录

* password验证

```
<Huawei>system-view 

[Huawei]user-interface console 0 #切换到console用户界面

[Huawei-ui-console0]authentication-mode password  #设置密码验证
Please configure the login password (maximum length 16):123	
[Huawei-ui-console0]set authentication password cipher 12

[Huawei-ui-console0]display current-configuration section user-interface #查看当前用户界面的配置
[V200R003C00]
#
user-interface con 0
 authentication-mode password
 set authentication password cipher %$%$]2+w03wanKt6.<Y~hSjE,.R`Uf5P-1K65TjY<)(r
145O.Rc,%$%$
user-interface vty 0 4
user-interface vty 16 20
#
return
```

* aaa验证

```
[R1]aaa
[R1-aaa]local-user han password cipher 91xueit.com
Info: Add a new user.
[R1-aaa]local-user han service-type terminal 
[R1-aaa]local-user han privilege level 0 #用户的
[R1-aaa]local-user admin password cipher 51cto.com
[R1-aaa]local-user admin privilege level 3
[R1-aaa]local-user admin service-type terminal
[R1-aaa]quit

配置Console口身份验证模式为aaa
[R1]user-interface console 0
[R1-ui-console0]authentication-mode aaa
[R1-ui-console0]quit

```

### 2.5 配置telent登录

```
[Huawei]user-interface vty 0 4	
[Huawei-ui-vty0-4]authentication-mode aaa
[Huawei-ui-vty0-4]aaa
[Huawei-aaa]local-user admin service-type telnet 
```

### 2.6 配置ssh登录

```
[AR1-aaa]local-user admin service-type ssh #用户支持ssh
[AR1-aaa]quit 
[AR1]ssh user admin authentication-type password #ssh用户admin认证模式是密码认证
 Authentication type setted, and will be in effect next time
[AR1]
[AR1]stelnet server enable  #开启ssh认证服务
The STELNET server is already started.
[AR1]user-interface vty 0 4
[AR1-ui-vty0-4]authentication-mode aaa 
[AR1-ui-vty0-4]protocol inbound ssh #开启ssh
[AR1-ui-vty0-4]quit 
```

## 3. IP和子网划分

### 3.1 十进制和二进制转化

128 1000 0000

192 1100 0000

224 1110 0000

240 1111 0000

248 1111 1000

252 1111 1100

254 1111 1110

255 1111 1111

被2整除的 后一位是0； 余数为1的，后一位是1

被4整除的 后二位是00； 余数写成对应的二进制

被8整除的 后三位是000； 余数写成对应的二进制

被8整除的 后四位是0000； 余数写成对应的二进制

### 3.2 等长子网划分

### 3.3 变长子网划分

### 3.4 点对点网络划分

### 3.5 网段合并

### 3.6 同一网段判断



## 4. 路由

在网络通信中，“路由（Route）” 它是指数据包从某一网络设备出发去往某个目的地的路径。

网络中的路由器（或三层设备）负责为数据包选择转发路径。

* 路由优先级

				1.  不同来源的路由规定不同的优先级（Preference），值越小，优先级越高。
				1.  路由来源优先级排序：直连(0)->OSPF(10)->IS-IS(15)->静态(60)->RIP(100)->BGP(255)

* 路由开销

			  1.  RIP 是通过跳数开决定开销
			  1.  OSPF是带宽计算链路的开销

* 等价路由

  同一种路由协议有多条可以到达目标网段的路由并且这些路由的开销又是相同的称为`等价路由`

* 路由畅通条件

  数据包有来有回

### 4.1 直连路由

把设备自动发现的路由信息称为直连路由（Direct Route），网络设备启动之后，当路由器接口状态为UP时，路由器就能够自动发现去往自己接口直接相连的网络的路由。

### 4.2 静态路由

手动在路由器上配置的路由信息称作静态路由（Static Route）,适合小规模网络。

#### 4.2.1 静态路由配置

* 配置AR1  GigabitEthernet0/0/0 和 GigabitEthernet0/0/1的接口ip

```
[AR1]interface GigabitEthernet 0/0/0
[AR1-GigabitEthernet0/0/0]ip address 192.168.1.1 24

[AR1]interface GigabitEthernet 0/0/1
[AR1-GigabitEthernet0/0/1]ip address 172.16.1.1 24 


```

* 配置到192.168.0.0 /24网段的静态路由

```
[AR1]ip route-static 192.168.0.0 24 172.16.1.2
#查看路由表
[AR1]dis ip routing-table 
Route Flags: R - relay, D - download to fib
------------------------------------------------------------------------------
Routing Tables: Public
         Destinations : 11       Routes : 11       

Destination/Mask    Proto   Pre  Cost      Flags NextHop         Interface

      127.0.0.0/8   Direct  0    0           D   127.0.0.1       InLoopBack0
      127.0.0.1/32  Direct  0    0           D   127.0.0.1       InLoopBack0
127.255.255.255/32  Direct  0    0           D   127.0.0.1       InLoopBack0
     172.16.1.0/30  Direct  0    0           D   172.16.1.1      GigabitEthernet0/0/1
     172.16.1.1/32  Direct  0    0           D   127.0.0.1       GigabitEthernet0/0/1
     172.16.1.3/32  Direct  0    0           D   127.0.0.1       GigabitEthernet0/0/1
    192.168.0.0/24  Static  60   0          RD   172.16.1.2      GigabitEthernet0/0/1
    192.168.1.0/24  Direct  0    0           D   192.168.1.1     GigabitEthernet0/0/0
    192.168.1.1/32  Direct  0    0           D   127.0.0.1       GigabitEthernet0/0/0
  192.168.1.255/32  Direct  0    0           D   127.0.0.1       GigabitEthernet0/0/0
255.255.255.255/32  Direct  0    0           D   127.0.0.1       InLoopBack0

```



* 配置AR2  GigabitEthernet0/0/0 和 GigabitEthernet0/0/1的接口ip

```
[AR2]interface GigabitEthernet 0/0/0
[AR2-GigabitEthernet0/0/0]ip address 172.16.0.2 30

[AR2]interface GigabitEthernet 0/0/1
[AR2-GigabitEthernet0/0/1]ip address 192.168.0.1 24
```

* 配置AR2到192.168.1.0/24的回程路由

```
[AR2]ip route-static 192.168.1.0 24 172.16.0.1
```



* 配置AR3  GigabitEthernet0/0/0 和 GigabitEthernet0/0/1的接口ip

```
[AR3]interface GigabitEthernet 0/0/0
[AR3-GigabitEthernet0/0/0]ip address 172.16.1.2 30

[AR3]interface GigabitEthernet 0/0/1
[AR3-GigabitEthernet0/0/1]ip address 172.16.0.1 30
```

* 配置AR3   192.168.0.0/24 或到 192.168.1.0/24 的静态路由

```

[AR3]ip route-static 192.168.0.0 24 172.16.0.1
[AR3]ip route-static 192.168.1.0 24 172.16.1.1
```

#### 4.2.2 测试连通性

```

PC1>ping 192.168.0.2

Ping 192.168.0.2: 32 data bytes, Press Ctrl_C to break
From 192.168.0.2: bytes=32 seq=1 ttl=125 time=78 ms
...

PC2>ping 192.168.1.2

Ping 192.168.1.2: 32 data bytes, Press Ctrl_C to break
From 192.168.1.2: bytes=32 seq=1 ttl=125 time=47 ms
...
```





### 4.2 动态路由

路由器使用动态路由协议（如RIP、OSPF等）而获得的路由信息被称为动态路由。动态路由适合规模较大的网络，能够针

对网络的变化自动选择最佳路径。
