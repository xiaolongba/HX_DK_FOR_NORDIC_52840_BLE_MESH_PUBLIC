<!--
 * @Description: In User Settings Edit
 * @Author: your name
 * @Date: 2019-07-28 13:31:42
 * @LastEditTime: 2019-08-30 22:33:40
 * @LastEditors: Please set LastEditors
 -->
# 前言
在开始真正的基于Nordic BLE芯片的SIG Mesh开发之前，我们需务必先了解Nordic官方提供的SDK包的框架，**不要一上来就操作猛如虎，后来发现自己250**

![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/crazy%20operation.png)

# 如何下载Mesh SDK
目前下载Mesh SDK开发包的方式有两种：
- [Nordic官网下载](https://www.nordicsemi.com/Software-and-Tools/Software/nRF5-SDK-for-Mesh/Download#infotabs)

  这种方式比较简单，但是我不推荐。原因是你要定期上官方查看是否有更新。众所周知，Nordic官网有的时候真的是卡，遇到暴脾气的工程师估计要骂娘了:laughing:因此，这种方式我就不细说了；
- Nordic的官方Github代码仓下载
  
  小编是比如推崇这种方式的，充分利用Git工具的优势，具体做法如下：
  - 下载
    ```c
    git clone https://github.com/NordicSemiconductor/nRF5-SDK-for-Mesh.git
    ```
    ![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Mesh%20downloading.gif)
  - 更新  
    ```c
    git pull
    ```
    ![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/mesh_sdk_update.png)

# SDK一览
介绍完如何下载SDK之后，现在让我们看看Nordic_MESH SDK的文件框架是怎么样的。
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/mesh_sdk_contents.png)

- .git
  
  这个存放一些git相关的内容，如更改的内容及新增的记录。只要是从Github下载下来的都有这个隐藏的文件夹；

- bin
  - blinky

    存放着Nordic 不同芯片不同版本协议栈的blinky工程的Hex文件，用户只要对号入座即可在这自己的平台上运行了；

  - bootloader
    - armcc
      
      存放不同芯片不同版本协议栈的带串口以及不带串口的bootloader Hex文件；其中**armcc**就是Keil ARMCC工具链
    - gccarmemb

      跟上面的**armcc**的一样，都是bootloader的Hex文件,只是工具链不同而已，而**gccarmemb**就是GNU ARM Embedded工具链
  - merged
  
    这个文件夹存放了一些例程合并的Hex文件,如bootloader和不同协议栈的合并、应用程序和不同协议栈的合并文件，是官方预先编译并合并好的Hex文件，用户可以直接下载至相对应的与型号和开发板即可工作；
  - ospace

    这个文件夹跟上面的文件夹有一些重合，也是包含一些官方预先编译好的相关示例工程的Hex文件以及一些使用到的第三方的库文件；
  - softdevice

    存放不同版本的协议栈的Hex文件，这个源码是不会公开的均是以Hex文件的形式提供。
- CMake
  
  CMake相关的一些文件，至于什么是CMake？可以参考此[链接](https://www.baidu.com/link?url=1UTIpsiOR1qjnqbkcT8CaQdM4AjGodTgAc4xgaAwAye&wd=&eqid=9629f3c600040a3d000000065d44084b)，简单地来说就是用它来生成makefiles，然后使用编译工具链生成我们最终的固件。

- doc

  该文件夹存放的是Nordic官方提供的一些入门的学习文章，如“模型”的概念、如何配置Mesh、Nordic Mesh的框架介绍等等，**强烈建议新人从这个文件夹开始你们的Mesh之旅**。

- examples

  从文件夹的命名，我们可以很容易理解该文件夹存放的就是官方提供给我们的示例工程，我们可以参考这些例程开始我们的项目；如果不理解这个示例工程是做什么用的，我们可以阅读该文件夹下的 **“README”**即可了解各个示例工程的功能。

- external

  这里存放的是Nordic使用到的第三方的功能模块的源码。

- mesh

  该文件夹是SDK最重要的组成部分，这里存放的是Nordic Mesh的源码，基本上我们以后用到的Mesh相关的功能都是在这个文件夹中实现的，但是我们这里了解即可，**强烈不建议去阅读这些源码！！！**

- models

  顾名思义，该处存放的是目前为止Nordic官方已经实现的Model，包含SIG Model以及私有Model的源码。同样的，我们只需要了解即可无须深究源码，除非你自己想要实现自已的Model。

- scripts

  这里存放的文件也是很有用处的，这是官方提供的一个完整的mesh串口接口的python脚本，用户可以利用这些脚本通过串口实现mesh网络的创建以及mesh数据包的收发，非常适合于那些不太想了解Mesh而想使用Mesh的用户，例如网关。

- tools

  该处存放的是Nordic官方提供的一些python脚本工具，用于空中升级以及一些文档的生成，这里我们目前暂时用不到，更多的是Nordic官网的临时产物，而且目前提供的工具也比较少。
- ......

  也许官方以后会新增更多的文件夹，分类地更加详细；然而，截止目前为止，仅有以上这几个文件夹。

# 总结

Nordic官方的Mesh SDK包大体上分类的还可以的，但是有一些内容还是看得人云里雾里的，如果没有查看 **“README”** 的话，如**external**文件夹里也存放有协议栈的相关内容；如**tools**文件夹中提供的一些python脚本，光看名字只能了解一个大概，但是压根不知道怎么使用:laughing:；同理**scripts**也是，没有相关的文档说明如何使用这些脚本。
      
