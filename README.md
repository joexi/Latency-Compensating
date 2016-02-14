[原文地址](https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization#Game_Design_Implications_of_Lag_Compensation)

## Overview
## 概要

Designing first-person action games for Internet play is a challenging process. Having robust on-line gameplay in your action title, however, is becoming essential to the success and longevity of the title. In addition, the PC space is well known for requiring developers to support a wide variety of customer setups. Often, customers are running on less than state-of-the-art hardware. The same holds true for their network connections.

设计第一人称动作型网络游戏是一项富有挑战性的工作。在动作游戏中加入在线元素正在成为一款游戏是否成功和长寿的必要因素。此外，众所周知的是在PC端的游戏总是要求开发者能够支持各种各样的玩家设置。通常情况下，玩家的设备总不会是最先进的，网络情况也是一样。	

While broadband has been held out as a panacea for all of the current woes of on-line gaming, broadband is not a simple solution allowing developers to ignore the implications of latency and other network factors in game designs. It will be some time before broadband truly becomes adopted the United States, and much longer before it can be assumed to exist for your clients in the rest of the world. In addition, there are a lot of poor broadband solutions, where users may occasionally have high bandwidth, but more often than not also have significant latency and packet loss in their connections.

虽然宽带被誉为能够解决所有在线游戏目前困境的万能药，但他并不足以让开发者忽略一些其他游戏设计过程中的网络问题。距离宽带在美国的普及依然有一段时间，更不用说在世界范围内的终端上普及了。此外，还有很多差劲的宽带，让玩家们偶尔拥有高速的带宽，但却需要再更多的时候面临显著的延迟和丢包。

Your game must behave well in this world. This discussion will give you a sense of some of the tradeoffs required to deliver a cutting-edge action experience on the Internet. The discussion will provide some background on how client / server architectures work in many on-line action games. In addition, the discussion will show how predictive modeling can be used to mask the effects of latency. Finally, the discussion will describe a specific mechanism, lag compensation, for allowing the game to compensate for connection quality.

你的游戏必须要能够在世界范围能有不错的体验。这篇文章会让你意识到想要在网络世界中想要获得顶尖的操作体验，你必须做一些权衡。本文会交代一些背景信息，有关于客户端/服务端架构师们在很多动作型游戏中是如何工作的。此外，本文还会说明如何使用**预演模型**来降低延迟的影响。最后，本文将会介绍一种特殊的机制，**延迟补偿**，用于补偿一些网络因素造成的影响。

## Basic Architecture of a Client / Server Game
## 基础的C/S型游戏架构

Most action games played on the net today are modified client / server games. Games such as Half-Life, including its mods such as Counter-Strike and Team Fortress Classic, operate on such a system, as do games based on the Quake3 engine and the Unreal Tournament engine. In these games, there is a single, authoritative server that is responsible for running the main game logic. To this are connected one or more "dumb" clients. These clients, initially, were nothing more than a way for the user input to be sampled and forwarded to the server for execution. The server would execute the input commands, move around other objects, and then send back to the client a list of objects to render. Of course, the real world system has more components to it, but the simplified breakdown is useful for thinking about prediction and lag compensation.

大部分当今的动作型网络游戏都是变种的C/S型游戏。例如《半条命》,《反恐精英》，《军团要塞》等都是基于这样的系统，那些基于雷神3以及虚幻引擎开发的游戏也是如此。在这些游戏中，会有一个独立的，权威的服务器来负责整个游戏游戏逻辑的执行。服务器会连接着一个或多个“哑巴”客户端。这些客户端们仅仅用于采集用户输入并将它们发送给服务器以供运算。服务器将会计算那些用户输入的信息，移动物体，然后再将结果反馈给客户端。当然，在实际情况中这个系统还会很多组件，但是简单归纳后可以分为**预演**和**延迟补偿**。

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
## 客户端预演

One method for ameliorating this problem is to perform the client's movement locally and just assume, temporarily, that the server will accept and acknowledge the client commands directly. This method is labeled as client-side prediction.

改良问题的一种方式就是在本地临时地假设客户端行为并且展现它们，服务器端会直接收到并且确认客户端的指令。这种方式被称之为**客户端预演**

Client-side prediction of movements requires us to let go of the "dumb" or minimal client principle. That's not to say that the client is fully in control of its simulation, as in a peer-to-peer game with no central server. There still is an authoritative server running the simulation just as noted above. Having an authoritative server means that even if the client simulates different results than the server, the server's results will eventually correct the client's incorrect simulation. Because of the latency in the connection, the correction might not occur until a full round trip's worth of time has passed. The downside is that this can cause a very perceptible shift in the player's position due to the fixing up of the prediction error that occurred in the past.

客户端对行为的预演需要我们抛弃“哑巴”或者最简化客户端的准则。这并不意味着在没有中央服务器的P2P游戏中，客户端就会接管整个模拟体系。依然会有一个已授权的服务器如上面所说那样在模拟系统中运作。拥有一个已授权的服务器意味着即使客户端的数据和服务器不一致，服务器的结果将会纠正可无端的错误模拟。由于网络的延迟，纠错可能要再完成一次交互之后才会发生。这么做的缺点在于当纠正之前对于玩家位置信息的错误预演时会造成非常明显位移。

To implement client-side prediction of movement, the following general procedure is used. As before, client inputs are sampled and a user command is generated. Also as before, this user command is sent off to the server. However, each user command (and the exact time it was generated) is stored on the client. The prediction algorithm uses these stored commands.

为了引入客户端的行为预演，可以使用一下的常用步骤。就像之前那样，客户端的输入被创建为用户指令。同样，用户指令被发送至服务器。然而，每一条用户指令（以及生成这些指令所需的实际时间）被也会被保存在客户端。预演算法会使用这些用户指令。

For prediction, the last acknowledged movement from the server is used as a starting point. The acknowledgement indicates which user command was last acted upon by the server and also tells us the exact position (and other state data) of the player after that movement command was simulated on the server. The last acknowledged command will be somewhere in the past if there is any lag in the connection. For instance, if the client is running at 50 frames per second (fps) and has 100 milliseconds of latency (roundtrip), then the client will have stored up five user commands ahead of the last one acknowledged by the server. These five user commands are simulated on the client as a part of client-side prediction. Assuming full prediction1, the client will want to start with the latest data from the server, and then run the five user commands through "similar logic" to what the server uses for simulation of client movement. Running these commands should produce an accurate final state on the client (final player position is most important) that can be used to determine from what position to render the scene during the current frame.

预演将从服务器端获取的最近一次确认的行为信息作为预演的起点，其中的“确认”意味着是上一条在经过服务器模拟后执行并同步了玩家位置信息（或者其他信息）的用户指令。只要有延时的存在，那么上一条确认指令某种意义上总是发生在过去的。比如，如果客户端以50fps的帧率运行游戏并且拥有100毫秒的延时，那么客户端在收到服务器的确认前就会存储5个用户指令,这5条指令就将被用于**客户端预演**。从客户端收到上一条指令开始，以和服务端相同的模拟方法来运行这5条指令，就是一个完整的预演。执行这些指令应当生成出决定了客户端上当前帧下在何位置进行渲染的最终的精确状态（玩家的最终位置信息是最重要的）。

In Half-Life, minimizing discrepancies between client and server in the prediction logic is accomplished by sharing the identical movement code for players in both the server-side game code and the client-side game code. These are the routines in the pm_shared/ (which stands for "player movement shared") folder of the HL SDK. The input to the shared routines is encapsulated by the user command and a "from" player state. The output is the new player state after issuing the user command. The general algorithm on the client is as follows:

在《半条命》中客户端和服务端之间通过预演逻辑来实现差异最小化的方式就是在客户端和服务端之间共享相同的行为代码。在HL SDK中pm_shared/目录下(即玩家行为共享) 有相关例程。共享例程的输入就是用户指令和玩家信息的封装。输出则是执行了用户指令后的玩家信息，客户端的常用算法如下所示：

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

最终目标状态中的来源或是其他状态信息就是**预演**的结果，并且被用来渲染那一帧的场景。被执行的那一部分指令就是被拷贝进共享数据体中的玩家所有状态信息以及被处理过的用户指令的一部分，而最终的结果数据会从目标状态中拷贝回去。

There are a few important caveats to this system. First, you'll notice that, depending upon the client's latency and how fast the client is generating user commands (i.e., the client's framerate), the client will most often end up running the same commands over and over again until they are finally acknowledged by the server and dropped from the list (a sliding window in Half-Life's case) of commands yet to be acknowledged. The first consideration is how to handle any sound effects and visual effects that are created in the shared code. Because commands can be run over and over again, it's important not to create footstep sounds, etc. multiple times as the old commands are re-run to update the predicted position. In addition, it's important for the server not to send the client effects that are already being predicted on the client. However, the client still must re-run the old commands or else there will be no way for the server to correct any erroneous prediction by the client. The solution to this problem is easy: the client just marks those commands which have not been predicted yet on the client and only plays effects if the user command is being run for the first time on the client.

这里有一些该系统重要的注意事项。首先，你会注意到，基于客户端的延迟以及客户端生成用户指令的速度（即客户端的帧率）客户端在收到服务器的确认并且丢弃那些被确认后的指令之前会不断地重复执行同样地命令（就像《半条命》中的进度条）。首先要考虑的就是那些通过共享代码生成出来的音效或者视觉特效。必须确保不会因为那些旧指令的重复执行导致多次播放脚步声等（音效）。此外，服务端也不应将那些在客户端已经预演的效果再次推送给客户端。然而，客户端依然需要重复执行那些旧的指令否则服务端将无法纠正客户端预演中的错误。这个问题的解决方案很简单，客户端只要讲那些没有被预演过的指令进行标记，并且只在该指令第一次在客户端执行的时候对特效进行处理。

The other caveat is with respect to state data that exists solely on the client and is not part of the authoritative update data from the server. If you don't have any of this type of data, then you can simply use the last acknowledged state from the server as a starting point, and run the prediction user commands "in-place" on that data to arrive at a final state (which includes your position for rendering). In this case, you don't need to keep all of the intermediate results along the route for predicting from the last acknowledged state to the current time. However, if you are doing any logic totally client side (this logic could include functionality such as determining where the eye position is when you are in the process of crouching—and it's not really totally client side since the server still simulates this data also) that affects fields that are not replicated from the server to the client by the networking layer handling the player's state info, then you will need to store the intermediate results of prediction. This can be done with a sliding window, where the "from state" is at the start and then each time you run a user command through prediction, you fill in the next state in the window. When the server finally acknowledges receiving one or more commands that had been predicted, it is a simple matter of looking up which state the server is acknowledging and copying over the data that is totally client side to the new starting or "from state".

另一个需要注意的点就是对于那些只在客户端存在而不需要经过验证的状态数据。如果你没有此类的数据，那么你可以很简单的将上一次从服务端获取的数据作为之后运行的“起点”，并且基于此来执行那些预演指令并且达到最终的状态（包括了你用作渲染的位置信息）。在这种情况下，你并不需要去维护所有的中间状态来完成从上一次确认状态到当前状态的预演。然而，如果你执行了一些仅存在于客户端的逻辑运算（这种逻辑可能包含了类似当角色蹲着时他的眼睛处于什么位置，当然，如果服务端也对这一类数据进行模拟的话，那它们也算不上纯碎的客户端数据）并且影响到了那些不会通过网络来进行同步的用户状态信息，那么你就需要存储这些预演时的中间状态。这就像是一个进度条，将“来源状态”视作“起点”，每一次你通过预演的方式执行那些用户指令时，你就在不断的填充这个进度条。当服务端最终确认接收了一条或多条被预演过的指令时，我们只需要简单的找出哪些状态是已经被服务器确认过的并且将对应的纯客户端数据进行拷贝，作为新的“来源状态”

So far, the above procedure describes how to accomplish client side prediction of movements. This system is similar to the system used in QuakeWorld.

截止目前，以上的内容介绍了如何完成对于行为的客户端预演，这个系统和《雷神》中使用的类似。

## Client-Side Prediction of Weapon Firing
## 关于枪械开火的客户端预演

Layering prediction of the firing effects of weapons onto the above system is straightforward. Additional state information is needed for the local player on the client, of course, including which weapons are being held, which one is active, and how much ammo each of these weapons has remaining. With this information, the firing logic can be layered on top of the movement logic because, once again, the state of the firing buttons is included in the user command data structure that is shared between the client and the server. Of course, this can get complicated if the actual weapon logic is different between client and server. In Half-Life, we chose to avoid this complication by moving the implementation of a weapon's firing logic into "shared code" just like the player movement code. All of the variables that contribute to determining weapon state (e.g., ammo, when the next firing of the weapon can occur, what weapon animation is playing, etc.), are then part of the authoritative server state and are replicated to the client so that they can be used on the client for prediction of weapon state there.
Predicting weapon firing on the client will likely lead to the decision also to predict weapon switching, deployment, and holstering. In this fashion, the user feels that the game is 100% responsive to his or her movement and weapon activation activities. This goes a long way toward reducing the feeling of latency that many players have come to endure with today's Internet-enabled action experiences.

基于上面的系统预演开火效果非常简单。客户端的本地玩家需要一些额外的状态信息，当然，包括了当前手持的武器，武器的激活状态，以及每一把武器剩余的弹药。有了这些信息，开火相关的逻辑就可以被分为行为逻辑的最上层，因为开火按钮的状态也是C/S之间交互的用户指令信息的一部分。如果客户端和服务端的开火逻辑不同的话，这部分就会变得很复杂。在《半条命》中，我们为了避免出现这种复杂的情况，我们把这些关于武器开火的实现代码放入“共享代码”中，就像那些“用户行为”代码一样。所有为了武器开火服务的变量们（例如弹药的数量，是否可以进行下一次开火，武器的动画是否正在播放，等等）都是服务器需要验证的部分，并且客户端都会有一份数据的拷贝来进行客户端的预演。
客户端武器开火的预演可能会导致需要预演武器的切换，装备和收起。在这种方式下，用户会觉得游戏是100%完全受他的行为以及武器的状态所影响的。这需要做非常多的工作来降低延迟带来的影响。

## Umm, This is a Lot of Work
## 厄，这里还有很多事情

Replicating the necessary fields to the client and handling all of the intermediate state is a fair amount of work. At this point, you may be asking, why not eliminate all of the server stuff and just have the client report where s/he is after each movement? In other words, why not ditch the server stuff and just run the movement and weapons purely on the client-side? Then, the client would just send results to the server along the lines of, "I'm now at position x and, by the way, I just shot player 2 in the head." This is fine if you can trust the client. This is how a lot of the military simulation systems work (i.e., they are a closed system and they trust all of the clients). This is how peer-to-peer games generally work. For Half-Life, this mechanism is unworkable because of realistic concerns about cheating. If we encapsulated absolute state data in this fashion, we'd raise the motivation to hack the client even higher than it already is3. For our games, this risk is too high and we fall back to requiring an authoritative server.

在客户端复制那些必须的属性以及中间过程中的状态信息是一项相当大的工作。对此，也许你会问，为什么不取消服务端做的那些事情，仅仅由客户端来汇报各自的行为呢?或者说，为毛不摆脱服务端，纯粹靠客户端来执行这些行为呢？这样，客户端就只需要将结果发送至服务端例如“我在X点，刚刚爆了个人的头。”，如果你可以信赖客户端，这当然是可以的。许多军事模拟系统就是这么做的（换言之，他们是一个封闭的系统，他们信赖每一个客户端）。这也是那些P2P游戏通常所做的。对于《半条命》来说，这个机制是不可行的，因为我们必须要考虑作弊行为。如果我们以这种形式封装状态的数据，我们就会提升客户端被“hack”的动机。对于我们游戏来说，这样的风险太高了，所以我们退而选择了由服务器进行验证。

A system where movements and weapon effects are predicted client-side is a very workable system. For instance, this is the system that the Quake3 engine supports. One of the problems with this system is that you still have to have a feel for your latency to determine how to lead your targets (for instant hit weapons). In other words, although you get to hear the weapons firing immediately, and your position is totally up-to-date, the results of your shots are still subject to latency. For example, if you are aiming at a player running perpendicular to your view and you have 100 milliseconds of latency and the player is running at 500 units per second, then you'll need to aim 50 units in front of the target to hit the target with an instant hit weapon. The greater the latency, the greater the lead targeting needed. Getting a "feel" for your latency is difficult. Quake3 attempted to mitigate this by playing a brief tone whenever you received confirmation of your hits. That way, you could figure out how far to lead by firing your weapons in rapid succession and adjusting your leading amount until you started to hear a steady stream of tones. Obviously, with sufficient latency and an opponent who is actively dodging, it is quite difficult to get enough feedback to focus in on the opponent in a consistent fashion. If your latency is fluctuating, it can be even harder.

玩家行为和武器效果的客户端预演系统是一个非常具有可行性的系统。比如《雷神3》引擎就支持这样的系统。这个系统存在一个问题就是在决定如何瞄准你的目标的时候，你依然能感受到延迟的影响（相对于瞬间发射）。换句话说，虽然你瞬间听到了武器发射的声音，且你的位置信息也已经更新到了最新，你射击的结果也依然要取决于延迟。比如说，你正在瞄准一个跑进你视野范围内的敌人，此时你有100毫秒的网络延迟，而敌人每秒钟会移动500个距离单位，那么你就要瞄准目标前方50个单位处才能击中他。延迟越大，瞄准的难度也就越大。而感受你的“延迟”却是十分困难的。《雷神3》尝试在你收到开枪指令的确认时播放一段提示音以减轻你的这种感觉，这样的话，你就可以很快的意识到在瞄准目标的时候需要提前多少距离来，并且调整你的准心直到你听到了确认音。当然，在一定的延迟下想要瞄准一个不断躲避的对手并不是一件容易的事情，如果你的延迟还不稳定的话，就更为困难了。

## Display of Targets
## 目标的显示

Another important aspect influencing how a user perceives the responsiveness of the world is the mechanism for determining, on the client, where to render the other players. The two most basic mechanisms for determining where to display objects are extrapolation and interpolation4.

影响用户在游戏世界中的交互反馈体验的另一个方面是如何确认其他玩家在客户端的渲染位置信息。其中2条最简单的用于确定玩家位置的机制是**外推法**和**内插法**

For extrapolation, the other player/object is simulated forward in time from the last known spot, direction, and velocity in more or less a ballistic manner. Thus, if you are 100 milliseconds lagged, and the last update you received was that (as above) the other player was running 500 units per second perpendicular to your view, then the client could assume that in "real time" the player has moved 50 units straight ahead from that last known position. The client could then just draw the player at that extrapolated position and the local player could still more or less aim right at the other player.

对于**外推法**，另一个玩家/对象会根据上一次已知的位置和方向以及速度模拟玩家之后的移动，所以，如果你处于100毫秒的延迟，而上一次你更新时，另一个玩家正在以每秒500单位的速度进行移动，那么你就可以假设该玩家已经在之前的位置上又向前方移动了50个单位。客户端可以在**外推法**计算出的位置上渲染玩家，而本地的玩家依然可以大致瞄准那些经过模拟后的玩家。

The biggest drawback of using extrapolation is that player's movements are not very ballistic, but instead are very non-deterministic and subject to high jerk5. Layer on top of this the unrealistic player physics models that most FPS games use, where player's can turn instantaneously and apply unrealistic forces to create huge accelerations at arbitrary angles and you'll see that the extrapolation is quite often incorrect. The developer can mitigate the error by limiting the extrapolation time to a reasonable value (QuakeWorld, for instance, limited extrapolation to 100 milliseconds). This limitation helps because, once the true player position is finally received, there will be a limited amount of corrective warping. In a world where most players still have greater than 150 milliseconds of latency, the player must still lead other players in order to hit them. If those players are "warping" to new spots because of extrapolation errors, then the gameplay suffers nonetheless.

使用**外推法**的最大弊端在于，玩家的行为并不总是像子弹那样简单，相反总是充满着不确定性。在大多数FPS游戏使用的虚拟玩家物理模型中，你可以从任意角度上瞬间给操作的对象施加一些不实际的力，产生一些非常大的加速度，这样的话外推法演算出的结果往往都是错误的。开发者可以将**外推法**的演算时间限制在一个合理的范围内（在《雷神世界》中，外推法演算的时间上线就是100毫秒）。这个限制是确切有效的，因为当一个真实的玩家坐标最终获得时，将会对之前的演算进行一系列的纠正。在一个大部分玩家依然存在150毫秒以上延迟的世界中，玩家依然需要通过预判的方式来射击目标。如果那些游戏角色被**外推法**修正到了错误的位置上，那么游戏依然会受到影响。

The other method for determining where to display objects and players is interpolation. Interpolation can be viewed as always moving objects somewhat in the past with respect to the last valid position received for the object. For instance, if the server is sending 10 updates per second (exactly) of the world state, then we might impose 100 milliseconds of interpolation delay in our rendering. Then, as we render frames, we interpolate the position of the object between the last updated position and the position one update before that (alternatively, the last render position) over that 100 milliseconds. As the object just gets to the last updated position, we receive a new update from the server (since 10 updates per second means that the updates come in every 100 milliseconds) we can start moving toward this new position over the next 100 milliseconds.

另一种用语确认游戏对象和角色渲染位置的方法就是**内插法**。**内插法**可以被看做总是以物体之前的相对位置来演算之后的移动。比如说，服务器以每秒10次的频率广播世界状态，那么我们就应该施加一个100毫秒的内插延迟在我们的渲染上。那么，当我们进行帧渲染时，我们对上次以及上上更新的位置（或者说渲染位置）之间的100毫秒进行差值演算。当对象正好移动到上一次更新的位置时，我们从服务器接收了新的位置信息（将10次每秒的更新视作每100毫秒更新一次），我们就可以在之后的100毫秒中往这个新的位置进行移动。

If one of the update packets fails to arrive, then there are two choices: We can start extrapolating the player position as noted above (with the large potential errors noted) or we can simply have the player rest at the position in the last update until a new update arrives (causing the player's movement to stutter).
The general algorithm for this type of interpolation is as follows:

如果有一个更新包没有正常收获，那么此时有2个选择：我们可以以上述的方式对玩家的位置进行**外推**演算（可能有很大的潜在误差），或者我们可以简单的让玩家待在圆度直到有下一次的位置更新（会造成玩家移动的卡顿）对于这种类型的内插算法通常如下：

1. Each update contains the server time stamp for when it was generated6
1. 每一次更新都带有服务器的时间戳来表明这次更新的生成时间

2. From the current client time, the client computes a target time by subtracting the interpolation time delta (100 ms)
2. 对于当前的客户端时间而言，客户端通过减去内插的时间增量（100毫秒）计算出一个目标时间。

3. If the target time is in between the timestamp of the last update and the one before that, then those timestamps determine what fraction of the time gap has passed.
3. 如果目标时间介于上一次更新时间和上上次更新时间之间，那么这个时间戳就可以计算出一个过去时间间隔内的分数

4. This fraction is used to interpolate any values (e.g., position and angles).
4. 这个分数就可以用于对任何数值进行差值计算。

In essence, you can think of interpolation, in the above example, as buffering an additional 100 milliseconds of data on the client. The other players, therefore, are drawn where they were at a point in the past that is equal to your exact latency plus the amount of time over which you are interpolating. To deal with the occasional dropped packet, we could set the interpolation time as 200 milliseconds instead of 100 milliseconds. This would (again assuming 10 updates per second from the server) allow us to entirely miss one update and still have the player interpolating toward a valid position, often moving through this interpolation without a hitch. Of course, interpolating for more time is a tradeoff, because it is trading additional latency (making the interpolated player harder to hit) for visual smoothness.

本质上，在上面的例子中，你可以把**内插法**，视作是在客户端缓冲100毫秒的数据.其他的玩家因此被绘制在了他们之前位于的地方，
