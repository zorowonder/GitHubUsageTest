Prepared by:Bi YongSen

Date Prepared:08-Oct-2015

#Broker 设计架构

##1.引言
东软载波(EastSoft) MQTT Broker是支持MQTT协议的分布式云平台，

支持满足MQTT 3.1.1协议的**Tcp/WebSocket/Http终端**接入。

本文主要阐述MQTT Broker的整体框架以及发展与扩展方向。

###1.1.目的
本文的主要目的是阐述东软载波MQTT Broker功能架构以及系统技术架构。

为后续的维护以及扩展提供技术方向以及实现依据。

###1.2.范围
主要涵盖：MQTT Broker系统功能架构以及系统技术架构，以及功能架构的分解。

###1.3.定义、首字母缩写词和缩略语

术语 | 释义
--- | ---
Hub    | Broker-Hub子系统的简称
Thing  | Broker-Thing子系统的简称

###1.4.参考资料
[AKKA](http://akka.io/)

[Netty](http://netty.io/)

[MQTT](http://mqtt.org/)

###1.5.公共约定
所有MQTT设备连接必须携带username和password

##2.Broker系统
该章节主要阐述东软MQTT Broker的系统结构。

###2.1.MQTT Broker系统组成结构
MQTT Broker主要由以下子系统组成：Hub和Thing

###2.2.MQTT Broker系统结构图

![Broker001](/Broker001.png)

###2.3.MQTT Broker子系统主要功能
Hub负责管理MQTT设备的接入，对MQTT协议报文进行初步处理，并分发到Thing。

Thing负责管理MQTT设备连接的会话，以及处理MQTT协议的逻辑。

###2.4.MQTT Broker技术使用
Broker使用AKKA框架，实现了高并发，分布式部署的集群系统。

使用Google Protocol Buffer进行高效序列化。

##3.Broker-Hub架构

###3.1.Hub功能架构
Hub为Broker面向MQTT设备的接入系统，主要有以下几项功能：

负责管理和维护MQTT设备的Tcp ，WebSocket 或者Http连接；

对MQTT设备的连接请求向租户管理门户进行验证；

初步处理和过滤MQTT协议报文，并转交给Thing处理。

###3.2.Hub技术架构

####3.2.1.Hub通信框架
Hub使用Netty框架以支持Tcp，WebSocket以及Http连接，每一个连接将在Hub System中映射为一个AKKA Actor，

这个Actor用来保存必要的，无状态的设备连接信息，并负责初步处理和过滤MQTT报文，

包括对发布和订阅的报文中的topic filter进行检查验证。

####3.2.2.Hub权限验证
Hub通过Http协议访问租户管理门户以验证请求连接的MQTT设备的合法性，

未通过验证的设备将被断开连接，无法继续进行通讯。

每个Hub的权限验证由唯一一个AKKA Actor维护，并使用断路器模式对租户管理门户在高访问量的情况下进行保护。

####3.2.3.Broker集群
Hub使用AKKA框架，支持多节点分布式部署，可由多个Hub以及Thing组成一个集群，拥有强大的处理能力和扩展性。

Hub与Thing部署在同一集群的不同节点上，通过AKKA Cluster技术实现集群内互相通信以及自动化管理。

##4.Broker-Thing架构

###4.1.Thing功能架构
Thing为Broker中处理MQTT逻辑，集轻量持久化，保证交付等功能于一体的子系统，

可以对MQTT 3.1.1协议进行完全的支持，目前最多支持**3级**topic的有效交付。

Thing可以向第三方传输数据流，供其它子系统做后续的分发以及处理。

###4.2.Thing技术架构

####4.2.1.Thing有效交付
Thing使用AKKA Persistence中AtLeastOnceDelivery来保证有效交付，接收方如未回复或超时回复，

发送方都会进行重试，重试次数及超时时间等参数都可以通过配置文件配置。

####4.2.2.Thing 会话（Session）管理
Thing会为每个连接的MQTT设备创建一个会话，用于保存连接状态，订阅状态以及必要的用户信息等，

Thing根据MQTT中描述的存在于设备Connect报文中的Clean Session标志位来管理会话的生命周期。

####4.2.3.Thing Topic管理
Thing中的topic是以AKKA Actor的形式存在的，例如eastsoft/thing/test这个topic会创建出3个Actor，

分别为：eastsoft，eastsoft/thing 以及eastsoft/thing/test，

topic级数较长的Actor由比自身少一级的Actor监管。

在多节点部署的情况下，Thing会使用AKKA Cluster Sharding技术将不同的topic相对均衡的分布在不同节点上

##5.Broker发展方向

###5.1.Thing向第三方系统提供实时数据


