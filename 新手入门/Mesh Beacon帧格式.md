<!--
 * @Description: In User Settings Edit
 * @Author: 临时工
 * @Date: 2019-08-17 13:34:18
 * @LastEditTime: 2019-12-05 21:39:37
 * @LastEditors: Please set LastEditors
 -->
# 前言
基础篇章已经讲解完成，我坚信大伙应该对SIG Mesh已经有了大概的概念。接下来，让我们继续前进深层次地了解这些概念是如何变成现实中看不见摸不着，但是又能监测到的Mesh信标是怎么样的。
# SIG Mesh Beacon载荷包
关注红旭无线Mesh教程的朋友，我相信大家应该对**SIG MESH协议各个层的作用**中提到的SIG Mesh载荷包还有印象吧？Mesh的数据包是BLE广播包的另外一种体现，其大体的帧格式基本一致。那么，它究竟长什么样子呢？如下图所示：
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Mesh_Beacon_Data.png)

从上图可知，其实Mesh Beacon的帧格式还是比较简单的。接下来让小编带着你们**剥洋葱**，一层一层地剖析它，每一个字段都是干什么的？起到什么作用？如果条件允许，建议听着[《洋葱--杨宗纬》](https://music.163.com/#/mv?id=5441126)这首歌再看这篇文章，效果更佳:smile:！为了让大家的理解不局限于纯理论中，此次小编直接采用真实的示例工程来拆段式地讲解最终的数据包是长什么样的。无疑这种效果是最佳的，**届时我们也会专门开设视频讲解，敬请期待！！！<---此处是软广**；接下来，交待下此时的相关开发环境及工具：
- nRF52840DK或者红旭无线的开发板
- light_switch_server的示例工程
- SEGGER Embedded Studio

将上述的示例工程编译下载之后，我们就可以看到Mesh Beacon数据包了；由于上面的示例工程是使能了proxy特性，所以实际上会有两种不同类型的广播包：一种是**不可连接非定向广播**，另外一种是**可连接非定向广播**。显然，前者是属于Mesh beacon包，而后者则是proxy特性相关的广播包。它们两者前后顺序在37、38、39三个广播信道分别发送相应的数据包。**注意：这里还是有必要提一下，MESH网络中的所有数据传送都是通过不可连接非定向广播包发送出去的**。接下来，让我们看看在空中传播的Mesh Beacon是些什么内容：

- Unprovisioned Mesh Beacon

  ![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/mesh_beacon_header.png)
  ![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/mesh_beacon_data_payload.png)

- Secure Network Beacon

  ![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Secure_Network_Beacon.png)

由上面的抓包的截图，我们可以看出跟帧格式的那幅图基本上是一一对应的。
## Unprovisioned Mesh Beacon
### Len
这个字段的意思是说这个mesh beacon包有效数据的长度大小 **(不包含Len字段占用的长度)**,也就是说上述的抓包图的整个mesh beacon的长度是25字节，**Len字段**占用一个字节，那么**Len字段**就表示接下来的有效数据的长度，这个跟普通的广播包是一模一样的套路。当然，这也是非常正常的现象；因为，mesh本来就是利用了BLE的广播包进行数据传送。
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/mesh_beacon_valid_data.png)

### Type
该字段用于往外告诉周边的设备说，它是一包Mesh Beacon数据包，有别于其他普通的BLE广播包。

- 0x29

    表示这个是PB-ADV数据类型，常用于采用广播承载时发送Generic Provisioning PDU
- 0x2A

    表示这个是Mesh-Message数据类型，用于入网之后进行数据交互的数据格式
- 0x2B
    
    表示这个是Mesh Beacon数据类型
- 0x01
    
    表示这个是一个普通的BLE广播包

所以，接受者一旦收到该广播包的数据时就可以通过上面的字段，判别是Mesh Beacon还是普通的BLE广播包。

### Beacon Type
这个用于向外表明当前的设备是否已经加入过Mesh网络：

- 0x00

    表示当前的设备还是unprovisioned状态,等待加入相应的mesh网络，即**Unprovisioned Device beacon**
- 0x01

    表示当前的设备已经加入了mesh网络，即**Secure Network beacon**

- 0x02-0xFF

    这个范围的值，保留用于以后使用

### Device UUID
该字段通俗地来说，表示每个设备的唯一标识号(128 bit)。你也可以简单地理解为设备的身份证号码。这个对于应用层的用户来说，我们不必过于关注其是如何生成的。因为，设备厂商会根据相关的标准去生成这些UUID，从而极大地保证UUID的唯一性。如果感兴趣的朋友，可以点击[此链接](https://tools.ietf.org/html/rfc4122#section-4.2)获取更多的信息。

### OOB Information
首先，映入眼帘的是**OOB**这个单词。我估计有很多读者看到这个单词的第一反应就是 **“What the hell?”**。
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/what.gif)

OOB的全称是**Out of Band，也就是带外数据的意思**，其实说白了就是采用与当前所使用的方式不同的方式去传送数据，如NFC、条形码、二维码等等。但是，前提是该设备必须具备这样的能力 **(有点类似于BLE中配对时的IO Capabilities)**，否则你将不需要设置该字段的内容。目前，Mesh Spec v1.0.1支持的OOB Information如下所示：

| Bit | Description |
|-----|-----|
| 0 | Other    |
| 1 | Electronic / URI    |
| 2 | 2D machine-readable code    |
| 3 | Bar code    |
| 4 | Near Field Communication (NFC)    |
| 5 | Number    |
| 6 | String    |
| 7 | Reserved for Future Use    |
| 8 | Reserved for Future Use    |
| 9 | Reserved for Future Use    |
| 10 | Reserved for Future Use    |
| 11 | On box    |
| 12 | Inside box    |
| 13 | On piece of paper    |
| 14 | Inside manual    |
| 15 | On device    |

那么，这个字段是干什么用的呢？主要还是用于驱动配置入网的过程 **(例如带外转送设备的公钥)**

### URI Hash
这个是一个可选的字段，也就是说它是可有可无的。它主要就是告诉provisioner当前你配置的设备是什么类型的东西，除此之外似乎也没有太大的作用。其值是由如下面的计算公式所得：
> URI Hash = s1(URI Data)[0-3]
## Secure Mesh Network Beacon
### Len
此处不再重复详述，查阅上面的**Unprovisioned Mesh Beacon**的**Len**章节即可。

### Type
此处不再重复详述，查阅上面的**Unprovisioned Mesh Beacon**的**Type**章节即可。

### Beacon Type
此处不再重复详述，查阅上面的**Unprovisioned Mesh Beacon**的**Beacon Type**章节即可。

### Flags
该字段主要包含**Key Refresh Flag**和**IV Update Flag**两个内容:

| Bits | Definition |
|-----|-----|
| 0 | Key Refresh Flag (0: False 1: True)    |
| 1 | IV Update Flag (0: Normal operation  1: IV Update active)    |
| 2–7 | Reserved for Future Use     |

其主要的作用就是用于告诉周边的设备，我要准备进行NetKey和AppKey或者IV Index更新了；一旦更新完毕之后，相应的标志位就会被清0。

### Network ID
顾名思义就是网络ID号，也就是说当前Secure Network Beacon所加入的Mesh网络的ID号；这个ID一般是不会变的，除非你加入了新的另外一个网络或者NetKey更新了才会变更。

### IV Index
同样的，该字段表示这个mesh网络当前的IV索引号。为什么要说是当前的呢？这表明这个IV Index是可以更新的。那么该IV Index又有什么作用呢？

1. 用于应用层和网络层的身份验证和加密
2. 当**Sequence Number**快要溢出时，发起**IV Update**程序，让**IV Index**值增加，从而防止**Sequence Number**被重复使用

### Authentication Value 
最后一个字段，看单词就知道肯定又是跟加密相关的。该值是通过以下的公式算出来的：

> Authentication Value = AES-CMAC <sub>BeaconKey</sub> (Flags || Network ID || IV Index) [0–7]

至于**BeaconKey**是哪里来的，我们不必过份追究；无非就是从NetKey派生出来的，通过一系列的公式算法计算出来的。
```c
salt = s1(“nkbk”)
P = “id128” || 0x01
BeaconKey = k1 (NetKey, salt, P) 
```
如果有读者希望深层次地了解,可以阅读Mesh Spec v1.0.1的第**3.8.6.3.4**章节。同样的，Network ID也是从NetKey中派生的 **(可以阅读Mesh Spec v1.0.1的第3.8.6.3.2章节)** ：
> Network ID  = k3 (NetKey)

总而言之，一个NetKey只能生成一个**BeaconKey**和**Network ID**,从而也是说只能生成一个**Authentication Value**。

