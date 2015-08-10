[原文地址](https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization#Game_Design_Implications_of_Lag_Compensation)

## Overview
## 概要

Designing first-person action games for Internet play is a challenging process. Having robust on-line gameplay in your action title, however, is becoming essential to the success and longevity of the title. In addition, the PC space is well known for requiring developers to support a wide variety of customer setups. Often, customers are running on less than state-of-the-art hardware. The same holds true for their network connections.

设计第一人称动作型网络游戏是一项富有挑战性的工作。在动作游戏中加入在线元素正在成为一款游戏是否成功和长寿的必要因素。此外，众所周知的是在PC端的游戏总是要求开发者能够支持各种各样的玩家设置。通常情况下，玩家的设备总不会是最先进的，网络情况也是一样。	

While broadband has been held out as a panacea for all of the current woes of on-line gaming, broadband is not a simple solution allowing developers to ignore the implications of latency and other network factors in game designs. It will be some time before broadband truly becomes adopted the United States, and much longer before it can be assumed to exist for your clients in the rest of the world. In addition, there are a lot of poor broadband solutions, where users may occasionally have high bandwidth, but more often than not also have significant latency and packet loss in their connections.

虽然宽带被誉为能够解决所有在线游戏目前困境的万能药，但他并不足以让开发者忽略一些其他游戏设计过程中的网络问题。距离宽带在美国的普及依然有一段时间，更不用说在世界范围内的终端上普及了。此外，还有很多差劲的宽带，让玩家们偶尔拥有高速的带宽，但却需要再更多的时候面临显著的延迟和丢包。

Your game must behave well in this world. This discussion will give you a sense of some of the tradeoffs required to deliver a cutting-edge action experience on the Internet. The discussion will provide some background on how client / server architectures work in many on-line action games. In addition, the discussion will show how predictive modeling can be used to mask the effects of latency. Finally, the discussion will describe a specific mechanism, lag compensation, for allowing the game to compensate for connection quality.

你的游戏必须要能够在世界范围能有不错的体验。这篇文章会让你意识到想要在网络世界中想要获得顶尖的操作体验，你必须做一些权衡。本文会交代一些背景信息，有关于客户端/服务端架构师们在很多动作型游戏中是如何工作的。此外，本文还会说明如何使用**预测模型**来降低延迟的影响。最后，本文将会介绍一种特殊的机制，**延迟补偿**，用于补偿一些网络因素造成的影响。

## Basic Architecture of a Client / Server Game
## 基础的C/S型游戏架构

Most action games played on the net today are modified client / server games. Games such as Half-Life, including its mods such as Counter-Strike and Team Fortress Classic, operate on such a system, as do games based on the Quake3 engine and the Unreal Tournament engine. In these games, there is a single, authoritative server that is responsible for running the main game logic. To this are connected one or more "dumb" clients. These clients, initially, were nothing more than a way for the user input to be sampled and forwarded to the server for execution. The server would execute the input commands, move around other objects, and then send back to the client a list of objects to render. Of course, the real world system has more components to it, but the simplified breakdown is useful for thinking about prediction and lag compensation.

大部分当今的动作型网络游戏都是变种的C/S型游戏。例如《半条命》,《反恐精英》，《军团要塞》等都是基于这样的系统，那些基于雷神3以及虚幻引擎开发的游戏也是如此。在这些游戏中，会有一个独立的，权威的服务器来负责整个游戏游戏逻辑的执行。服务器会连接着一个或多个“哑巴”客户端。这些客户端们仅仅用于采集用户输入并将它们发送给服务器以供运算。服务器将会计算那些用户输入的信息，移动物体，然后再将结果反馈给客户端。当然，在实际情况中这个系统还会很多组件，但是简单归纳后可以分为**预测**和**延迟补偿**。

With this in mind, the typical client / server game engine architecture generally looks like this:

考虑到这一点，典型的客户端/服务端游戏引擎架构大致上是这样的：

![](https://developer.valvesoftware.com/w/images/6/6f/Lagcomp1.png)

For this discussion, all of the messaging and coordination needed to start up the connection between client and server is omitted. The client's frame loop looks something like the following:

以上不包含消息传递以及客户端服务端之间的链接协议。客户端的帧循环内会处理以下事件

* Sample clock to find start time
* 捕捉时钟信息来定位开始时间

* Sample user input (mouse, keyboard, joystick)
* 扑捉用户输入（鼠标，键盘，摇杆）

* Package up and send movement command using simulation time
* 基于模拟时间来打包并发送移动指令

* Read any packets from the server from the network system
* 读取从服务端收到的数据包

* Use packets to determine visible objects and their state
* 通过读取的数据包来同步物体的信息

* Render Scene
* 渲染场景

* Sample clock to find end time
* 捕捉时钟信息来定位结束时间

* End time minus start time is the simulation time for the next frame
* 利用结束时间减去开始时间得到下一帧的仿真时间

Each time the client makes a full pass through this loop, the "frametime" is used for determining how much simulation is needed on the next frame. If your framerate is totally constant then frametime will be a correct measure. Otherwise, the frametimes will be incorrect, but there isn't really a solution to this (unless you could deterministically figure out exactly how long it was going to take to run the next frame loop iteration before running it...).

每一次客户端都会完整的执行这一个循环，“帧时间”是用来确认下一帧将会需要多少时间。如果你的帧率是完全不变的，那么你测量出的帧时间将会是正确无误。否则，帧时间就是不正确地，但这并没有完全意义上的解决方案（除非你能够在执行之前就完全确定的判断出下一个帧循环将会消耗多长时间）。

The server has a somewhat similar loop:

服务端也有一个类似的循环
* Sample clock to find start time
* 捕捉时钟信息来定位开始时间

* Read client user input messages from network
* 通过网络获取客户端的用户输入

* Execute client user input messages
* 执行客户端的用户输入

* Simulate server-controlled objects using simulation time from last full pass
* 利用上一次循环的仿真时间来模拟服务器控制的物体

* For each connected client, package up visible objects/world state and send to client
* 将可见物体/世界的状态打包并发送给每一个链接着的客户端

* Sample clock to find end time
* 捕捉时钟信息来定位结束时间

* End time minus start time is the simulation time for the next frame
* 利用结束时间减去开始时间得到下一帧的仿真时间

In this model, non-player objects run purely on the server, while player objects drive their movements based on incoming packets. Of course, this is not the only possible way to accomplish this task, but it does make sense.

在这种模式下，非玩家对象只会由服务端驱动，而玩家的对象则基于他们的传入数据进行移动。当然，并非是达到这种效果唯一的方式，但他确实有一定道理。

## Contents of the User Input messages
## 用户输入消息的内容

In Half-Life engine games, the user input message format is quite simple and is encapsulated in a data structure containing just a few essential fields:

在《半条命》引擎的游戏中，用户输入消息的格式化方式非常简单，并且一个拥有几个字段的数据结构就足以封装它了。

``` c
typedef struct usercmd_s
{
	// Interpolation time on client
	short		lerp_msec;   
	// Duration in ms of command
	byte		msec;      
	// Command view angles.
	vec3_t	viewangles;   
	// intended velocities
	// Forward velocity.
	float		forwardmove;  
	// Sideways velocity.
	float		sidemove;    
	// Upward velocity.
	float		upmove;   
	// Attack buttons
	unsigned short buttons; 
	//
	// Additional fields omitted...
	//
} usercmd_t;
```

The critical fields here are the msec, viewangles, forward, side, and upmove, and buttons fields. The msec field corresponds to the number of milliseconds of simulation that the command corresponds to (it's the frametime). The viewangles field is a vector representing the direction the player was looking during the frame. The forward, side, and upmove fields are the impulses determined by examining the keyboard, mouse, and joystick to see if any movement keys were held down. Finally, the buttons field is just a bit field with one or more bits set for each button that is being held down.

在这里关键的字段是msec, viewangles, forward, side, and upmove, and buttons fields.msec字段对应相关指令的模拟毫秒数（就是帧时间）。viewangles意味着玩家在这一帧的过程中面朝方向的矢量信息。forward, side, and upmove字段通过检查键盘，鼠标和遥感来确定方向按键是否被按住。

Using the above data structures and client / server architecture, the core of the simulation is as follows. First, the client creates and sends a user command to the server. The server then executes the user command and sends updated positions of everything back to client. Finally, the client renders the scene with all of these objects. This core, though quite simple, does not react well under real world situations, where users can experience significant amounts of latency in their Internet connections. The main problem is that the client truly is "dumb" and all it does is the simple task of sampling movement inputs and waiting for the server to tell it the results. If the client has 500 milliseconds of latency in its connection to the server, then it will take 500 milliseconds for any client actions to be acknowledged by the server and for the results to be perceptible on the client. While this round trip delay may be acceptable on a Local Area Network (LAN), it is not acceptable on the Internet.

使用上述的数据结构以及C/S架构方式，那么核心的模拟系统就是这样的，首先，客户端创建并发送一条用户指令给服务器。服务器执行这条指令并且把计算的所有结果返回给客户端。最终，客户端基于最新的数据渲染整个场景。这个核心结构想起来非常简单，但当用户的互联网连接有大量延迟的情况下就无法在真实的环境下良好的运行。这最主要的问题就在于客户端是完全“哑巴”的，他要做的事情就是采集用户的输入行为然后等待服务器钙素它究竟发生了什么。如果客户端和服务端之间有500毫秒的延迟，那么每一次客户端要从服务器端确认那些客户端采集的信息并展现出来都需要消耗500毫秒。虽然这种交互的延迟对于局域网来说也许是可以接受的，但在互联网上显然不是。

## Client Side Prediction
## 客户端预测

One method for ameliorating this problem is to perform the client's movement locally and just assume, temporarily, that the server will accept and acknowledge the client commands directly. This method is labeled as client-side prediction.

改良问题的一种方式就是在本地临时地假设客户端行为并且展现它们，服务器端会直接收到并且确认客户端的指令。这种方式被称之为**客户端预测**

Client-side prediction of movements requires us to let go of the "dumb" or minimal client principle. That's not to say that the client is fully in control of its simulation, as in a peer-to-peer game with no central server. There still is an authoritative server running the simulation just as noted above. Having an authoritative server means that even if the client simulates different results than the server, the server's results will eventually correct the client's incorrect simulation. Because of the latency in the connection, the correction might not occur until a full round trip's worth of time has passed. The downside is that this can cause a very perceptible shift in the player's position due to the fixing up of the prediction error that occurred in the past.

客户端对行为的预测需要我们抛弃“哑巴”或者最简化客户端的准则。这并不意味着在没有中央服务器的P2P游戏中，客户端就会接管整个模拟体系。依然会有一个已授权的服务器如上面所说那样在模拟系统中运作。拥有一个已授权的服务器意味着即使客户端的数据和服务器不一致，服务器的结果将会纠正可无端的错误模拟。由于网络的延迟，纠错可能要再完成一次交互之后才会发生。这么做的缺点在于当纠正之前对于玩家位置信息的错误预测时会造成非常明显位移。

To implement client-side prediction of movement, the following general procedure is used. As before, client inputs are sampled and a user command is generated. Also as before, this user command is sent off to the server. However, each user command (and the exact time it was generated) is stored on the client. The prediction algorithm uses these stored commands.

为了引入客户端的行为预测，可以使用一下的常用步骤。就像之前那样，客户端的输入被创建为用户指令。同样，用户指令被发送至服务器。然而，每一条用户指令（以及生成这些指令所需的实际时间）被也会被保存在客户端。预测算法会使用这些用户指令。

For prediction, the last acknowledged movement from the server is used as a starting point. The acknowledgement indicates which user command was last acted upon by the server and also tells us the exact position (and other state data) of the player after that movement command was simulated on the server. The last acknowledged command will be somewhere in the past if there is any lag in the connection. For instance, if the client is running at 50 frames per second (fps) and has 100 milliseconds of latency (roundtrip), then the client will have stored up five user commands ahead of the last one acknowledged by the server. These five user commands are simulated on the client as a part of client-side prediction. Assuming full prediction1, the client will want to start with the latest data from the server, and then run the five user commands through "similar logic" to what the server uses for simulation of client movement. Running these commands should produce an accurate final state on the client (final player position is most important) that can be used to determine from what position to render the scene during the current frame.

预测将从服务器端获取的最近一次确认的行为信息作为预测的起点，其中的“确认”意味着是上一条在经过服务器模拟后执行并同步了玩家位置信息（或者其他信息）的用户指令。只要有延时的存在，那么上一条确认指令某种意义上总是发生在过去的。比如，如果客户端以50fps的帧率运行游戏并且拥有100毫秒的延时，那么客户端在收到服务器的确认前就会存储5个用户指令,这5条指令就将被用于**客户端预测**。从客户端收到上一条指令开始，以和服务端相同的模拟方法来运行这5条指令，就是一个完整的预测。执行这些指令应当生成出决定了客户端上当前帧下在何位置进行渲染的最终的精确状态（玩家的最终位置信息是最重要的）。

In Half-Life, minimizing discrepancies between client and server in the prediction logic is accomplished by sharing the identical movement code for players in both the server-side game code and the client-side game code. These are the routines in the pm_shared/ (which stands for "player movement shared") folder of the HL SDK. The input to the shared routines is encapsulated by the user command and a "from" player state. The output is the new player state after issuing the user command. The general algorithm on the client is as follows:

在《半条命》中客户端和服务端之间通过预测逻辑来实现差异最小化的方式就是在客户端和服务端之间共享相同的行为代码。在HL SDK中pm_shared/目录下(即玩家行为共享) 有相关例程。共享例程的输入就是用户指令和玩家信息的封装。输出则是执行了用户指令后的玩家信息，客户端的常用算法如下所示：

``` c

"from state" <- state after last user command acknowledged by the server;

"command" <- first command after last user command acknowledged by server;

while (true)
{
	run "command" on "from state" to generate "to state";
	if (this was the most up to date "command")
		break;

	"from state" = "to state";
	"command" = next "command";
};

```

The origin and other state info in the final "to state" is the prediction result and is used for rendering the scene that frame. The portion where the command is run is simply the portion where all of the player state data is copied into the shared data structure, the user command is processed (by executing the common code in the pm_shared routines in Half-Life's case), and the resulting data is copied back out to the "to state".

最终目标状态中的来源或是其他状态信息就是**预测**的结果，并且被用来渲染那一帧的场景。被执行的那一部分指令就是被拷贝进共享数据体中的玩家所有状态信息以及被处理过的用户指令的一部分，而最终的结果数据会从目标状态中拷贝回去。

There are a few important caveats to this system. First, you'll notice that, depending upon the client's latency and how fast the client is generating user commands (i.e., the client's framerate), the client will most often end up running the same commands over and over again until they are finally acknowledged by the server and dropped from the list (a sliding window in Half-Life's case) of commands yet to be acknowledged. The first consideration is how to handle any sound effects and visual effects that are created in the shared code. Because commands can be run over and over again, it's important not to create footstep sounds, etc. multiple times as the old commands are re-run to update the predicted position. In addition, it's important for the server not to send the client effects that are already being predicted on the client. However, the client still must re-run the old commands or else there will be no way for the server to correct any erroneous prediction by the client. The solution to this problem is easy: the client just marks those commands which have not been predicted yet on the client and only plays effects if the user command is being run for the first time on the client.

这里有一些该系统重要的注意事项。首先，你会注意到，基于客户端的延迟已经客户端生成用户指令的速度（即客户端的帧率）客户端在收到服务器的确认并且丢弃那些被确认后的指令之前会不断地重复执行同样地命令（就像《半条命》中的滑动窗口）。首先要考虑的就是那些通过共享代码生成出来的音效或者视觉特效。由于命令会被不断地执行，那么不产生脚步声就显得很重要。
