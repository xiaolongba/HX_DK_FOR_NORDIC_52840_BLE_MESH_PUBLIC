<!--
 * @Description: In User Settings Edit
 * @Author: Helon CHEN
 * @Date: 2019-08-15 21:08:42
 * @LastEditTime: 2019-08-30 22:00:02
 * @LastEditors: Please set LastEditors
 -->
# 前言
经过前面几个基础章节地讲解，我相信大家就算不能很熟悉地了解SIG MESH,也应该有一定的认知。因此，接下来是时候给大家演示如何使用SES搭建SIG MESH的开发环境了。
# 什么是SES
SES是[SEGGER Embedded Studio](https://www.segger.com/products/development-tools/embedded-studio/)的缩写，后继小编将用SES来代替它。SES是SEGGER公司开发的一个跨平台IDE **(支持Windows、Linux、MaC OS)**。至于SEGGER公司是谁？如何有谁没有听说过，那么他肯定是一个假嵌入式人，它就是大名鼎鼎的、我们人手都有的JLINK调试工具就是它们家搞得。从用户体验上来看，其是优于**IAR和MDK**的。同时，使用Nordic的BLE芯片是可以免费使用这个IDE，没有版权的纠纷 **(Nordic官方跟SEGGER就这事已经谈妥了)**。
# 前期准备
在我们开始环境搭建之前，我们还需要下载如下工具：

- [SEGGER Embedded Studio](https://www.segger.com/products/development-tools/embedded-studio/)
- SIG MESH的SDK包
    
    下载方式和对应的方法可以参考[Nordic MESH SDK 文档框架简介](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PRIVATE/blob/master/%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/Nordic%20MESH%20SDK%20%E6%96%87%E6%A1%A3%E6%A1%86%E6%9E%B6%E7%AE%80%E4%BB%8B.md)
- nRF SDK包

    看过我前面文章的朋友应该知道，SIG MESH是基于低功耗蓝牙的一套应用层协议，因此我们还需要下载nRF SDK。**但是，注意版本不要随便下，应该看SIG MESH的SDK包的Release Notes**。
    ![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PRIVATE/blob/master/Material%20library/BLE_MESH_Release_Notes.png)

    - SDK下载
    
        遗憾的是nRF SDK包似乎没有上传至Github，因此只能从官方下载了，下载地址[请点我](https://www.nordicsemi.com/Software-and-Tools/Software/nRF5-SDK)
    - 离线API手册

        这个也是一个非常重要的资料，可以很方便地查阅各个API的功能及使用方法，具体的下载链接[请点我](http://developer.nordicsemi.com/nRF5_SDK/)。当然，如果你觉得下载太麻烦且懒得下载，你也可以在我们红旭的服务器上查看 **(由于服务器是在国内，所以打开的响应速度会很快)**，具体的链接[请点我](http://nordic.wireless-tech.cn/nrf5/index.html)
# 环境搭建
下载完SIG MESH以及nRF SDK开发包之后，两者可以不用放在同一个路径或者目录下，它们彼此相互独立又互相依依赖。因为我们知道，SIG MESH是基于低功耗蓝牙的一套应用层协议，那么如何让它们两者之间关联起来呢？具体的操作如下：

1. 使用SES打开SIG MESH的示例工程，这里小编以**light_switch_server**为例；
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PRIVATE/blob/master/Material%20library/light_switch_server_nrf52840.png)

2. <code>Tools</code>--><code>options</code>--><code>Building</code>--><code>Golbal Macros</code>，在这个选项填充nRF SDK开发包的绝对路径，如下所示 **(这里是小编的地址，读者可以根据自己的路径做相应的修改，注意左斜杠与右斜杠之分)**；
    ```c
    F:/Bluetooth/Nordic/SDK/nRF5_SDK_15.3.0_59ac345
    ```
3. 完成上面的操作之后，如果设置正确的话那么此时就可以直接编译SIG MESH开发包中的示例工程了。

为了更清晰明了地去阐述上述的操作，请看下图：
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PRIVATE/blob/master/Material%20library/Global_Macro_Setting.gif)
    
# 最后
这里在结尾处，还是要提醒一下广大读者的就是，上面的操作调试下载是需要提前下载低功耗蓝牙协议栈的，即在跑SIG MESH相关的代码之前是需要事先下载SoftDevice，这一点是要大家注意一下的。