<!--
 * @Author: 临时工
 * @Date: 2019-07-28 13:26:25
 * @LastEditTime: 2019-12-05 21:37:18
 * @LastEditors: Please set LastEditors
 * @Description: In User Settings Edit
 -->


# 前言
继Zigbee、Thread、WiFi Mesh之后，物联网行业中的组网阵营又冒出了一匹黑马---**BLE Mesh**。然而在这之前，BLE的组网能力在江湖上是排不上名号的。鉴于IPHONE 4S对BLE的普及起到了火箭级的推动作用，使得BLE几乎是智能穿戴届的**一哥**。可是随着越来越复杂的IoT应用，相较于之前点对点的数据传输，组网的需求越来越旺盛。因此，基于BLE的广播模式的多对多网状网络将彻底打破人们固有的成见，这也引起了无线通讯半导体芯片各领头养的关注如**CSR、Broadcom、NXP、Nordic、Dialog**等公司都推出过自己私有的蓝牙Mesh协议 **（Telink也因其较早推出BLE Mesh而一战而红）**。然而，原本是一件让物联网振奋人兴的事却因为各大厂商之间的MESH很难做到互联互通，为了避免重走当年Zigbee的老路。这个时候有位老大哥就站出来了![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/%E5%A4%A7%E4%BD%AC%E5%B8%A6%E5%B8%A6%E6%88%91.jpg)

这个转折点就出现在2014年，CSR决定将自己的私有蓝牙Mesh协议捐赠给蓝牙技术联盟（Bluetooth SIG），以加速制定统一的蓝牙Mesh物联网协议，尽快结束蓝牙阵营内部的纷争。最终，在2018年7月18日SIG正式推出蓝牙Mesh标准，即**SIG MESH**。

# 亮点
一个新生事物出来后少不了需要与 **“前辈”** 们对比的，中国不是有句古话说 **“是螺子是马拿出来溜溜”**。
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Comparison%20table.jpg)

从上面的表格可以看出，SIG MESH具备低功耗、低成本以及可以直接与我们的智能手机相互通讯。除了这些之外，SIG MESH还具备这样的优势：
- 节点设置自由，加入和脱离都比较容易，这也就是说当网络中某个节点出现故障时，整个网络仍可保持通信正常，具有组网便捷、抗干扰能力强等优点
- Mesh网的任何一个节点都可以连接手机 **（前提是该节点具备Proxy特性）**
- 去中心化，所有节点平等，解决了节点连接中心点困难的状况
- 每个节点都可以作为操控平台，极大的便利了家居使用的各种场景
- 可以做到银行等级的信息加密，保证传输信息安全的问题

当然啦，任何一门技术都不太可能一统天下，也有他们的局限性的，SIG MESH也是如此。
- 通讯延迟，既然在mesh网络中数据通过中间节点进行多跳转发，每一跳至少都会带来一些延迟，随着无线mesh网络规模的扩大，跳接越多，积累的总延迟就会越大。一些对通信延迟要求高的应用，可能面临无法接受的延迟过长的问题
- 数据吞吐量，SIG MESH主要用于传输少量数据的指令，而不是用来传输影像资料

# 什么是SIG MESH
SIG MESH目前采用的是基于flooding **(泛洪)** 协议的MESH网络技术。在目前发布的v1.0版本的Spec，其仅支持泛洪的方式，但是在未来的修订版本中可能会加入基于路由协议的MESH网络，具体什么时间会发布？我们只需耐心等待即可，因为你急你也改变不了什么:bowtie:。

![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Mesh_topology.png)

SIG Mesh网络由Mobile Phone、Node组成，其中Mobile是智能手机，作为Mesh网络的控制端。Node是网络中的节点设备。BLE Mesh网络是采用广播的方式实现的，基本步骤是： 

  1. 由节点A广播消息出去；

  2. 当节点B收到节点A的消息后，再把节点A的消息广播出去；

  3. 以此类推，利用感染的方式，一传十，十传百，让所有在无线范围内的装置都收到此消息。

  ![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Flooding%20Broadcast.png)
 
SIG MESH还会对网络中的数据进行特殊的加密，防止通过监听和中间人攻击等手段窃取网络数据。

# 澄清
经常听到有人问 **“请教一下你们会搞蓝牙5.0的mesh吗？”**，其实这种问法本身就是有误的，因为蓝牙5.0 Specification和 Mesh Specification发布的时间都比较相近，从而导致这样的 **“笑话”**
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Bluetooth%205.0%20VS%20BLE%20Mesh.png)