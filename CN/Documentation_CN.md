[TOC]

# 快速开始

通过简短的介绍让你快速配置、运行 kRPC，并能自己动手写一个简单的 Python 脚本来控制游戏。



## 服务插件

### 安装运行

第一步：以下地址，下载安装 kRPC 服务端插件：

* [Github](https://github.com/krpc/krpc/releases/download/v0.4.8/krpc-0.4.8.zip)
* [SpaceDock](https://spacedock.info/mod/69/kRPC)
* [Curse](https://mods.curse.com/ksp-mods/kerbal/220219-krpc-control-the-game-using-c-c-java-lua-python)
* [CKAN](https://forum.kerbalspaceprogram.com/index.php?/topic/90246-the-comprehensive-kerbal-archive-network-ckan-package-manager-v1180-19-june-2016/)

第二步：启动 KSP

第三步：你会看到一个欢迎窗口

![kRPC 欢迎窗口](https://krpc.github.io/krpc/_images/server-window.png)

第四步：点击 "Start Server"，如果没有爆 bug，你会看到一个绿灯 （绿灯亮时，表示服务在线）

第五步：如果嫌遮挡视线，可以点右上角叉叉隐藏这个窗口。



### 配置

可以点击上面窗口中的 "Edit" 来配置服务器。

* 协议：这时服务器使用的协议，它可以控制客户端连接到服务器的类型。比如 Python，或者其他通过 TCP/IP 进行通信的大多数客户端，需要选择 "Protobuf over TCP"。
* 地址：服务器监听的 IP 地址，只允许使用 "localhost" 或本机的 IP 地址，默认使用 "localhost"。
* 端口：设置为本机中未被占用的端口号，大多情况下，使用默认的即可。

此外，还有一些高级配置，可以点击 "Show advanced settings" 来显示隐藏的高级设置。

* 自动启动服务器：启用此功能后，每当游戏启动，服务器也会自动启动。
* 自动接受新客户端：启用此功能后，会自动允许新的客户端连接。如果禁用此项，则会弹出窗口，询问是否允许新客户端连接。

其他的高级设置项，与 [服务器的性能](https://krpc.github.io/krpc/internals.html#server-performance-settings) 有关。【TODO】



## Python 服务端

### Windows 

kRPC 支持Python 2.7 和 Python3.x.

1. 如果你没有安装 python,  需要先 [下载 python](https://www.python.org/downloads/windows) 然后安装。安装的时候，请将 pip 也装上。
2. 命令行进入`\python 的安装目录\scripts`，运行 `pip.exe install krpc` 来安装 kRPC 的 python模块。
3. 启动 Python IDE ，开始码码吧。



## Hello World

启动 KSP ，启动 kRPC 服务，然后运行以下 python 代码

```python
import krpc
conn = krpc.connect(name='Hello World')
vessel = conn.space_center.active_vessel
print(vessel.name)
```

上面的代码将执行：

​	第一行：导入 kRPC python 模块

​	第二行：打开一个连接到 krpc服务

​    第三行：获取待操作的飞船对象

​    第四行：输出飞船名字

运行以上代码后，你应该看到：

![](https://krpc.github.io/krpc/_images/hello-world.png)



至此，你已经写完你第一个与 KSP 通信的脚本。

---







# 教程和示例

下面将介绍如何使用 kRPC 在亚轨道飞行时发送一些信息，并且能安全再入着陆。包括：

1. 控制火箭 "油门"
2. 控制飞船指向特定方向
3. 当某个条件达成时，执行特定动作
4. 获取飞船的各种资源剩余量 (燃料、电量等)
5. 跟踪飞船和轨道数据 



以下代码使用下图的二级火箭。

![](https://krpc.github.io/krpc/_images/SubOrbitalFlight.png)



#### 第一步：准备发射

第一件事，我们需要打开连接，连接上服务器。

```java
    Connection connection = Connection.newInstance("Sub-orbital flight");
    KRPC krpc = KRPC.newInstance(connection);
    SpaceCenter spaceCenter = SpaceCenter.newInstance(connection);
```

接着，我们获取表示飞船的对象，之后可以通过此对象来操作飞船。

```java
    SpaceCenter.Vessel vessel = spaceCenter.getActiveVessel();	
```

然后，就可以准备发射火箭了，下面的代码将 "油门" 调到最大，并且指示飞船将俯仰、航向保持指向 90°（垂直向上）。最后，等待 1秒，这些设置就能生效了。

```java
    vessel.getAutoPilot().targetPitchAndHeading(90, 90);
    vessel.getAutoPilot().engage();
    vessel.getControl().setThrottle(1);
    Thread.sleep(1000);
```



#### 第二步：发射

下面的代码相当于你手动操作时，按的空格键

```java
    System.out.println("Launch!");
    vessel.getControl().activateNextStage();
```

毋庸置疑，固体燃料助推器的燃料耗尽后，肯定会抛弃的。那么，我们可以使用一个事件来监控火箭的固体燃料量，此事件在火箭没有固体燃料时触发。

```java
    {
      ProcedureCall solidFuel = connection.getCall(vessel.getResources(), "amount", "SolidFuel");
      Expression expr = Expression.lessThan(
        connection,
        Expression.call(connection, solidFuel),
        Expression.constantFloat(connection, 0.1f));
      Event event = krpc.addEvent(expr);
      synchronized (event.getCondition()) {
        event.waitFor();
      }
    }

    System.out.println("Booster separation");
    vessel.getControl().activateNextStage();
```



以上代码中，`vessel.resources` 返回一个用于获取飞船信息的 `Resources` 对象。接着使用 `Expression` 监听事件，在达到条件时触发。



#### 第三步：到达远地点

当飞船到达一定高度时，我们执行重力转弯。以下代码示例的是，当飞船到达 10km 时触发事件

```java
ProcedureCall meanAltitude = connection.getCall(vessel.flight(null), "getMeanAltitude");
Expression expr = Expression.greaterThan(
    connection,
    Expression.call(connection, meanAltitude),
    Expression.constantDouble(connection, 10000));
Event event = krpc.addEvent(expr);
synchronized (event.getCondition()) {
    event.waitFor();
}
```



上面这段代码中，调用 `vessel.flight()` 用来获取飞船的各种信息，比如它的方向、速度等。

接着我们将飞船调到俯仰 60°，航向 90°. 

```java
System.out.println("Gravity turn");
vessel.getAutoPilot().targetPitchAndHeading(60, 90);
```



现在我们使用事件，等待远地点到达 100km 时触发降低油门、分离火箭、关闭自动驾驶

```java
{
    ProcedureCall apoapsisAltitude = connection.getCall(
        vessel.getOrbit(), "getApoapsisAltitude");
    Expression expr = Expression.greaterThan(
        connection,
        Expression.call(connection, apoapsisAltitude),
        Expression.constantDouble(connection, 100000));
    Event event = krpc.addEvent(expr);
    synchronized (event.getCondition()) {
        event.waitFor();
    }
}

System.out.println("Launch stage separation");
vessel.getControl().setThrottle(0);
Thread.sleep(1000);
vessel.getControl().activateNextStage();
vessel.getAutoPilot().disengage();
```

这段代码中，`vessel.orbit` 会返回一切关于飞船轨道的信息。



#### 第四步：再入



TODO 













## 亚轨道飞行







## 参考框架

## 入轨

## 俯仰、航向与滚转

## 与模块互动

## 对接

## 可视化界面

## 自动控制







---



# Java











