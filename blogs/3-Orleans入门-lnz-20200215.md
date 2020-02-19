# Orleans 入门

## Orleans简介

Orleans是由微软设计开发并开源的一款基于Actor模型的跨平台、分布式框架。它可以直接构建分布式大规模计算应用程序，而无需学习和应用复杂的并发或其他缩放模式。

官网：http://dotnet.github.io/orleans/

源码：https://github.com/dotnet/orleans

它具有以下特性：

* 默认扩展：Orleans处理构建分布式系统的复杂性，使应用程序能够轻易扩展
* 低延迟：Orleans允许在内存中保持你的状态，让应用程序快速响应传入的请求
* 简化并发：Orleans允许你写简单的单线程C#代码来使用actor间的异步消息传递处理并发。

## Orleans核心

### Grains

Grain是Orleans应用的基石，是隔离、分布和持久化的最小单元。Grain是包含用户定义的身份，行为和状态的实体。

**Grain生命周期**：当有工作需要grain处理的时候，Orleans确保一个Orleans Silos上有一个grain的实例。当silo上没有grain实例时，运行时会创建一个。这个过程称作激活。当一个grain使用Grain持久化，运行时在激活时从持久化存储中自动读取状态。 Orleans透明地控制激活和注销的过程。当编写一个grain的时候，开发者只要假设所有的grain永远是被激活的。

**Grain Identity**：无论grain是否处于活动状态，它都需要具有标识，以便silo可以按需激活它。它可以用GUID, long, string来表示，也可以用 long+string 或 Guid+string 来组合表示。

### Silo

一个Orleans silo是一个执行Orleans grain的宿主服务器，负责托管grain。silo到silo的消息和其他的client到silo的消息都通过同一个监听端口。通常一个机器运行一个silo。

一定数量的silo可以组成一个Orleans集群。Orleans运行时完全自动化管理集群，Silo通过读取共享存储得知其他的silo的状态。任何时候，一个silo可以通过在共享存储中注册来加入一个集群。这样集群可以在运行时动态的扩容。 Orlean通过从集群中移除不响应的silo来保证集群的可靠性和可用性。运行时使群集群托管的grain可以相互通信，就好像它们在单个进程中一样。


## 一般开发流程

- 定义Grain Interface

    Grain通过接口彼此交互，接口的所有方法必须返回一个`Task`或`Task<T>`

    ```cs
    public interface IUserGrain : IGrainWithGuidKey
    {
        Task<UserInfo> GetUserInfo();
    }
    ```

- 实现Grain类

    所有的grain类都需要继承于`Grain`,可以选择重写`OnActivateAsync`和`OnDeactivateAsync`虚方法，运行时将在激活或停用grain时调用，这使我们有机会执行其他初始化和清理操作。

    ```cs
    public class UserGrain : Grain, IUserGrain
    {
        public Task<UserInfo> GetUserInfo() 
        {
            return Task.FromResult(userInfo);
        }
    }
    ```
    然后就可以通过获取代理对象来调用grain的方法:

    ```cs
        var userGrain = GrainFactory.GetGrain<IUserGrain>(userGuid);
        var userInfo = await userGrain.GetUserInfo();
    ```


- 配置Silo

    将`Microsoft.Orleans.Server`NuGet包添加到项目中。
    ```cs
    PM> Install-Package Microsoft.Orleans.Server
    ```
    通过`ISiloBuilder.Configure`方法配置`ClusterOptions`，将其指定为群集选择，然后配置Silo端点。

    调用`ConfigureApplicationParts`将带有Grain类的程序集显式添加到应用程序设置中。由于WithReferences扩展，它还会添加任何引用的程序集。

    完成这些步骤后，将构建并启动Silo。
    以下是一个简单的控制台示例:

    ```cs
        var builder = new SiloHostBuilder()
            .UseLocalhostClustering()
            .Configure<ClusterOptions>(options =>
            {
                options.ClusterId = "clusterId";
                options.ServiceId = "serviceId";
            })
            .ConfigureApplicationParts(parts => parts.AddApplicationPart(typeof(IUserGrain).Assembly).WithReferences())
            .Configure<EndpointOptions>(options => options.AdvertisedIPAddress = IPAddress.Loopback)
            .ConfigureLogging(logging => logging.AddConsole());

        var host = builder.Build();
        await host.StartAsync();

        Console.WriteLine("Press Enter to terminate...");
        Console.ReadLine();

        await host.StopAsync();
    ```

- 配置Client

    首先安装Microsoft.Orleans.Client包

    ```cs
    PM> Install-Package Microsoft.Orleans.Client
    ```
    
    为 `ClientBuilder` 配置ClusterId,调用 `ConfigureApplicationParts` 将带有Grain接口的程序集显式添加到应用程序设置中。

    完成这些步骤后，就可以`Connect()`在其上构建客户端连接到Silo集群。
    以下是一个简单的客户端连接到本地Silo示例:
    ```cs
        var builder = new ClientBuilder()
            .UseLocalhostClustering()
            .Configure<ClusterOptions>(options =>
            {
                options.ClusterId = "clusterId";
                options.ServiceId = "serviceId";
            })
            .ConfigureLogging(logging => logging.AddConsole());
        var client = builder.Build();
        await client.Connect();
    ```

