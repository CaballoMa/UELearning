# UE4Gameplay 关键元素
利用“GameMode”制定和检测胜利规则,并支配世界内的“元素”，通过“GameState”记录游戏世界的关键状态,使用PlayerState保存玩家的状态。通过GameState和PlayerState来持久化游戏世界的数据。”为对局恢复(断线重连,观战)进行,为世界的重建提供支持。
## GameMode
Game Mode 和 Game State是游戏的定义，包括游戏规则和获胜条件等内容，它仅存在于服务器上。它通常不应有太多在游戏过程中会发生变化的数据，也绝对不应有客户端需要了解的临时数据。

基类：AGameModeBase，默认的游戏模式。
AGameMode：AGameModeBase子类，继承于AInfo，一个流程/规则的状态机。

(UWorld)BeginPlay——> (AGameMode)StartPlay——> (AGameMode)StartMatch——> Spawn Actors and begin the game.

GameMode仅存于服务器（属于Server，通过修改GameState影响客户端），通过GameState同步信息，PlayerState是服务器客户端都有。

### GameMode中的Func：
* InitGame

    在所有Actor激活之前调用(执行PrelnitializeComponents之前) Beginplay。
* PreLogin

    接受或拒绝尝试加入服务器的玩家。如它将ErrorMessage设为一个非空字符串,会导致Login 函数失败。
* PostLogin

    成功登录后调用。这是首个在PlayerController上安全调用复制函数之处。OnPostLogin暴露给了蓝图中，用于方便添加额外的逻辑。
* HandleStartingNewPlayer

    在PostLogin后或无缝游历后调用,可在蓝图中覆盖,修改新玩家身上发生的事件。它将默认创建一个玩家 pawn
* RestartPlayer

    调用开始生成一个玩家 pawn。如需要指定 Pawn 生成的地点，还可使用RestartPlayerAtPlayerStart和RestartPlayerAtTransform函数。OnRestartPlayer可在蓝图中实现，在此函数完成后添加逻辑。
* SpawnDefaultPawnAtTransform

    实际生成玩家Pawn，可在蓝图中覆盖。
* Logout

    玩家离开游戏或被摧毁时调用。可实现OnLogout执行蓝图逻辑。（断线重连）

* URL作为启动参数,指定GameMode

UE4Editor.exe/Game/Maps/MyMap?game=MyGameMode-game

* 配置默认的GameMode

可在DefaultEngine.ini文件的/Script/Engine.WorldSettings/部分中设置地图前缀(和URL法的别名) 。这些前缀设置所有拥有特定前缀的地图的默认游戏模式。
[/Script/EngineSettings.GameMapsSettings]

+GameModeMapPrefixes=(Name="DM",GameMode="/Script/ MyGameMode. MyGameMode")

+GameModeClassAliases=(Name="DM",GameMode="/Script/ MyGameMode. MyGameMode"))
## GameState(全局信息)
GameState包含游戏的状态，其中可以包括联网玩家列表、得分、棋类游戏中棋子的位置，或者在开放世界场景中完成的任务列表。游戏状态存在于服务器和所有客户端上，可以自由复制以保持所有机器处于最新状态。

包含要复制到游戏中的每个客户端的信息,简而言之,它表示每个联网玩家的"游戏状态"。它通常包含有关游戏分数、比赛是否已开始和基于世界场景玩家人数要生成的AI数量的信息，以及其他特定于游戏的信息。

对于多人游戏,每个玩家的机器上都有一个游戏状态实例,而服务器的实例为权威实例。

* GetServerWorldTimeSeconds

    GetTimeSeconds的服务器版本,保持客户端和服务器上时间的同步。
    
* PlayerArray
    
    存储了所有玩家的APlayerState，方便遍历和获取玩家数据信息。
    
* HasBegunPlay
    
    游戏世界中的Actor已执行Beginplay,则返回true。

## PlayState(玩家个人信息)
玩家状态 是游戏玩家的状态，例如人类玩家或模拟玩家的机器人。作为游戏的一部分而存在的非玩家A1将不会拥有玩家状态,在玩家状态中适当的示例数据包括玩家姓名或得分、比赛中MOBA等的等级,或玩家当前是否在CTF游戏中携带旗帆,所有玩家的玩家状态存在于所有机器上(与玩家控制器不同），并且可以自由复制以保持同步。

# 网络同步

* 网络角色NetRole
    
    authority 
    
    autonomous 
    
    simulate

* 值复制(server)
    
    相关性的概念（相关时同步actor+最新状态）
    
    Actor状态的保持(ActorChannel：每个Actor的同步信息，某Actor不相关的时候就在本地销毁掉该Actor的ActorChannel)
    
    Server-->Client(可靠的，及时的）

* RPC(server client) valid(damage10000: 100,return false.)

    两种设置：
    
    Reliable(pitfall)
    
    Unreliable(server→client,udp)

    类型：
    
    Multicast
    
    Run on Server client-->server(客户端只能通过RPC给Server发)
    
    Run on OwningClient：角色+背包（发光）server-→clientspecific show.





