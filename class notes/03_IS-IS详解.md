# 03_IS-IS详解

### 1.IS-IS概述

`IS-IS(Intermediate System to Intermediate System)`中间系统到中间系统，是一种**链路状态路由协议**，在运营商网络中被广泛使用。

`IS-IS`是`ISO(International Organization for Standardization)`国际标准化组织为它的`CLNP`无连接网络协议设计的一种动态路由协议。随着`TCP/IP`的留下，为了提供对`IP`路由的支持，`IETF`对其进行了扩充和修改，使它能够同时应用在`TCP/IP`和`OSI`中。扩展后的`IS-IS`，我们将其称为集成`IS-IS`。

![image-20250817215525476](https://gitee.com/chouhama/pic/raw/master/20250817215525560.png)

------

`IS-IS`与`OSPF`有很多相似之处：

​	运行`IS-IS`的直连设备之间会通过`Hello`报文发现彼此，然后建立邻居关系，并交互链路状态信息。在`IS-IS`中，链路状态信息叫做`LSP(Link-State Packet)`链路状态报文。每一台运行`IS-IS`的设备都会产生`LSP`，设备产生的`LSP`会被泛洪到网络中适当的范围，所有的设备都将自己产生的、以及网络中泛洪的`LSP`存储在自己的`LSDB`中。

​	`IS-IS`设备基于自己的`LSDB`采用`SPF`算法进行计算，最终得到`IS-IS`路由信息。另外，与`OSPF`一样，`IS-IS`也支持层次化的网络架构，支持`VLSM`，支持手工路由汇总等功能。

------

**术语介绍：**

以下的术语大多都是和`OSI`协议栈相关的：

- `ISO`国际标准化组织：全球性非政府组织，`1946`年成立，促进各领域的标准化实现。`OSI`参考模型就是它的杰作。
- `IS(Intermediate System)`：指的是`OSI`中的路由器。

- `IS-IS`中间系统到中间系统：用于在`IS`之间实现动态路由信息交互的协议。
- `CLNP`无连接网络协议：这是`OSI`的无连接网络协议，他与`TCP/IP`中的`IP`协议的功能类似。
- `LSP`链路状态报文：这是`IS-IS`用于描述链路状态信息的关键数据，类似`OSPF`的`LSA`。
  - `LSP`分为两种：`Level-1`和`Level-2`。

------

#### 1.1.NSAP

在`TCP/IP`协议栈中，`IP`地址用于标识网络中的设备，从而实现网络层寻址。

在`OSI`协议栈中，`NSAP`网络服务接入点被视为`CLNP`地址，它是一种用于在`OSI`协议栈中定位资源的地址。`IP`地址只用于标识设备，并不标识该设备的上层协议类型或服务类型，而`NSAP`地址中除了包含用于标识设备的地址信息，还包含用于表示上层协议类型或服务类型的内容，类似于`TCP/IP`中的`IP`地址与`TCP`或`UDP`端口号的组合。

------

`NSAP`地址由`IDP`初始域部分和`DSP`域指定部分两部分构成，而`IDP`及`DSP`这两部分又被进一步划分，如下图所示：

![image-20250817221055269](https://gitee.com/chouhama/pic/raw/master/20250817221055312.png)

在`NSAP`地址中，`IDP`和`DSP`都是可变长的，这使得`NSAP`地址的总长度并不固定，最短为`8Byte`，最长则可以达到`20Byte`。

- `AFI(Authority and Format Identifier)`授权组织和格式标识符：长度为`1Byte`，用于标识地址分配机构。另外，该字段值同时也指定了地址的格式。一个在实验室环境中经常被使用到的`AFI`值为`49`，该值表示本地管理，即私有地址空间。

  - | AFI值  |                用途说明                |            适用场景             |
    | :----: | :------------------------------------: | :-----------------------------: |
    | **49** | 私有地址（相当于IP中的192.168.0.0/16） | **最常用**，企业/运营商私有网络 |
    | **47** |      国际代码（ISO 3166国家代码）      |       跨国组织（需注册）        |
    | **39** |              美国DOD代码               |          军事/政府网络          |
    | **00** |       本地管理（类似IPv6的FE80）       |         实验室测试环境          |

- `IDI(Initial Domain Identifier)`初始域标识符：该字段用于表示域，其长度是可变的。

- `DSP`高位部分：也就是`DSP`中的高比特位部分（在二进制数中，最靠近左边的比特位被视为高比特位），该字段的长度是可变的，它用于在一个域中进一步划分区域。

- 系统ID`(system Identification)`：用于在一个区域内标识某台设备。在华为路由器上，系统`ID`的长度固定为`6Byte`，而且通常采用`16`进制呈现，例如`0122.a2f1.0031`。在网络部署过程中，必须保证域内设备的系统`ID`的唯一性。考虑到在以太网环境中，设备的`MAC`地址具有全局唯一性，而且正好长度也是`6Byte`，因此使用`MAC`地址作为系统`ID`也是一个不错的方案。

- `NSEL`：长度为`1Byte`，用于标识上层协议类型或服务类型。

在`IS-IS`中，`NSAP`的`IDP`及`DSP`高位部分加在一起被称为区域地址，区域地址就是区域`ID`。

![image-20250817232500816](https://gitee.com/chouhama/pic/raw/master/20250817232500866.png)

------

##### 1.1.1.NET

`NET(Network Entity Title)`网络实体名称，用于在网络层标识一台设备，可以简单看作`NSEL`为`0x00`的`NSAP`。由于`NSEL`为`0x00`，因此`NET`不标识任何上层协议类型，只用于标识设备本身。

在`TCP/IP`环境中部署`IS-IS`，必须为每一台准备运行`IS-IS`的设备分配`NET`，否则`IS-IS`将无法正常工作。一旦网络管理员为一台设备指定了`NET`，该设备便可以从`Net`中解析出区域`ID`，以及设备的系统`ID`。系统`ID`相当于`OSPF`中的`Router-ID`。

在`IS-IS`中`SPF`计算时通过`System-ID`识别拓扑节点。`Net`中的`Area-ID`决定路由器所属的`IS-IS`区域。

```
[Area ID] + [System ID] + [NSEL] 
```



![image-20250817233020082](https://gitee.com/chouhama/pic/raw/master/20250817233020142.png)

|       方案       |                     实现方式                      |     适用场景     |
| :--------------: | :-----------------------------------------------: | :--------------: |
| **MAC地址转换**  |           使用设备MAC地址（去掉分隔符）           |     小型网络     |
| **IPv4地址映射** | 将IPv4地址转为3段16进制（如10.1.1.1→`0a01.0101`） |     便于记忆     |
|  **序列化编码**  |     按设备部署顺序分配（0000.0001→0000.0002）     | 大规模可扩展网络 |

也可像如下方式配置：

```
49.0001.1921.6800.1001.00
49.0001.0010.0100.1001.00
```

![image-20250817234058343](https://gitee.com/chouhama/pic/raw/master/20250817234058410.png)

**配置命令如下：**

```
isis 1
	network-entity 49.0001.0000.0000.0001.00
```

**提问：**

如果某台`IS-IS`设备配置了`NET`：`49.1524.2011.2102.0000.2192.00`，那么该设备所属的区域`ID`是（）？

A.`49.1524.2011.2102.0000.2192`

B.`49.1524.2011.2102.0000`

C.`49.1524.2011.2102`

D.`49.1524.2011`

------

##### 1.1.2.多NET

每台运行`IS-IS`的网络设备至少需要拥有一个`NET`，一台设备也可以同时配置多个`NET`，但是这些`NET`的`System ID`必须相同。

多`NET`的作用大多是使用在**区域迁移**，当路由器需要从一个区域迁移到另一个区域时，同时保留旧`NET`和新`NET`，形成过渡期双区域身份，避免因突然变更`Area ID`导致邻居关系中断和路由震荡。

其他情况可能是多区域融合或网络角色转换。

------

### 2.IS-IS的基本概念

#### 2.1.区域划分

通过区域划分可在网络规模较大的情况下，减少泛洪所造成的开销与设备的负担。

`IS-IS`采用两级分层结构：骨干网络及常规区域。

需要注意的是，`IS-IS`在区域上与`OSPF`不同：

- `OSPF`以接口划分区域
- `IS-IS`以设备划分区域

![image-20250818141528785](https://gitee.com/chouhama/pic/raw/master/20250818141535942.png)

------

#### 2.2.路由器的分类

在`IS-IS`中根据路由器所处区域的不同，分为以下三种`IS-IS`路由器：

![image-20250818142040873](https://gitee.com/chouhama/pic/raw/master/20250818143608553.png)

- `Level-1`路由器：常规区域路由器。
  - 它只与属于同一区域的`Level-1`和`Level-1-2`路由器形成邻接关系，这种邻接关系称为`Level-1`邻接关系，`Level-1`路由器无法与`Level02`路由器建立邻接关系。
  - `Level-1`路由器只负责维护`Level-1`的链路状态数据库`LSDB`，该`LSDB`只包含区本区域的路由信息。`Level-1`路由器想要访问其他区域时，前提必须通过`Level-1-2`路由器接入骨干区域。

- `Level-2`路由器：骨干路由器。
  - `Level-2`路由器可以与同一或者不同区域的`Level-2`路由器或者`Level-1-2`路由器形成邻接关系。`Level-2`路由器维护一个`Level-2`的`LSDB`，该`LSDB`包含整个`IS-IS`域的所有路由信息。
  - 所有`Level-2`级别的路由器组成路由域的骨干网，负责在不同区域间通信。路由域中`Level-2`级别的路由器必须是物理连续的，以保证骨干网的连续性。

- `Level-1-2`路由器：用于连接常规区域与骨干区域，类似于`ABR`。
  - `Level-1-2`路由器维护两个`LSDB`。

```
[AR1-isis-1]is-level ?
  level-1    Level-1
  level-1-2  Level-1-2
  level-2    Level-2
```

华为路由器默认的`IS-IS`路由器类型是`Level-1-2`

------

**总结：**

- `Level-1`路由器只能与**同区域的**`Level-1`和`Level-1-2`路由器建立邻接关系。
- `Level-1`路由器默认只能学习到本区域的`LSP`和一条由`Level-1-2`路由器下发的默认路由。
- `IS-IS`的骨干区域不仅仅以区域号规定，所有同区域或不同区域的`Level-2`路由器组成了骨干区域，但前提是它们彼此之间互联。
- `Level-1-2`路由器的区域号应与其相连的`Level-1`路由器一致。

------

#### 2.3.网络类型概述

在`IS-IS`中支持两种网络类型：

- `Broadcast`广播
- `p2p(Point-to-Point)`点到点

![image-20250818152509881](https://gitee.com/chouhama/pic/raw/master/20250818152509959.png)

当设备的接口激活`IS-IS`后，`IS-IS`会自动根据该接口的数据链路层封装决定接口的网络类型。

不同的网络类型在实际邻居关系建立和`LSP`泛洪时都会有显著区别，与`OSPF`一样在广播类型的`IS-IS`中建立邻接关系时，存在一个类似于`DR/BDR`的概念，叫做`DIS`。

------

#### 2.4.度量值

##### 2.4.1.默认值

`IS-IS`与`OSPF`一样采用`Cost`作为路由度量值，`Cost`越小，路由越优。`IS-IS`的`Cost`也与设备的接口有关，每一个激活了`IS-IS`的接口都会维护接口`Cost`。

但是，`IS-IS`接口的`Cost`在缺省情况下与接口的带宽无关！无论接口的带宽是多少，缺省`Cost`值均为`10`。虽然，你可以手动的根据自己的需求修改接口的`Cost`值。

但是，这种设计方式在某些场景会存在问题，例如可能会导致设备选择`Cost`更优的低带宽路径，而不是选择`Cost`更差的高带宽路径。如下图：

![image-20250818154227532](https://gitee.com/chouhama/pic/raw/master/20250818154227622.png)

![image-20250818161538963](https://gitee.com/chouhama/pic/raw/master/20250818161539064.png)

------

##### 2.4.2.Narrow与Wide

在华为设备中默认使用的`IS-IS Cost`类型为`Narrow`，当使用这类`Cost`时，`IS-IS`接口`Cost`的长度为`6bit`，也就是说这个接口所支持的`Cost`值范围是`1-63`。

另外，`Cost`的长度为`10 bit`，这就导致接收到的路由最大`Cost`值为`1023`。很显然，这在现网大规模网络下会成为瓶颈，完全是不够看的。

![image-20250818161610070](https://gitee.com/chouhama/pic/raw/master/20250818161610148.png)

所以，`IS-IS`引入了`Wide`类型的`Cost`，当`IS-IS`使用`Wide`类型的`Cost`时，接口`Cost`变成了`24bit`，这使得设备的接口支持更大的`Cost`值范围。命令如下：

```
isis 1
cost-style wide
```

在现网中，需确保`IS-IS`域内所有的路由器配置一致的`IS-IS Cost`类型，否则会影响接收。

![image-20250818161641495](https://gitee.com/chouhama/pic/raw/master/20250818161641602.png)

------

##### 2.4.3.自动计算开销

`IS-IS`接口默认`10`的`Cost`在某些场景下肯定是不合适的，`IS-IS`提供了自动计算接口`Cost`的功能。这个功能被激活后，设备将自动根据接口的带宽值进行该接口`Cost`值的计算，与`OSPF`类似。

设备将使用参考带宽值（缺省`100Mbps`，可以在`IS-IS`视图下通过`bandwidth-reference`命令修改）除以接口的带宽值，再将所得结果乘以10，得到接口的`Cost`值。

只有当`Cost`类型修改为`Wide`或`Wide-compatible`宽度量兼容模式时，上述计算才会发生，如果设备的`IS-IS Cost`类型为`Narrow、Narrow-compatible、Compatible`，则激活了自动接口`Cost`计算功能后，设备将采用下表所示的对应关系为接口设置缺省`Cost`值。

|        **接口带宽范围**        | **Cost值** |
| :----------------------------: | :--------: |
|      接口带宽 ≤ 10 Mbit/s      |     60     |
| 10 Mbit/s < 带宽 ≤ 100 Mbit/s  |     50     |
| 100 Mbit/s < 带宽 ≤ 155 Mbit/s |     40     |
| 155 Mbit/s < 带宽 ≤ 622 Mbit/s |     30     |
| 622 Mbit/s < 带宽 ≤ 2.5 Gbit/s |     20     |
|       带宽 > 2.5 Gbit/s        |     10     |

在`IS-IS`视图下，执行`auto-cost enable`命令后，可激活自动计算接口`Cost`的功能。默认未开启。

![image-20250818161812963](https://gitee.com/chouhama/pic/raw/master/20250818161813058.png)

------

##### 2.4.4.Cost的区域性

当你手动修改`Cost`值时需要注意，加上`level-1`关键字，那么该命令配置的接口的`Level-1 Cost`值，如果加上`level-2`关键字，那么该命令配置的是接口的`Level-2 Cost`。如果你没加上这两个关键字，默认则是两个都修改了。

![image-20250818162021114](https://gitee.com/chouhama/pic/raw/master/20250818162021197.png)

------

#### 2.5.IS-IS的三张表

与`OSPF`一样，`IS-IS`也维护着三张非常重要的数据表：

##### 2.5.1.邻居表

两台直连的`IS-IS`设备首先必须建立邻居关系，然后才能够开始交互`LSP`。`IS-IS`将设备直连链路上发现的邻居加载到自己的邻居表中。

使用命令`display isis peer`命令可查看设备的`IS-IS`邻居表，在邻居表中能够看到邻居的系统`ID`、状态、保活时间及类型等信息。

![image-20250818173121652](https://gitee.com/chouhama/pic/raw/master/20250818173121738.png)

|     **参数**      |                           **说明**                           |   **示例值/状态**   |                          **重要性**                          |
| :---------------: | :----------------------------------------------------------: | :-----------------: | :----------------------------------------------------------: |
|   **System Id**   | 邻居路由器的唯一标识符，格式为6字节的MAC地址（通常以点分十六进制表示）。 |  `0000.0000.0002`   |    用于唯一标识IS-IS网络中的设备，类似OSPF中的Router ID。    |
|   **Interface**   |        本地与邻居建立IS-IS邻接关系的物理或逻辑接口。         |      `GE0/0/0`      |         帮助定位连接的具体链路（如千兆以太网接口）。         |
|  **Circuit Id**   | IS-IS电路的标识符，由本地路由器的System ID + 两位数字组成，标识链路在拓扑中的逻辑编号。 | `0000.0000.0001.02` |      用于区分同一设备上的不同邻接关系（如多链路场景）。      |
|     **State**     | 邻接关系的当前状态： • `Up`：邻接正常，可交换路由信息。 • `Init`或`Down`：邻接异常。 |        `Up`         |          直接反映邻居连通性，`Up`是正常通信的前提。          |
|   **HoldTime**    | 邻居保持时间（秒），表示在收到下一个Hello报文前，邻居关系将保持的有效时间。超时后邻接断开。 |        `24s`        |     需与Hello间隔配合，若持续递减到0，说明邻居通信故障。     |
|     **Type**      | 邻居的层级类型： • `L1`：Level-1路由器（同一区域） • `L2`：Level-2路由器（骨干区域） • `L1L2`：同时支持。 |        `L2`         | 决定路由传递范围： • `L2`邻居参与区域间路由，`L1`仅限本区域。 |
|      **PRI**      | 邻居的DIS（指定中间系统）选举优先级，范围0-127，值越高越优先成为广播网络中的DIS。 |        `64`         |  仅影响广播网络（如以太网）中的DIS选举，点对点链路中无效。   |
| **Total Peer(s)** |                  当前IS-IS进程的邻居总数。                   |         `1`         | 快速确认网络规模或邻接状态是否完整（如预期应有多个邻居但显示1，可能配置错误）。 |

------

##### 2.5.2.LSDB

在邻居关系建立完成且`LSP`交互后，`IS-IS`设备将自己产生的、以及网络中所泛洪的`LSP`收集后存储在自己的`LSDB`中。与`OSPF`类似，`IS-IS`也使用`LSDB`存储链路状态信息，这些信息将被用于网络拓扑绘制及路由计算。

![image-20250818173737347](https://gitee.com/chouhama/pic/raw/master/20250818173737452.png)

由于`R1`是一台`Level-2`的路由器，所以它只维护`Level-2`的`LSDB`，从以上信息可以看出，`R1`的`Level-2 LSDB`中存在三个`LSP`。

每个`LSP`都采用`LSP ID(Link-State Packet ID`链路状态报文标识)进行标识，`LSP`由三部分组成：

- 系统`ID`6个字节：产生该`LSP`的`IS-IS`路由器的系统`ID`。
- 伪节点`ID`1个字节：该字段值存在`0`或非`0`两种情况。对于普通的`LSP`，该字段总是`0`。对于伪节点`LSP`，`DIS`负责为该字段分配一个非`0`的值。
- 分片号1个字节：如果`LSP`过大，导致始发设备需要对其进行分片，那么该设备通过为每个`LSP`分片设置不同的分片号来对它们进行标识及区分。同一个`LSP`的不同分片必须拥有相同的系统`ID`及伪节点`ID`。

![image-20250818180645599](https://gitee.com/chouhama/pic/raw/master/20250818180645706.png)

如果`LSPID`后有`*`号，代表该`LSP`是本设备产生的。

`Seq Num`与`OSPF`中`LSA`的序列号概念非常相似，用来表示一个`LSP`的新旧。

------

##### 2.5.3.路由表

每台`IS-IS`设备都会基于自己的`LSDB`运行相应的算法，最终计算出最优路由。`IS-IS`计算出的路由存放在`IS-IS`路由表中，通过`display isis route`命令可以查看设备的`IS-IS`路由表。

`IS-IS`路由表中的路由未必最终一定被加载到设备的全局路由表，这还取决于路由优先级等因素。

![image-20250818181105216](https://gitee.com/chouhama/pic/raw/master/20250818181105330.png)

------

#### 2.6.报文

`IS-IS`的协议报文直接采用数据链路层封装，所以`IS-IS`相比`OSPF`少了`IP`层的封装，相比之下`IS-IS`报文的封装效率更高。

在以太网环境中，`IS-IS`报文载荷直接封装在以太网数据帧中。`IS-IS`使用了以下几种`PDU(Protocol Data Unit)`协议数据单元：

------

##### 2.6.1.IIH（IS-IS Hello）

`IIH PDU`用于建立及维护`IS-IS`的邻居关系。在`IS-IS`中存在三种`IIH PDU`：

- `Level-1 LAN IIH`：广播，`Level-1`设备
- `Level-2 LAN IIH`：广播，`Level-2`设备

- `P2P IIH`：点到点

![image-20250819190346562](https://gitee.com/chouhama/pic/raw/master/20250819190346687.png)

如果设备类型为`Level-1-2`，在缺省情况下，它在`Broadcast`类型的接口上发送及侦听这两种类型的`LAN IIH`。

**时间：**

**广播网络：**

|       参数       | 默认值 |          华为CLI配置命令          |
| :--------------: | :----: | :-------------------------------: |
|  Hello Interval  |  10秒  |       `isis timer hello 10`       |
| Hello Multiplier |   3    | `isis timer holding-multiplier 3` |
|    Dead Time     |  30秒  |         自动计算（10×3）          |

**点到点网络：**

|       参数       | 默认值 |          华为CLI配置命令          |
| :--------------: | :----: | :-------------------------------: |
|  Hello Interval  |  10秒  |       `isis timer hello 10`       |
| Hello Multiplier |   3    | `isis timer holding-multiplier 3` |
|    Dead Time     |  30秒  |         自动计算（10×3）          |

**Dead Time = Hello Interval × Hello Multiplier**

- **Hello Interval**：发送Hello报文的时间间隔（默认10秒）。
- **Hello Multiplier**：乘数因子（默认3）。
- **Dead Time**：若在此时间内未收到Hello，则认为邻居失效（默认10×3=30秒）

```
[AR2-GigabitEthernet0/0/0]isis timer ?
  csnp                Set CSNP packet sending interval
  hello               Set hello packet sending interval
  holding-multiplier  Set holding multiplier value
  ldp-sync            Ldp-Sync
  lsp-retransmit      Set retransmission interval of the same LSP packet on P2P 
                      links
  lsp-throttle        Set minimum interval between sending a batch of LSPs or 
```

------

##### 2.6.2.LSP（Link-State Packet）

`IS-IS`使用`LSP`承载链路状态信息，`LSP`类似`OSPF`中的`LSA`。

`LSP`存在`Level-1 LSP`及`Level-2 LSP`之分，具体发送哪一种`LSP`视`IS-IS`邻居关系的类型而定。

![image-20250820132136747](https://gitee.com/chouhama/pic/raw/master/20250820132143999.png)

- `PDU`长度（`PDU Length`）：指示该`PDU`的总长度（单位字节）
- 剩余生存时间（`Remaining Lifetime`）：指示该`LSP`的剩余存活时间（单位为秒）
- `LSP`标识符（`LSP ID`）：`LSP ID`由三部分组成：该设备的系统`ID`、伪节点`ID`、分片编号。
- 序列号（`Sequence Number`）：该`LSP`的序列号。在`IS-IS`中，`LSP`序列号的作用与`OSPF`中`LSA`序列号的作用类似，主要用于区分`LSP`的新旧。
- 校验和（`Checksum`）：校验和。
- `P(Partition Repair)`：如果设备支持区域划分修复特性，则其产生的`LSP`中该比特位将被设置为1。
- `ATT(Attached bits)`：关联位，实际上该字段共包含四个比特位（分别对应四种度量值类型），华为只使用了其中一个比特位（`Default Metric`）。
  - 如果`Level-1-2`路由器连接着`IS-IS`骨干网时，会在自己产生的`Level-1 LSP`中，将`ATT`置1。
- `OL(Overload bit)`：过载位。正常情况下，`IS-IS`设备产生的`LSP`中该比特位被设置位`0`；如果该比特位被设置为`1`，则意味着该`LSP`的始发设备希望通过该比特位声明自己已经“过载”，而收到该`LSP`的其他`IS-IS`设备在进行路由计算时，只会计算到达该`LSP`始发设备的直连路由，而不会计算穿越该设备、到达远端目的网段的路由。
- `IS`类型（`IS Type`）：用于指示产生该`LSP`的路由器是`Level-1`还是`Level-2`类型，如果该字段值为二进制`01`，则表示`Level-1`路由器；如果为二进制的`11`，则表示`Level-2`路由器。

------

##### 2.6.3.CSNP（Complete Sequence Number PDU，完全序列号报文）

`CSNP`存在`Level-1 CSNP`与`Level-2 CSNP`之分，不同的`IS-IS`邻居关系交互不同类型的`CSNP`。

一个`IS-IS`设备发送的`CSNP`包含该设备`LSDB`中所有的`LSP`摘要。`CSNP`主要用于确保`LSDB`的同步，与`OSPF`的`DD`报文类似。

`CSNP`包含`LSDB`中所有`LSP`的摘要信息，一条摘要信息包含四个重要信息：

- `LSP ID`
- 序列号
- 剩余生存时间
- 校验和

**在广播网络中，`CSNP`由`DIS`定期发送（缺省的发送周期为`10s`）。**

**在点到点网络上，`CSNP`只在第一次建立邻接关系时发送。**

![image-20250820133533764](https://gitee.com/chouhama/pic/raw/master/20250820133533898.png)

------

##### 2.6.4.PSNP（Partial Sequence Number PDU，部分序列号报文）

`PSNP`存在`Level-1 PSNP`与`Level-2 PSNP`之分，与`CSNP`不同，`PSNP`中只包含部分`LSP`的摘要信息。`PSNP`主要用于请求`LSP`更新。

另外，`PSNP`还用于在`P2P`网络中对收到的`LSP`进行确认，所以，类似于`OSPF`的`LSR`及`LSAck`。

当`IS-IS`路由器发现`LSDB`不同步时，发送`PSNP`来请求邻居发送新的`LSP`。

在点到点网络中，当收到`LSP`时，使用`PSNP`对收到的`LSP`进行确认。

![image-20250820133725122](https://gitee.com/chouhama/pic/raw/master/20250820133725248.png)

------

#### 2.7.IS-IS PDU的组成

`IS-IS PDU`由三部分组成：

- 通用头部
- `PDU`特有头部：取决于`PDU`类型，如`IIH、LSP、SNP`
- `TLV`

![image-20250819191153481](https://gitee.com/chouhama/pic/raw/master/20250819191153652.png)

------

##### 2.7.1.通用头部

![image-20250819194413497](https://gitee.com/chouhama/pic/raw/master/20250819194413623.png)

![image-20250819195007636](https://gitee.com/chouhama/pic/raw/master/20250819195007771.png)

------

##### 2.7.2.TLV

`TLV（Type-Length-Value）`是`IS-IS`协议的核心设计机制，其核心作用是 **以灵活、可扩展的方式封装协议数据**，使 `IS-IS` 能够适应复杂的网络需求（如多协议支持、动态拓扑变化等）。

1. **灵活扩展：**无需修改协议头部即可支持新功能（如`IPv6`，流量工程等）

2. **动态适配：**不同类型的数据（如邻居、`IP`地址、认证信息）通过独立的`TLV`封装，互补干扰。
3. **高效解析与兼容性：**接收方通过`Type`字段快速识别`TLV`类型，仅处理支持的`TLV`，忽略未知类型。

`TLV(Type-Length-Value)`：每个元素的描述如下：

- 类型（`Type`）：该字段的长度为`1Byte`，它标识了这个`TLV`的类型，`IS-IS`定义了丰富的`TLV`类型，不同的`TLV`类型用于携带不同的信息。
- 长度`Length`：该字段的长度为`1Byte`，它存储的数据用于指示后面的值的长度。由于不同的`TLV`类型所描述的信息不同，因此信息的长度可能也有所不同，本字段指示了该`TLV`中值的长度。
- 值`value`：该字段的长度是可变的，所占用的字节数在长度字段中描述。本字段的值就是该`TLV`所携带的有效内容。

`TLV`就类似于**模块**，当需要什么功能时，添加对应的模块即可。

| **TLV** **Type** | **名称**                                                     | **PDU**类型     |
| ---------------- | ------------------------------------------------------------ | --------------- |
| 1                | Area Addresses                                      区域地址 | IIH、 LSP       |
| 2                | IS Neighbors（LSP）                               中间系统邻接 | LSP             |
| 4                | Partition Designated Level2 IS                区域分段指定L2中间系统 | L2 LSP          |
| 6                | IS Neighbors（MAC Address）                        中间系统邻接 | LAN IIH         |
| 7                | IS Neighbors（SNPA Address）                       中间系统邻接 | LAN IIH         |
| 8                | Padding                                              填充    | IIH             |
| 9                | LSP Entries                                         LSP条目  | SNP             |
| 10               | Authentication Information                             验证信息 | IIH、 LSP、 SNP |
| 128              | IP Internal Reachability Information                 IP内部可达性信息 | LSP             |
| 129              | Protocols Supported                                 支持的协议 | IIH、 LSP       |
| 130              | IP External Reachability Information                IP外部可达性信息 | LSP             |
| 131              | Inter-Domain Routing Protocol Information        域间路由选择协议信息 | L2 LSP          |
| 132              | IP Interface Address                                 IP接口地址 | IIH、 LSP       |

------

#### 2.8.邻接关系

##### 2.8.1.邻居状态机

| **状态** |            **触发条件**            |                 **行为说明**                  |
| :------: | :--------------------------------: | :-------------------------------------------: |
| **Down** |         初始状态或邻居超时         |            不发送Hello，不生成LSP             |
| **Init** | 收到对端Hello但未包含本端System ID |    等待对端确认（广播网络需检查两次Hello）    |
|  **Up**  |    收到包含本端System ID的Hello    | 邻接建立完成，开始同步LSDB（广播网络选举DIS） |

------

##### 2.8.2.广播网络

在广播网络中，使用三次握手建立邻接关系。

![image-20250819223249388](https://gitee.com/chouhama/pic/raw/master/20250819223249540.png)

1. 假设`R1`率先在`GE0/0/0`接口上激活了`IS-IS`，由于`R1`是`Level-1`路由器，因此它的`GE0/0/0`接口的`Level`为`Level-1`，它将在该接口上周期性地发送`Level-1 LAN IIH`，这些`PDU`以组播地形式发送，目的`MAC`地址是`0180-c200-0014`，该`Level-1 LAN IIH`中记录了`R1`的系统`ID` `(0000.0000.0001)`，还包含多个`TLV`，其中区域地址`TLV`记录了`R1`的区域`ID` （`49.0012`）。
2. `R2`在其`GE0/0/0`接口上收到了`R1`发送的`IIH`，它会针对`PDU`中的相关内容进行检查（比如是否处在相同区域），检查通过后，`R2`其`IS-IS`邻居表中将`R1`的状态设置为`Init`，并在自己发送的`IIH`报文中增加`IS`邻居`TLV`，在该`TLV`中写入`R1`的接口`MAC`地址。
3. `R1`收到`R2`发送的含有`R1 GE0/0/0`接口`MAC`地址的`IIH`报文后，在其`IS-IS`邻居表中将`R2`的状态设置为`UP`，然后发送包含对端设备`MAC`地址的`IIH`报文。
4. `R2`收到该`IIH`后，在其`IS-IS`邻居表中将`R1`的状态设置为`UP`。双方的邻居状态到此建立完毕。

本例使用的是`Level-1`邻居关系的建立过程为例，`Level-2`除了使用`Level-2 LAN IIH`建立邻居关系以及`MAC`地址为`0180-c200-0015`外其他没区别。

**实验观察：**

![image-20250819232224811](https://gitee.com/chouhama/pic/raw/master/20250819232231969.png)

![image-20250819232626700](https://gitee.com/chouhama/pic/raw/master/20250819232626883.png)

![image-20250819232712200](https://gitee.com/chouhama/pic/raw/master/20250819232712365.png)

![image-20250819232758912](https://gitee.com/chouhama/pic/raw/master/20250819232759095.png)

![image-20250819232835713](https://gitee.com/chouhama/pic/raw/master/20250819232835900.png)

**抓包发现**：邻居关系建立完成后，过了一段时间`DIS`才同步。

![image-20250819233048203](https://gitee.com/chouhama/pic/raw/master/20250819233048327.png)

------

##### 2.8.3.DIS与伪节点

在广播网络中，`IS-IS`需要在所有的路由器中选举一个路由器作为`DIS(Designated Intermediate System,指定中间系统)`。`DIS`的概念与`OSPF`中的`DR`很像，主要作用是在`LAN`中虚拟出一个**伪节点**，并产生伪节点`LSP`，用来描述这个网络上有哪些网络设备。

生成了伪节点后，设备仅需在其泛洪的`LSP`中描述自己与伪节点的邻居关系即可，无需再描述自己与其他非为节点的邻居关系。

以下图为例，本身对于`R1`它需要泛洪三条`LSP`用来描述它的所有邻居关系，但现在只需要一条。

![image-20250819225446104](https://gitee.com/chouhama/pic/raw/master/20250819225446240.png)

伪节点`LSP`用于描述伪节点与`LAN`中所有设备（包括`DIS`）的邻居关系，从而区域内的其他`IS-IS`设备能够根据伪节点`LSP`计算出该`LAN`内的拓扑。伪节点`LSP`由`DIS`生成。

所以，伪节点及伪节点`LSP`减小了网络中所泛洪的`LSP`的体积，另外，当拓扑发生变化，网络中需要泛洪的`LSP`数量也减少了，对设备造成的负担自然也就相应减小了。

为了确保`LSDB`的同步，`DIS`会在`LAN`内周期性地泛洪`CSNP`，`LAN`中地其他设备收到该`CSNP`后，会执行一致性检查，以确保本地`LSDB`与`DIS`同步。

`DIS`周期性发送`CSNP`的时间间隔为`10s`，可以在`DIS`相应的接口上使用`isis timer csnp`命令修改。可以加上`Level1`或`Level2`参数，指定对什么类型生效，如果不加，则对当前级别的生效。

------

**选举规则**

1. 比较接口`DIS`优先级，优先级最高的设备称为该`LAN`的`DIS`。`DIS`优先级数值越大，则优先级越高。
2. 如果`DIS`优先级相等，则接口`MAC`地址最大设备将成为该`LAN`的`DIS`。

接口的默认`DIS`优先级为`64`，可通过在接口视图下执行`isis dis-priority`命令修改接口的`DIS`优先级，取值范围为`0-127`。修改优先级时，可指定`level-1`或`level-2`参数，若未指定则同时修改。另外，即使接口的`DIS`优先级设置为`0`，该设备依旧会参与`DIS`选举。

------

**其他**

- 非`DIS`依旧会与其他所有该`LAN`中的`IS-IS`设备建立邻居关系，所以，`DIS`其实并没有减少`LAN`中的邻居关系数量。
- 在一个`LAN`中，`Level-1`及`Level-2`的`DIS`独立选举，互补干扰。
- `IS-IS`没有定义备份`DIS`，当`DIS`发生故障时，立即启动新的`DIS`选举过程。
- `DIS`可以抢占，即使已经选举出`DIS`，如果有优先级更高的，则该设备将抢占当前`DIS`的角色，成为新的`DIS`。

------

**实验：**

![image-20250819233428776](https://gitee.com/chouhama/pic/raw/master/20250819233428915.png)

![image-20250819233929797](https://gitee.com/chouhama/pic/raw/master/20250819233929967.png)

------

##### 2.8.4.广播网络LSP同步过程

**端到端建立的情况**

![image-20250820134117148](https://gitee.com/chouhama/pic/raw/master/20250820134117275.png)

当`AR1`与`AR2`成功建立完成邻居关系后，就开始`LSP`的交互过程。

1. 一旦双方邻居进入`up`状态后，即泛洪自己的`LSP`。
   - 其中，`DIS`泛洪的`LSP`是为了描述当前`LAN`中的所有路由器。
   - 其他，`LSP`都是真正的由`IS-IS`路由器泛洪出来的**路由信息**。

![image-20250820140106367](https://gitee.com/chouhama/pic/raw/master/20250820140106500.png)

![image-20250820140259619](https://gitee.com/chouhama/pic/raw/master/20250820140259770.png)

![image-20250820140321647](https://gitee.com/chouhama/pic/raw/master/20250820140321786.png)

2. `DIS`负责周期性（默认`10s`）发送`CSNP`报文（`LSDB`摘要），告知当前`LAN`中所有路由器。
3. 如果，其他路由器在对比`CSNP`后发现自己少了某些路由，会组播发送`PSNP`来请求。
   - 但在我现在的实验中，你抓不到任何的`PSNP`报文，因为一开始互相已经泛洪了，不缺任何路由。

**问题：**为啥，没有`PSNP`呢？`PSNP`本身不也是类似于`LSAck`的存在么？

因为，在广播网络中`IS-IS`不依赖显示`ACK`来确认`LSP`。你可以简单理解为，**沉默即同意**。

- 一开始互相泛洪的情况下，如果成功把对端发送的`LSP`加入到`LSDB`后，下次发送的`LSP`的序列号就会加`1`。
  - 如果路由器B收到路由器A的LSP（Seq=5），且**没有用自己的LSP反驳**（即没有发送Seq≥5的相同LSP），说明B认可该LSP。
  - 此时，B会在下次发送LSP时**递增自己的序列号**，表示“我已同步你的信息”。

------

**后加的情况**

![image-20250820141253545](https://gitee.com/chouhama/pic/raw/master/20250820141253699.png)

------

##### 2.8.5.点到点网络

在`P2P`网络中，`IS-IS`邻居关系的建立过程存在两种方式：

- 两次握手
- 三次握手

**两次握手**

邻居关系的建立过程不存在确认机制，只要设备在其接口上收到`P2P IIH`，并且对`PDU`中的内容检查通过后，便单方面将该邻居的状态视为`UP`。这显然是不可靠的，因为即使双方的互联链路存在单通故障，也依然会有一方认为邻居关系已经建立，此时网络就必然会出现问题。

![image-20250820144211257](https://gitee.com/chouhama/pic/raw/master/20250820144211450.png)

**三次握手**

![image-20250820151415199](https://gitee.com/chouhama/pic/raw/master/20250820152946552.png)

三次握手机制，设备将在`P2P IIH`中增加一个特殊的`TLV`，`P2P`三向邻接`TLV`（`Point-to-Point Three-Way Adjacency TLV`），用于实现三次握手。

1. 假设`R3`率先在接口上激活了`IS-IS`，开始在该接口上发送`P2P IIH`。在该`IIH`中，包含`R3`的系统`ID`、区域`ID`等信息，此外还有一个关键的`TLV`，`P2P`三向邻接`TLV`，由于此时`R3`还没有在该接口上收到`P2P IIH`，因此它将该`TLV`中的邻接状态设置为`Down`。
2. `R4`收到`R3`发送的`P2P IIH`，它会针对该`PDU`中的相关内容进行检查，检查通过后，`R4`在其`IS-IS`邻居表中将邻居`R3`的状态设置为`Initial`，并在自己发送的`IIH`的`P2P`三向邻接`TLV`中，将邻接状态设置为`Init`，并且在该`TLV`的邻居系统`ID`字段中写入`R3`的系统`ID`。
3. `R3`收到该`P2P IIH`后，发现在该`PDU`中，`P2P`三向邻接`TLV`的邻接状态为`Init`，且邻居系统`ID`字段填写的是自己的系统`ID`，于是他认为自己与邻居完成了两次握手过程，接下来，它在自己发送的`IIH`三向邻接`TLV`中，将该邻接状态设置为`UP`，然后在该`TLV`的邻居系统`ID`字段中写入`R4`的系统`ID`。
4. `R4`收到后，认为自己与邻居完成了三次握手过程，便在`IS-IS`邻居表中，将该邻居的状态设置为`UP`。然后发送`IIH`，并且在三向邻接`TLV`中，将邻接状态设置为`UP`，然后在邻居系统`ID`字段中写入`R3`的系统`ID`。
5. `R3`收到后，也认为自己与对方完成了三次握手，便在`IS-IS`邻居表中，将该邻居的状态设置为`UP`。到此，邻居关系建立完成。

**在华为路由器上默认是三次握手，可以通过`isis ppp-negotiation 2-way`命令修改。**

------

**抓包**

1. 彼此先发送了`Down`的`IIH`

![image-20250820153704987](https://gitee.com/chouhama/pic/raw/master/20250820153712159.png)

![image-20250820153726482](https://gitee.com/chouhama/pic/raw/master/20250820153726619.png)

2. `AR1`发送`Init`的`IIH`

![image-20250820153759195](https://gitee.com/chouhama/pic/raw/master/20250820153759346.png)

3. `AR2`发送`up`的`IIH`

![image-20250820153829152](https://gitee.com/chouhama/pic/raw/master/20250820153829301.png)

4. `AR1`发送`up`的`IIH`

![image-20250820153851681](https://gitee.com/chouhama/pic/raw/master/20250820153851822.png)

------

##### 2.8.6.点到点网络LSP同步过程

![image-20250820155309621](https://gitee.com/chouhama/pic/raw/master/20250820155309779.png)

1. 邻居`up`后全量泛洪`LSP`
2. 收到对方的`LSP`后，向对方发送`PSNP`用来确认。
3. 互相发送`CSNP`，如果不一致，则继续发送`PSNP`进行请求。`P2P`链路上`CSNP`仅在初始同步时发送。

![image-20250820155351693](https://gitee.com/chouhama/pic/raw/master/20250820155351887.png)

**与广播类型相比，只少了`DIS`发送的`LSP`，其他基本一致。**

------

##### 2.8.7.SPT分析

![image-20250819235135745](https://gitee.com/chouhama/pic/raw/master/20250819235135889.png)

**对R1 LSDB分析：**

`AR1`已经通过下图中列出的`LSP`学习到了所有`LAN`中邻居宣告的网段。其中，具体的地址是**实节点**，网段是**叶子**。

![image-20250819235232952](https://gitee.com/chouhama/pic/raw/master/20250819235233105.png)

------

后续，`AR2`和`AR4`建立了邻居关系，并在`R4`上宣告`4.4.4.4/32`后，`R1`会得到`AR4`泛洪的`LSP`，其中就包含`AR4`建立`IS-IS`的接口地址以及`4.4.4.4/32`这个网段。

那么，`AR1`就已经通过`24.1.1.4`得知实节点和`Cost`了（`AR1 --> AR2` + `AR2 --> AR4`）：

![image-20250819235919446](https://gitee.com/chouhama/pic/raw/master/20250819235919584.png)

注意，上图中`Cost`指的是`Ar4`到达`Ar2`的`Cost`值。`IP-Internal 24.1.1.0`中的`Cost`也就是`24.1.1.2`到达`24.1.1.4`的`Cost`。

------

##### 2.8.8.邻居关系建立须知

1. 建立`IS-IS`邻居关系的两台设备必须是同一个`Level`的设备。
2. 两台之连设备如需建立`Level-1`邻居关系，则两者的区域`ID`必须相同。
3. 建立`IS-IS`邻居关系的两台`IS-IS`设备，直连接口需使用相同的网络类型。

------

##### 2.8.9.LSP的处理机制

![image-20250820155543589](https://gitee.com/chouhama/pic/raw/master/20250820155543762.png)

------

### 3.进阶

#### 3.1.路由计算

![image-20250821132324551](https://gitee.com/chouhama/pic/raw/master/20250821132324723.png)

根据之前所学的基础知识可得出，`R2`和`R3`作为`Level-1-2`路由器，会将`R1`产生的`Level-1 LSP`传递给所有`Level-2`路由器，所以`Level-2`路由器会拥有当前整个`IS-IS`网络的完整`LSP`。

而`R1`根据`LSDB`中的`Level-1 LSP`计算出`Area 49.0001`内的拓扑，另外，`R2`和`R3`会向`Level-1`区域下的`LSP`中设置`ATT`标志位，用于向区域内的`Level-1`路由器宣布可以通过自己到达其他区域，`R1`根据该标志位，计算出指向`R2`和`R3`的默认路由。

------

分析此实验中`AR1`与`AR4`的路由信息：

- `AR1`：从下述两张图中可分析出，当前`AR1`通过`IS-IS`学习到了所有`Level-1`邻居通告的网段，包括`AR2、AR3`的`GE0/0/1`接口。
  - 但是，从路由表中可得知，除了`24.1.1.0/30`和`35.1.1.0/30`之外，还有两条`0.0.0.0/0`的默认路由。
  - 可是，没有从`LSDB`看到这两条默认路由啊？原因很简单，`IS-IS Level-1`路由器学习到了默认路由，和`OSPF`不一样，这条默认路由并不是邻居通告给它的，而是由自己生成的。
  - 如何生成呢？只要邻居发送的`LSP`中`ATT`标志位置位，则生成默认路由且下一跳为邻居直连接口。

![image-20250821134004199](https://gitee.com/chouhama/pic/raw/master/20250821134004385.png)

![image-20250821134028400](https://gitee.com/chouhama/pic/raw/master/20250821134028573.png)

- `AR4`：

![image-20250821135405744](https://gitee.com/chouhama/pic/raw/master/20250821135405928.png)

![image-20250821135418303](https://gitee.com/chouhama/pic/raw/master/20250821135418485.png)

------

##### 3.1.1.次优路径

![image-20250821132324551](https://gitee.com/chouhama/pic/raw/master/20250821140143515.png)

从上图我们可以分析出，当前`AR1`去往`4.4.4.4/32`的最优路径是`AR1-->AR2-->AR4`，`AR1`去往`5.5.5.5/32`的最优路径是`AR1-->AR3-->AR5`。

但是，当我用`tracert`命令在`AR1`测试时，实际路径却不是这样：

![image-20250821140304840](https://gitee.com/chouhama/pic/raw/master/20250821140305002.png)

经过测试发现，当前`AR1`上已经产生了次优路径了。原因如下：

- `AR1`当前有两条默认路由，即负载分担，所以当跨区域的流量产生时，`AR1`选路不稳定，可能走`AR2`也可能走`AR3`这就是所谓的**次优路径**。

------

##### 3.1.2.路由渗透

![image-20250821132324551](https://gitee.com/chouhama/pic/raw/master/20250821140919632.png)

在此实验中需要在`AR2`和`AR3`上执行路由渗透，将`Level-2`中的明细路由传递给`Level-1`区域的`AR1`，此时`Ar1`就可以学习到其他区域的详细路由了，路由渗透这个操作需要在`Level-1-2`路由器上执行。

```
[AR2]isis 1
[AR2-isis-1]import-route isis level-2 into level-1

[AR3]isis 1
[AR3-isis-1]import-route isis level-2 into level-1
```

**观察AR1的LSDB**

![image-20250821141259612](https://gitee.com/chouhama/pic/raw/master/20250821141259804.png)

**观察AR1的路由表**

![image-20250821141344665](https://gitee.com/chouhama/pic/raw/master/20250821141344844.png)

------

**拓展：**

正常情况下路由渗透是把`Level-2`的路由渗透给`Level-1`。实际上也可以做`Level-1`的渗透给`Level-2`。

因为，默认情况下`Level-1-2`路由器会将在`Level-1`学到的所有路由通告给`Level-2`，可以通过`import-route isis level-1 into level-2`命令渗透时，加上过滤参数，只将需要的路由渗透过去。

------

##### 3.1.3.SPT计算

对于使用`SPF`算法计算出`SPT`以及最短路径的方式，实际上的诀窍就是搞清楚当前路由器的**实节点**与**叶子节点**。

- **实节点**：当前路由器计算的拓扑图中的路由器节点，一般是接口链路地址。
- **叶子节点**：也可以叫做**附加节点**，简单来说就是附加在**实节点**后的网段及掩码信息。

在你搞明白什么是**实节点**，什么是**叶子节点**后，你就能很轻松的知道`IS-IS`到底是什么计算整个网络中的路径和最优路径的。

1. 以当前设备为**树根**计算出到达每一个**实节点**的`Cost`值。
2. 将所有**附加节点**附着在**实节点**的背后，当计算**附加节点**（网段）的`Cost`值时，只需要计算本节点到达对应实节点的`Cost`加上对应**实节点**到达**附加节点**的`Cost`。

用下面的物理拓扑画出现在`AR1`的逻辑拓扑（**路由渗透后**）：

![image-20250821132324551](https://gitee.com/chouhama/pic/raw/master/20250821142146132.png)

![image-20250821142659727](https://gitee.com/chouhama/pic/raw/master/20250821142659905.png)

------

##### 3.3.4.默认路由

在`IS-IS`中，主要通过以下三种方式来控制默认路由的生成与发布。

**方式一：在`Level-1-2`设备上，控制器产生的`LSP`中的`ATT`比特位**

前文已经学过，`Level-1`区域中的`Level-1-2`路由器如果与骨干网相连，那它在向`Level-1`区域发送的`LSP`中会将`ATT`置为`1`，区域内的`Level-1`路由器将自动根据该`LSP`生成默认路由。

在`Level-1-2`路由器的`IS-IS`配置视图下，如果执行`attached-bit advertise always`命令，则该`Level-1-2`路由器无论是否连接到`IS-IS`骨干网，都会将`ATT`置位。

如果配置`attached-bit advertise never`命令，则该路由器无论如何都会将`ATT`设置为`0`。

------

**方式二：在`Level-1`设备上，通过配置使其即使收到了`ATT`置位的`LSP`也不会自动生成默认路由**

在`Level-1`路由器的`IS-IS`配置视图下执行`attached-bit avoid-learning`命令，即使收到了`ATT`置位的也不会生成默认路由。

------

**方式三：发布默认路由**

通过`default-route-advertise`命令，用于在`IS-IS`中发布默认路由。可通过参数指定下发的路由级别。

在`Level-1`路由器上使用命令：`default-route-advertise level-1`向`Level-1-2`路由器下发默认路由。还可以加上`avoid-learning`参数，使本端不学习默认路由。默认路由的传播范围为域内。

------

#### 3.2.认证

`IS-IS`认证是基于网络安全性的要求而实现的一种认证手段，通过在`IS-IS`报文中增加认证字段对报文进行认证。当本地路由器接收到远端路由器发送过来的`IS-IS`报文，如果发现认证密码不匹配，则将收到的报文进行丢弃，达到自我保护的目的。

根据报文的种类，认证可以分为以下三类：

- **接口认证：**在接口视图下配置，对`Level-1`和`Level-2`的`Hello`报文进行认证。

  - ```
    [AR2]int g0/0/0	
    [AR2-GigabitEthernet0/0/0]isis authentication-mode md5 cipher Huawei@123
    
    [AR1]int g0/0/0
    [AR1-GigabitEthernet0/0/0]isis authentication-mode md5 cipher Huawei@123
    ```

- **区域认证：**在`IS-IS`进程视图下配置，对`Level-1`的`CSNP、PSNP`和`LSP`报文进行认证。

  - ```
    [AR1-isis-1]area-authentication-mode md5 cipher Huawei@123
    
    [AR2-isis-1]area-authentication-mode md5 cipher Huawei@123
    
    [AR3-isis-1]area-authentication-mode md5 cipher Huawei@123
    ```

- **路由域认证：**在`IS-IS`进程视图下配置，对`Level-2`的`CSNP、PSNP`和`LSP`报文进行认证。

  - ```
    [AR5-isis-1]domain-authentication-mode md5 cipher Huawei@123
    ```

**注意：**区域认证和路由域认证主要是用来防护开启该功能的设备的。

- 当本端开启了认证后，会对收到的`LSP`进行检查，如果这个`LSP`没有携带认证信息或者密码错误的话，都会被本地路由器忽略，不加入到`LSDB`。
- 本端发送的`LSP`中也会携带认证信息，如果一台没有开启认证的设备收到后，会加入到`LSDB`中，不会忽略。

有一个例外，如果你在配置认证时加了`all-send-only`参数后，本端路由器仅对发送的`PDU`进行身份验证，不检查接收到的`PDU`，也就是所有的`PDU`都收。

```
domain-authentication-mode md5 cipher Huawei@123 all-send-only
```

------

根据报文的认证方式，可以分为以下四类：

- **简单认证：**不加密
- `MD5`认证：通过将配置的密码进行`MD5`算法加密之后再加入报文中，提高密码的安全性。
- `Keychian`认证：通过配置随时间变化的密码链表进一步提升网络的安全性。
- `HMAC-SHA256`认证：通过将配置的密码进行`HMAC-SHA256`算法加密之后再加入报文中，提高密码的安全性。

------

#### 3.3.路由汇总

##### 3.3.1.Level-1汇总

 在一个`Level-1`区域内，每一台`IS-IS`路由器都会产生自己的`Level-1 LSP`；`IS-IS`允许设备对其始发的路由执行汇总。

![image-20250821192601563](https://gitee.com/chouhama/pic/raw/master/20250821192601762.png)

![image-20250821193046506](https://gitee.com/chouhama/pic/raw/master/20250821193046701.png)

在上图中，`AR1、AR2、AR3`属于`Level-1`区域，`R1`将`3`个回环口地址都宣告进`IS-IS`网络中，`AR2、AR3`都可以学到这些明细路由。为了简化`AR2、AR3`的路由表，可以在`AR1`上部署路由汇总。

```
[AR1]isis 1
[AR1-isis-1]summary 192.168.0.0 255.255.0.0 level-1
```

![image-20250821193126721](https://gitee.com/chouhama/pic/raw/master/20250821193126940.png)

------

##### 3.3.2.Level-1-2汇总

如下图，`AR1`在`Level-1`区域中泛洪`3`条`LSP`，`AR4`作为`Level-1-2`路由负责将该`LSP`转发给`AR5`。此时可以在`AR4`上配置路由汇总，将其通告给`AR5`。

![image-20250821193405016](https://gitee.com/chouhama/pic/raw/master/20250821193405209.png)

![image-20250821193810456](https://gitee.com/chouhama/pic/raw/master/20250821193810656.png)

```
[AR4]isis 1
[AR4-isis-1]summary 192.168.0.0 255.255.0.0 level-2
```

![image-20250821194013413](https://gitee.com/chouhama/pic/raw/master/20250821194013612.png)

------

##### 3.3.3.Level-2汇总

在下图中，`R6`为`Level-2`路由器，将三个回环口都宣告进`IS-IS`网络中，`R4`和`R5`都会学习到三条明细路由。此时可以在`AR6`上执行路由汇总。

![image-20250821194212061](https://gitee.com/chouhama/pic/raw/master/20250821194212249.png)

![image-20250821194454755](https://gitee.com/chouhama/pic/raw/master/20250821194454958.png)

```
[AR6]isis 1
[AR6-isis-1]summary 172.16.0.0 255.255.0.0 level-2
```

![image-20250821194552361](https://gitee.com/chouhama/pic/raw/master/20250821194552573.png)

------

#### 3.4.Silent-Interface

```
[AR5]int g0/0/0
[AR5-GigabitEthernet0/0/0]isis silent
```

- `advertise-zero-cost`：`IS-IS`会将该接口的直连网段以**开销=0**的形式通告出去。如果不加参数，以实际开销发布。

------

