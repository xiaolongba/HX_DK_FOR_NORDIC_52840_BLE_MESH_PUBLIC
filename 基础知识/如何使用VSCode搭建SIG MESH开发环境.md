<!--
 * @Description: In User Settings Edit
 * @Author: Helon CHEN
 * @Date: 2019-08-17 13:36:51
 * @LastEditTime: 2019-08-30 22:37:24
 * @LastEditors: Please set LastEditors
 -->
# 前言
继上一章节讲完了**如何使用SES搭建SIG MESH开发环境**，我相信大部分人都已经可以愉快地使用SES开始自己的SIG MESH之旅了。然而，此次小编却剑走偏锋采用**CMake+Vscode+nRF5-SDK-for-Mesh**的组合来搭建此次MESH的开发环境。因此，本章节仅适合有一定基础且又进取的人，从而也不太适合啥都要现成的工程师 **（如果有的话，看到这里就可以移动你的鼠标到右上角，并单击）** 首先，采用这样的组合方式在业内不是属于首次，互联网上已经大量充斥着类似的文章。然而，更多的是基于纯应用层的应用，即没有跟硬件绑定在一起的相关场景。只有零星的几篇跟嵌入式开发相关的，因此小编觉得还是有必要借这样的机会写一写的，顺带促进自己的成长。

# 优势
那么这样的组合的方式有什么样的优势呢？小编认为是如下几种：

1. 开源免费，没有版权纠纷。因为CMake以及Vscode均是开源免费的工具软件；
2. 适合于模块化以及大项目工程开发且跨平台，不受平台限制；
3. 提升工程师自身的水平，更了解编译的原理

# 缺点
就我目前的了解，除了入门的门槛有点高之外也没有什么其他的缺点了。当然了，集成化的IDE在调试方面还是会优于这种方式的。

# 准备工作
在开始环境搭建之前，我们还需要安装如下的工具软件：

| 工具名称 | 作用 |
|-----|-----|
| [nRF5x Command Line Tools](https://www.nordicsemi.com/Software-and-Tools/Development-Tools/nRF-Command-Line-Tools/Download#infotabs) | 用于下载以及合并固件，安装时应该将其添加到系统环境变量，必选    |
| [SEGGER J-Link](https://www.segger.com/downloads/jlink) | 用于代码调试，必选    |
| [Python 2.7](https://www.python.org/downloads/) | 用于运行OTA固件的相关python脚本，安装时应该将其添加到系统环境变量，必选    |
| [Python 3.5](https://www.python.org/downloads/) | 用于运行下载或者合并固件的相关脚本，安装时应该将其添加到系统环境变量且是32Bit的安装包（起码3.5.1版本以上）；**此处要注意：安装时应将其安装在盘符的根目录下，否则会跟Python2.7冲突**，必选    |
| [CMake](https://cmake.org/download/) | 用于生成本地的构建文件，安装时应该将其添加到系统环境变量（最新版本即可），必选    |
| [Ninja](https://github.com/ninja-build/ninja/releases) | 基于CMake生成的本地构建文件生成相应的Hex、ELF、MAP等文件，手动将其添加到系统环境变量（最新版本即可），必选    |
| [GNU ARM Embedded Toolchain](https://developer.arm.com/open-source/gnu-toolchain/gnu-rm/downloads/) | 工具链，下载**7-2018-q2-update (7.3.1)** ，其他版本的不要下载，安装时将其添加到系统环境变量，必选    |
| [Doxygen](http://www.doxygen.nl/)、[Graphviz](http://www.graphviz.org/)、[Mscgen](http://www.mcternan.me.uk/mscgen/)| 如果还要生成离线的nRF5-SDK-for-Mesh API文档你就必须下载这三个软件(目前官方暂没有提供现成的API手册，那么就只能自己生成了)，三个工具软件均要添加到系统环境变量，可选    |
| [Visual Studio Code](https://code.visualstudio.com/)| **Cortex Debug插件**，必选；**CMake、CMake Tools、CMake Tools Helper插件**，可选    |

除了上面提到的软件之外，当然你还要下载 **[nRF5-SDK-for-Mesh]((https://www.nordicsemi.com/Software-and-Tools/Software/nRF5-SDK-for-Mesh/Download#infotabs))** 和 **[nRF5-SDK]((https://www.nordicsemi.com/Software-and-Tools/Software/nRF5-SDK))** 两个SDK开发包。

# 环境搭建
安装完上述的软件之后，我们就可以直接用Visual Studio Code打开nRF5-SDK-for-Mesh了，如下所示：
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/open_nRF5_SDK_for_Mesh_with_vscode.gif)

同时，打开之后还需要安装上述提及到的Visual Studio Code插件：
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Extensions_for_debug.png)

## 配置Cortex Debug插件
该插件主要是使能在Visual Studio Code中调试Cortex-M内核的MCU，是一个非常不错的插件。由于我们使用的是JLINK调试仿真器，所以我们要告诉这个插件我们JLINK的位置，这样我们再按下F5时，其才会调用JLINK的GDB来进行调试。为实现上面的目的，我们可以这样做：<code>Manage</code>--><code>Settings</code>--><code>Extensions</code>--><code>Cortex-Debug Configuration</code>--><code>JLink GDBServer Path</code>--><code>Edit in settings.json</code>,然后填充自己的JLinkGDBServerCL的路径，下面是小编的路径
```c
C:\\Program Files (x86)\\SEGGER\\JLink\\JLinkGDBServerCL.exe
```
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Cortex_Debug_Configuration.gif)

完成上面的配置之后，我们还需要再配置下我们当前调试的是什么型号的MCU、什么调试接口等等，只有这些都完成了才能完美地使用该插件。以下是小编的配置内容，具体的操作请看如下的动图：
```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [            
        {
        "cwd": "${workspaceRoot}",
        "executable": "${workspaceRoot}/build/examples/beaconing/beaconing_nrf52840_xxAA_s140_6.1.1.elf",// 根据自身的情况填充自己的实际路径,这里我仅是我当前演示时使用的示例工程
        "name": "Debug Microcontroller",
        "request": "launch",
        "type": "cortex-debug",
        "servertype": "jlink",//调试仿真器的选择,可以支持opennocd、pyocd、pe以及stutil
        "runToMain": true,//一调试就停在main函数
        "device": "nRF52840_xxAA",//设备的型号
        "interface": "swd",//SWD接口
        "ipAddress": null,
        "serialNumber": null, 
        },
    ]
}
```
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/Cortex_Debug_Configuration_Json.gif)

## 编译
完成上面的配置之后，接下来我们就要利用CMake对工程进行编译并生成ELF、HEX等文件了。但是，由于我们今天的主角是nRF52840_xxAA，然而工程默认的是nRF52832_xxAA，所以我们还要修改一下相关的文件，如下图所示：

![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/modification_for_platform_cmake.png)

只有这个时候，我们才可以真正地对工程进行编译，具体的操作如下：
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/CMake_build_project.gif)

然后，我们就可以在根目录看到新建了一个**build**文件夹。
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/project_builded.png)

## 调试下载
编译完成之后，此时应该就会生成相应的ELF、HEX等文件了，那么我们这个时候就可以利用Cortex Debug插件进行调试下载，具体的操作请看如下的动图：
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/project_debug.gif)

## 如何生成nRF5-SDK-for-Mesh离线手册
目前，官方并没有把nRF5-SDK-for-Mesh离线手册放在官方的服务器上，而是将生成的方法告诉了我们；目前，我也搞不明白官方的想法。但是，没有关系我们按照官方的方法操作也是可以得到的，具体的操作如下所示：
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/doc_generation.gif)

最后，我们可以在**build**文件夹下，找到生成的离线文件：
![](https://github.com/xiaolongba/HX_DK_FOR_NORDIC_52840_BLE_MESH_PUBLIC/blob/master/Material%20library/doc_generation.png)