# 在EAIDK上部署TVM Runtime

很高兴在上周收到了EAIDK-310开发板，非常感谢OPEN AI LAB和极术社区。

在这篇文章中，我想介绍如何从零开始，在EAIDK-310上部署TVM Runtime, 并且运行TVM的几个教程代码。我将TVM官方提供的例子适配于EAIDK-310, 并上传到了：https://github.com/wkcn/TVM_tutorials_for_EAIDK    因为我也是第一次学习TVM的部署，文章中可能有些错漏，欢迎大家指出，谢谢！

# 概述
概述部分来自《EAIDK-310用户使用手册》以及TVM官网“关于”部分。

## EAIDK
EAIDK(Embedded AI Development Kit),是以 Arm SoC 为硬件平台、Tengine(Arm 中国周易平台)为核心的人工智能基础软件平台 AID、集成典型应用算法,所形成的 “软硬一体化”的 AI 开发套件;是专为 AI 开发者精心打造,面向边缘计算的人工智能开发套件。

硬件平台具备语音、视觉等传感器数据采集能力,及适用于多场景的运动控制接口。

软件平台支持视觉处理与分析、语音识别、语义分析、SLAM 等应用和主流开源算法,满足AI 教育、算法应用开发、产品原型开发验证等需求。

EAIDK-310 是 EAIDK 产品系列中第二款套件,主芯片采用具备主流性能 Arm SoC 的RK3228H,搭载 OPEN AI LAB 嵌入式 AI 开发平台 AID(包含支持异构计算库 HCL、嵌入式深度学习框架 Tengine、以及轻量级嵌入式计算机视觉加速库 BladeCV)。为 AI 应用提供简洁、高效、统一的 API 接口,加速终端 AI 产品的场景化应用落地。

## TVM
TVM是一个针对CPU、GPU和专用加速器的开放式深度学习编译器堆栈，它旨在缩小以生产力为重点的深度学习框架与性能或效率为导向的硬件后端之间的差距。TVM提供以下主要功能：

将Keras, MXNet, PyTorch, Tensorflow, CoreML, DarkNet中的深度学习模型，编译成各种硬件后端上可部署的最小模块。

基础架构可在更多后端自动生成和优化张量运算符，同时得到更好的性能。

# 前期准备
在部署之前，需要做一些前期准备，包括“远程连接EAIDK-310开发板”，“在开发板上安装TVM Runtime第三方依赖库”，大家可以选择性跳过。

## 远程连接EAIDK-310开发板
该部分介绍了如何通过ssh或vnc连接开发板，如果您全程使用外接显示器，可以跳过该部分。

EAIDK-310开发板预先安装了Fedora 28操作系统，开箱即用，默认开启了ssh服务端, 桌面环境为lxde, 网络管理器为NetworkManager. 初始帐号和密码均为openailab. 如果我们不想全程使用外接显示器，可以将开发板的操作系统设置为自动登录，并自动连接WIFI。这样在同一个局域网下，就可以使用ssh或vnc连接开发板。

有两种方法可以连接EAIDK-310开发板：
1. 通过HDMI接口连接显示器，进行配置
连接电源、HDMI, 鼠标、键盘后，开机，输入登录密码openailab后进入桌面，选择右下角的“网络”图标，选择WIFI并填写密码后，将会连接WIFI，并保存帐号密码。由于进入桌面后WIFI才会自动连接，因此我们需要设置自动登录。

设置自动登录：
sudo vim /etc/lxdm/lxdm.conf
找到autologin这一项，取消注释，改为autologin=openailab, 保存。

2. 通过串口调试的方法进行连接
EAIDK-310的调试串口波特率为1.5M Hz, 电平为TTL(5V), 需要支持1.5M Hz以上波特率的串口转USB, 如CH340系列。我尝试过使用Arduino实现串口转USB, 但失败了，原因是Arduino不支持1.5M Hz以上波特率。

按《EAIDK-310 用户使用手册》中的5.4串口调试说明进行连接后，可输入以下命令连接WIFI:

### 列出所有WIFI
nmcli device wifi list
### 连接WIFI
sudo nmcli device wifi connect <WIFI名称> password <WIFI密码>

经过配置后，开发板连接上了WIFI，在局域网内可以使用ssh进行连接。
可以通过设置固定IP或者在WIFI管理界面中查看的方式，获得开发板的IP地址。
通过命令：ssh openailab@<IP地址> ，如ssh openailab@192.168.43.33  (开发板在我的局域网下的IP为192.168.43.33) 进行连接，密码为openailab



# 
