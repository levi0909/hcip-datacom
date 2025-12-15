# 05_BGP详解

### 1.BGP的基本概念

#### 1.1.AS

`OSPF、IS-IS`等`IGP`路由协议在**组织机构内部**广泛应用，随着网络规模扩大，网络中路由数量不断增长，`IGP`已无法管理大规模网络，`AS`的概念由此诞生。

`AS`指的是由一个单一的机构或组织所管理的一系列`IP`网络及其设备所构成的集合。可以简单的将`AS`理解为一个独立的机构或者企业所管理的网络，例如一家网络运营商的网络等。另一个关于`AS`的例子是，一家全球性的大型企业在其网络的规划上将全球各个区域划分为一个个的`AS`，例如中国区是一个`AS`，韩国区是另一个`AS`。

![image-20250825135758555](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135336507.png)

例如上图，总部是`AS100`分部是`AS200`，现在总部和分部的路由器通过专线连接并建立`BGP`邻居关系，随后因为运行了`BGP`路由协议的路由器，同时也运行了各自公司内部的`IGP`协议，公司内部路由都会出现在其路由表中。通过`BGP`将自己公司的路由通告给对方，然后再使用重分发让公司内部的其他路由器学习到对方的路由。

------

不同`AS`通过`AS`号区分，`AS`号存在`16 bit`、`32bit`两种表示方式。`LANA`负责`AS`号的分发。你需要明白一点不管是`IP`地址还是`AS`号，都根据你使用的场景分为私有和公有。如果是在公网范围使用的一定要通过机构申请，如果是私有的那无所谓你用什么。

当不同的`AS`之间需要进行通信时，就需要使用`BGP`路由协议进行路由传递。

- 两字节的`AS`范围为：`1-65535`

- 四字节的`AS`范围为：`65536-4294967295`

华为默认使用的还是两字节的。使用命令：`private-4-byte-as enable`可开启四字节范围。

其中，`64512-65534`是私有范围

------

#### 1.2.BGP 概述

`BGP(Border Gateway Protocol)`边界网关协议，几乎是当前唯一被用于在不同`AS`之间实现路由交互的`EGP`协议。`BGP`适用于大型的网络环境，例如运营商网络，或者大型企业网等。`BGP`支持`VLSM`、`CIDR`，支持自动路由汇总、手工路由汇总。

`BGP`是一种实现自治系统`AS`之间的路由可达，并选择最佳路由的**矢量性协议**。

`BGP`使用`TCP`作为传输层协议，这使得协议报文的交互更加可靠和有序。`BGP`使用目的`TCP`端口`179`，两台互为对等体的`BGP`路由器首先会建立`TCP`连接，随后协商各项参数并建立对等体关系。

初始情况下，两者会同步双方的`BGP`路由表，在`BGP`路由表同步完成后，路由器不会周期性地发送`BGP`路由更新，而只发送增量更新或在需要时进行触发性更新，这大大减小了设备的负担及网络带宽损耗，由于`BGP`往往被用于承载大批量的路由信息，如果依然像`IGP`协议那样，周期性地交互路由信息，显然是相当低效和不切实际的。

`BGP`定义了多种路径属性（`Path Attribute`）用于描述路由，就像一个人的身高、体重、学历、特征和经历等属性一样，一条`BGP`路由同样携带者多种属性，路径属性将影响`BGP`路由的优选。

`BGP`的发展过程中，经历了数个版本，目前在`IPv4`环境中，`BGPv4(BGP Version 4)`被广泛使用。

![image-20250825155223670](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135336510.png)

------

##### 1.2.1.查看全球BGP路由条目

**远程路由器查看**

1. 通过`telnet`登录镜像路由器，地址为`route-views.routeviews.org`，用户为`rviews`。

2. 输入命令：`show bgp all summary`查看`IPv4 BGP`路由表信息。

![image-20250825154424396](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135336500.png)

- **网络条目 (Network entries):** `1,026,032` 条
- **路径条目 (Path entries):** `18,110,895` 条 (到达同一目的网络可能存在多条路径)
- **BGP路径/最佳路径属性:** `2,970,774 / 178,442` 条 (当前优选的最佳路径为178,442条)
- **AS路径信息 (AS-PATH entries):** `2,776,468` 条
- **团体属性 (Community entries):** `260,406` 条
- **扩展团体属性 (Extended community entries):** `2,926` 条

**网站查看**

可以通过网站http://www.cidr-report.org/as2.0/查看到各大机构路由发布和汇聚情况。

从下图可看出最新的全球`BGP`路由表中，路由前缀与汇总的条目：

![image-20250825154608314](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135336522.png)

------

##### 1.2.2.对等体关系类型

`BGP`建立对等体关系与`IGP`建立邻居关系区别很大，`IGP`要求建立邻居关系的两台设备必须是直连的，而`BGP`采用`TCP`，两台路由器只要能够通信，并且能够基于`TCP 179`端口建立连接，就可以建立`BGP`对等体关系，因此`BGP`的对等体关系是可以**跨设备**建立的。

`BGP`由两种对等体关系类型：

- `EBGP`：位于不同`AS`的`BGP`路由器之间的`BGP`对等体关系。两台路由器之间要建立`EBGP`对等体关系，必须满足两个条件：
  - 两个路由器所属`AS`不同
  - 在配置`EBGP`时，`peer`命令所指定的对等体`IP`地址要求路由可达，并且`TCP`连接能够正确建立。
- `IBGP`：位于相同`AS`的`BGP`路由器之间的`BGP`对等体关系。

![image-20250825160222191](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135336516.png)

------

##### 1.2.3.TCP连接源地址

缺省情况下，`BGP`使用报文出接口作为`TCP`连接的本地接口。在部署`IBGP`对等体关系时，建议使用`LoopBack`地址作为更新源地址。因为`LoopBack`接口非常稳定，而且可以借助`AS`内的`IGP`和冗余拓扑来保证可靠性。

一般而言在`AS`内部，网络具备一定的冗余性。在`R1`与`R3`之间，如果采用直连接口建`IBGP`邻居关系，那么一旦接口或者直连链路发生故障，`BGP`会话也就断了，但是事实上，由于冗余链路的存在，`R1`与`R3`之间的`IP`连通性其实并没有`DOWN`（仍然可以通过`R4`到达彼此）。

![image-20250825164830336](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135336503.png)

在部署`EBGP`对等体关系时，通常使用直连接口的`IP`地址作为源地址，如若使用`Loopback`接口建立`EBGP`对等体关系，需要考虑`TTL`。因为默认`EBGP`对等体发送的报文`TTL`为`1`。

------

#### 1.3.BGP报文详解

##### 1.3.1.报文类型

`BGP`有五种报文，分别是：`Open、Updata、Notification、Keepalive、Route-refresh`。

![image-20250826123603925](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135336732.png)

------

##### 1.3.2.通用头部

`BGP`报文属于应用层，所以`BGP`所有报文都是封装`TCP`里的，依赖`TCP`通道进行传输。

![image-20250826130413396](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135336738.png)

- `Marker`：`16 Byte`，用于标明`BGP`报文边界，所有`bit`均为`1`。
  - 主要有两个作用：
    - **定界符：** 在TCP流中标识BGP报文的开始。
    - **差错检测：** 通过检查全 `0xFF` 的值来快速发现严重错误，并重置连接。
- `Length`：`2 Byte`，`BGP`报文总长度（包括报文头在内），以`Byte`为单位。
- `Type`：`1 Byte`，用于表明携带的`BGP`报文的类型。

| TYPE值 | 报文类型     |
| ------ | ------------ |
| 1      | OPEN         |
| 2      | UPDATE       |
| 3      | NOTIFICATION |
| 4      | KEEPALIVE    |
| 5      | REFRESH      |

------

##### 1.3.3.Open报文

`Open`报文是`TCP`连接建立之后发送的第一个报文，用于建立`BGP`对等体之间的连接关系，报文格式如下：

![image-20250826131540464](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135336734.png)

- `Version`：`BGP`的版本号。对于`BGP 4`来说，其值为`4`。
- `My AS(autonomous system)`：本地`AS`号。通过比较两端的`AS`号可以判断对端是否和本段处于相同`AS`。换句话说，通过该字段知道是建立`iBGP`还是`eBGP`。
- `Hold Time`：保持时间。在建立对等体关系时两端要协商`Hold Time`，并保持一致。如果在这个时间内未收到对端发来的`Keepalive`报文或`Update`报文，则认为`BGP`连接中断。
- `BGP Identifier`：`BGP`标识符（`Router-ID`），以`IP`地址的形式表示，用来识别`BGP`路由器。

![image-20250826151258438](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135336737.png)

------

##### 1.3.4.Update报文

`Update`报文用于在对等体之间传递路由信息，可以用于发布、撤销路由。一个`Update`报文可以通告具有相同路径属性的多条路由，这些路由保存在`NLRI`（网络层可达信息）中。

同时`Update`报文还可以携带多条不可达路由，用于告知对方撤销路由，这些保存在`Withdrawn routes`字段中。

![image-20250826131959917](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135336742.png)

- `Withdrawn Routes Length`：用来表明当前的`Update`报文是更新路由还是撤销路由。
  - 当该字段为`0`时，表示**更新路由**，路由条目将出现在`NLRI`字段。
  - 当该字段非`0`时，表示**撤销路由**，被撤销的路由条目出现在`Withdrawn Routes`字段。
- `Path attributes`：与`NLRI`相关的所有路径属性列表，每个路径属性由一个`TLV`组成。
- `NLRI`：更新路由的前缀和掩码。
- `Withdrawn Routes`：撤销路由的前缀和掩码。

**通告路由**

![image-20250826134720041](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135336739.png)

**撤销路由**

![image-20250826134758556](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135336895.png)

------

##### 1.3.5.Notification报文

当`BGP`检测到错误状态时（对等体关系建立时、建立之后都可能发生），就会向对等体发送`Notification`，报纸对端错误原因。之后`BGP`连接将会立即中断。

简单来讲，就是当出现任何问题影响到对等体建立时，只要通道还保持畅通，就会向对端发送`Notification`。

![image-20250826140731450](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135336879.png)

- `Error code、Error subcode`：差错码、差错子码，用于告知对端具体的错误类型。
- `Data`：用于辅助描述详细的错误内容，长度不固定。

当你主动断开对等体关系时，也会产生`Notification`，如下：

![image-20250826140843013](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135336897.png)

------

##### 1.3.6.Keepalive报文

`BGP`路由器收到对端发送的`Keepalive`报文，将对等体状态置为已建立，同时后需定期发送`Keepalive`报文用于保持连接。

`Keepalive`报文格式中只包含报文头，没有附加其他任何字段。

![image-20250826141830262](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135336894.png)

------

##### 1.3.7.Route-refresh报文

`Route-refresh`报文用来要求对等体重新发送指定地址族的路由信息，一般为本端修改了相关路由策略之后让对方重新发送`Update`报文，本端执行新的路由策略重新计算`BGP`路由。

- `AFI`：`Address Family Identifier`，地址族标识，如`IPv4`。
- `Res`：保留，`8 bit`必须为`0`。
- `SAFI`：`Subsequent Address Family Identifier`，子地址族标识。

![image-20250826142215302](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135336911.png)

![image-20250826153420866](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135336924.png)

------

#### 1.4.BGP状态机

![image-20250826142849229](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337112.png)

1. **Idle (空闲状态)**

   - **描述**：这是BGP初始状态。BGP进程正在检测是否有足够的资源来建立对等连接（例如，是否配置了邻居）。如果没有资源，它会保持在Idle状态。同时，路由器会拒绝所有进入的BGP连接请求。

   - **动作**：BGP进程启动后，会初始化资源，并**启动一个连接重传计时器（ConnectRetry timer）**，然后尝试发起一个TCP连接到其对等体。成功后，状态转换为 **Connect**。

   - **触发转换的事件**：管理员手动启动BGP进程或重置一个已有的对等会话。
   - 任何状态中收到`Notification`报文或`TCP`连接断开等`Error`事件后，`BGP`都会切换到`Idle`状态。

   ![](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337064.png)

2. **Connect (连接状态)**

   - 在该阶段`BGP`启动连接重传定时器（`Connect Retry`，默认`32s`），并等待`TCP`三次握手完成。

   - **如果TCP连接成功建立**：BGP会向对端发送一个**OPEN报文**，并转换到 **OpenSent** 状态。
   - **如果TCP连接建立失败**（例如，对端未响应、端口`179`不可达）：状态转换为 **Active** 状态，并重启`ConnectRetry`计时器。
   - **如果ConnectRetry计时器超时**：状态仍转换为 **Active**，并重启`ConnectRetry`计时器。

   - **注意**：在此状态下，任何其他的事件（如收到其他BGP消息）都会导致TCP连接中断，并回到`Idle`状态。

3. **Active(活跃状态)**

   - 从`Connect`转换到`Active`状态后`BGP`继续试图建立`TCP`连接

     - **如果TCP连接成功建立**：BGP会向对端发送一个**OPEN消息**，关闭ConnectRetry计时器，并转换到 **OpenSent** 状态。

     - **如果ConnectRetry计时器超时**：BGP会重启该计时器，并**同时尝试重新发起TCP连接**，然后状态转换回 **Connect** 状态。

   - **如果TCP连接始终无法建立**，`BGP`会在`Connect`和`Active`状态之间来回振荡，直到底层网络问题被解决。重传定时器从`Connect`状态到`Active`状态时不会重置会一直倒计时。

   - **排查重点**：邻居长时间停留在Active状态通常意味着**IP层连通性问题**（错误的对端IP地址、ACL阻止了TCP 179端口、中间防火墙拦截等）。

![在这里插入图片描述](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337071.png)

------

4. **OpenSent**
   - **描述**：BGP已经发送了自己的OPEN消息，并正在等待对端发来的OPEN消息。
   - **动作：**
     - **当收到对端的OPEN消息：**BGP会检查消息中的参数（例如，AS号、版本号、Hold Time、BGP标识符等）。
       - **如果参数均可接受**：BGP会发送一个**KEEPALIVE消息**，并协商Hold Time（取两者中较小的值），然后状态转换为 **OpenConfirm**。
       - **如果参数有错误**（例如，对端AS号与配置不匹配）：BGP会发送一个**NOTIFICATION消息**（说明错误原因）并断开TCP连接，状态回到 **Idle**。
     - **在此状态下，任何其他BGP消息（如KEEPALIVE）的到达都被视为错误**，会触发NOTIFICATION消息并复位连接。
     - **如果TCP连接中断**：状态会回到 **Idle**。

![在这里插入图片描述](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337077.png)

------

5. **OpenConfirm**
   - **描述**：BGP已经交换并接受了彼此的OPEN消息，正在等待对端发来的KEEPALIVE消息（或自己先发送，然后等待确认）以最终确认会话参数。
   - **动作：**
     - BGP会启动一个**保持计时器（Hold Timer）**。
     - **如果收到KEEPALIVE消息**：这意味着对端也认可我们的OPEN消息参数。BGP会**重置Hold Timer**，状态转换为最终的 **Established** 状态。
     - **如果Hold Timer超时**：BGP会认为对端失效，发送NOTIFICATION消息并断开连接，状态回到 **Idle**。
     - **如果收到NOTIFICATION消息**：处理错误，断开连接，回到 **Idle**。

![在这里插入图片描述](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337083.png)

------

6. **Established**

- **描述**：**BGP对等会话已成功建立！** 这是唯一可以交换UPDATE消息（包含路由信息）的状态。
- **动作：**
  - BGP可以与其对等体交换**UPDATE**（通告/撤销路由）、**KEEPALIVE**（保活）和**NOTIFICATION**（错误通知）消息。
  - **每当收到KEEPALIVE或UPDATE消息**，都会**重置Hold Timer**。如果Hold Timer超时，就意味着与对端的连接已中断。
  - **如果收到NOTIFICATION消息**：处理错误，断开连接，回到 **Idle**。
  - **如果发生任何错误**（例如，收到格式错误的消息）：BGP会发送NOTIFICATION消息并断开连接，回到 **Idle**。

------

##### 1.4.1.总结报文与状态机

1. 交互`TCP`报文
   - `Idle`
   - `Connect`
   - `Active`
2. 交互`Open`报文
   - `OpenSent`
3. 交互`Keepalive`报文
   - `OpenConfirm`
4. 交互`Update、Keepalive`
   - `Established`

------

##### 1.4.2.邻居关系建立过程

1. 设备使用命令指定对等体地址进入`Idle`状态。
2. 本端设备发送`TCP SYN`请求建立连接，进入`Connect`状态。
   - 若`TCP`连接建立失败，进入`Active`状态继续尝试建立`TCP`。
   - 若`TCP`连接建立成功，进入`OpenSent`状态，并发送`Open`报文。
3. 当本端收到正确的`Open`报文后，进入`OpenConfirm`状态并发送`Keepalive`报文。
4. 当收到正确的`Keepalive`报文之后进入`Established`状态，开始交互`Update`报文相互学习路由条目。

在任意状态机中，如果发生了`Error`错误，即收到`Notification`报文将直接切换到`Idle`状态。

![image-20250826142857814](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337121.png)

------

**对等体建立过程**

![ ](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337227.png)

**路由交互过程**

![在这里插入图片描述](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337232.png)

------

##### 1.4.3.抓包分析

![image-20250826152320808](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337244.png)

此实验中，`AR1`和`AR2`使用直连接口建立`iBGP`邻居关系，并且`AR1`将`1.1.1.1/32`宣告到`BGP`中。命令如下：

```
[AR1]bgp 65000	
[AR1-bgp]peer 12.1.1.2 as-number 65000
[AR1-bgp]network 1.1.1.1 32
[AR1-bgp]quit

[AR2]bgp 65000
[AR2-bgp]peer 12.1.1.1 as-number 65000
[AR2-bgp]quit
```

![image-20250826153021329](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337269.png)

------

在`AR2`执行`refresh bgp all import`，触发路由更新：

![](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337292.png)

------

在`AR1`执行`undo peer 12.1.1.2`，观察`Notification`报文：

![image-20250826153534380](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337302.png)

------

#### 1.5.对等体关系建立演示

在`BGP`中对等体关系分为`iBGP`与`eBGP`两种，`iBGP`用在相同`AS`的路由器之间，`eBGP`用在不同`AS`的路由器之间。

通常在`iBGP`中推荐使用`LoopBack`接口建立对等体关系，因为其非常稳定。

![image-20250826154611987](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337395.png)

在`eBGP`中一般都是使用直连接口地址建立邻居，少数情况会使用`LoopBack`或其他接口，但是默认情况下`eBGP`发送的报文中`TTL`为1。所以你必须修改参数才能实现。

##### 1.5.1.iBGP直连口

```
[AR1]bgp 65000	
[AR1-bgp]peer 12.1.1.2 as-number 65000
[AR1-bgp]quit

[AR2]bgp 65000
[AR2-bgp]peer 12.1.1.1 as-number 65000
[AR2-bgp]quit
```

##### 1.5.2.iBGP回环口

需要先使用`IGP`打通设备之间的`LoopBack`。

```
[AR1]ospf 1
[AR1-ospf-1]area 0
[AR1-ospf-1-area-0.0.0.0]network 12.1.1.1 0.0.0.0
[AR1-ospf-1-area-0.0.0.0]network 1.1.1.1 0.0.0.0
[AR1-ospf-1-area-0.0.0.0]quit
[AR1-ospf-1]quit

[AR2]ospf 1
[AR2-ospf-1]area 0
[AR2-ospf-1-area-0.0.0.0]network 12.1.1.2 0.0.0.0
[AR2-ospf-1-area-0.0.0.0]network 2.2.2.2 0.0.0.0
[AR2-ospf-1-area-0.0.0.0]quit
[AR2-ospf-1]quit
```

然后配置`BGP`：

```
[AR1]bgp 65000
[AR1-bgp]peer 2.2.2.2 as-number 65000
[AR1-bgp]peer 2.2.2.2 connect-interface LoopBack 0

[AR2]bgp 65000	
[AR2-bgp]peer 1.1.1.1 as-number 65000
[AR2-bgp]peer 1.1.1.1 connect-interface loo0
```

![image-20250826162251115](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337394.png)

##### 1.5.3.eBGP直连接口

```
[AR1]bgp 65000
[AR1-bgp]peer 12.1.1.2 as-number 65001

[AR2]bgp 65001
[AR2-bgp]peer 12.1.1.1 as-number 65000
```

![image-20250826162449451](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337413.png)

##### 1.5.4.eBGP回环口

![image-20250826162635865](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337478.png)

```
bgp 65000
 peer 2.2.2.2 as-number 65001 
 peer 2.2.2.2 ebgp-max-hop 255 
 peer 2.2.2.2 connect-interface LoopBack0
 #
 ipv4-family unicast
  undo synchronization
  peer 2.2.2.2 enable

bgp 65001
 peer 1.1.1.1 as-number 65000 
 peer 1.1.1.1 ebgp-max-hop 255 
 peer 1.1.1.1 connect-interface LoopBack0
 #
 ipv4-family unicast
  undo synchronization
  peer 1.1.1.1 enable
```

![image-20250826164210816](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337485.png)

------

##### 1.5.5.对等体表

![image-20250826164236761](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337497.png)

- `Peer`：对等体地址
- `V`：`Version`版本号
- `AS`：对等体`AS`号
- `Up/Down`：该对等体已经存在`Up`或者`Down`的时间
- `State`：对等体状态，这里显示的为`BGP`状态机的状态
- `PrefRcv`：`prefix received`，从该对等体收到的路由前缀数目。

------

##### 1.5.6.路由表

![image-20250826164703107](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337579.png)

- `valid`：有效的。
- `best`：最优。
- `internal`：从`IBGP`学到的路由。
- `damped`：这是`BGP`的惩罚机制，用于惩罚频繁`up/down`的不稳定路由，用来减少路由震荡。
  - 当该路由的惩罚值超过抑制阈值时，它会被“阻尼”（抑制），**路由器不会将其通告给其他BGP邻居**。
- `history`：路由器检测到某条路由发生翻动，将其标记为“历史”状态，并开始计算惩罚值。**处于此状态的路由不会被使用**。
- `suppressed`：表示路由被抑制。
- `Stale`：这个状态码与BGP的**无中断重启（Graceful Restart）** 功能相关。它表示这条路由信息是从一个正在重启中的对等体那里保留下来的“陈旧”信息。路由器会暂时保留这条路由，等待对等体重启完成后更新它，从而避免路由中断。**你的路由表中没有出现这个状态码。**

![image-20250826164647981](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337576.png)

------

#### 1.6.BGP路由生成

与`IGP`路由协议不同，`BGP`自身并不会发现并计算产生路由，`BGP`将`IGP`路由表中的路由注入到`BGP`路由表中，并通过`Update`报文传递给`BGP`对等体。

所以，`BGP`不生产路由，它只是路由的搬运工。

![image-20250826165929498](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337591.png)

`BGP`注入路由的方式有两种：

- `Network`
- `import-route`
- 聚合路由(路由汇总)

------

##### 1.6.1.Network注入路由

![image-20250826170525689](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337656.png)

1. `AS200`内的`BGP`路由器通过`IGP`协议学习到了两条路有，在`BGP`进程内通过`network`命令注入这两台路由，这两条路由将会出现在本地的`BGP`路由表中。

------

![image-20250826170630442](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337673.png)

2. `AS200`内的`BGP`路由器通过`Update`报文将路由传递给`AS300`内的`BGP`路由器。
3. `AS300`内的`BGP`路由器收到路由后，将这两条路由加入到本地的`BGP`路由表中。

------

##### 1.6.2.import-route注入路由

`network`方式注入路由虽然精准，但是只能一条条`network`，如果路由条目比较多那就比较麻烦，所以可以使用`import-route`方式将其他协议的路由注入到`BGP`路由表中，比如：

- 直连路由
- 静态路由
- `OSPF`路由
- `IS-IS`路由

但是不能在`BGP`里引入`BGP`，为啥？因为，一台设备上只能有一个`BGP`进程。

![image-20250826170927289](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337674.png)

------

#### 1.7.路由通告原则

`BGP`通过`Netowrk`、`import-route`、聚合路由生成`BGP`路由后，通过`Update`报文将`BGP`路由传递给对等体。

- 只发布最优路由
  - 只发布最优且有效（下一跳地址可达）路由。
  - ![image-20250826193303736](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337751.png)
- 从`EBGP`对等体获取的路由，会发布给所有对等体。

![image-20250826193357791](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337761.png)

- `IBGP`水平分割
- `BGP`同步规则：
  - 如果同步规则开启，当一台路由器从自己的`IBGP`对等体学习到一条`BGP`路由时，它将不能使用这条路由或吧这条路有通告给自己的`EBGP`对等体，除非它又从`IGP`协议学习到这条路由，也就是要求`IBGP`路由域`IGP`路由同步。同步规则主要用于规避`BGP`**路由黑洞问题**。

------

##### 1.7.1.水平分割

水平分割指的是**从`IBGP`对等体获取的`BGP`路由，不会再发送给其他`IBGP`对等体**。这个概念非常好理解，但是水平分割的存在会极大的影响路由的传递，它的存在究竟是否有意义呢？

![image-20250826193716131](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337765.png)

如上图，`R1`与`R2`和`R3`分别建立了`EBGP`邻居关系，`AS 64513`内部的路由器两两建立了`IBGP`邻居关系。当前`R1`将`10.1.1.0/24`发布到`BGP`后，`R2`会将这条路由通告给`R3`与`R4`。此时，如果没有水平分割，`R4`会将这条路有再次通告给`R3`，`R3`会将这条路有通告给`R2`。那现在，`R2`就会从`IBGP`内部学到`10.1.1.0/24`的路由，环路产生。

所以，水平分割的存在是有很大意义的，那我们应该怎么保留它并解决它所造成的问题呢，方法有三种：

1. **全互联：**一个`AS`内部的所有`BGP`路由器两两之间建立`iBGP`邻居关系。注意，`iBGP`邻居关系非直连也可以建立。
   - 虽然这种方式可以顺利解决，但是这种方法会衍生出另一个问题，以及当网络非常庞大时，这种方式明显是很低效的。

![image-20250826194430144](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337863.png)

2. **BGP联邦**
3. **反射器**

------

##### 1.7.2路由黑洞

从上面的学习过程我们知道，两台`BGP`路由器之间可以跨设备建立`IBGP`邻居关系。但是它会带来一个麻烦：**路由黑洞**。

![image-20250826195820089](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337853.png)

如上图，`R1、R2、R3、R7`这四台路由器都运行了`BGP`，其中`R1`与`R3`、`R2`与`R7`建立了`EBGP`邻居关系。`R3`和`R7`跨设备建立了`IBGP`邻居关系。`AS 34567`内使用`OSPF`实现全互联。

现在`R1`将`1.0.0.0/8`路由发布到`BGP`并通告给了`R3`，`R3`会将这条`BGP`路由通过`IBGP`通告给`R7`并且这条路由的`Next_Hop`是`R3`建立邻居的地址，`R7`再将其通告给`R2`，最终`R2`成功学习到这条路由，并加载到自己的路由表中。

现在`R2`收到一个去往`1.0.0.0/8`的数据包，`R2`将这个数据包转发给`R7`。`R7`收到后查看路由表发现下一跳是`R3`，触发递归查询（不是直连），假设是下一跳是`R4`。

当`R4`收到这个目的地址是`1.0.0.0/8`的数据包时，麻烦来了。因为`R4`并没有运行`BGP`，所以它的路由表里不存在`1.0.0.0/8`的路由，所以**路由黑洞**产生了。因为`R4`查找不到路由，所以会将这个数据包丢弃。同样的问题在`R6`上也会存在。

------

##### 1.7.3.同步规则

为了规避**路由黑洞**问题，`BGP`引入了同步规则（`BGP Synchronization`）。

同步规则指的是：当一台路由器从自己的`IBGP`对等体学习到一条`BGP`路由时，它将不能直接使用这条路由或把这条路由通告给自己的`EBGP`邻居（包括`IBGP`水平分割）。除非它又从`IGP`协议学习到这条路有，也就是要求`IBGP`路由与`IGP`路由同步。

![image-20250826195820089](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337857.png)

如上图，现在因为有了**同步规则**后，`R7`从`R3`学习到的这条`BGP`路由不能放到路由表里，也不会发送给`R2`。你如果想完美的规避同步规则和路由黑洞，现在的方式就是在`R3`上将`BGP`路由`1.0.0.0/8`引入到`OSPF`中。那么此时，`AS 34567`内的所有路由器都有了这条路有，那路由黑洞就消失了。

但你现在看完，你肯定觉得这个**同步规则**有点太*了，所以在华为或者绝大多数设备上**同步规则**都是默认关闭的。打开的命令如下：

`synchronization`

------

实际上在这个例子中，想要规避掉路由黑洞有很多种方法：

1. 实现全互联：`AS 34567`内的所有路由器都开启`BGP`，并使用全互联的方式打破水平分割。
2. 路由同步：`IGP`路由与`BGP`同步。
3. `MPLS`：通过标签的方式替代传统`IP`寻址。

但是，其实在这里有更好的方式，只是因为还没学，所以没有将他放在上面。

我们现在应该把这个例子中的问题分为两部分：

**第一部分：**出现路由黑洞的根本原因是因为当前的`IBGP`邻居关系不连贯（非直连），导致路由表不同步。那还是很简单啊，咱们就让这里域内所有的路由器都运行`BGP`不就好了吗。

**第二部分：**但是水平分割的问题还存在啊，你难道要全互联么？太麻烦了吧！！真正的工作场景中，打破水平分割的方式，一定不是全互联。而是**联邦**或**路由反射器**（用的最多）。

------

##### 1.7.4.作业

![image-20250826203422575](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337922.png)

**第一步观察路由黑洞现象：**将图中的`OSPF`邻居与`BGP`对等体都先创建好，并在`AR1`宣告`1.0.0.0/8`的路由。观察现象：

此时，你会遇到一个问题，`AR7`收到路由后并不是有效的，因为下一跳不可达。注意：从`EBGP`邻居学习到的路由，转发给`IBGP`邻居时，下一跳不会改变依旧是`EBGP`邻居的接口地址。

![image-20250826204242284](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337919.png)

通过下述命令实现`AR3`将`EBGP`路由传递给`IBGP`邻居时修改下一跳：

```
[AR3-bgp]peer 7.7.7.7 next-hop-local
```

![image-20250826204645697](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135337953.png)

------

此时，在`AR2`的路由表中已经成功通过`BGP`学习到`1.0.0.0/8`的路由。

![image-20250826204755282](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338060.png)

使用`tracert`发现数据包到达`AR7`后就没有然后了：

![image-20250826204923471](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338046.png)

这是因为数据包达到`AR7`后，`AR7`查看路由表发现`1.0.0.0/8`的下一跳是`3.3.3.3`，触发路由递归原则，查看`3.3.3.3`发现下一跳是`AR4`，遂数据包发送到`AR4`。

![image-20250826204846332](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338066.png)

但是`AR4`上压根就没有`1.0.0.0/8`的路由，所以数据包被丢弃。

![image-20250826205105004](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338109.png)

------

**第二步开启路由同步规则**：在`AR3`上开启路由同步规则。

- 尴尬，你虽然可以在模拟器上看到`undo synchronization`但是，不支持你启用同步规则！

------

**第三步：重分布**：

- 在`AR3`把`BGP`引入到`OSPF`

```
[AR3]ospf 1 
[AR3-ospf-1]import-route bgp
```

可是现在依旧`ping`不通为啥呢？

- `AR1`上并没有`AR7`或`AR2`的接口地址，所以你需要在`AR3`把`OSPF`引入到`BGP`

```
[AR3]bgp 34567
[AR3-bgp]import-route ospf 1
[AR3-bgp]quit
```

- 此时在`AR7`上可以成功`ping`通`1.0.0.1`了，但是`AR2`依旧不行，因为`AR1`没有`AR2`的接口地址

![image-20250826205904826](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338094.png)

- 在`AR7`上将直连引入`BGP`和`OSPF`

```
[AR7]bgp 34567
[AR7-bgp]import-route direct 
[AR7-bgp]quit

[AR7]ospf 1	
[AR7-ospf-1]import-route direct 
[AR7-ospf-1]quit
```

![image-20250826210015007](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338123.png)

------

#### 1.8.Router-ID

`BGP Router-ID`是网络设备的`BGP`协议标识符，长度为`32bit`，与`IPv4`地址的格式相同，例如`192.168.1.1`。在规划`BGP`网络是，需确保设备的`Router-ID`的唯一性。

`BGP`的`Router-ID`可以通过两种方式获取：

1. 让`BGP`自动获取
   1. 选择`LoopBack`接口中地址最大的
   2. 选择物理接口中地址最大的
2. 手动配置（推荐）

------

### 2.路径属性

任何一条`BGP`路由都拥有多个**路径属性**（`Path Attributes`），当路由器将`BGP`路由通告给它的对等体时，一并被通告的还有路由所携带的各个路径属性。

当`BGP`发现了多条到达同一个目的网段的路由时，每条`BGP`路由的路径属性值将会作为该路由是否被优选的依据，`BGP`将按照一定的规则进行决策，最终在这些路由中选择一条最优的路由。

我们也可以根据实际需求，对`BGP`路由的路径属性进行修改，从而影响`BGP`路由优选的决策。



![image-20250827133720457](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338204.png)

------

#### 2.1.路径属性分类

![image-20250827140919023](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338229.png)

**`BGP`路径属性分为两大类：**

- **公认（Well-known）**：公认属性是所有`BGP`路由器都必须能够识别的属性。
- **可选（Optional）**：可选属性不要求所有的`BGP`路由器都必须能够识别。

**公认属性可以分为两类：**

- **公认必遵：**`BGP`路由器使用`Update`报文通告路由更新时必须携带的路径属性。
- **公认任意**：不要求`Update`报文通告路由更新时必须携带的路径属性。

**可选属性可以分为两类：**

- **可选过渡（传递）：**即使`BGP`路由器不能够识别该路径属性，依然会接受携带该路径属性的路由更新，且当需要将该路由通告给其他对等体时必须携带该路径属性。
- **可选非过度（不传递）：**如果`BGP`路由器不能识别该路径属性，那么路由器就会忽略携带该路径属性的路由，且不会通告给其他对等体。

![image-20250827140045557](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338246.png)

------

从上述内容中，我们可以总结出**公认属性**规定的是通告的路由必须**携带**或可选的属性。**可选属性**规定的是**接收**路由时遇到无法识别的属性，是接收且传递，还是不接收不传递。

举个例子，公认必遵就类似于**简体字**在一定范围内，大家的书面文件一定得是以**简体字**形式呈现。而**公认任意**则类似于**繁体字**虽然没要求你必须写，但是你如果想在你的文件里用**繁体字**书写，也是可以的。

而**可选过度**类似于**英文**，虽然我可能看不懂具体内容，但我知道它是有效的，所以我会接收并同步给其他人，因为我看不懂，不代表别人也看不懂。

但是**可选非过度**就类似于**甲骨文、象形语言**，如果我看不懂，当我收到后我不会进行接收，也不会进行传播。

------

#### 2.2.路径属性详解

##### 2.2.1.AS_Path

![image-20250827143354346](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338269.png)

`AS_Path`是**公认必遵属性**，它描述了一条`BGP`路由在传递过程中所经过的`AS`号。一台路由器在将`BGP`路由通告给自己的`EBGP`邻居时，会将本地的`AS`号插入到该路由原有`AS_Path`之前。一条`BGP`路由在一个`AS`内（`IBGP`）传递时，该路由所携带的`AS_Path`属性不会改变。只有在`EBGP`对等体之间才会改变。

**作用一：**

通过`AS_Path`可以避免`EBGP`路由的环路，如果路由器收到一条`BGP`路由并且发现该路由携带的`AS_Path`中出现了自己的`AS`号，该路由器会忽略这条路由的更新，避免环路产生。

**作用二：**

`AS_path`可以用于`BGP`路由选路，`AS_path`本身是一个列表，所以这个列表越短该路由约优。

------

**举例：**

如果一台`BGP`路由器收到一条路由的`AS_Path`是`300 500 100 200 400`，那么对于这台路由器来说，他如果要访问这条路由，他需要依次经过`AS 300 500 100 200 400`，所以`300`离他最近，`400`离他最远。

如果现在这台路由器本地`AS`为`1000`，当他把这条路由通告给它的`EBGP`邻居时，会在`AS_Path`加上自己的本地`AS`号，`AS 1000 300 500 100 200 400`。

------

![image-20250827143721640](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338330.png)

如上图，请问`R4`会通过哪个邻居学习`10.3.0.0/16`。

------

**AS_Path类型**

`AS_Path`默认使用的类型是`AS_SEQENCE`，标识`AS_Path`内的`AS`号是一个有序的列表。

当使用`BGP`聚合路由时可以选择使用`AS_SET`类型。

如下图，`R3`针对`R1`和`R2`的`EBGP`路由做了路由聚合，默认情况下汇总后的路由只会携带当前`AS`号，也就是在哪做的聚合，就携带哪个`AS`的`AS`号。但是这样很容易引发路由环路，所以可以在做路由聚合时，加上参数`as-set`参数，使`R3`向外更新路由时携带明细路由的`AS`号，但此时这些明细路由的`AS`会被放在一个大括号里，并且部分先后顺序。

![image-20250827144846351](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338296.png)

------

`AS_Path`属性可以用来匹配路由，使用`as-path-filter`工具即可做到。另外，搭配上**正则表达式**即可通过匹配`AS_Path`属性匹配路由。

|     **语法/模式**      |                **含义**                |               **示例**                |                 **示例解释**                  |
| :--------------------: | :------------------------------------: | :-----------------------------------: | :-------------------------------------------: |
|     **基本元字符**     |                                        |                                       |                                               |
|          `^`           |           匹配 AS 路径的开始           |                `^100`                 |             路径**起始于** AS 100             |
|          `$`           |           匹配 AS 路径的结束           |                `100$`                 |             路径**结束于** AS 100             |
|          `_`           |  匹配任意一个 AS 号（空格或路径起止）  |                `_100_`                |             路径中**包含** AS 100             |
|          `.`           |        匹配任意单个字符（慎用）        |                 `65.`                 |         匹配如 `650`, `651`, `65x`等          |
|          `*`           |   匹配前一个字符或模式**0次或多次**    |                `123*`                 |          匹配 `12`, `123`, `1233`...          |
|          `+`           |   匹配前一个字符或模式**1次或多次**    |                `123+`                 |        匹配 `123`, `1233`, `12333`...         |
|          `?`           |    匹配前一个字符或模式**0次或1次**    |                `123?`                 |               匹配 `12`或 `123`               |
|          `\|`          |             **或** 操作符              |             `^(100\|200)`             |        路径起始于 AS 100 **或** AS 200        |
|         `( )`          |               将模式分组               |             `(123 456)+`              |         匹配一次或多次 `123 456`序列          |
|         `[ ]`          |       匹配括号内任意**单个**字符       |               `65[0-5]`               |              匹配 `650`到 `655`               |
|      **常用模式**      |                                        |                                       |                                               |
|          `^$`          |   匹配**空** AS 路径（本地起源路由）   | `ip as-path access-list 10 permit ^$` |            允许本路由器始发的路由             |
|          `.*`          |          匹配**任意** AS 路径          | `ip as-path access-list 20 permit .*` |          允许所有路由（常用在最后）           |
|        `^ASN$`         |     匹配**直接来自**该 ASN 的路由      |               `^65001$`               |     匹配从 EBGP 邻居 AS 65001 学来的路由      |
|        `_ASN_`         |       匹配**经过**该 ASN 的路由        |               `_65400_`               |    匹配路径中任何位置包含 AS 65400 的路由     |
|        `^ASN_`         |  匹配**起源于**该 ASN 或其下游的路由   |               `^65001_`               |  匹配源自 AS 65001 的路由（可能经过其他AS）   |
|        `_ASN$`         |     匹配**下一跳是**该 ASN 的路由      |               `_65001$`               | 匹配下一个 AS 是 65001 的路由（即从它学来的） |
|      `^AS1 AS2$`       |       匹配**精确的 AS 路径序列**       |              `^100 200$`              |       匹配精确路径为 `100 → 200`的路由        |
|      `_AS1 AS2_`       |     匹配路径中包含**该序列**的路由     |              `_100 200_`              | 匹配路径中包含 `... → 100 → 200 → ...`的路由  |
|      **高级模式**      |                                        |                                       |                                               |
| `^([0-9]+ ){n}[0-9]+$` |       匹配特定**长度**的 AS 路径       |        `^([0-9]+ ){2}[0-9]+$`         |        匹配恰好包含 **3 个 AS** 的路径        |
|     `^65[0-9]{2}`      | 匹配 AS 号范围（正则匹配，非数值比较） |             `^65[0-9]{2}`             |    匹配 AS 号 `6500`- `6599`**开头**的路径    |
|      `_123(45)?_`      |             组合使用元字符             |             `_123(45)?_`              |      匹配包含 `123`**或** `12345`的路径       |

![image-20250831172804078](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338365.png)

**实验一：**

在`AR3`上匹配路由始发`AS 100`的路由将其过滤掉。

```
[AR3]ip as-path-filter 1 deny _100$
[AR3]ip as-path-filter 1 permit .*

[AR3]bgp 300
[AR3-bgp]peer 23.1.1.1 as-path-filter 1 import 
[AR3-bgp]quit
```

![image-20250831173035983](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338426.png)

**实验二：**

在`AR3`上匹配路由始发`AS 200`的路由将其`MED`值设置为`1000`：

```
[AR3]ip as-path-filter 2 permit _200$

[AR3]route-policy MED permit node 10	
[AR3-route-policy]if-match as-path-filter 2
[AR3-route-policy]apply cost 1000
[AR3-route-policy]quit	

[AR3]route-policy MED permit node 20
[AR3-route-policy]quit

[AR3]bgp 300	
[AR3-bgp]peer 34.1.1.2 route-policy MED export 
[AR3-bgp]quit
```

![image-20250831173301494](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338423.png)

------

##### 2.2.2.Origin

`Origin`属性是**公认必遵**属性，用于描述`BGP`路由的来源。根据路由被引入到`BGP`的方式不同，存在三种类型的`Origin`。

![image-20250827151246949](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338428.png)

当去往同一个目的地存在多条不同`Origin`属性的路由时，在其他条件都相同的情况下，`BGP`将按照如下顺序优选路由：`IGP > EGP > Incomplete`。

对于`Origin`名称你很有可能会对他产生误解，首先`IGP`不代表是通过`IGP`学习到的，而是只要使用`network`方式注入路由的都是`IGP`。`EGP`指的是通过`EGP`路由协议学习到的，但现在已经不使用`EGP`了。`Incomplete`指的是通过`import-route`注入路由。

![image-20250827151825547](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338474.png)

------

##### 2.2.3.Next_Hop（看PPT）

`Next_Hop`是一个**公认必遵**属性，用于指定到达目标网络的下一跳地址。

当路由器学习到`BGP`路由后，需对`BGP`路由的`Next_Hop`属性进行检查，该属性指（`IP`地址）必须在本地路由可达，如果不可达，则这条`BGP`路由不可用。

在不同的场景中，设备对`BGP`路由的缺省`Next_Hop`属性值的设置规则如下：

- 当`BGP`路由器将本地路由表中的路由通过`network`或`import-route`命令发布到`BGP`时，在该路由器的`BGP`路由表中，这些路由的`Next_Hop`属性显示`0.0.0.0`，因为该路由器是这些路由的始发者。

- 当`BGP`路由器使用`aggregate`命令通告一条`BGP`汇总路由时，在该路由器的`BGP`路由表中，该汇总路由的`Next_Hop`属性显示`127.0.0.1`。
- 当`BGP`路由器将本地始发的`BGP`路由通告给自己的`IBGP`对等体时，路由的`Next_Hop`属性值将被设置为这台路由器的`BGP`更新源`IP`地址。
- 当`BGP`路由器将非本地始发的`BGP`路由通告给自己的`IBGP`对等体时，缺省情况下该路由原有的`Next_Hop`属性不会被改变。
- 当路由将一条`EBGP`路由通告给自己的`IBGP`对等体时，缺省情况下，该路由器不会改变该路由原有的`Next_Hop`属性值。

- 当路由器将一条`BGP`路由通告给自己的`EBGP`对等体时，无论这条路由是否为该路由器始发，路由的`Next_Hop`属性值都被设置为这台路由器的更新源`IP`地址。

==更新源`IP`地址是设备建立对等体使用的接口`IP`地址。==

------

##### 2.2.4.Local_Preference

`Local_Preference`（本地优先级），该属性是**公认任意属性**，可以用于告诉`AS`中的路由器，哪条路径是离开本`AS`的首选路径。该值越大则`BGP`路由越优，缺省情况下值为`100`。该属性只能传递给`IBGP`对等体，不能传递给`EBGP`对等体。

![image-20250827154100316](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338538.png)

![image-20250827154147237](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338607.png)

![image-20250827154317804](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338589.png)

------

##### 2.2.5.Community

如下图，当`AS 100`内有大量的路由被引入`BGP`，这些路由分别用于生产及办公网络。现在`AS200`的`BGP`路由器需要分别针对这些路由执行不同的策略，如果使用`ACL、IP-Prefix`这些工具，效率非常地下。

有了`Community`属性后，可以为不同种类的路由打上不同的`Community`属性值，这些属性会伴随着`BGP`路由更新给`AS 200`，那么`AS200`的`BGP`路由器只需要根据`Community`来执行差异化的策略即可，而不用去关心路由前缀。

![image-20250827164506901](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338610.png)

------

`Community`（团体）属性为可选过渡属性，是一种路由标记，用于简化路由策略的执行。可以将某些路由分配一个特定的`Community`属性值，之后就可以基于`Community`的值而不是网络前缀和掩码信息来匹配路由并执行相应的策略了。

`Community`属性值长度为`32bit`，可以使用两种形式呈现：

- 十进制整数格式。
- `AA:NN`格式，其中`AA`表示`AS`号，`NN`是自定义的编号。

![image-20250827165100014](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338601.png)

`RFC1997`定义了几个公认的`Community`属性值：

![image-20250827165215062](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338648.png)

------

**实验一**

在下图中，`R1、R2、R3`分别建立了`EBGP`邻居关系，`R1`将业务`A`与业务`B`的路由通过`EBGP`传递到其他两个`AS`中。为了对这两种不同业务的路由做区分，在`R1`上部署路由策略，是其通告的业务`A`路由携带`100:1`的`Community`属性，业务`B`路由携带`100:2`的`Community`属性。

![image-20250831163714228](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338737.png)

**network方法：**

```
[AR1]route-policy comm1 permit node 10
[AR1-route-policy]apply community 100:1
[AR1-route-policy]quit

[AR1]route-policy comm2 permit node 20
[AR1-route-policy]apply community 100:2
[AR1-route-policy]quit

[AR1]bgp 100
[AR1-bgp]network 172.16.1.0 24 route-policy comm1
[AR1-bgp]network 172.16.2.0 24 route-policy comm1
[AR1-bgp]network 172.16.3.0 24 route-policy comm2	
[AR1-bgp]networ
```

![image-20250831164226397](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338763.png)

**import-route方法：**

```
[AR1]ip ip-prefix comm1 permit 172.16.1.0 24
[AR1]ip ip-prefix comm1 permit 172.16.2.0 24
[AR1]ip ip-prefix comm2 permit 172.16.3.0 24
[AR1]ip ip-prefix comm2 permit 172.16.4.0 24

[AR1]route-policy comm1 permit node 10	
[AR1-route-policy]if-match ip-prefix comm1
[AR1-route-policy]apply community 100:1
[AR1-route-policy]quit	

[AR1]route-policy comm1 permit node 20
[AR1-route-policy]if-match ip-prefix comm2
[AR1-route-policy]apply community 100:2
[AR1-route-policy]quit

[AR1]bgp 100
[AR1-bgp]import-route direct route-policy comm1
[AR1-bgp]quit
```

![image-20250831164733945](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338776.png)

但此时，`R2`和`R3`学习到的路由并没有携带任何`community`属性，需要在`R1`增加如下命令，允许通告带有`community`属性。

![image-20250831164853207](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338779.png)

```
[AR1-bgp]peer 12.1.1.2 advertise-community
```

同理，在`R2`上也要针对`R3`配置该命令：

```
[AR2-bgp]peer 23.1.1.2 advertise-community
```

![image-20250831165046900](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338817.png)

![image-20250831165026403](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338834.png)

------

**实验二：**

![image-20250831165515531](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338931.png)

在上个实验中已经成功通过`community`属性区分开两个业务了，现在`R2`上要求通过`route-policy`根据`community`属性把业务`A`的路由过滤掉。并且在业务`B`原有`community`属性的基础上增加`no-export`属性。

```
[AR2]ip community-filter 1 permit 100:1	
[AR2]ip community-filter 2 permit 100:2

[AR2]route-policy FromR1 deny node 10
[AR2-route-policy]if-match community-filter 1
[AR2-route-policy]quit
[AR2]route-policy FromR1 permit node 20
[AR2-route-policy]quit

[AR2]route-policy ToR3 permit node 10
[AR2-route-policy]if-match community-filter 2
[AR2-route-policy]apply community no-export additive 
[AR2-route-policy]quit

[AR2]bgp 200
[AR2-bgp]peer 12.1.1.1 route-policy FromR1 import 
[AR2-bgp]peer 23.1.1.2 route-policy ToR3 export 
```

![image-20250831170015579](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338941.png)

------

**实验三：**

当然你如果想实现上述过滤路由的目的，可以依赖其他工具，比如`Filter-Policy`。先记得还原上个实验的环境。

![image-20250831170159478](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338951.png)

要求在`AR2`上使用`Filter-Policy`过滤掉业务`A`的路由。

```
[AR2]acl 2000
[AR2-acl-basic-2000]rule deny source 172.16.1.0 0.0.0.0
[AR2-acl-basic-2000]rule deny source 172.16.2.0 0.0.0.0
[AR2-acl-basic-2000]rule permit
[AR2-acl-basic-2000]quit

[AR2]bgp 200	
[AR2-bgp]peer 12.1.1.1 filter-policy 2000 import 
[AR2-bgp]quit
```

![image-20250831170550120](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338962.png)

上述的演示是过滤掉如方向的路由，我们也可以过滤掉出方向，现在要求`R2`不发送`172.16.3.0`的路由给`R3`：

```
[AR2]acl 2001
[AR2-acl-basic-2001]rule deny source 172.16.3.0 0.0.0.255
[AR2-acl-basic-2001]rule permit
[AR2-acl-basic-2001]quit

[AR2]bgp 200	
[AR2-bgp]peer 23.1.1.2 filter-policy 2001 export 
[AR2-bgp]quit
```

![image-20250831170743398](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338989.png)

------

**实验四**

![image-20250831170159478](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135338951.png)

同样的需求使用`ip-prefix`也可以做到，要求`AR3`使用`ip-prefix`将`172.16.4.0/24`过滤掉。

```
[AR3]ip ip-prefix deny deny 172.16.4.0 24

[AR3]bgp 300	
[AR3-bgp]peer 23.1.1.1 ip-prefix deny import 
[AR3-bgp]quit
```

------

##### 2.2.6.MED

`MED(Multi-Exit Discriminator)`属性是一个**可选非过渡属性**，是一种度量值。当到达同一个目的网段存在多条`BGP`路由时，在其他条件相同的情况下，`MED`属性值越小的`BGP`路由将被优选。

`MED`属性用于向外部对等体指出进入本`AS`的首选路径，即当进入本`AS`的入口有多个时，`AS`可以使用`MED`动态地影响其他`AS`进入地路径。

![image-20250828191154260](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339040.png)

如上图，`AS100 R4`可以通过`R2`和`R3`学习到`AS200`的路由。当我们希望`R4`可以优先通过`R3`学习到`AS200`的路由，当发生故障再从`R2`进入`AS200`。此时，就可以在`R3`上通过`Route-Policy`设置其通告给`R4`的路由`MED`值比`R2`小，那么`R4`默认情况下，优选`R3`通告的路由。

缺省情况下，只有当路由来自同一个相邻的`AS`时，`BGP`才会进行`MED`属性的比较。可以通过`compare-different-as-med`命令，使`BGP`路由器在收到不同`AS`号的`EBGP`邻居通告的相同前缀和掩码的路由通过`MED`属性进行比较。

------

一台`BGP`路由器将路由通告给`EBGP`对等体时，是否携带`MED`属性，需要根据以下条件进行判断（不对`EBGP`对等体使用策略的情况下）：

- 如果该`BGP`路由是本地始发（本地通过`network`或`import-route`命令引入）的，则缺省携带`MED`属性发送给`EBGP`对等体。

  - 如果路由器通过`IGP`学习到一条路由，并通过`network`或`import-route`的方式将路由引入`BGP`，产生的`BGP`路由的`MED`值继承路由在`IGP`中的`metric(cost)`。例如下图中，如果`R2`通过`OSPF`学习到`10.0.1.0/24`，并且该路由在`R2`的全局路由表中`Cost`为`100`，那么当`R2`将路由`network`进`BGP`后，产生的`BGP`路由的`MED`值为`100`。

    ![image-20250828193803206](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339112.png)

  - 如果路由器将本地直连、静态路由通过`network`或`import-route`引入到`BGP`，那么这条`BGP`路由的`MED`为`0`，因为直连、静态的`COST`为`0`。

- 如果该`EBGP`路由从对等体学到，那么该路由传递给`EBGP`对等体时缺省不会携带`MED`属性。

  - 例如下图中， `R3`从`R2`学习到一条携带了`MED`属性的`BGP`路由，则它将该路由通告给`R4`时，缺省是不会携带`MED`属性的。这就是所谓的：`MED`不会跨`AS`传递。

  ![](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339212.png)

  - 可以使用`default med`命令修改缺省的`MED`值，`default med`命令只对本设备上用`import-route`命令引入的路由和`BGP`的聚合路由生效。例如在`R2`上配置`default med 999`，那么`R2`通过`import-route`及`aggregate`命令产生的路由传递给`R3`时，路由携带的`MED`为`999`。

- 在`IBGP`对等体之间传递路由时，`MED`值会被保留并传递，除非部署了策略，否则`MED`值在传递过程中不会发生改变也不会丢失。

------

##### 2.2.7.Atomic_Aggreagte及Aggregator

`Atomic_aggreagte`属于**公认任意属性**，而`Aggregator`属于**可选过渡属性**。

------

**Atomic_Aggreagte**

这是一个**警告标识**，他的唯一作用是向`BGP`对等体发出信号：“这条路由是经过聚合的，其`AS`路径信息可能不完整，因此可能会丢失`AS_Path`。”

这个属性本身不携带任何信息，要么存在要么不存在。

------

**Aggregator**

**这个属性记录了是谁执行了这次路由聚合操作，通常与`Atomic_Aggregate`一同出现：**

1. **执行聚合的路由器的AS号** (AS Number)
2. **执行聚合的路由器的BGP Router ID** (IP Address)

这个属性存在的意义是，当网络管理员需要溯源这个聚合路由是谁汇总的，为什么没有携带`AS_Path`时，可通过这个属性精准确定位置。

![image-20250828224847774](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339219.png)

------

##### 2.2.8.Preferred-Value

`Preferred-Value`（协议首选值）是华为设备的特有属性，该属性仅在本地有效。

当`BGP`路由表中存在到相同目的地的路由时，将优先选择`Preferred-Value`值高的路由。

------

### 3.路由汇总

**路由汇总**几乎是每一种动态路友协议都支持的功能，而`BGP`路由协议本身承载的路由条目就是极大的，所以**路由汇总**对于`BGP`显得尤为重要。

`BGP`支持路由**自动汇总**和**手动汇总**。对于`BGP`路由的自动汇总，是受限于特定的场景的，首先要求路由器打开`BGP`路由自动汇总的开关（通过在`BGP`配置视图下使用`summary automatic`命令开启，缺省未配置），另外，`BGP`的自动汇总功能仅对使用`import-route`方式引入`BGP`的路由有效。

#### 3.1.自动汇总

![image-20250829145801438](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339240.png)

如上图，`AR1`与`AR2`建立`EBGP`邻居关系，`AR1`中通过`import-route direct`引入直连，并开启自动汇总。

```
bgp 65000
 router-id 1.1.1.1
 peer 12.1.1.2 as-number 65001 
 #
 ipv4-family unicast
  undo synchronization
  summary automatic
  import-route direct
  peer 12.1.1.2 enable

bgp 65001
 router-id 2.2.2.2
 peer 12.1.1.1 as-number 65000 
 #
 ipv4-family unicast
  undo synchronization
  peer 12.1.1.1 enable
```

![image-20250829145745216](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339242.png)

如果要完美一些，记得加上`Route-Policy`：

```
[AR1]ip ip-prefix bgp permit 172.16.0.0 16 less-equal 24	
[AR1]route-policy bgp permit node 10
[AR1-route-policy]if-match ip-prefix bgp
[AR1]bgp 65000	
[AR1-bgp]import-route direct route-policy bgp
```

![image-20250829150239809](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339268.png)

------

#### 3.2.手动汇总

自动汇总仅对使用`import-route`命令引入到`BGP`的路由产生作用，另外`BGP`自动汇总后的路由只能是主类网络路由（`A、B、C`类）。

当你需要更精确的控制时，手工路由汇总就是一个最佳的解决方案。`BGP`支持手工路由汇总，而且配置非常简单，在`BGP`配置视图下使用`aggregate`命令即可配置手工路由汇总，`aggregate`命令包含着多个可选参数。

------

如下图，`AR1`使用`import-route`的方式将四条路由通告给`EBGP`对等体`AR2`。`AR2`在将该路由通告给`AR3`。现在使用`AR2`执行手动路由汇总。

![image-20250829151537955](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339389.png)

##### 3.2.1.普通手动汇总

```
[AR2]bgp 65001
[AR2-bgp]aggregate 172.16.0.0 16
```

**现象：**

此时在`AR3`上查看`BGP`路由表，会发现明细路由依旧存在，只是多了一条汇总后的路由。从路由汇总的角度来说，其实意义并不大，不仅没有做到减少路由，还额外增加了一条。

![image-20250829151758734](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339415.png)

另外，`AR2`做完路由汇总后，因为这条路有丢失了`AS_Path`属性，所以这条路由还会被通告给`AR1`，并且加载到了`AR1`的路由表中，产生了环路的风险。

![image-20250829152144637](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339432.png)

![image-20250829153119035](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339428.png)

------

##### 3.2.2.detail-suppressed参数

```
[AR2]bgp 65001
[AR2-bgp]aggregate 172.16.0.0 16 detail-suppressed 
```

当在汇总时加上这个参数后，`R2`只通告汇总路由，抑制明细路由。

![image-20250829153303561](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339436.png)

![image-20250829154010261](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339460.png)

此时，在`AR2`的`BGP`路由表中明细路由前都打上了`S`标记，代表被抑制，不能通告给其他邻居。

![image-20250829153349084](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339566.png)

------

##### 3.2.3.as-set参数

虽然，我们现在通过`detail-suppressed`参数实现了明细路由抑制，但是汇总路由的`AS_Path`依旧没有携带汇总前的信息，导致可能出现环路。

当我们增加`as-set`参数后，产生的汇总路由将会继承明细路由的路径属性，其中`AS_Path`最为关键。

```
[AR2]bgp 65001
[AR2-bgp]aggregate 172.16.0.0 255.255.0.0 detail-suppressed as-set
```

此时，`AR3`学习到的汇总路由会明确继承明细路由的`AS`号，需要注意的是即使你增加了`as-set`参数，`Aggregator`以及`Atoic-aggregate`依旧存在。

![image-20250829154322270](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339605.png)

当前`AR1`上就已经学习不到这条汇总路由了，因为它在这条路有的`AS_path`属性上看到了自己的`AS`号，所以将其忽略。

![image-20250829154518753](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339620.png)

------

##### 3.2.4.suppress-policy参数

这个参数可以实现有选择的抑制明细路由，而不是一刀切全部抑制掉。我们可以通过搭配`Route-Policy`实现这个需求。

![image-20250829151537955](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339612.png)

要求`R2`将汇总路由以及除`172.16.1.0/24`之外的明细路由通告给`AR3`，使用`suppress-policy`可实现。`suppress-policy`叫做抑制策略，所以在`Route-Policy`中被匹配上的路由将会被抑制。

```
[AR2]ip ip-prefix no-subnet permit 172.16.1.0 24
[AR2]route-policy no-subnet permit node 10
Info: New Sequence of this List.
[AR2-route-policy]if-match ip-prefix no-subnet
[AR2-route-policy]quit

[AR2]bgp 65001
[AR2-bgp]aggregate 172.16.0.0 16 suppress-policy no-subnet
```

![image-20250829163939234](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339637.png)

------

##### 3.2.5.origin-policy参数

当我们在`R2`上配置`aggregate 172.16.0.0 16`时，只要`R2`的路由表中存在`172.16.1.0/24`或`172.16.2.0/24`，只要有任意一条路由存在，汇总路由都会产生。只有当所有`172.16.`开头的明细路由都消失了，路由汇总才会失败。

![image-20250829151537955](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339661.png)

那现在我有一个需求，在我当前的环境中，我认为`172.16.3.0/24`这条路由非常重要，如果这条明细路由在汇总点消失了，那么汇总路由也应该消失，而其他的明细路由是否存在我并不关心。实现这个需求就可以使用`origin-policy`参数。

```
[AR2]ip ip-prefix test1 permit 172.16.3.0 24

[AR2]route-policy test1 permit node 10
[AR2-route-policy]if-match ip-prefix test1
[AR2-route-policy]quit

[AR2]bgp 65001	
[AR2-bgp]aggregate 172.16.0.0 255.255.0.0 origin-policy test1
[AR2-bgp]quit
```

此时`AR3`是可以学习到汇总路由的：

![image-20250829165830950](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339739.png)

当`AR2`路由表中的`172.16.3.0/24`消失，那么`AR3`就无法学习到汇总路由：

![image-20250829165916804](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339807.png)

------

此时，有个非常有意思的现象，当我在刚刚的例子中，加上`detail-suppressed`参数后，那么`R2`只针对`172.16.3.0/24`进行抑制，因为你使用了`origin-policy`后，只有这条路由被认为与该汇总路由强相关。

```
[AR2-bgp]aggregate 172.16.0.0 255.255.0.0 origin-policy test1 detail-suppressed
```

![image-20250829170235665](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339853.png)

------

另外，如果你再添加上`as-set`参数，那么`R2`产生的汇总路由只会继承`Route-Policy test1`匹配的明细路由的路径属性。

------

##### 3.2.6.attribute-policy参数

![image-20250829170719269](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339856.png)

在汇总时通过`attribute-policy`参数调用一个`Route-Policy`，可以设置汇总路由的路径属性。例如要将`R2`所产生的汇总路由的`MED`数值值设置为`200`：

```
[AR2]route-policy test2 permit node 10
[AR2-route-policy]apply cost 200
[AR2-route-policy]quit

[AR2]bgp 65001
[AR2-bgp]aggregate 172.16.0.0 16 detail-suppressed attribute-policy test2
[AR2-bgp]quit
```

![image-20250829170654628](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339861.png)

------

### 4.复位BGP

在上述的很多实验中我们发现一个现象，当我给路由添加或修改了路径属性后，该属性在对等体的路由表中没有立马生效。这个现象的原因是因为`BGP`不是**周期性更新**的，而你修改路径属性的操作也不会使本机向外通告路由。

此时，我们就可以通过手动复位`BGP`的方式，让对方或本机再次通告`BGP`路由，复位分为两种：

- **硬复位：**通过拆除并重新建立`TCP`会话来实现。
  - **reset bgp all：**该命令将复位路由器的所有`BGP`连接。
  - **reset bgp as-number：**该命令将复位路由器与特定`AS`的`BGP`连接。
  - **reset bgp peer-address：**该命令将复位路由器与特定对等体的`BGP`连接。
  - **reset bgp internal：**该命令将复位路由器的所有`IBGP`连接。
  - **reset bgp external：**该命令将复位路由器的所有`EBGP`连接。
- **软复位：**在不中断`TCP`会话的情况下，重新通告所有路由。
  - **refresh bgp all import**: 该命令将使路由器的所有`BGP`连接在入方向触发软复位。
  - **refresh bgp all export**: 该命令将使路由器的所有`BGP`连接在出方向触发软复位。
  - **reset bgp peer-address import**: 该命令将使路由器针对特定的对等体在入方向触发软复位。
  - **refresh bgp external import**: 该命令将使路由器的所有`EBGP`连接在入方向触发软复位。
  - **refresh bgp external export**: 该命令将使路由器的所有`EBGP`连接在出方向触发软复位。
  - **refresh bgp internal import**: 该命令将使路由器的所有`IBGP`连接在入方向触发软复位。
  - **refresh bgp internal export**: 该命令将使路由器的所有`IBGP`连接在出方向触发软复位。
- `import`用在从邻居收到的路由执行策略时的复位。
- `export`用在对发送给邻居的路由执行策略时的复位。

------

在路由上使用`display bgp peer verbose`命令，可查看`BGP`对等体是否支持`Route-refresh`特性：

![image-20250831174612363](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339884.png)

------

### 5.IBGP防环

通告上述的学习我们已经了解到`EBGP`的防环是使用`AS_PATH`路径属性实现的，而`IBGP`的防环是使用**水平分割**实现的。**水平分割**导致路由器无法将从`IBGP`对等体学习到的路由传递给另一个`IBGP`对等体，如果想解决这个问题可以使用**全互联**。但是**全互联**在`AS`内`BGP`路由器数量较多的情况下，会增加设备的负担、降低网络的扩展性。

![image-20250831190607112](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339941.png)

当前`BGP`提供了两种方式来解决**水平分割**的问题：

- **路由反射器**
- **联邦**

------

#### 5.1.路由反射器

路由反射器（`Route Reflector,RR`）是一种用于解决`AS`内部`BGP`路由传递问题的技术，在一些大型的`BGP`组网中常被应用。如下图的例子中就可以使用**路由反射器**解决，将`R1`指定为路由反射器，而`R2、R3`被指定为它的客户（`client`），此时`R1`便会将自己学习到的`IBGP`路由在遵循规则的情况下进行**反射**。

![image-20250831191042352](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135339998.png)

我们将`RR`以及它的`client`所构成的系统称为**路由反射簇(Cluster)**，如下图中`R2、R3、R4、R5`就组成了一个`Cluster`，`RR`与所有的客户建立`IBGP`对等体关系，而`client`之间则无需建立`IBGP`对等体关系，优化了网络中`IBGP`对等体关系数量。实际上，所有的路由反射器配置都是在`RR`路由器上完成的，而`client`是不需要做任何额外配置的，甚至`client`并不知道自己是`RR`的`client`。所以，`RR`对比联邦配置非常简单。

![](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340022.png)

------

##### 5.1.1.规则

在路由反射器场景中，有以下几种角色：

- `RR`：路由反射器
- `client`：路由反射器的客户
- 非`client`

**路由反射器**并不是在任何场景下都会将`BGP`路由进行反射的，否则`IBGP`路由的传递将变得混乱不堪。

- 如果`RR`从自己的`client`学习到一条`IBGP`路由，则它会将路由反射给所有除该对等体之外的所有`IBGP`邻居。
  - 在`BGP`视图下执行`undo reflect between-clients`命令可关闭路由在客户之间的反射行为。

- 如果`RR`从自己的非`client`学习到一条`IBGP`路由则它会将路由反设备所有除该`client`之外的所有`IBGP`邻居。

- 当路由反射器执行路由反射时，它只将自己使用的、最优的`BGP`路由进行反射。

**总结：**非非互传，`RR`不会将非`client`的路由传递给非`client`。

------

##### 5.1.2.路由环路

**路由反射器**解决的是**水平分割**产生的路由不传递问题，但是**环路问题**怎么解决呢？

`RR`的设定使得`IBGP`水平分割原则失效，这就可能导致环路的产生，为此`RR`会为`BGP`路由添加两个特殊的路径属性来避免出现环路：

- `Originator_ID`
- `Cluster_List`

这两个属性都属于**可选非过渡**类型。

------

**Originator_ID**

`Originator_ID`是一个**可选非过渡**属性，该属性长度为`32bit`，其格式与`ipv4`地址格式相同。

当一条`BGP`路由被`RR`反射给其他路由器时，如果该条路由已经携带了`Originator_ID`属性，则保留该属性，否则`RR`为这条路有添加`Originator_ID`属性，并将该属性设置为该路由在本地`AS`内的始发路由器的`Router-ID`。

当路由器从对等体收到一条`IBGP`路由，并且该路由所携带的`Originator_ID`与自己的`BGP Router-ID`相同时，它意识到从自己始发的路由又被通告回来了，它就会忽略这条路由的更新。

![image-20250831213603945](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340037.png)

如上图，假设`R3`和`R2`都是`RR`，当`R1`通告`10.1.0.0/16`的路由后，`R3`会将这条路由加上`Originator_ID=1.1.1.1`并传递给邻居`R2`，当`R2`收到这条路由后，因为`Originator_ID`已经在这条路由上存在，所以他会原封不动的给到`R1`。`R1`收到后发现`Originator_ID`与自己的`Router-ID`一致，忽略这条路由。

------

**Cluster_List**

`Cluster_List`也是一个**可选非过渡**属性，该属性的值是可变长的，它可以包含一个或者多个`Cluster-ID`（路由反射簇标识符）。

路由反射簇是路由反射器与其客户一起构成的，在一个`AS`内可以有多个路由反射簇，每个路由反射簇都拥有自己的`Cluster-ID`。

`Cluster-ID`是一个可配置的`32bit`数值，缺省为`RR`的`Router-ID`。当一条`BGP`路由被`RR`反射时，如果该条路由已经存在`Cluster_List`属性，那么`RR`会将本地的`Cluster-ID`添加到路由的`Cluster_List`属性值之前，而如果该路由并不存在`Cluster_List`，那么`RR`会为其创建并将`Cluster-ID`添加上去。

当`RR`收到一条`BGP`路由后，若发现该条路由携带`Cluster_List`属性，并且`Cluster_List`属性值中包含着自己的`Cluster-ID`时，它意识到自己反射出去的`BGP`路由又被通告回来了，此时它将忽略关于这条路由的更新。

![image-20250831214603434](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340032.png)

如上图，`R1、R2、R3`是`RR`，`R4`是`R1`的`client`，`R2`是`R3`的`client`，`R1`是`R2`的`client`。此时`R4`通告路由`10.1.0.0/16`给`R1`，`R1`会为该路由添加`Originator_ID=4.4.4.4`和`Cluster_List=1.1.1.1`属性，然后传递给`R3`。`R3`收到后，发现该路由已经存在`Cluster_List`遂将自己的`Cluster-ID`添加到`Cluster_List`列表中，然后传递给`R2`。`R2`收到后也重复该动作，`R1`收到后发现`Cluster_List`中存在自己的`Cluster-ID`，所以将该路由忽略。

需要注意的是，当`RR`收到`EBGP`邻居的路由后，并不会为该路由创建`Originator_ID`和`Cluster_List`属性，因为这不是路由反射的行为，而是一个正常通告行为。

另外，当携带`Originator_ID`和`Cluster_List`属性的路由通告给`EBGP`对等体时，这条路有的这两个属性会被移除。

------

##### 5.1.3.实验

如下图，`AR3`理所当然的是`RR`，但是你至少要为其配置两个`client`，此例中`AR2`和`AR4`为`client`。在`AR1`通告`192.168.1.0/24`的路由，最终观察是否在`AR4`和`AR5`上学到，并且是否有路由属性。

![image-20250831220213025](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340068.png)

**步骤一：配置基本BGP**

```
[AR1]bgp 100
[AR1-bgp]router-id 1.1.1.1
[AR1-bgp]peer 12.1.1.2 as-number 200
[AR1-bgp]network 192.168.1.0 24
[AR1-bgp]quit

[AR2]bgp 200
[AR2-bgp]router-id 2.2.2.2
[AR2-bgp]peer 12.1.1.1 as-number 100
[AR2-bgp]peer 23.1.1.2 as-number 200
[AR2-bgp]peer 23.1.1.2 next-hop-local
[AR2-bgp]quit

[AR3]bgp 200
[AR3-bgp]router-id 3.3.3.3
[AR3-bgp]peer 23.1.1.1 as-number 200
[AR3-bgp]peer 34.1.1.2 as-number 200
[AR3-bgp]peer 35.1.1.2 as-number 200
[AR3-bgp]quit

[AR4]bgp 200
[AR4-bgp]router-id 4.4.4.4
[AR4-bgp]peer 34.1.1.1 as-number 200
[AR4-bgp]quit

[AR5]bgp 200
[AR5-bgp]router-id 5.5.5.5
[AR5-bgp]peer 35.1.1.1 as-number 200
[AR5-bgp]quit
```

**步骤二：配置路由反射器**

```
[AR3-bgp]peer 23.1.1.1 reflect-client 
[AR3-bgp]peer 34.1.1.2 reflect-client
```

**验证：**

因为我在做的时候用的都是直连接口建立的对等体关系，所以`AR3`通告给`AR4`和`AR5`的路由下一跳默认是`R1`建立对等体的地址。

![image-20250831221525384](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340122.png)

![image-20250831221928780](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340200.png)

此时，观察路由的详细信息中，可发现`Originator=2.2.2.2`，`Cluster_List=3.3.3.3`。

------

#### 5.2.联邦

除了路由反射器可以解决**水平分割**带来的路由传递问题之外，**联邦**也可以达到同样的效果。联邦（`Confederation`）也被称为**联盟**，大致的思想是在一个大的`AS`内创建若干个小的`AS`（类似于子`AS`），是的`AS`内部出现一种特殊的`EBGP`对等体关系，从而解决`IBGP`路由在`AS`内的传递问题。

------

![image-20250831222813052](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340204.png)

如上图，`R3`收到收到`R1`的路由后，会将该路由传递给`R4`，而`R4`因为水平分割不会将该路由传递给`R5`。如果需要用**联邦**解决该问题的话，需要将其分割成多个子`AS`：

![image-20250831222939988](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340219.png)

如上图，`AS 3456`由两个子`AS`组成，`R3`收到`R1`的路由后传递给`IBGP`对等体`R4`，`R4`将该路由传递给自己的`EBGP`对等体`R5`，随后`R5`将该路由传递给自己的`EBGP`和`IBGP`对等体，从而实现将路由传递到整个`BGP`。

需要注意，`AS 3456`内部部署联邦时，`R3、R4、R5、R6`创建`BGP`进程时所使用的`AS`号是其所属的成员`AS`号。在它们与`AS`外部的路由器建立对等体关系时，实际上用的是`AS 3456`，也就是联邦`AS`。

从这里你就能感觉到，联邦需要你一开始就细致的规划好你的整个拓扑，不是你一拍脑门子就能配置的。

------

##### 5.2.1.AS_PATH

当家一定还记得着重介绍的`AS_Path`路径属性，该属性的作用是`EBGP`防环与选路。那在联邦的场景中多个子`AS`之间是不是也可以利用`AS_Path`属性进行防环呢？

`BGP`设计了四种`AS_Path`类型：

- `AS_Sequence`
- `AS_Set`
- `AS_Confed_Sequence`
- `AS_Confed_Set`

前面两种前文已经介绍过了。联邦使用了后面两种类型，它们的作用和前两种类似，只不过它们被用于联邦。

当路由在联邦`EBGP`对等体之间传递时，子`AS`号被写入这种特殊类型的`AS_Path`片段中。而当路由被传出联邦时，成员`AS`号会从`AS_Path`里移除。

![image-20250831224215407](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340244.png)

1. `R1`向`EBGP`对等体`R3`通告`10.10.0.0/16`路由条目，此时该路由的`AS_Path`属性值为`100`，该`AS_Path`属性的类型是`AS_Sequence`。
2. `R3`将该`EBGP`路由传递给`IBGP`对等体`R4`，此时的`AS_Path`值依旧为`100`。

3. 当`R4`将该路由传递给它的联邦`EBGP`对等体`R5`时，将在路由原有`AS_Path`属性的基础上，增加一个`AS_Confed_Sequence`类型的片段（列表）值为`64512`。
4. `R5`收到后这条路由的`AS_Path`为`(64512) 100`。
   - 发送给`R6`时需要在`AS_Confed_Sequence`片段上增加自己的成员`AS`号`64513`，此时这条路由的`AS_Path`为`(64513 64512) 100`。
   - 发送给`EBGP`对等体`R2`时，因为其不是联邦成员，所以`R5`会将`AS_Path`属性中的`AS_Confed_Sequence`片段移除，并在`AS_Sequence`中增加联邦`AS`号`3456`，此时这条路由的`AS_Path`为`3456 100`。

------

##### 5.2.2.实验

![image-20250831225637633](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340250.png)

**命令分析：**

配置联邦时会涉及到两条命令：

- `confederation id`：该命令用来指定当前的**联邦AS**。
- `COnfederation peer-as`：该命令用来指定同属一个联邦的邻居成员`AS`，如果路由器不与其他成员`AS`建立`EBGP`邻居关系，则不需要配置。

- `AR1`

```
[AR1]int g0/0/0
[AR1-GigabitEthernet0/0/0]ip add 13.1.1.1 30
[AR1-GigabitEthernet0/0/0]quit

[AR1]int loo0
[AR1-LoopBack0]ip add 192.168.1.1 24
[AR1-LoopBack0]quit

[AR1]bgp 100
[AR1-bgp]router-id 1.1.1.1
[AR1-bgp]peer 13.1.1.2 as 300
[AR1-bgp]network 192.168.1.0 24
[AR1-bgp]quit
```

- `AR2`

```
[AR2]int g0/0/0
[AR2-GigabitEthernet0/0/0]ip add 25.1.1.1 30
[AR2-GigabitEthernet0/0/0]quit

[AR2]bgp 200
[AR2-bgp]router-id 2.2.2.2
[AR2-bgp]peer 25.1.1.2 as-number 300
[AR2-bgp]quit
```

- `AR3`

```
[AR3]int g0/0/0
[AR3-GigabitEthernet0/0/0]ip add 13.1.1.2 30
[AR3-GigabitEthernet0/0/0]quit

[AR3]int g0/0/1
[AR3-GigabitEthernet0/0/1]ip add 34.1.1.1 30
[AR3-GigabitEthernet0/0/1]quit

[AR3]int loo0
[AR3-LoopBack0]ip add 3.3.3.3 32
[AR3-LoopBack0]quit

[AR3]ospf 1 router-id 3.3.3.3
[AR3-ospf-1]area 0
[AR3-ospf-1-area-0.0.0.0]network 34.1.1.1 0.0.0.0
[AR3-ospf-1-area-0.0.0.0]network 3.3.3.3 0.0.0.0
[AR3-ospf-1-area-0.0.0.0]quit
[AR3-ospf-1]quit

[AR3]bgp 64512
[AR3-bgp]router-id 3.3.3.3
[AR3-bgp]confederation id 300
[AR3-bgp]peer 13.1.1.1 as 100
[AR3-bgp]peer 4.4.4.4 as 64512	
[AR3-bgp]peer 4.4.4.4 connect-interface loo0	
[AR3-bgp]peer 4.4.4.4 next-hop-local
[AR3-bgp]quit
```

- `AR4`

```
[AR4]int g0/0/0
[AR4-GigabitEthernet0/0/0]ip add 34.1.1.2 30
[AR4-GigabitEthernet0/0/0]quit

[AR4]int g0/0/1
[AR4-GigabitEthernet0/0/1]ip add 45.1.1.1 30
[AR4-GigabitEthernet0/0/1]quit

[AR4]int loo0
[AR4-LoopBack0]ip add 4.4.4.4 32
[AR4-LoopBack0]quit

[AR4]ospf 1 router-id 4.4.4.4
[AR4-ospf-1]area 0
[AR4-ospf-1-area-0.0.0.0]network 34.1.1.2 0.0.0.0
[AR4-ospf-1-area-0.0.0.0]network 45.1.1.1 0.0.0.0
[AR4-ospf-1-area-0.0.0.0]network 4.4.4.4 0.0.0.0
[AR4-ospf-1-area-0.0.0.0]quit
[AR4-ospf-1]quit

[AR4]bgp 64512
[AR4-bgp]router-id 4.4.4.4
[AR4-bgp]confederation id 300		
[AR4-bgp]confederation peer-as 64513
[AR4-bgp]peer 3.3.3.3 as 64512
[AR4-bgp]peer 3.3.3.3 connect-interface loo0
[AR4-bgp]peer 5.5.5.5 as 64513	
[AR4-bgp]peer 5.5.5.5 connect-interface loo0
[AR4-bgp]peer 5.5.5.5 ebgp-max-hop 2
[AR4-bgp]quit
```

- `AR5`

```
[AR5]int g0/0/0
[AR5-GigabitEthernet0/0/0]ip add 45.1.1.2 30
[AR5-GigabitEthernet0/0/0]quit

[AR5]int g0/0/1
[AR5-GigabitEthernet0/0/1]ip add 56.1.1.1 30
[AR5-GigabitEthernet0/0/1]quit

[AR5]int g0/0/2
[AR5-GigabitEthernet0/0/2]ip add 25.1.1.2 30
[AR5-GigabitEthernet0/0/2]quit

[AR5]int loo0
[AR5-LoopBack0]ip add 5.5.5.5 32
[AR5-LoopBack0]quit

[AR5]ospf 1 router-id 5.5.5.5
[AR5-ospf-1]area 0
[AR5-ospf-1-area-0.0.0.0]network 45.1.1.2 0.0.0.0
[AR5-ospf-1-area-0.0.0.0]network 56.1.1.1 0.0.0.0
[AR5-ospf-1-area-0.0.0.0]network 5.5.5.5 0.0.0.0
[AR5-ospf-1-area-0.0.0.0]quit
[AR5-ospf-1]quit

[AR5]bgp 64513
[AR5-bgp]router-id 5.5.5.5
[AR5-bgp]confederation id 300	
[AR5-bgp]confederation peer-as 64512
[AR5-bgp]peer 4.4.4.4 as 64512
[AR5-bgp]peer 4.4.4.4 connect-interface loo0
[AR5-bgp]peer 4.4.4.4 ebgp-max-hop 2
[AR5-bgp]peer 4.4.4.4 next-hop-local
[AR5-bgp]peer 6.6.6.6 as 64513
[AR5-bgp]peer 6.6.6.6 connect-interface loo0
[AR5-bgp]peer 6.6.6.6 next-hop-local
[AR5-bgp]peer 25.1.1.1 as 200
[AR5-bgp]quit
```

- `AR6`

```
[AR6]int g0/0/0
[AR6-GigabitEthernet0/0/0]ip add 56.1.1.2 30
[AR6-GigabitEthernet0/0/0]quit

[AR6]int loo0
[AR6-LoopBack0]ip add 6.6.6.6 32
[AR6-LoopBack0]quit

[AR6]ospf 1 router-id 6.6.6.6
[AR6-ospf-1]area 0
[AR6-ospf-1-area-0.0.0.0]network 6.6.6.6 0.0.0.0
[AR6-ospf-1-area-0.0.0.0]network 56.1.1.2 0.0.0.0
[AR6-ospf-1-area-0.0.0.0]quit
[AR6-ospf-1]quit

[AR6]bgp 64513
[AR6-bgp]router-id 6.6.6.6
[AR6-bgp]confederation id 300	
[AR6-bgp]peer 5.5.5.5 as 64513
[AR6-bgp]peer 5.5.5.5 connect-interface loo0
[AR6-bgp]quit
```

![image-20250831233036002](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340302.png)

![image-20250831233049138](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340425.png)

------

### 6.BGP路由优选

`BGP`是一个应用非常广泛且用于大规模网络场景下的路由协议，`BGP`定义了多种路径属性，并且拥有丰富的路由策略工具，这些特点使得`BGP`在路由操控和路径决策上非常灵活。

当一台路由器学习到多条到达相同目的网段的`BGP`路由时，它将进行路由优选的决策，在这些路由中选择出一条最优的路由，如下图：

![image-20250901144233537](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340418.png)

------

`BGP`定义了一整套详细的**最优路径选择算法（`Best Path Selection Algorithm`）**，这使得`BGP`路由器能够在任何复杂的、高冗余性的网络环境下选择出路由路径。当到达同一个目的网段存在多条路由时，`BGP`会通过**最优路径算法**定义的次序进行路由选路。

但是，任何一条`BGP`路由在参与优选之前都必须先经过检查。设备会检查`BGP`路由的`Next_Hop`是否可达，如果不可达，则`BGP`路由被视为不可用，该路由无论如何不会被优选，也不会被设备使用或通告给其他对等体。

#### 6.1.最优路径选择算法

1. 优选`Preferred_Value`属性值最大的路由。
2. 优选`Local_Preference`属性值最大的路由。
3. 本地始发的`BGP`路由优于从其他对等体学习到的路由。
   - 本地始发的路由类型按优先级从高到低的排列顺序为：
     - 通过手工汇总的方式发布的路由
     - 通过自动汇总的方式发布的路由
     - 通过`network`命令发布的路由
     - 通过`import-route`命令发布的路由
4. 优选`AS_Path`属性值最短的路由。
5. 优选`Origin`属性最优的路由，按优先级从高到低的排列顺序为：
   1. `IGP`
   2. `EGP`
   3. `Incomplete`
6. 优选`MED`属性值最小的路由。
7. 优选从`EBGP`对等体学来的路由（`EBGP`路由优先级高于`IBGP`路由）。
8. 优选到`Next_Hop`的`IGP`度量值最小的路由。

9. 优选`Cluster_List`最短的路由。
10. 优选`Router-ID(Originator_ID)`最小的设备通告的路由。
11. 优选具有最小`IP`地址（`Peer`命令所指定的地址）的对等体通告的路由。

这些规则依序排列，`BGP`进行路由优选时，从第一条规则开始执行。如果根据第一条规则无法做出判断（值一样），则继续执行下一条规则。如果当前的规则可以决策出最优的路由，则不再继续往下执行。

------

#### 6.2.案例1：Preferred_Value

当到达同一个目的网段存在多条`BGP`路由时，路由器首先会比较这些路由的`Preferred_Value`属性值，优选`Preferred_Value`属性值最大的路由。

这是一个华为私有的路径属性，只在本地有效，这个属性值不会被传递给任何`BGP`对等体。

接下来会使用`EBGP`邻居进 行演示，但是`Preferred_value`也可以使用在`IBGP`之间。

![image-20250901151026397](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340422.png)

如上图，`R4`与`R2`和`R3`分别建立了`EBGP`邻居，现要求`R4`学习到的`192.168.10.0/24`这条路由有选`AR3`，通过`Preferred_Value`属性控制。

![image-20250901152351749](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340471.png)

此时，可观察到虽然当前`R4`优选`R2`为下一跳，但当前的`PrefVal`为`0`。通过命令使`R4`通过`PrefVal`优选`R3`作为下一跳。

```
[AR4]bgp 400
[AR4-bgp]peer 34.1.1.1 preferred-value 6000
[AR4-bgp]quit
```

由于`PrefVal`是一个本地属性，所以不需要路由刷新。

![image-20250901152619375](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340436.png)

![image-20250901152753869](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340482.png)

上图中很明确告诉我们当前从`34.1.1.1`学习到的路由是最优的因为`Pref-Val`，而`24.1.1.1`学习到的不是最优的因为`not preferred for PreVal`。

上例中，非常粗暴的针对`R3`过来的路由全部修改了`Pref-Val`，我们也可以针对某些路由进行修改，使用`Route-Policy`即可。

```
ip ip-prefix preferred permit 192.168.10.0 24
route-policy preferred permit node 10
	if-match ip-prefix preferred
	apply preferred-value 6000
	quit
route-policy preferred permit node 20
	quit
bgp 400
	peer 34.1.1.1 route-policy preferred import
```

------

#### 6.3.案例2：Local_Preference

当到达同一个目的网段存在多条`BGP`路由时，在其他条件相同的情况下，`BGP`将优选这些路由中`Local_Preference`属性值最大的路由。如果路由在传递到本地时并不携带`Local_Preference`属性，则`BGP`在决策时使用缺省的`Local_Preference`属性值（100）来计算，这个缺省值可以使用`default local-preference`命令修改。

`Local_Preference`（本地优先级），该属性是**公认任意属性**，可以用于告诉`AS`中的路由器，哪条路径是离开本`AS`的首选路径。该值越大则`BGP`路由越优，缺省情况下值为`100`。==该属性只能传递给`IBGP`对等体，不能传递给`EBGP`对等体。==

但是，如果在`EBGP`的场景中，使用`import`方向也可以做到使用`Local_Preference`优选路由。

![image-20250901155507904](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340616.png)

**修改之前：**

![image-20250901161130187](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340596.png)

**方法一：出方向**

要求在`AR3`上通告给`AR1`的`10.0.45.0/24`这条路由的`LocPrf`为`200`：

```
[AR3]ip ip-prefix test1 permit 10.0.45.0 24

[AR3]route-policy test1 permit node 10	
[AR3-route-policy]if-match ip-prefix test1
[AR3-route-policy]apply local-preference 200
[AR3-route-policy]quit

[AR3]route-policy test1 permit node 20
[AR3-route-policy]quit

[AR3]bgp 200
[AR3-bgp]peer 1.1.1.1 route-policy test1 export 
[AR3-bgp]quit
```

![image-20250901161416292](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340583.png)

![image-20250901161449871](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340619.png)

**方法二：入方向**

要求在`AR1`收到`AR2`发送的`10.0.45.0/24`这条路由的`LocalPref`为`300`：

```
[AR1]ip ip-prefix test1 permit 10.0.45.0 24

[AR1]route-policy test1 permit node 10	
[AR1-route-policy]if-match ip-prefix test1
[AR1-route-policy]apply local-preference 300
[AR1-route-policy]quit

[AR1]route-policy test1 permit node 20
[AR1-route-policy]quit

[AR1]bgp 200
[AR1-bgp]peer 2.2.2.2 route-policy test1 import 
[AR1-bgp]quit
```

![image-20250901161642011](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340658.png)

------

#### 6.4.案例3：本地始发最优

![image-20250901155507904](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340616.png)

##### 6.4.1.本地最优

```
[AR1]int loo100
[AR1-LoopBack100]ip add 100.100.100.1 24
[AR1-LoopBack100]quit

[AR2]int loo100
[AR2-LoopBack100]ip add 100.100.100.2 24
[AR2-LoopBack100]quit

[AR3]int loo100
[AR3-LoopBack100]ip add 100.100.100.3 24
[AR3-LoopBack100]quit

[AR1]bgp 200
[AR1-bgp]network 100.100.100.0 24
[AR1-bgp]quit

[AR2]bgp 200
[AR2-bgp]network 100.100.100.0 24
[AR2-bgp]quit

[AR3]bgp 200
[AR3-bgp]network 100.100.100.0 24
[AR3-bgp]quit
```

![image-20250901162332501](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340707.png)

------

##### 6.4.2.本地始发中最优选的

此时，在`AR1`上创建一个回环口`200.200.200.200/32`。

```
[AR1]int loo200
[AR1-LoopBack200]ip add 200.200.200.200 32
[AR1-LoopBack200]quit
```

**import-route**

```
[AR1]bgp 200	
[AR1-bgp]import-route direct 
```

![image-20250901162722956](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340760.png)

**network**

```
[AR1]bgp 200
[AR1-bgp]network 200.200.200.200 32
[AR1-bgp]quit
```

![image-20250901162805281](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340794.png)

**手动汇总**

```
[AR1]bgp 200	
[AR1-bgp]aggregate 200.200.0.0 255.255.0.0 detail-suppressed 
[AR1-bgp]quit

[AR1]int loo210
[AR1-LoopBack210]ip add 200.200.0.1 16

[AR1]bgp 200
[AR1-bgp]network 200.200.0.0 16
[AR1-bgp]quit
```

![image-20250901163132717](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340801.png)

**手工汇总与自动汇总**

```
[AR1]int loo1
[AR1-LoopBack1]ip add 10.100.1.1 24
[AR1-LoopBack1]quit
[AR1]int loo2
[AR1-LoopBack2]ip add 10.200.1.1 24
[AR1-LoopBack2]quit

[AR1-bgp]summary automatic
```

![image-20250901163717340](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340798.png)

```
[AR1-bgp]aggregate 10.0.0.0 8 detail-suppressed
```

![image-20250901163836889](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340852.png)

从这里你没有办法判断出，哪一条是手工，哪一条是自动。

![image-20250901163935216](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340881.png)

------

#### 6.5.案例4：AS_Path

优选`AS_Path`属性值最短的路由，在`AR2`配置`Route-Policy`使其通告给对等体`AR1`的`10.0.45.0/24`这条路有的`AS_Path=400 100`

![](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340963.png)

```
[AR2]ip ip-prefix as_path permit 10.0.45.0 24

[AR2]route-policy as_path permit node 10
[AR2-route-policy]if-match ip-prefix as_path
[AR2-route-policy]apply as-path 400 additive
[AR2-route-policy]quit

[AR2]bgp 200
[AR2-bgp]peer 1.1.1.1 route-policy as_path export
[AR2-bgp]quit
```

![image-20250901181825892](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340988.png)

------

#### 6.6.案例5：Origin

![image-20250901233447943](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135340982.png)

**修改前：**

![image-20250901233549562](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341003.png)

当前`R4`和`R5`宣告`10.0.45.0/24`的方法都是`network`，所以无法使用`Origin`选路。故将`R4`修改为`import-route`方式：

```
[AR4]bgp 100
[AR4-bgp]undo network 10.0.45.0 24
[AR4-bgp]import-route direct 
[AR4-bgp]quit
```

**修改后：**

![image-20250901233623928](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341048.png)

------

#### 6.7.案例6：MED

默认情况下`BGP`只会对来自同一个`AS`的相同路由比较`MED`值，可以通过命令：`compare-different-as-med`开启来自不同`AS`的相同路由也比较`MED`值。

![image-20250901234401213](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341075.png)

现在将`R4`修改为`network`方式宣告路由`10.0.45.0/24`。

**修改前：**

![image-20250901234442700](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341151.png)

由于当前的`10.0.45.0`是通过`network`宣告直连口发布的，所以`MED`为`0`。

由于`MED`值一般用于影响`EBGP`对等体进入本`AS`的路径，所以在`R4`和`R5`上进行`MED`值配置。现要求`R4`配置为`200`，`R5`配置为`100`。

```
[AR4]ip ip-prefix med permit 10.0.45.0 24

[AR4]route-policy med permit node 10	
[AR4-route-policy]if-match ip-prefix med	
[AR4-route-policy]apply cost 200
[AR4-route-policy]quit
[AR4]route-policy med permit node 20
[AR4-route-policy]quit

[AR4]bgp 100
[AR4-bgp]peer 24.1.1.1 route-policy med ex	
[AR4-bgp]peer 24.1.1.1 route-policy med export 
[AR4-bgp]quit

[AR5]ip ip-prefix med permit 10.0.45.0 24

[AR5]route-policy med permit node 10	
[AR5-route-policy]if-match ip-prefix med
[AR5-route-policy]apply cost 100
[AR5-route-policy]quit
[AR5]route-policy med permit node 20
[AR5-route-policy]quit

[AR5]bgp 300
[AR5-bgp]peer 35.1.1.1 route-policy med export 
[AR5-bgp]quit
```

![image-20250901235051677](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341155.png)

此时，`AR1`的路由表中依旧是优选`2.2.2.2`作为下一跳，说明没有比较`MED`值。需要使用如下命令：

```
[AR1]bgp 200
[AR1-bgp]compare-different-as-med
```

![image-20250901235212047](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341192.png)

------

#### 6.8.案例7：优选EBGP

该规则其实非常好理解，当路由前缀和掩码相同的情况下，如果前面的规则一致，优先选择`EBGP`路由。

![image-20250901235532810](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341181.png)

还是用该拓扑为例，在`AR1`上配置静态路由`10.0.45.0/24`通过`network`通告对等体，并且为其配置`AS_Path`。

```
[AR1]ip route-static 10.0.45.0 24 null0

[AR1]ip ip-prefix ebgp permit 10.0.45.0 24

[AR1]route-policy ebgp permit node 10	
[AR1-route-policy]if-match ip-prefix ebgp
[AR1-route-policy]apply as-path 500 additive 
[AR1-route-policy]quit

[AR1]route-policy ebgp permit node 20
[AR1-route-policy]quit

[AR1]bgp 200
[AR1-bgp]network 10.0.45.0 24	
[AR1-bgp]peer 2.2.2.2 route-policy ebgp export 
[AR1-bgp]quit
```

- 为什么要`network`呢？为了避免`Origin`选路规则。

  - 如果不`network`会出现如下现象：

  ![image-20250902000207102](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341241.png)

- 为什么要添加`AS_Path`呢？为了避免`AS_Path`选路规则。

  - 如果不添加`AS_Path`会出现如下现象：

  ![image-20250902000137406](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341248.png)

**最终现象：**

![image-20250902000229514](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341347.png)

------

#### 6.9.案例8：IGP Cost

该规则为优选到`Next_Hop`的`IGP`度量值最小的路由。

**恢复配置：**

![image-20250902000357404](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341333.png)

想要实现该规则非常简单，当前`AR1`到达`AR2`和`AR3`的`BGP`对等体源地址的`Cost`是一样的，当前在未修改的情况下优选`2.2.2.2`，我只需将`AR1`的`G0/0/0`接口的`OSPF Cost`修改为`100`即可。

```
[AR1]int g0/0/0
[AR1-GigabitEthernet0/0/0]ospf cost 100
[AR1-GigabitEthernet0/0/0]quit
```

![image-20250902000620850](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341345.png)

![image-20250902000637462](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341365.png)

------

#### 6.10.BGP路由负载分担

在大型网络中，到达同一目的地通常会存在多条有效`BGP`路由，设备只会优选一条最优的`BGP`路由，将该路由加载到路由表中使用，这一特点往往会造成很多流量负载不均衡的情况。

从`BGP`路由优选的规则中你就能看出来，如果不人工干预最终肯定不会形成负载均衡。所以，负载均衡需要在`BGP`视图中手动开启，命令为：`maximum load-balancing ibgp 2`。

虽然开启了负载均衡，但是该命令的作用仅仅是使`BGP`可以将多条相同前缀和掩码的路由加入到路由表。当`BGP`路由器要将路由通告给对等体时，依旧会根据路由优选规则选择出最优选的通告给对等体。

在设备上使能`BGP`负载分担功能后，只有满足条件的多条`BGP`路由才会成为等价路由，进行负载分担。

##### 6.10.1.负载分担条件

1. `Preferred-Value`属性值相同。
2. `Local_Preference`属性值相同。
3. 都是聚合路由或者非聚合路由
4. `AS_Path`属性长度相同。
5. `Origin`类型相同。
6. `MED`属性值相同。
7. 都是`EBGP`路由或都是`IBGP`路由。
8. `AS`内部`IGP`的`Metric`相同。
9. `AS_Path`属性完全相同。

上面说到的负载分担条件，并不是按照顺序排列的。实际上如果你仔细看了路由优选规则你就会发现，前八个才是真正“有意义的“。而最后几个只是单纯为了”决出个高下”。

------

##### 6.10.2.实验

![image-20250902001628925](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341443.png)

依旧是这张拓扑图，恢复环境后查看路由表：

![image-20250902001659390](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341461.png)

在`AR1`上开启负载均衡功能。

```
[AR1]bgp 200
[AR1-bgp]maximum load-balancing ibgp ?
  INTEGER<1-8>  Specify maximum equal cost routes
[AR1-bgp]maximum load-balancing ibgp 2
[AR1-bgp]quit
```

此时，并不会出现负载均衡现象，为啥呢？

- 因为`AR1`学习到的两条路有的`AS_Path`不是完全相同的。

```
[AR3]ip ip-prefix load permit 10.0.45.0 24

[AR3]route-policy load permit node 10	
[AR3-route-policy]if-match ip-prefix load	
[AR3-route-policy]apply as-path 100 overwrite 
Warning: The AS-Path lists of routes to which this route-policy is applied will 
be overwritten. Continue? [Y/N]y
[AR3-route-policy]quit

[AR3]route-policy load permit node 20
[AR3-route-policy]quit

[AR3]bgp 200
[AR3-bgp]peer 1.1.1.1 route-policy load export 
[AR3-bgp]quit
```

![image-20250902002406949](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341522.png)

此时再查看`AR1`的全局路由表：

![image-20250902002444193](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341519.png)

------

#### 6.11.案例9：优选Cluster_List最短的路由

![image-20250902002637660](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341548.png)

为了实现该实验，需要改动一下拓扑：

- 只在`AR5`上将`10.0.45.0/24`发布到`BGP`。
- 配置`R1`为路由反射器，`R3`为`R1`的客户端。
- `R2、R3`之间基于回环口建立`IBGP`对等体关系。

通过上述更改后，`R2`将收到`R3`通告的`BGP`路由`10.0.45.0/24`以及`R1`反射的`BGP`路由`10.0.45.0/24`。

```
[AR2]int g0/0/1	
[AR2-GigabitEthernet0/0/1]shutdown 
[AR2-GigabitEthernet0/0/1]quit

[AR2]bgp 200
[AR2-bgp]peer 3.3.3.3 as 200
[AR2-bgp]peer 3.3.3.3 connect-interface LoopBack 0
[AR2-bgp]quit

[AR3]bgp 200
[AR3-bgp]peer 2.2.2.2 as 200	
[AR3-bgp]peer 2.2.2.2 connect-interface loo0
[AR3-bgp]peer 2.2.2.2 next-hop-local
[AR3-bgp]quit

[AR1]bgp 200
[AR1-bgp]peer 2.2.2.2 reflect-client 
[AR1-bgp]peer 3.3.3.3 reflect-client 
[AR1-bgp]quit
```

做完这些之后，查看`BGP`路由表会发现下一跳也是完全一致的，此时你需要查看详细信息：

![image-20250902003500375](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341598.png)

![image-20250902003524637](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341622.png)

------

#### 6.12.案例10：优选Router-ID

此规则为优先选择`Router-ID`最小的对等体通告的路由。

![image-20250902003724931](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341630.png)

恢复到最初的环境，查看最优路由详细信息：

![image-20250902003812859](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341730.png)

如上图，你什么都不用改就会发现当前选择最优路由的依据是`Router-ID`。

此时，将`AR2`的`Router-ID`修改为`100.100.100.100`，然后再观察现象：

```
[AR2]bgp 200
[AR2-bgp]router-id 100.100.100.100
Warning: Changing the parameter in this command resets the peer session. Continu
e?[Y/N]:y
```

![image-20250902003954700](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341736.png)

------

#### 6.13.案例10：优选Origin-ID

如果`BGP`路由携带`Originator-ID`属性，则在本条规则的优选过程中，将比较`Originator_ID`的大小，并优选`Originator_ID`最小的`BGP`路由。

需要注意的是，`Originator_ID`携带的内容为原始路由发布者的`Router-ID`。

![image-20250902005126898](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341742.png)

```
[AR2]bgp 200
[AR2-bgp]undo peer 24.1.1.2 
[AR2-bgp]peer 24.1.1.2 as 200
[AR2-bgp]quit

[AR3]bgp 200
[AR3-bgp]undo peer 35.1.1.2 
[AR3-bgp]peer 35.1.1.2 as 200
[AR3-bgp]quit

[AR4]undo bgp 100
Warning: All BGP configurations will be deleted. Continue? [Y/N]: y
[AR4]bgp 200
[AR4-bgp]router-id 4.4.4.4
[AR4-bgp]peer 24.1.1.1 as 200
[AR4-bgp]network 10.0.45.0 24
[AR4-bgp]quit

[AR5]undo bgp 300
Warning: All BGP configurations will be deleted. Continue? [Y/N]: y
[AR5]bgp 200
[AR5-bgp]router-id 5.5.5.5
[AR5-bgp]peer 35.1.1.1 as 200
[AR5-bgp]network 10.0.45.0 24
[AR5-bgp]quit

[AR3]ospf 1 
[AR3-ospf-1]area 0
[AR3-ospf-1-area-0.0.0.0]network 35.1.1.0 0.0.0.3
[AR3-ospf-1-area-0.0.0.0]quit
[AR3-ospf-1]quit

[AR2]ospf 1
[AR2-ospf-1]area 0
[AR2-ospf-1-area-0.0.0.0]network 24.1.1.0 0.0.0.3
[AR2-ospf-1-area-0.0.0.0]quit
[AR2-ospf-1]quit

[AR2]bgp 200	
[AR2-bgp]peer 1.1.1.1 reflect-client 
[AR2-bgp]quit

[AR3]bgp 200
[AR3-bgp]peer 1.1.1.1 reflect-client 
[AR3-bgp]quit
```

![image-20250902005157159](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341802.png)

此时，不是特别能够证明是比较`Originator_ID`，所以将`AR4`的`Router-ID`修改为`200.200.200.200`:

```
[AR4]bgp 200
[AR4-bgp]router-id 200.200.200.200
Warning: Changing the parameter in this command resets the peer session. Continu
e?[Y/N]:y
[AR4-bgp]quit
```

![image-20250902005339977](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341819.png)

------

#### 6.14.案例11：优选IP地址最小的

当前面所有的规则都无法比较出优选路由时，此时会根据对等体地址大小来进行优选，对等体地址较小者发送的路由较优选。

![image-20250902005628525](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341813.png)

修改前面的拓扑，`R2、R3`都与`R4`相连，`R4`作为`RR`客户端，只在`R4`上将路由发布到`BGP`，此时`R2、R3`反射的`BGP`路由将拥有相同的`Originator_ID`：

```
[AR3]int g0/0/1
[AR3-GigabitEthernet0/0/1]ip add 34.1.1.1 30
[AR3-GigabitEthernet0/0/1]quit
[AR3]bgp 200
[AR3-bgp]undo peer 35.1.1.2
[AR3-bgp]peer 34.1.1.2 as 200
[AR3-bgp]quit

[AR4]int loo0
[AR4-LoopBack0]ip add 192.168.10.1 24
[AR4-LoopBack0]quit
[AR4]int g0/0/1
[AR4-GigabitEthernet0/0/1]ip add 34.1.1.2 30
[AR4-GigabitEthernet0/0/1]quit
[AR4]bgp 200
[AR4-bgp]peer 34.1.1.1 as 200
[AR4-bgp]network 192.168.10.0 24
[AR4-bgp]quit
```

![image-20250902010241452](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902135341869.png)

------

### 7.补充

#### 7.1.关于联邦下一跳问题

![image-20250910151002924](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250910151003061.png)

如上图，`AR1`通过`EBGP`将`192.168.1.0`通告给`AR2`，`AR2`再通过`IBGP`通告给`AR3`。此时因为`AR2`针对`AR3`使用了`next-hop-local`命令，所以`AR3`学习到的这条路有的下一跳是`AR2`建立对等体的地址。

当`AR3`将这条路由条目通告给它的联邦`EBGP`邻居时下一跳不会发生改变，因为本质上`AR4`学习到的这条路由是`IBGP`路由（遵循`IBGP`规则，保持下一跳属性不变）。另外，即使`AR3`针对`AR4`使用`next-hop-local`命令也不会发生改变。

![image-20250910152213680](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250910152213832.png)

**next-hop-local**

|                    场景                     | 是否需要 `next-hop-local`？ |                             原因                             |
| :-----------------------------------------: | :-------------------------: | :----------------------------------------------------------: |
|         **向 EBGP 对等体通告路由**          |        **从不**需要         |      默认行为就是将自己的地址作为下一跳通告给外部邻居。      |
| **向 IBGP 对等体通告 \*从EBGP\*学来的路由** |      **几乎总是**需要       |          这是解决 IBGP 下一跳不可达问题的标准方法。          |
| **向 IBGP 对等体通告 \*从IBGP\*学来的路由** |  **不需要**（且通常无效）   | 因为 BGP 水平分割规则，你本来就不会传播这类路由。如果使用了路由反射器，反射器会有特殊处理。 |

------

**如果想要想改下一跳，可通过下面的方法：**

```
[AR3]ip ip-prefix test permit 192.168.1.0 24

[AR3]route-policy test permit node 10	
[AR3-route-policy]if-match ip-prefix test
[AR3-route-policy]apply ip-address next-hop 3.3.3.3
[AR3-route-policy]quit
[AR3]route-policy test permit node 20

[AR3]bgp 64512
[AR3-bgp]peer 4.4.4.4 route-policy test export 
[AR3-bgp]quit
```

![image-20250910152635956](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250910152636104.png)

------

#### 7.2.引入问题

![image-20250910154012358](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250910154012503.png)

在`AR2`上将`BGP`路由引入到`OSPF`，你会发现没有任何路由被成功引入到`OSPF`，为啥呢？？

**技术上可以，设计上不可以。**

因为，`IBGP`路由可能包含了当前`AS`或整个互联网的路由条目，一旦引入到`OSPF`。会导致`IGP`数据库爆炸，路由器因`CPU`和内存耗尽而崩溃。

另外，`BGP`的精华在于基于路径属性的策略控制（选路），一旦引入`IGP`，所有这些属性都会丢失，路由决策退回至简单的度量值。

违背了`BGP/IGP`的分层设计原则，`IGP`的核心任务是为`BGP`下一跳提供可达性，而不是承载外部路由。

最后，引入`BGP`路由到`OSPF`中可能会出现路由环路问题。

------

可以通过命令实现引入`IBGP`路由：

```
[AR2-ospf-1]import-route bgp permit-ibgp
```

![image-20250910161907545](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250910161907703.png)

