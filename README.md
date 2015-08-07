[原文地址](https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization#Game_Design_Implications_of_Lag_Compensation)


## Overview
## 概要


Designing first-person action games for Internet play is a challenging process. Having robust on-line gameplay in your action title, however, is becoming essential to the success and longevity of the title. In addition, the PC space is well known for requiring developers to support a wide variety of customer setups. Often, customers are running on less than state-of-the-art hardware. The same holds true for their network connections.

设计第一人称动作型网络游戏是一项富有挑战性的工作。在动作游戏中加入在线元素正在成为一款游戏是否成功和长寿的必要因素。此外，众所周知的是在PC端的游戏总是要求开发者能够支持各种各样的玩家设置。通常情况下，玩家的设备总不会是最先进的，网络情况也是一样。	

While broadband has been held out as a panacea for all of the current woes of on-line gaming, broadband is not a simple solution allowing developers to ignore the implications of latency and other network factors in game designs. It will be some time before broadband truly becomes adopted the United States, and much longer before it can be assumed to exist for your clients in the rest of the world. In addition, there are a lot of poor broadband solutions, where users may occasionally have high bandwidth, but more often than not also have significant latency and packet loss in their connections.

虽然宽带被誉为能够解决所有在线游戏目前困境的万能药，但他并不足以让开发者忽略一些其他游戏设计过程中的网络问题。具体宽带在美国的普及依然有一段时间，更不用说在世界范围内的终端上普及了。此外，还有很多差劲的宽带，让玩家们偶尔拥有高速的带宽，但却需要再更多的时候面临显著的延迟和丢包。

Your game must behave well in this world. This discussion will give you a sense of some of the tradeoffs required to deliver a cutting-edge action experience on the Internet. The discussion will provide some background on how client / server architectures work in many on-line action games. In addition, the discussion will show how predictive modeling can be used to mask the effects of latency. Finally, the discussion will describe a specific mechanism, lag compensation, for allowing the game to compensate for connection quality.

你的游戏必须要能够在世界范围能有不错的体验。这篇文章会让你意识到想要在网络世界中想要获得顶尖的操作体验，你必须做一些权衡。本文会交代一些背景信息，有关于客户端/服务端架构师们在很多动作型游戏中是如何工作的。此外，本文还会说明如何使用**预测模型**来降低延迟的影响。最后，本文将会介绍一种特殊的机制，**延迟补偿**，用于补偿一些网络因素造成的影响。

## Basic Architecture of a Client / Server Game
## 基础的C/S型游戏架构

Most action games played on the net today are modified client / server games. Games such as Half-Life, including its mods such as Counter-Strike and Team Fortress Classic, operate on such a system, as do games based on the Quake3 engine and the Unreal Tournament engine. In these games, there is a single, authoritative server that is responsible for running the main game logic. To this are connected one or more "dumb" clients. These clients, initially, were nothing more than a way for the user input to be sampled and forwarded to the server for execution. The server would execute the input commands, move around other objects, and then send back to the client a list of objects to render. Of course, the real world system has more components to it, but the simplified breakdown is useful for thinking about prediction and lag compensation.

大部分当今的动作型网络游戏都是变种的C/S型游戏。例如《半条命》,《反恐精英》，《军团要塞》等都是基于这样的系统，那些基于雷神3以及虚幻引擎开发的游戏也是如此。在这些游戏中，会有一个独立的，权威的服务器来负责整个游戏游戏逻辑的执行。服务器会连接着一个或多个“哑巴”客户端。这些客户端们仅仅用于采集用户输入并将它们发送给服务器以供运算。服务器将会计算那些用户输入的信息，移动物体，然后再将结果反馈给客户端。当然，在实际情况中这个系统还会很多组件，但是简单归纳后可以分为**行为预测**和**延迟补偿**。

With this in mind, the typical client / server game engine architecture generally looks like this:

考虑到这一点，典型的客户端/服务端游戏引擎架构大致上是这样的：

![](https://developer.valvesoftware.com/w/images/6/6f/Lagcomp1.png)

For this discussion, all of the messaging and coordination needed to start up the connection between client and server is omitted. The client's frame loop looks something like the following:

以上不包含消息传递以及客户端服务端之间的链接协议。客户端的帧循环会处理以下事件

* Sample clock to find start time
* 作为时钟标准来定位开始时间

* Sample user input (mouse, keyboard, joystick)
* 定义用户输入（鼠标，键盘，摇杆）

* Package up and send movement command using simulation time
* 基于模拟时间来打包并发送移动指令

* Read any packets from the server from the network system
* 读取从服务端收到的数据包

* Use packets to determine visible objects and their state
* 通过读取的数据包来同步物体的信息

* Render Scene
* 渲染场景

* Sample clock to find end time
* 作为时钟标准来定位结束时间

* End time minus start time is the simulation time for the next frame
* 结束时间减去开始时间就是下一帧的模拟时间
