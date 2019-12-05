<!--
 * @Description: In User Settings Edit
 * @Author: 临时工
 * @Date: 2019-09-13 15:08:28
 * @LastEditTime: 2019-12-05 21:39:30
 * @LastEditors: Please set LastEditors
 -->
# 前言
 我相信看过我们[PB-GATT入网过程](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/%E6%96%B0%E6%89%8B%E5%85%A5%E9%97%A8/PB-GATT%E5%85%A5%E7%BD%91%E8%BF%87%E7%A8%8B.md)的读者，应该对入网的全过程有了一个相对程度的了解吧；当然啦，如果短期间看不明白也没有关系，那就多看几遍。接下来，小编仍然还是给大家讲解Mesh入网的全过程，只不过此次是PB-ADV的方式入网；那么，接下来先讲讲什么是PB-ADV。
 
 ![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Time_To_Start.png)

 - 什么是PB-ADV?

    对于有一些了解的读者，看到这个名词时肯定会说“这不就是通过广播包互相收发数据，从而配置入网嘛”，其实对于总的概括来说是没有错的。但是，我们有一些关键的地方我们仍然还要注意：
    
    1. 与PB-GATT不同的是，PB-ADV使用**Generic Provisioning PDUs**配置new device,而PB-GATT则是通过**Proxy PDUs**，这两种是完全不同的PDUs；前者是通过37，38，39的广播通道进行通讯，而后者则是通过Proxy service进行数据收发，也就是数据通道进行通讯；
    2. PB-GATT是基于连接的模型进行一序列的入网操作，而PB-ADV则基于会话的模型进行相应的入网操作；前者，传输配置数据的最大PDU是由ATT的MTU的大小来决定；而后者的MTU则是最大就只有24个字节，且使用连接建立程序 **(Link Establishment procedure)** 进行会话的建立;

讲了这么多，接下来我们还是采用同样的套路，先看PB-ADV的帧格式，之后再分别层层剖析。
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/PB_ADV_Packet.png)

由上图可知，PB-ADV PDU主要由**Link ID**，**Transaction Number**，**Generic Provisioning PDU**三个部分组成；

## Link ID
该字段所表示的含义比较好理解，其起到的就是一个标识符的作用；主要用于在配置过程辨别同一个LINK ID下，Provisioner与unprovisioned device的数据交互；不同LINK ID的packets会被彼此忽略，因为一个会话的生命周期时有且仅有一个Link ID,不同的会话其LINK ID是不一样的。**注意，LINK ID是由Provisioner随机产生的**。

## Transaction Number
该字段主要用于辨别每个单独的**Generic Provisioning PDU**；即从0x00开始，该值在每个新的**Generic Provisioning PDU**上累加 **(对于Provisioning Bearer Control PDU，其Transaction Number一直都是0x00)**，如果累加到最大值时，则又从0x00重新开始；对于Provisioner和Unprovisioned device它们两者的最大值是不一样的：

- Provisioner

    最大只能达到0x7F;

- Unprovisioned Device

    最大可能达到0xFF;

但是，也有两种情况是不需要累加的：

1. 如果一个**Provisioning PDU**放不下去时，所有的分包均是同样的**Transaction Number**的，不需要累加；
2. 如果**Provisioning PDU**需要被重传，那么也是不需要累加的，还是使用旧的**Transaction Number**;
## Generic Provisioning PDU
该PDU的详细情况，小编不在此处展开，准备将其放在[Generic Mesh Provisioning Control](#Generic-Mesh-Provisioning-Control)和[Generic Mesh Provisioning Payload](#Generic-Mesh-Provisioning-Payload)两个章节中细述。
# 入网流程
同理，这个步骤跟我们上一章节[PB-GATT入网过程](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/%E6%96%B0%E6%89%8B%E5%85%A5%E9%97%A8/PB-GATT%E5%85%A5%E7%BD%91%E8%BF%87%E7%A8%8B.md)中的**入网流程**小章节所细述的内容是一模一样，小编这里就不再细述；下图所示的是PB-ADV方式入网的全过程：
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Provisioning_Process_ADV.png)

与PB-GATT相比之下，多了**Generic Mesh Provisioning Link Open、Generic Mesh Provisioning Link Ack、Generic Mesh Provisioning Link Close**，那么这三者之间有什么关联呢？我们在上述的内容也有提到这一点，PB-ADV是采用会话的模型进行配置数据的收发，那么会话模型就需要这三个PDU来建立，具体如下图所示：
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Establishment_of_Link.png)

> 注意：上述的这三个PDU均属于**Provisioning Bearer Control PDU**的范畴，也就是说这三个PDU根据不同的场景所组装成不同的PDU就表示**Provisioning Bearer Control PDU**；

## Generic Mesh Provisioning Link Open
Link Open消息用于在Provisioner和Unprovisioned Device之间建立链接,Unprovisioned Device如果接收到该消息，应使用Link ACK消息应答此消息。具体的帧格式如下：
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Link_Open_message.png)

其中Device UUID指的是选中的Unprovisioned Device的UUID。
## Generic Mesh Provisioning Link Ack
该消息用于发送应答消息，表示其收到Link Open的消息了，具体的帧格式如下：
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Link_Ack_message.png)

这里值得注意的是：应答消息是不携带任何参数的。
## Generic Mesh Provisioning Link Close
该消息用于关闭Link会话，由于该消息是没有被应答的，因此为了保险起见，Link关闭消息最少要发送3次且Provisioner和Unprovisioned Device均可以发送该消息，并且携带**关闭原因**这个参数。具体的帧格式如下图所示：
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Link_Close_message.png)

> 注意：关于**Link Establishment procedure**我们还有以下几点需要大家了解下：
>1. 对于Provisioner而言，其在准备建立Link会话时会先设置一个60秒的定时器，接着发送**Link Open**消息；如果在60秒后没有收到Unprovisioned Device的**Link Ack**的消息，则认为此次Link会话建立失败；
>2. 对于Unprovisioned Device而言，其在收到**Link Open**消息之后，会给Provisioner回应**Link Ack**的消息，紧接着同样也开启一个60秒定时器；如果在60秒后没有收到任何的Provisioning PDU，则认为此次Link会话建立失败；

## Generic Provisioning PDU
除了上述的之外，这两种方式最大的不同是：

- PB-GATT

    通过**Proxy PDUs**进行配置数据的收发
- PB-ADV

    通过**Generic Provisioning PDUs**进行配置数据的收发

至于**Proxy PDUs**,我们在**PB-GATT入网过程**已经细述过，该小章节我们着重描绘**Generic Provisioning PDUs**的内容，如下图所示：
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Generic_Provisioning_PDU.png)

由上图可知，该PDU主要由**Generic Provisioning Control**和**Generic Provisioning Payload**组成。

### Generic Mesh Provisioning Control
该字段主要表明Generic Provisioning Control域包含的是哪一种类型的GPCF,至于一共有哪几种类型的Generic Provisioning Control,上述的Generic Provisioning PDU示意图已经描述地很清楚了：

- Transaction Start             -0b00
- Transaction Acknowledgment    - 0b01
- Transaction Continuation      - 0b10
- Provisioning Bearer Control   - 0b11

#### Transaction Start
Transaction Start PDU用于启动分段消息的转输，也就是说每一个Provisioning PDU的消息传输都是使用该PDU开始传输。而Provisioning PDU到底都是些什么帧格式呢？在上一章节[PB-GATT入网过程](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/%E6%96%B0%E6%89%8B%E5%85%A5%E9%97%A8/PB-GATT%E5%85%A5%E7%BD%91%E8%BF%87%E7%A8%8B.md)中已经详述过，即

| Type | Name |
|-----|-----|
|0x00|Provisioning Invite|
|0x01|Provisioning Capabilities|
|0x02|Provisioning Start|
|0x03|Provisioning Public Key|
|0x04|Provisioning Input Complete|
|0x05|Provisioning Confirmation|
|0x06|Provisioning Random|
|0x07|Provisioning Data|
|0x08|Provisioning Complete|
|0x09|Provisioning Failed|
|0x0A-0xFF|RFU|

该PDU的具体帧格式如下表所示：

![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Transaction_Start_PDU.png)

| Field | Size (bits) |Description|
|-----|-----|-----|
|SegN|6|The last segment number|
|GPCF|2|0b00 = Transaction Start|
|TotalLength|16|The number of octets in the Provisioning PDU|
|FCS|8|Frame Check Sequence of the Provisioning PDU|

##### SegN
该字段其实就是表示除了Transaction Start中携带了一些Provisioning PDU,剩下的Provisioning PDU还需要多少个分段的包才能发送完成；
##### GPCF
对于该PDU，该字段固定为0x00
#### TotalLength
表示要表达的Provisioning PDU总有多少字节
##### FCS
该字段则表示对Provisioning PDU进行计算后的帧检验值

为了上述更易于理解，我们可以查看如下所示的捉包图：
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Transaction_Start.png)

#### Transaction Acknowledgment
Transaction应答PDU用于应答接收到的Provisioning PDU,跟其他的应答消息一样，它也是不携带任何参数的。
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Transaction_Acknowledgment_PDU.png)

| Field | Size (bits) |Description|
|-----|-----|-----|
|Padding|6|0b000000; all other values Prohibited|
|GPCF|2|0b01 = Transaction Acknowledgment|

#### Transaction Continuation
使用Transaction Start PDU没有办法将整个Provisioning PDU传送完成，就应该使用该字段传输分包的Provisioning PDU。具体的帧格式如下所示：

![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Transaction_Continuation_PDU.png)

| Field | Size (bits) |Description|
|-----|-----|-----|
|SegmentIndex|6|Segment number of the transaction|
|GPCF|2|0b10 = Transaction Continuation|

##### SegmentIndex
该字段表示当前的Provisioning PDU是第几个分包；

##### GPCF
固定为0b10,表示Transaction Continuation；

##### Data
表示各个分包Provisioning PDU的内容；

#### Provisioning Bearer Control
该类型的PDU是上述的[Generic Mesh Provisioning Link Open](#Generic-Mesh-Provisioning-Link-Open)、[Generic Mesh Provisioning Link Ack](#Generic-Mesh-Provisioning-Link-Ack)、[Generic Mesh Provisioning Link Close](#Generic-Mesh-Provisioning-Link-Close)的统称，这也就是说该PDU是由根据不同的**BearerOpcode**类型来表示不同的PDU。
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Provisioning_Bearer_Control_PDU.png)
### Generic Mesh Provisioning Payload
该字段所描述的PDU内容其实是根据[Generic Mesh Provisioning Control](#Generic-Mesh-Provisioning-Control)设置的不同而不同，其本质就是用来传输Provisioning PDU，至于怎么传输，传输哪些内容则是看当前处于入网的哪个阶段，Provisioning PDU有多少字节。**例如：当前Provisioner与Unprovisioned Device进入到交换公钥的阶段，这个时候就必须使用[Transaction Start PDU](#Transaction-Start-PDU)和[Transaction Continuation PDU](#Transaction-Continuation-PDU)来传输Provisioning PDU**。
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Provisioning_Public_Key.png)


同时，我们在刚看上面[前言](#前言)中的PB-ADV图时，肯定会有读者在想“**Generic Provisioning PDU**不是最大只支持24字节吗？为什么下面的**Generic Provisioning Payload**是0~64字节，这不是比24字节还大吗？” 我想有这样疑问的读者，现在应该知道为什么了吧？**没错，放不下去我还可以分包啊**。

## 总结
在[入网流程](#入网流程)中的最前面提及到整个PB-ADV流程中可以看出，基本上跟PB-GATT是一模一样的，所以这个章节就不再重复。不同的点我在上述也已经一一细述。但是还有一些细节，小编觉得还是有必要要了解一下：

- 跟普通的广播包一样，Generic Provisioning PDU的发送也是要插入一个20~50ms的随机时间，这样就可以大大降低信道冲突的可能性，从而提高鲁棒性；
- 当收到Transaction Acknowledgment的消息，表示Provisioning PDU传输完成。尤其对于需要分包发送的场合，Unprovisioned Device接收完Provisioning PDU之后会给Provisioner发送这个应答，这个时候可能读者会问 **“Unprovisioned Device怎么知道已经接收完Provisioning PDU呢？”**不知道读者还有没有印象，在[Transaction Start](#Transaction-Start)中有个**SegN**字段，Unprovisioned Device可以根据它判断整个Provisioning PDU是否已传输完成；

- 在一些需要应答的数据交互，双方将第一条消息发送出去之后，如果在30秒内没有收到应答，那么此次入网失败；

- 当Unprovisioned Device接收完Provisioning PDU之后，它会计算Provisioning PDU的FCS，并与[Transaction Start](#Transaction-Start)中的**FCS**字段的内容相比较，如果匹配则发送应答消息；

- 当且仅当Provisioner发送给Unprovisioned Device Provisioning PDU时，才需要发送应答。否则是不需要发送应答信息的；也就是说Unprovisioned Device给Provisioner发送的消息是不需要发送应答的；
