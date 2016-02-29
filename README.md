[原文地址](https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization#Game_Design_Implications_of_Lag_Compensation)

## Overview
## 概要

Designing first-person action games for Internet play is a challenging process. Having robust on-line gameplay in your action title, however, is becoming essential to the success and longevity of the title. In addition, the PC space is well known for requiring developers to support a wide variety of customer setups. Often, customers are running on less than state-of-the-art hardware. The same holds true for their network connections.

设计第一人称动作型网络游戏是一项富有挑战性的工作。其中网络部分是否足够优秀正在成为一款游戏是否成功和持久的必要因素。此外，众所周知的是在PC端的游戏总是要求开发者能够提供给玩家各种各样的设置。通常情况下，玩家的设备总不会是最先进的，网络情况也是一样。	

While broadband has been held out as a panacea for all of the current woes of on-line gaming, broadband is not a simple solution allowing developers to ignore the implications of latency and other network factors in game designs. It will be some time before broadband truly becomes adopted the United States, and much longer before it can be assumed to exist for your clients in the rest of the world. In addition, there are a lot of poor broadband solutions, where users may occasionally have high bandwidth, but more often than not also have significant latency and packet loss in their connections.

虽然宽带被誉为能够解决所有在线游戏目前困境的万能药，但他并不足以让开发者忽略一些其他游戏设计过程中的网络问题。距离宽带在美国的普及依然还有有一段时间，更不用说在世界范围内的终端上普及了。此外，还有很多差劲的宽带，让玩家们偶尔拥有高速的带宽，但却需要再更多的时候面临显著的延迟和丢包。

Your game must behave well in this world. This discussion will give you a sense of some of the tradeoffs required to deliver a cutting-edge action experience on the Internet. The discussion will provide some background on how client / server architectures work in many on-line action games. In addition, the discussion will show how predictive modeling can be used to mask the effects of latency. Finally, the discussion will describe a specific mechanism, lag compensation, for allowing the game to compensate for connection quality.

你的游戏必须要在这种情况下依然有不错的体验。这篇文章会让你意识到想要在网络世界中想要提供顶级的操作体验，你必须做一些权衡。本文会交代一些背景信息，有关于客户端/服务端架构师们在那些动作型游戏中是如何工作的。此外，本文还会说明如何使用**预演模型**来降低延迟的影响。最后，本文将会介绍一种特殊的机制，**延迟补偿**，用于补偿一些网络因素造成的影响。

## Basic Architecture of a Client / Server Game
## 基础的C/S型游戏架构

Most action games played on the net today are modified client / server games. Games such as Half-Life, including its mods such as Counter-Strike and Team Fortress Classic, operate on such a system, as do games based on the Quake3 engine and the Unreal Tournament engine. In these games, there is a single, authoritative server that is responsible for running the main game logic. To this are connected one or more "dumb" clients. These clients, initially, were nothing more than a way for the user input to be sampled and forwarded to the server for execution. The server would execute the input commands, move around other objects, and then send back to the client a list of objects to render. Of course, the real world system has more components to it, but the simplified breakdown is useful for thinking about prediction and lag compensation.

大部分当今的动作型网络游戏都是变种的C/S型游戏。例如《半条命》，《反恐精英》，《军团要塞》等都是基于这样的系统，那些基于雷神3以及虚幻引擎开发的游戏也是如此。在这些游戏中，会有一个独立的，权威的服务器来负责整个游戏游戏逻辑的执行。服务器会连接着一个或多个“哑巴”客户端。这些客户端们仅仅用于采集用户输入并将它们发送给服务器以供运算。服务器将会计算那些用户输入的信息，移动物体，然后再将结果反馈给客户端。当然，在实际情况中这个系统还会很多组件，但是简单归纳后可以分为**预演**和**延迟补偿**。

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

基于上述的数据结构以及C/S架构方式，核心的模拟系统如下。首先，客户端创建并发送一条用户指令给服务器。服务器执行这条指令并且把计算的所有结果返回给客户端。最终，客户端基于最新的数据渲染整个场景。这个核心结构想起来非常简单，但当用户的互联网连接有大量延迟的情况下就无法在真实的环境中良好的运行。这最主要的问题就在于客户端是完全“哑巴”的，他要做的事情就是采集用户的输入行为然后等待服务器告诉它究竟发生了什么。如果客户端和服务端之间有500毫秒的延迟，那么每一次客户端要从服务器端确认那些客户端采集的信息并展现出来都需要消耗500毫秒。虽然这种交互的延迟对于局域网来说也许是可以接受的，但在互联网上显然不是。

## Client Side Prediction
## 客户端预演

One method for ameliorating this problem is to perform the client's movement locally and just assume, temporarily, that the server will accept and acknowledge the client commands directly. This method is labeled as client-side prediction.

改良这个问题的一种方式就是直接展现客户端的移动，并且假设服务器端已经收到并且确认客户端的指令。这种方式被称之为**客户端预演**

Client-side prediction of movements requires us to let go of the "dumb" or minimal client principle. That's not to say that the client is fully in control of its simulation, as in a peer-to-peer game with no central server. There still is an authoritative server running the simulation just as noted above. Having an authoritative server means that even if the client simulates different results than the server, the server's results will eventually correct the client's incorrect simulation. Because of the latency in the connection, the correction might not occur until a full round trip's worth of time has passed. The downside is that this can cause a very perceptible shift in the player's position due to the fixing up of the prediction error that occurred in the past.

客户端对行为的预演需要我们抛弃“哑巴”或者最简化客户端的准则。这并不意味着客户端就会接管整个模拟体系而成为没有中央服务器的P2P游戏。依然会有一个权威服务器如上面所说那样在模拟系统中运作。拥有一个权威服务器意味着即使客户端的数据和服务器不一致，服务器的结果将会纠正客户端的错误模拟。由于网络的延迟，纠错可能要再完成一次交互之后才会发生。这么做的缺点在于当纠正之前对于玩家位置信息的错误预演时会造成非常明显位移。

To implement client-side prediction of movement, the following general procedure is used. As before, client inputs are sampled and a user command is generated. Also as before, this user command is sent off to the server. However, each user command (and the exact time it was generated) is stored on the client. The prediction algorithm uses these stored commands.

为了引入客户端的行为预演，可以使用以下的常用步骤。就像之前所说的那样，客户端的输入被创建为用户指令。同样，用户指令会被发送至服务器。然而，每一条用户指令（以及生成这些指令时的实际时间）被也会被保存在客户端。预演算法会使用这些用户指令。

For prediction, the last acknowledged movement from the server is used as a starting point. The acknowledgement indicates which user command was last acted upon by the server and also tells us the exact position (and other state data) of the player after that movement command was simulated on the server. The last acknowledged command will be somewhere in the past if there is any lag in the connection. For instance, if the client is running at 50 frames per second (fps) and has 100 milliseconds of latency (roundtrip), then the client will have stored up five user commands ahead of the last one acknowledged by the server. These five user commands are simulated on the client as a part of client-side prediction. Assuming full prediction1, the client will want to start with the latest data from the server, and then run the five user commands through "similar logic" to what the server uses for simulation of client movement. Running these commands should produce an accurate final state on the client (final player position is most important) that can be used to determine from what position to render the scene during the current frame.

预演将从服务器端获取的最近一次确认的行为信息作为预演的起点，其中的“确认”意味着是上一条在经过服务器模拟后执行并同步了玩家位置信息（或者其他信息）的用户指令。只要有延时的存在，那么上一条确认指令某种意义上总是发生在过去的。比如，如果客户端以50fps的帧率运行游戏并且拥有100毫秒的延时，那么客户端在收到服务器的确认前就会存储5个用户指令,这5条指令就将被用于**客户端预演**。从客户端收到上一条指令开始，以和服务端相同的模拟方法来运行这5条指令，就是一个完整的预演。执行这些指令应当生成出决定了客户端上当前帧下在何位置进行渲染的最终的精确状态（玩家的最终位置信息是最重要的）。

In Half-Life, minimizing discrepancies between client and server in the prediction logic is accomplished by sharing the identical movement code for players in both the server-side game code and the client-side game code. These are the routines in the pm_shared/ (which stands for "player movement shared") folder of the HL SDK. The input to the shared routines is encapsulated by the user command and a "from" player state. The output is the new player state after issuing the user command. The general algorithm on the client is as follows:

在《半条命》中客户端和服务端之间最小化预演逻辑中差异的方式就是在客户端和服务端之间共享相同的行为代码。在HL SDK中pm_shared/目录下(即玩家行为共享) 有相关例程。共享例程的输入就是用户指令和玩家信息的封装。输出则是执行了用户指令后的玩家信息，客户端的常用算法如下所示：

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

这里有一些该系统中需要注意的重要事项。首先，你会注意到，基于客户端的延迟以及客户端生成用户指令的速度（即客户端的帧率）客户端在收到服务器的确认并且丢弃那些被确认的指令之前都会不断地重复执行他们（就像《半条命》中的进度条）。首先要考虑的就是那些通过共享代码生成出来的音效或者视觉特效。必须确保不会因为那些旧指令的重复执行导致多次播放脚步声等（音效）。此外，服务端也不应将那些在客户端已经预演的效果再次推送给客户端。然而，客户端依然需要重复执行那些旧的指令否则服务端将无法纠正客户端预演中的错误。这个问题的解决方案很简单，客户端只要讲那些没有被预演过的指令进行标记，并且只在该指令第一次在客户端执行的时候对特效进行处理。

The other caveat is with respect to state data that exists solely on the client and is not part of the authoritative update data from the server. If you don't have any of this type of data, then you can simply use the last acknowledged state from the server as a starting point, and run the prediction user commands "in-place" on that data to arrive at a final state (which includes your position for rendering). In this case, you don't need to keep all of the intermediate results along the route for predicting from the last acknowledged state to the current time. However, if you are doing any logic totally client side (this logic could include functionality such as determining where the eye position is when you are in the process of crouching—and it's not really totally client side since the server still simulates this data also) that affects fields that are not replicated from the server to the client by the networking layer handling the player's state info, then you will need to store the intermediate results of prediction. This can be done with a sliding window, where the "from state" is at the start and then each time you run a user command through prediction, you fill in the next state in the window. When the server finally acknowledges receiving one or more commands that had been predicted, it is a simple matter of looking up which state the server is acknowledging and copying over the data that is totally client side to the new starting or "from state".

另一个需要注意的点就是对于那些只在客户端存在而不需要经过验证的状态数据。如果你没有此类的数据，那么你可以很简单的将上一次从服务端获取的数据作为之后运行的“起点”，并且基于此来执行那些预演指令并且达到最终的状态（包括了你用作渲染的位置信息）。在这种情况下，你并不需要去维护所有的中间状态来完成从上一次确认状态到当前状态的预演。然而，如果你执行了一些仅存在于客户端的逻辑运算（这种逻辑可能包含了类似当角色蹲着时他的眼睛处于什么位置，当然，如果服务端也对这一类数据进行模拟的话，那它们也算不上纯碎的客户端数据）并且影响到了那些不会通过网络来进行同步的用户状态信息，那么你就需要存储这些预演时的中间状态。这就像是一个进度条，将“来源状态”视作“起点”，每一次你通过预演的方式执行那些用户指令时，你就在不断的填充这个进度条。当服务端最终确认接收了一条或多条被预演过的指令时，我们只需要简单的找出哪些状态是已经被服务器确认过的并且将对应的纯客户端数据进行拷贝，作为新的“来源状态”

So far, the above procedure describes how to accomplish client side prediction of movements. This system is similar to the system used in QuakeWorld.

截止目前，以上的内容介绍了如何完成对于行为的客户端预演，这个系统和《雷神》中使用的类似。

## Client-Side Prediction of Weapon Firing
## 关于枪械开火的客户端预演

Layering prediction of the firing effects of weapons onto the above system is straightforward. Additional state information is needed for the local player on the client, of course, including which weapons are being held, which one is active, and how much ammo each of these weapons has remaining. With this information, the firing logic can be layered on top of the movement logic because, once again, the state of the firing buttons is included in the user command data structure that is shared between the client and the server. Of course, this can get complicated if the actual weapon logic is different between client and server. In Half-Life, we chose to avoid this complication by moving the implementation of a weapon's firing logic into "shared code" just like the player movement code. All of the variables that contribute to determining weapon state (e.g., ammo, when the next firing of the weapon can occur, what weapon animation is playing, etc.), are then part of the authoritative server state and are replicated to the client so that they can be used on the client for prediction of weapon state there.
Predicting weapon firing on the client will likely lead to the decision also to predict weapon switching, deployment, and holstering. In this fashion, the user feels that the game is 100% responsive to his or her movement and weapon activation activities. This goes a long way toward reducing the feeling of latency that many players have come to endure with today's Internet-enabled action experiences.

基于上面的系统预演开火效果非常简单。客户端的本地玩家需要一些额外的状态信息，当然，包括了当前手持的武器，武器的激活状态，以及每一把武器剩余的弹药。有了这些信息，开火相关的逻辑就可以基于行为逻辑来实现，因为开火按钮的状态也是C/S之间交互的用户指令信息的一部分。如果客户端和服务端的开火逻辑不同的话，这部分就会变得很复杂。在《半条命》中，我们为了避免出现这种复杂的情况，我们把这些关于武器开火的实现代码放入“共享代码”中，就像那些“用户行为”代码一样。所有为了武器开火服务的变量们（例如弹药的数量，是否可以进行下一次开火，武器的动画是否正在播放，等等）都是服务器需要验证的部分，并且客户端都会有一份数据的拷贝来进行客户端的预演。
客户端武器开火的预演可能会导致需要预演武器的切换，装备和收起。在这种方式下，用户会觉得游戏是100%完全受他的行为以及武器的状态所影响的。这需要做非常多的工作来降低延迟带来的影响。

## Umm, This is a Lot of Work
## 厄，这里还有很多事情

Replicating the necessary fields to the client and handling all of the intermediate state is a fair amount of work. At this point, you may be asking, why not eliminate all of the server stuff and just have the client report where s/he is after each movement? In other words, why not ditch the server stuff and just run the movement and weapons purely on the client-side? Then, the client would just send results to the server along the lines of, "I'm now at position x and, by the way, I just shot player 2 in the head." This is fine if you can trust the client. This is how a lot of the military simulation systems work (i.e., they are a closed system and they trust all of the clients). This is how peer-to-peer games generally work. For Half-Life, this mechanism is unworkable because of realistic concerns about cheating. If we encapsulated absolute state data in this fashion, we'd raise the motivation to hack the client even higher than it already is3. For our games, this risk is too high and we fall back to requiring an authoritative server.

在客户端复制那些必须的属性以及中间过程中的状态信息是一项相当大的工作。对此，也许你会问，为什么不取消服务端做的那些事情，仅仅由客户端来汇报各自的行为呢?或者说，为毛不摆脱服务端，纯粹靠客户端来执行这些行为呢？这样，客户端就只需要将结果发送至服务端例如“我在X点，刚刚爆了个人的头。”，如果你可以信赖客户端，这当然是可以的。许多军事模拟系统就是这么做的（换言之，他们是一个封闭的系统，他们信赖每一个客户端）。这也是那些P2P游戏通常所做的。对于《半条命》来说，这个机制是不可行的，因为我们必须要考虑作弊行为。如果我们以这种形式封装状态的数据，我们就会提升客户端被“hack”的可能性。对于我们游戏来说，这样的风险太高了，所以我们退而选择了由服务器进行验证。

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

	每一次更新都带有服务器的时间戳来表明这次更新的生成时间

2. From the current client time, the client computes a target time by subtracting the interpolation time delta (100 ms)

	对于当前的客户端时间而言，客户端通过减去内插的时间增量（100毫秒）计算出一个目标时间。

3. If the target time is in between the timestamp of the last update and the one before that, then those timestamps determine what fraction of the time gap has passed.

	如果目标时间介于上一次更新时间和上上次更新时间之间，那么这个时间戳就可以计算出一个过去时间间隔内的分数

4. This fraction is used to interpolate any values (e.g., position and angles).

	这个分数就可以用于对任何数值进行差值计算。

In essence, you can think of interpolation, in the above example, as buffering an additional 100 milliseconds of data on the client. The other players, therefore, are drawn where they were at a point in the past that is equal to your exact latency plus the amount of time over which you are interpolating. To deal with the occasional dropped packet, we could set the interpolation time as 200 milliseconds instead of 100 milliseconds. This would (again assuming 10 updates per second from the server) allow us to entirely miss one update and still have the player interpolating toward a valid position, often moving through this interpolation without a hitch. Of course, interpolating for more time is a tradeoff, because it is trading additional latency (making the interpolated player harder to hit) for visual smoothness.

本质上，在上面的例子中，你可以把**内插法**，视作是在客户端缓冲100毫秒的数据.其他的玩家因此被绘制在了他们之前位于的地方,即你的确切延迟加上你插值的时间。对于偶然的丢包事件，我们可以把内插的时间从100毫秒修改为200毫秒。这可以使得我们能够完全丢掉一个更新包而依然能够通过插值顺利使玩家处于合理的位置上。当然，**内插法**更多时候像是一种交易，他付出一些额外的延迟（使得经过内插演算的玩家更难以瞄准）来换取更平滑的视觉表现。

In addition, the above type of interpolation (where the client tracks only the last two updates and is always moving directly toward the most recent update) requires a fixed time interval between server updates. The method also suffers from visual quality issues that are difficult to resolve. The visual quality issue is as follows. Imagine that the object being interpolated is a bouncing ball (which actually accurately describes some of our players). At the extremes, the ball is either high in the air or hitting the pavement. However, on average, the ball is somewhere in between. If we only interpolate to the last position, it is very likely that this position is not on the ground or at the high point. The bounciness of the ball is "flattened" out and it never seems to hit the ground. This is a classical sampling problem and can be alleviated by sampling the world state more frequently. However, we are still quite likely never actually to have an interpolation target state be at the ground or at the high point and this will still flatten out the positions.

此外， 上述关于**内插法**的类型（客户端只跟踪最近的2次更新，并且总是直接向最近一次更新的位置进行移动）需要在服务器以固定的时间间隔进行每一次更新。这个方法依然会受到那些难以解决的视觉质量问题的影响。视觉质量问题如下：
试想一下，被插值处理的对象是一个弹性球（实际上和我们的玩家角色很相似）在极端情况下，球既不是在高空中也没有撞击地面，球应该在中间的某处，如果我们只是将其向上一次更新的位置进行差值，那么很可能这个位置既不是在地面上也不是在高空中，球的反弹力被“数据化”出来，它看起来便永远都不会砸到地上。这是一个景点的采样问题，并且可以通过更多的对世界状态的采样来解决。然而，我们似乎可能没有一个被插值对象的状态是处于地面或者高空中的，他依然会处于中间的位置。

In addition, because different users have different connections, forcing updates to occur at a lockstep like 10 updates per second is forcing a lowest common denominator on users unnecessarily. In Half-Life, we allow the user to ask for as many updates per second as he or she wants (within limit). Thus, a user with a fast connection could receive 50 updates per second if the user wanted. By default, Half-Life sends 20 updates per second to each player the Half-Life client interpolates players (and many other objects) over a period of 100 milliseconds.7

此外，由于不同的玩家基于不同的链接，强制所有用户的更新步调一致是没有必要的。在 《半条命》中，我们允许用户按自己的需求每秒进行更新（有上限）。这样，拥有高链接速度的用户就可以每秒钟更新50次如果他期望如此，而默认状态下，《半条命》每秒发送20次更新给每一个玩家，《半条命》的客户端则对玩家（以及更多的其他的对象）进行100毫秒时间区间内的差值处理。

To avoid the flattening of the bouncing ball problem, we employ a different algorithm for interpolation. In this method, we keep a more complete "position history" for each object that might be interpolated.

要解决弹球的“flattening”问题，我们采用了另一个不同的算法来进行插值。在这种方法中，我们保留了每个需要插值处理的对象更完整的“坐标历史”。

The position history is the timestamp and origin and angles (and could include any other data we want to interpolate) for the object. Each update we receive from the server creates a new position history entry, including timestamp and origin/angles for that timestamp. To interpolate, we compute the target time as above, but then we search backward through the history of positions looking for a pair of updates that straddle the target time. We then use these to interpolate and compute the final position for that frame. This allows us to smoothly follow the curve that completely includes all of our sample points. If we are running at a higher framerate than the incoming update rate, we are almost assured of smoothly moving through the sample points, thereby minimizing (but not eliminating, of course, since the pure sampling rate of the world updates is the limiting factor) the flattening problem described above.

坐标历史就是时间戳，物体的坐标以及方向（也可能包括其他我们需要插值的数据）。每一次我们从服务器获取更新的时候就创建一条新的坐标历史条目，包括了时间戳和该时间戳对应的坐标和方向。为了进行插值，我们用上面的方式计算目标时间，但是我们回过头在坐标历史中搜寻一对跨越目标时间的更新。我们可以使用它们来进行插值并且计算出那一帧状态下的最终位置。这使我们可以平滑的沿着包含了所有采样坐标的曲线。如果我们以帧率高于数据更新频率的方式运行游戏，我们可以几乎保证能沿着采样坐标平滑地移动，从而最大限度减少（并不是完全消除，因为世界的更新的采样频率是有限制的）上面所说的“flattening”问题。

The only consideration we have to layer on top of either interpolation scheme is some way to determine that an object has been forcibly teleported, rather than just moving really quickly. Otherwise we might "smoothly" move the object over great distances, causing the object to look like it's traveling way too fast. We can either set a flag in the update that says, "don't interpolate" or "clear out the position history," or we can determine if the distance between the origin and one update and another is too big, and thereby presumed to be a teleportation/warp. In that case, the solution is probably to just move the object to the latest know position and start interpolating from there.

对于这些插值方式我们唯一需要考虑的是需要一种方式来确认一个对象是被远程传送的而不是以非常快的速度进行移动。否则我们就会“平滑”地将这个对象移动一大段距离，让这个物体看起来就像是以很快的速度进行移动一样。我们可以在更新信息中设置一个标识位表示“不要插值”或者“清空坐标历史”，或者我们认为当2个坐标点之间的距离太大时，就可以认为是传送/跳跃。在这个例子中，解决方案可能就是直接将对象移动到上一次已知的位置上并且基于此再进行之后的插值。

## Lag Compensation
## 延迟补偿

Understanding interpolation is important in designing for lag compensation because interpolation is another type of latency in a user's experience. To the extent that a player is looking at other objects that have been interpolated, then the amount of interpolation must be taken into consideration in computing, on the server, whether the player's aim was true.
了解**内插法**对于设计延迟补偿机制是十分重要的，因为**内插法**在用户的体验中其实是另一种意义上的延迟。在服务器的计算中，当玩家看到其他已经经过插值处理的对象并且进行瞄准时，其内插的量必须被考虑进计算中。

Lag compensation is a method of normalizing server-side the state of the world for each player as that player's user commands are executed. You can think of lag compensation as taking a step back in time, on the server, and looking at the state of the world at the exact instant that the user performed some action. The algorithm works as follows:

延迟补偿是一种服务器端每一个用户指令被执行之后每一个玩家的世界状态的标准化方式。你可以把延迟补偿视作在服务器端将时间倒退一步，然后看着在用户执行指令后的世界精确的状态。算法如下：

* Before executing a player's current user command, the server:
	
	在执行一条当前的用户指令时，服务端会：
	
	* Computes a fairly accurate latency for the player
	
	相对准确地计算该玩家的延迟
	
	* Searches the server history (for the current player) for the world update that was sent to the player and received by the player just before the player would have issued the movement command
	
	搜索服务器端对于当前用户在提交这个行为指令发送给用户的更新历史。
	
	* From that update (and the one following it based on the exact target time being used), for each player in the update, move the other players backwards in time to exactly where they were when the current player's user command was created. This moving backwards must account for both connection latency and the interpolation amount8 the client was using that frame.
	
	对于数据更新涉及的每一个玩家，实时地将其他玩家移动到该条用户指令产生时真正处于的位置，位置的回滚必须要同时考虑到网络的延迟以及客户端对于该帧的插值的量。
	
* Allow the user command to execute (including any weapon firing commands, etc., that will run ray casts against all of the other players in their "old" positions).

	允许用户指令的执行（包括了所有武器射击的指令等，这会对所有物体之前的位置发射射线）


* Move all of the moved/time-warped players back to their correct/current positions
	
	将所有移动后/位置错乱的玩家移动回他们正确的/当前的位置
	
Note that in the step where we move the player backwards in time, this might actually require forcing additional state info backwards, too (for instance, whether the player was alive or dead or whether the player was ducking). The end result of lag compensation is that each local client is able to directly aim at other players without having to worry about leading his or her target in order to score a hit. Of course, this behavior is a game design tradeoff.

注意当我们把玩家位置进行回滚的时候，，可能也会需要将其的附加状态进行回滚（比如当一个玩家是否避开了射击可能影响他是否存活）。延迟补偿的最终结果就是使得每一个客户端的玩家在瞄准他们的目标时不需要考虑提前量。当然，这种行为也是一种游戏设计的权衡

## Game Design Implications of Lag Compensation
## 一些关于延迟补偿的游戏设计启示

The introduction of lag compensation allows for each player to run on his or her own clock with no apparent latency. In this respect, it is important to understand that certain paradoxes or inconsistencies can occur. Of course, the old system with the authoritative server and "dumb" or simple clients had it's own paradoxes. In the end, making this tradeoff is a game design decision. For Half-Life, we believe deciding in favor of lag compensation was a justified game design decision.

延迟补偿的引入是为了使每一个玩家可以在没有明显延迟的情况下各自执行自己的时钟。为此，有必要意识这会导致一些悖论或者矛盾的发生。当然，那些包含着权威服务器和哑巴客户端或者说简单客户端的传统的系统也会有自己的悖论。最后采取这种权衡方式也是游戏设计角度的决定。对于《半条命》来说，我们认为采用延迟补偿是一个正确的游戏设计。

The first problem of the old system was that you had to lead your target by some amount that was related to your latency to the server. Aiming directly at another player and pressing the fire button was almost assured to miss that player. The inconsistency here is that aiming is just not realistic and that the player controls have non-predictable responsiveness.

传统系统首要的问题就在于当你总是需要一些预判来瞄准目标，具体的值则取决你和服务器之间的延迟。直接瞄准目标然后设计几乎是肯定无法击中的。这里的矛盾点就在于瞄准本身并不是真实的，用户的操作响应速度是无法预知的。

With lag compensation, the inconsistencies are different. For most players, all they have to do is acquire some aiming skill and they can become proficient (you still have to be able to aim). Lag compensation allows the player to aim directly at his or her target and press the fire button (for instant hit weapons9). The inconsistencies that sometimes occur, however, are from the points of view of the players being fired upon.

对于延迟补偿机制来说，矛盾的地方就不同了。对于大部分玩家来说，他们所要做的就是学习一些瞄准的技巧并且能够熟练使用（你依然是需要瞄准的）。延迟补偿是的玩家能够直接瞄准他们的目标然后射击。矛盾之处就在于被击中玩家的视点。

For instance, if a highly lagged player shoots at a less lagged player and scores a hit, it can appear to the less lagged player that the lagged player has somehow "shot around a corner"10. In this case, the lower lag player may have darted around a corner. But the lagged player is seeing everything in the past. To the lagged player, s/he has a direct line of sight to the other player. The player lines up the crosshairs and presses the fire button. In the meantime, the low lag player has run around a corner and maybe even crouched behind a crate. If the high lag player is sufficiently lagged, say 500 milliseconds or so, this scenario is quite possible. Then, when the lagged player's user command arrives at the server, the hiding player is transported backward in time and is hit. This is the extreme case, and in this case, the low ping player says that s/he was shot from around the corner. However, from the lagged player's point of view, they lined up their crosshairs on the other player and fired a direct hit. From a game design point of view, the decision for us was easy: let each individual player have completely responsive interaction with the world and his or her weapons.

比如说，一个高延迟的玩家向一个低延迟玩家射击，对于那个低延迟的玩家看起来就像高延迟玩家“拐角射击”。在这个案例中，那个低延迟的玩家已经拐过了墙角。而那个高延迟的家伙看到的依然是过去的景象。对于他来说，他望向其他玩家的视线总是笔直的直线。当他看到有其他的玩家经过拐角时按下了射击按钮。在这个时候那个低延迟的玩家可能早就跑过了拐角处，甚至可能已经蹲到了箱子的后面.如果那个高延迟的家伙有足够高的延迟，比如说500毫秒之类的，那么这种情况就很有可能发生。那么，当这个高延迟玩家的用户指令抵达服务器端，那么这个已经躲在箱子后面的家伙可能就会在之前的某个位置上被击中。这是一个极端的例子，在这个案例中，低延迟的玩家在拐角之后被射杀。然而，从高延迟的玩家的视点看去，目标是在拐角前被击杀的。从游戏设计的角度来看，我们的决定很简单，让每一个独立的玩家都能和武器以及环境完全地互相作用。

In addition, the inconsistency described above is much less pronounced in normal combat situations. For first-person shooters, there are two more typical cases. First, consider two players running straight at each other pressing the fire button. In this case, it's quite likely that lag compensation will just move the other player backwards along the same line as his or her movement. The person being shot will be looking straight at his attacker and no "bullets bending around corners" feeling will be present.

此外，上述的矛盾在正常的游戏中是很少出现的。对于第一人称射击游戏而言，还有2个典型的案例。首先，想象一下两个玩家互相冲向对方并且按下了射击按钮。在这个案例中，看起来延迟补偿应该会将玩家沿着他们的行动轨迹即同一条直线进行回滚。被射中的人会面向对方，不会有任何“拐角射击”的问题出现。

The next example is two players, one aiming at the other while the other dashes in front perpendicular to the first player. In this case, the paradox is minimized for a wholly different reason. The player who is dashing across the line of sight of the shooter probably has (in first-person shooters at least) a field of view of 90 degrees or less. In essence, the runner can't see where the other player is aiming. Therefore, getting shot isn't going to be surprising or feel wrong (you get what you deserve for running around in the open like a maniac). Of course, if you have a tank game, or a game where the player can run one direction, and look another, then this scenario is less clear-cut, since you might see the other player aiming in a slightly incorrect direction.

另一个例子中，两个玩家，其中一个在另一个从侧面冲向自己时瞄准了对方。在这个案例中，矛盾则因为完全不同的原因最小化了。那个冲向射击者视线的玩家可能呈90度（至少从第一人称角度来看）或更小。从本质上来说，冲刺的那个人并不知道另一个玩家是不是在瞄准。所以，被击中也没什么好奇怪或者觉得哪里有什么不对的（当你想疯子一样冲向一片开阔区域是，这就是你应得的下场）。当然，如果你是在坦克游戏或者那些玩家只能往一个方向移动的游戏中看着对方，那么这个场景就不是那么明确了，因为你可能会意识到对方正在瞄准在一个错误的方向上。

## Conclusion
## 结论

Lag compensation is a tool to ameliorate the effects of latency on today's action games. The decision of whether to implement such a system rests with the game designer since the decision directly changes the feel of the game. For Half-Life, Team Fortress and Counter Strike, the benefits of lag compensation easily outweighed the inconsistencies noted above.

延迟补偿是一种改善现在动作类游戏体验的工具。由于延迟补偿会改变玩家在游戏中的感觉，所以是否引入延迟补偿就要由游戏的设计者来决定。对于《半条命》，《军团要塞》，《反恐精英》来说，延迟补偿的利大于弊。

## Footnotes
## 脚注

1. In the Half-Life engine, it is possible to ask the client-side prediction algorithm to account for some, but not all, of the latency in performing prediction. The user could control the amount of prediction by changing the value of the "pushlatency" console variable to the engine. This variable is a negative number indicating the maximum number of milliseconds of prediction to perform. If the number is greater (in the negative) than the user's current latency, then full prediction up to the current time occurs. In this case, the user feels zero latency in his or her movements. Based upon some erroneous superstition in the community, many users insisted that setting pushlatency to minus one-half of the current average latency was the proper setting. Of course, this would still leave the player's movements lagged (often described as if you are moving around on ice skates) by half of the user's latency. All of this confusion has brought us to the conclusion that full prediction should occur all of the time and that the pushlatency variable should be removed from the Half-Life engine.
	
	在《半条命》的游戏中，你可以让客户端的预测算法中考虑进部分执行预测时的延迟情况。用户可以通过在控制台修改 “pushlantency”变量以达到控制预测量的效果。这个变量是一个用于表明最大预测时间毫秒数的负数。如果这个数大于（对负数而言）玩家当前的延迟数，那么就充分预测那些到当前为止发生的事情。在这种情况下，用户就几乎感觉不到自己移动时的延迟。由于一些误传的谣言，很多玩家认为将pushlatency设置为当前延迟的一半是最正确的选择。当然，这依然会使玩家的移动有实际延迟一半的延迟的感觉（经常表现为滑步）。这些东西总结下来就是你总是执行完整的预测，然pushlatency这个变量应该从《半条命》游戏中移除。
	
2. http://www.quakeforge.net/files/q1source.zip

3. A discussion of cheating and what developers can do to deter it is beyond the scope of this paper.
	
	关于作弊和开发者如何反作弊已经超出了本文的讨论范围。
	
4. Though hybrids and corrective methods are also possible.

	虽然混合以及纠正的方式也是可以的。
	
5. "Jerk" is a measure of how fast accelerative forces are changing. 

	Jerk 是描述加速度变化速度的单位。
	
6. It is assumed in this paper that the client clock is directly synchronized to the server clock modulo the latency of the connection. In other words, the server sends the client, in each update, the value of the server's clock and the client adopts that value as its clock. Thus, the server and client clocks will always be matched, with the client running the same timing somewhat in the past (the amount in the past is equal to the client's current latency). Smoothing out discrepancies in the client clock can be solved in various ways.

	本文假设客户端的时钟是直接和服务器同步以计算出链接的延迟的。即是说，当服务器端每次将需要更新的内容发送给客户端时带上服务器的本地时间，客户端采用这个时间作为自己的本地时间。所以，服务器和客户端时钟总是能够匹配上的，当然客户端的时间总是会稍慢于服务端（具体慢多少取决于双方之间的延迟）。将客户端的时间的差异平滑地修正则有很多的方法。
	
7. The time spacing of these updates is not necessarily fixed. The reason why is that during high activity periods of the game (especially for users with lower bandwidth connections), it's quite possible that the game will want to send you more data than your connection can accommodate. If we were on a fixed update interval, then you might have to wait an entire additional interval before the next packet would be sent to the client. However, this doesn't match available bandwidth effectively. Instead, the server, after sending every packet to a player, determines when the next packet can be sent. This is a function of the user's bandwidth or "rate" setting and the number of updates requested per second. If the user asks for 20 updates per second, then it will be at least 50 milliseconds before the next update packet can be sent. If the bandwidth choke is active (and the server is sufficiently high framerate), it could be 61, etc., milliseconds before the next packet gets sent. Thus, Half-Life packets can be somewhat arbitrarily spaced. The simple move to latest goal interpolation schemes don't behave as well (think of the old anchor point for movement as being variable) under these conditions as the position history interpolation method (described below). 

	这些更新之间的间隔不一定非要是固定的。因为在高实时性的游戏（特别是对于网络较差的用户）中，很有可能游戏会希望发送大于你带宽负荷的游戏数据。如果我们是固定时间更新的，那么在你下一次发送更新之前就必须要等待一个完整的时间间隔。然而这并不是有效利用带宽的方式。相反，在服务器中，会在每一个数据包发送给客户端之后再决定下一次发送更新包的时间.这是个关于用户带宽设置以及每秒钟的更新频率的函数。如果用户希望每秒钟更新20次，那么在下一次发送前至少需要等待50毫秒。如果带宽阻塞（并且服务器有更高的帧刷新率）那么这个值可能是61毫秒或者更多。所以，《半条命》可以以任意间隔发送。在这种方式下对于简单移动到下一个目标点的坐标历史插值方式不会有很好的效果（考虑到移动的起始锚点是个变量）。

8. Which Half-Life encodes in the lerp_msec field of the usercmd_t structure described previously. 
	
	《半条命》将上面提到的usercmd_t结构压缩到lerp_msec字段中

9. For weapons that fire projectiles, lag compensation is more problematic. For instance, if the projectile lives autonomously on the server, then what time space should the projectile live in? Does every other player need to be "moved backward" every time the projectile is ready to be simulated and moved by the server? If so, how far backward in time should the other players be moved? These are interesting questions to consider. In Half-Life, we avoided them; we simply don't lag compensate projectile objects (that's not to say that we don't predict the sound of you firing the projectile on the client, just that the actual projectile is not lag compensated in any way). 
	
	对于发射子弹的武器来说，延迟补偿有跟多的问题。比如说，如果服务器端有一个字段对象，那么它应该处于哪一个时间间隔中呢，当它在服务器上移动的时候，其他的玩家要不要回滚呢?如果回滚又应该回滚多少呢？这是个有趣的问题。在《半条命》中，我们避免了这种情况，我们不对子弹物体做延迟补偿处理（这并不代表我们不会预演在客户端发射子弹时的音效，只是子弹物体本身不会参与延迟补偿）

10. This is the phrase our user community has adopted to describe this inconsistency. 

	这是在社区中用于描述这种不一致性的通俗短语

	
