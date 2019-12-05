<!--
 * @Description: In User Settings Edit
 * @Author: 临时工
 * @Date: 2019-08-06 22:05:02
 * @LastEditTime: 2019-12-05 21:36:54
 * @LastEditors: Please set LastEditors
 -->
# 前言
不管是Bluetooth low energy还是SIG MESH的载荷包，都是由不同的协议层从上到下层层拼装，最后由PHY层将数据发送出去；因此，层数越多就意味着越复杂。正因为如此，我们想要深层次的了解SIG MESH就务必要先了解SIG MESH协议栈各层的作用，只有这样你才能玩弄MESH于股掌之中，而不是云里雾里。

# SIG MESH 协议层一览
在我们开始讲解各个层的作用之前，先看看整个SIG MESH协议层是长什么样的。
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/mesh_system_architecture.png)

由上图可知，SIG MESH由**模型层**、**基础模型层**、**访问层**、**上层传输层**、**下层传输层**、**网络层**、**承载层**组成；这个时候可能读者会问，为什么**低功耗蓝牙核心规范**不被包含在内呢？
> 在回答这个问题之前，我们先看看Mesh数据包的格式：
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Mesh_packet.svg?sanitize=true)
从上面的mesh数据包的分解图来看，这个时候我想读者们明白为啥**低功耗蓝牙核心规范**不被包括在内了吧？mesh的数据包实质是利用低功耗蓝牙收发数据，至于mesh的数据包是如何拼装、怎么组装、以及加解密等等那才是mesh协议栈做的事情。因此，我们也可以简单地说mesh协议栈就是一个基于BLE的Big profile。另外，小编要提到的是：**最大的不分组的能给应用层用的有效载荷包就是11字节！！！注意，我这里是仅仅指的是不分组时的最大PDU，实际一次可以发送最大380字节的mesh数据，但是要分包**

从上面的内容，我想起码对各个层的作用有了一个大概的了解了，对于这个层数据包的详解，我们会在后面的章节根据实际的案例捉包讲解。这里小编就不在此处开展，本章节的重要是让大家知道mesh协议栈各层的作用是什么。

## 承载层
mesh数据包的收发是需要媒介的，从上面的内容我们可以知道，其利用了低功耗蓝牙进行数据的收发；承载层定义了节点之间是如何传送网络信息的，其定义了两个承载：
- GATT

    那么GATT承载的作用到底是什么呢？就是让不支持mesh协议栈的设备与mesh网络的节点间接地进行通信；而其又是通过什么方式与网络的节点进行交流呢？就是使用“代理协议”，没错！就是Proxy节点，其是可以实现这些GATT特性的，并且同时支持GATT承载和ADV承载 **（有的人可能会问为啥还要支持ADV承载？这不是废话吗，如果不支持的话怎么让不支持mesh协议的设备的数据送到mesh网络中去啊，就是因为正常mesh网络中的数据收发都是通过ADV承载实现的）**
    
    ![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/talking_about_proxy_node.gif)
    
    也就是说支持代理协议的节点是支持**mesh proxy service** 以及广播行为的，为了让大家更好的理解，我们可以看看下所示的拓扑图：
    ![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/mesh_proxy_service.png)

- ADV

    ADV承载利用BLE的GAP广播和扫描来传送和接收网络PDU，如下图所示：
    ![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/mesh_message_ad_type.png)

## 网络层
网络层是SIG MESH最重要的组成部分，其功能可以分为以下几个部分：

1. 定义了被承载层转输的底层传输层PDU的网络PDU格式，初看可能会有点拗口，但是你可以上面章节的Mesh数据包分解图，你就能理解了。底层传输PDU被合并并组装成网络PDU，然后再被转送到承载层；
2. 它解密并验证和转发在输入接口接收到的转送进来的信息到上层传输层，和或者输出接口，并加密和验证并转发派发到输出网络接口的输出信息；这里读者可能会有疑问，什么是接口？还特么分输入和输出？
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/doubts_emoticon.png)

    不管是输入还是输出接口，它们都叫做 **“网络接口”**。

    - 输入接口
      
      决定传入的mesh信息派发到网络层还是丢弃掉，也就是说决定将mesh信息转入到网络层还是直接丢弃掉；
    - 输出接口
      
      决定传出的mesh信息到承载层还是丢弃掉，同时不管是GATT承载还是ADV承载都要将TTL等于1的所有信息全丢掉；同理，也就是说决定将组装好的网络层信息传送到承载层还是丢弃掉；

在这里，细心的读者可能已经察觉到**代理**以及**中继**的特性是网络层的一种行为，也就是说代理和中继的功能在这一层得到实现。因为上面的内容不断地再强调转发、转发、还是转发。只不过一个是通过ADV承载将网络层的PDU转发，另外一个则是在GATT和ADV承载之间转发。前者的行为属于中继，而后者则是代理。

## 底层传输层
从上面章节的Mesh数据包分解图可以看出，底层传输层从上层输层接收上层传输PDU，然后将这个信息发送到对端设备的底层输层，我想应该有人不知道什么是对端设备吧？
>比如A-》B，那么就A而言，B就是A的对端设备

但是，上层传输层的**加密访问载荷包**最大是380字节，而底层传输层想要传送这么大的数据量怎么办呢？这个时候只能采用分段重组的方式了。所以如果要传送的数据量大于11字节，那么此时就必须要分段了。而对于对端设备而言，如果接收到的不是分包的上层传输层PDU的底层传输PDU的话，它的底层传输层则直接把该PDU传给上层传输层，否则需要将这些分包的上层传输层PDU的底层传输PDUs重组，一旦完成重组刚上传至上层传输层；可能文字描述不易理解，我们可以结合下图就可以很清楚地理解了。
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/segmentation_reassembly.png)

显然，又会有细心的读者发现，原来分段重组是在底层传输层处理实现的啊，随着慢慢地一步步地剖析，是不是感觉自己对mesh又懂了一些:bowtie:。

## 上层传输层
上层传输层的主要职责如下：

1. 从访问层接收访问层载荷包并使用**application key**对其进行加密和验证，然后将这些信息发送到对端设备的上层传输层，而对端设备的上层传输层会用相同的**application key**去验证接收到的信息的合法性，这一点我们仍然可以从上面章节的mesh数据分解图可以看出；
2. 内部生成的上层传输层控制消息，同样的将这个信息传送到对端设备的上层传输层；然而，控制信息则仅在网络层被加密与验证，这是1和2之间的不同之处；

该章节的内容不断地有提及到控制消息，那么控制消息是什么呢？

| 值 | 操作码 | 含义 |
|-----|-----|------|
| 0x00 | –    | Reserved for lower transport layer |
| 0x01 | Friend Poll   | Sent by a Low Power node to its Friend node to request any messages that it has stored for the Low Power node |
| 0x02 | Friend Update   | Sent by a Friend node to a Low Power node to inform it about security updates |
| 0x03 | Friend Request   | Sent by a Low Power node the all-friends fixed group address to start to find a friend |
| 0x04 | Friend Offer   | Sent by a Friend node to a Low Power node to offer to become its friend |
| 0x05 | Friend Clear   | Sent to a Friend node to inform a previous friend of a Low Power node about the removal of a friendship |
| 0x06 | Friend Clear Confirm   | Sent from a previous friend to Friend node to confirm that a prior friend relationship has been removed |
| 0x07 | Friend Subscription List Add   | Sent to a Friend node to add one or more addresses to the Friend Subscription List |
| 0x08 | Friend Subscription List Remove   | Sent to a Friend node to remove one or more addresses from the Friend Subscription List |
| 0x09 | Friend Subscription List Confirm   | Sent by a Friend node to confirm Friend Subscription List updates |
| 0x0A | Heartbeat   | Sent by a node to let other nodes determine topology of a subnet |
| 0x0B~0x7F | RFU   | Reserved for Future Use |

为了保证原汁原味，上面的表格我就不翻译了，好像翻译之后的意思有点不太对劲。还没有英文表达的含义准确。同时我们也从上表中可知，**朋友和低功耗**特性原来是在上层传输层实现的,另外的一个重要特性---心跳包也是在这一层实现的。

## 访问层
访问层负责定义应用如何利用上层传输层，包括：
- 定义应用数据的格式
- 定义并控制在上层传输层执行的加密和解密过程
- 在将数据上传到更高的层之前，对来自上层传输层的数据
进行验证，判断其是否适用于该网络和应用

该层已经非常接近应用层了，基本上这一层表示的数据来源于Mesh Model Spec规定的那些参数值了。

## 基础模型层
该层定义了访问层的状态、信息以及配置和管理网状网络所需的模型，至于相关状态的详细定义则不是Mesh spec的部分，我们可以从**Mesh Model Spec**获取得到这些细节的内容；同理，相关定义的信息的细节也是从**Mesh Model Spec**中可以获取得到；然而，配置和管理网状网络所需的模型主要是下面4个：
1. Configuration Server model
2. Configuration Client model
3. Health Server model
4. Health Client model

对于上面的这些模型的详解，我们后继会另外新的章节专门讲解，此处简单地了解即可。

## 模型层
模型层涉及模型的实施，因此涉及一个或多个模型规格 中定义的行为、消息、状态、状态绑定等的实现，更多的详细定义我们可以从**Mesh Model Spec**获知，这里不详述，小编也不建议去花太多时间去看，仅当遇到了相应的模型时再相应的查阅即可。
