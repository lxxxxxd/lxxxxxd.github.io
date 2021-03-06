## libnetwork设计与实现

简介：进行docker容器网络管理，实现了容器网络模型（CNM）

### CNM

容器网络模型（The Container Network Model），为容器提供抽象网络环境，这种抽象的网络环境可以支持多种网络驱动：bridge，null，host，overlay，remote。模型基于以下三个组件：

*sandbox*：负责容器网络栈配置，容器网络网卡管理，路由表和DNS服务设置。实现为一个Linux Network Namespace或者FreeBSD Jail或者其他类似的概念。一个sandbox包含多个指向各自网络的endpoint。

*network*：一组可以直接相互通信的端点，实现可能是一个网桥，VLAN等。network包含多个endpoint。

*endpoint*：endpoint帮助sandbox加入一个network，实现为veth 对，Open vSwitch软件交换机端口或者类似的东西。如果建立连接，endpoint只属于一个特定的网络和一个特定的sandbox。

### CNM对象

NetworkController：为用户（Docker Engine）提供一个操作入口，爆露了一个简单的API来分配和管理网络资源。libnetwork提供了多个可用的驱动。NetworkController对象允许用户为给定的网络绑定一个特定的驱动。

Driver：不是用户可以见的对象，Driver提供了真正的网络实现。NetworkController提供了对于libnetwork透明的被驱动直接处理的API（options / labels）来配置网络驱动。网络驱动包括：induilt（Bridge，Host，None，Overlay），remote（plugin接入），这些驱动可以满足多种使用和部署场景。现在，特定的一个驱动负责管理一个网络。在未来，多个驱动参与各种网络管理功能。

Network: 就是CNM:network，NetworkController提供API创建和管理Network对象。当Network对象创建或者更新后，对应的网络驱动Driver会收到通知。libnetwork在抽象层让Network对象提供了一组属于同一个网络的endpoint之间的连接，并与其他的完全隔离。Driver对象真正提供连接和隔离。连接可以是在同一台主机上或者穿插在多个主机上。因此，Network对象在集群中有全局作用域。

Endpoint：指服务端点，在服务之间提供连接，这里的服务是容器爆露端口实现的。Network对象提供API创建和管理endpoint对象。一个endpoint只附属于一个network。endpoint创建操作被对应的Driver（为对应的Sandbox分配资源）对象调用。因为endpoint代表一个Service而不是一个特定的容器，endpoint在集群中具有全局作用域。

Sandbox：容器的网络配置，比如说IP地址，MAC地址，路由，DNS项。当用户请求在一个network上创建endpoint的时候，Sandbox对象开始创建。处理Network的Driver负责分配所需的网络资源（如IP地址），并将名为SandboxInfo的信息传递回libnetwork。libnetwork将使用特定的操作系统的结构（例如：netns for Linux）将网络配置填充到Sandbox表示的容器中。沙盒可以有多个端点连接到不同的网络。由于沙盒与给定主机中的特定容器关联，因此它具有表示容器所属主机的本地作用域。

### CNM属性

Options: 提供了一种通用的和灵活的机制，可以将驱动程序特定的配置选项直接从用户传递给驱动程序。选项只是key-value对，键由字符串表示，值由泛型对象（如GO Interface{}）表示。Libnetwork只在key与net包中定义的已知labels匹配时才对选项进行操作。选项还包括如下所述的标签。选项通常不可见（在UI中），而标签是可见的。

Labels: Labels与Options非常相似，实际上只是Options的一个子集。标签通常是最终用户可见的，并在UI中使用--Labels选项显式表示。它们从UI传递到驱动程序，以便驱动程序可以利用它并执行任何特定于驱动程序的操作（例如，从网络中分配IP地址的子网）。

### CNM 生命周期

CNM的消费者，比如Docker，通过CNM对象和它的api进行交互，将他们管理的容器连网。

1.register：驱动程序向NetworkController注册。内置驱动程序注册在libnetwork内部，而远程驱动程序通过插件机制注册到libnetwork（插件机制是WIP）。每个驱动程序处理特定的网络类型。

2.NetworkController对象是使用libnetwork.New()API创建的，用于管理网络的分配，并可以选择使用特定于Driver的Options配置Driver。

3.通过提供名称（name）和网络类型(networkType)，使用控制器的NewNetwork()API创建网络。networkType参数有助于选择相应的驱动程序并将创建的网络绑定到该驱动程序。从现在起，网络上的任何操作都将由该驱动程序处理。

4.controller.NewNetwork()API还接受可选的options参数，该参数携带驱动程序特定的选项和标签，驱动程序可以使用这些选项和标签来实现其目的。

5.可以调用network.CreateEndpoint()在给定的网络中创建新的端点。这个API还接受驱动程序可以使用的可选选项参数。这些“选项”同时带有共识的标签和特定于Driver标签。驱动程序将依次使用driver.CreateEndpoint调用，当在网络中创建endpoint时，它可以选择保留IPv4/IPv6地址。驱动程序将使用driverapi中定义的InterfaceInfo接口分配这些地址。需要IP/IPv6来完成endpoint作为服务定义以及endpoint公开的端口，因为实际上服务endpoint只是一个网络地址和应用程序容器正在侦听的端口号。

6.endpoint.Join()可用于将容器附加到endpoint。如果该容器不存在，则联接操作将创建沙箱。Drivers可以使用Sandbox的Key标识附加到同一容器的多个端点。这个API还接受驱动程序可以使用的可选Options。

    虽然这不是LibNetwork的直接设计问题，但我们强烈鼓励Docker这样的用户在容器的Start（）生命周期中调用endpoint.Join（），在容器运行之前调用它。作为Docker集成的一部分，这将得到处理。

    endpoint join（）API的常见问题之一是，为什么我们需要一个API来创建端点，而另一个API来连接端点。

    答案是基于这样一个事实：端点表示一个服务，该服务可能由容器支持，也可能不由容器支持。创建endpoint时，它将保留其资源，以便以后可以将任何容器附加到终结点并获得一致的网络行为。

7.当容器停止时，可以调用endpoint.Leave()。驱动程序可以清除它在Join（）调用期间分配的状态。当最后一个引用endpoint离开网络时，LibNetwork将删除Sandbox。但是只要endpoint仍然存在，LibNetwork就会保留IP地址，并在容器（或任何容器）再次连接时重用。这样可以确保容器的资源在停止和重新启动时被重用。

8.endpoint.Delete()用于从网络中删除endpoint。这将导致删除终结点并清除缓存的sandbox.Info。

9.network.Delete()用于删除网络。如果存在网络上存在的任何端点，LIbNETs将不允许删除。

### Drivers
#### API
Drivers本质上是libnetwork的扩展，为上面定义的所有libnetworkapi提供了实际的实现。因此，所有Network和Endpoint APIs都有1-1对应关系，其中包括：

    driver.Config

    driver.CreateNetwork

    driver.DeleteNetwork

    driver.CreateEndpoint

    driver.DeleteEndpoint

    driver.Join

    driver.Leave

这些面向Driver的APIs使用唯一标识符(networkid、endpointid，…)而不是name（如面向用户的APIs中所示）。

这些APIs仍在发展，并且可以根据驱动程序的需求对它们进行更改，特别是在多主机网络方面。
### Driver语义

    Driver.CreateEndpoint

此方法通过方法interface和AddInterface传递接口EndpointInfo。

如果接口返回的值为非零，则驱动程序应使用其中的接口信息(例如，将一个或多个地址视为静态提供)，如果不能，则必须返回错误。如果该值为nil，则驱动程序应恰好分配一个新接口，并使用AddInterface记录它们；如果不能，则返回错误。如果接口为非零，则禁止使用AddInterface。
