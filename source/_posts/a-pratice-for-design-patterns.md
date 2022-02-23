title: 我有一把新锤子，问题当成钉子看 —— 从实践来谈设计模式 
date: 2022-02-23 00:13:13
tags:
- .NET
- Web Api
- CAP
categories:
- .NET
---


相信大家都知道设计模式，很多人都能背出一些常用的设计模式，但是很少有人知道怎么去使用它。甚至有很多人觉得设计模式没有用，哪怕在很多高水平的程序员之间，面对设计模式到底是不是一种“屠龙术”也有着很激烈的争论。

今天文章主题我们不争论设计模式有没有用，而是通过一次具体问题的实践，来看看我们到底怎么去使用设计模式，怎么让一段代码演变得更有扩展性。

## 问题的开始
我们假设这样一个场景，现在有一台物联人脸识别设备，不断地往消息队列MQTT发送消息，比如心跳包、上下线通知、人脸抓拍事件等等，我们系统要做的是：连接消息队列并监听，根据不同的消息主题及消息内容做出不同的处理。那么我们一开始的代码可能会这么写：
```csharp
public class MqttClientService{
    public IManagedMqttClient MqttClient { get; set; }

    public void Start(){
	// 省略消息队列连接过程
        MqttClient.UseApplicationMessageReceivedHandler(HandleReceivedMessage);
    }

    private Task HandleReceivedMessage(MqttApplicationMessageReceivedEventArgs e){
        // 处理消息内容
	    var topic = e.ApplicationMessage.Topic;
        if(topic.EndsWith("heatbeat")){
            // 处理心跳包业务
        }
        else if(topic.EndsWith("basic")){
            // 处理上下线基础业务
        }
        else if(topic.EndsWith("recognize")){
            // 处理人脸识别结果业务
        }
        else if(topic.EndsWith("snap")){
            // 处理陌生人抓拍业务
        }
        else if(topic.EndsWith("ack")){
            // 处理设备指令回复业务
        }
}
```


## 引入策略模式
我们可以看到，在最开始的代码中，我们使用简单的条件控制结构来处理业务逻辑，这样就会出现一个弊端，当我们的业务逻辑非常复杂，或者消息种类非常多的情况下，代码就会变得非常冗长，我们经常能看到一个代码文件达到几千行甚至上万行。那么这时候大家就会想到策略模式，我们就会定义一个Handler接口，并为每一种业务写一个Handler实现：
```csharp
public class MqttMessage
{
    public MqttMessage(string topic, string message)
    {
        Message = message;
        Topic = topic;
    }

    public string Message { get; set; }
    public string Topic { get; set; }
}

public interface IMessageHandler
{
    public Task Handle(MqttMessage message);
}

public class HeartbeatHandler: IMessageHandler
{
    public Task Handle(MqttMessage message){
        // 处理心跳包业务
    }
}

public class RecognizeHandler: IMessageHandler
{
    public Task Handle(MqttMessage message){
        // 处理人脸识别结果业务
    }
}

......

```

这时候，我们最开始的代码会变成这样:
```csharp
private Task HandleReceivedMessage(MqttApplicationMessageReceivedEventArgs e){
    // 处理消息内容
    var topic = e.ApplicationMessage.Topic;
    var message = Encoding.UTF8.GetString(e.ApplicationMessage.Payload);
    IMessageHandler handler;
    if(topic.EndsWith("heatbeat")){
        // 处理心跳包业务
         handler = new HeartbeatHandler();
    }
    else if(topic.EndsWith("basic")){
        // 处理上下线基础业务
         handler = new BasicHandler();
    }
    else if(topic.EndsWith("recognize")){
        // 处理人脸识别结果业务
         handler = new RecognizeHandler();
    }
    else if(topic.EndsWith("snap")){
        // 处理陌生人抓拍业务
         handler = new SnapHandler();
    }
    else if(topic.EndsWith("ack")){
        // 处理设备指令回复业务
         handler = new AckHandler();
    }
    handler.Handle(new MqttMessage(topic, message));
}

```

## 引入工厂模式
到这里，大家又会发现，创建Handler的过程还是很长，并且具有明显的业务特征：它是一个生产Handler的流水线，我们可以把这一部分代码抽出来，于是就引出了简单工厂模式：
```csharp
public class MessageHandlerFactory {

    public IMessageHandler CreateHandler(string topic)
    {
        IMessageHandler handler;
        if(topic.EndsWith("heatbeat")){
            // 处理心跳包业务
             handler = new HeartbeatHandler();
        }
        else if(topic.EndsWith("basic")){
            // 处理上下线基础业务
             handler = new BasicHandler();
        }
        else if(topic.EndsWith("recognize")){
            // 处理人脸识别结果业务
             handler = new RecognizeHandler();
        }
        else if(topic.EndsWith("snap")){
            // 处理陌生人抓拍业务
             handler = new SnapHandler();
        }
        else if(topic.EndsWith("ack")){
            // 处理设备指令回复业务
             handler = new AckHandler();
        }
        return handler;
    }
}
```

这时，原来的业务代码块就成了：
```csharp
private readonly MessageHandlerFactory _handlerFactory = new MessageHandlerFactory();

private Task HandleReceivedMessage(MqttApplicationMessageReceivedEventArgs e){
    // 处理消息内容
    var topic = e.ApplicationMessage.Topic;
    var message = Encoding.UTF8.GetString(e.ApplicationMessage.Payload);
    IMessageHandler handler = _handlerFactory.CreateHandler(topic);
    handler.Handle(new MqttMessage(topic, message));
}
```

## 控制反转
到这里，我在原来的代码块已经几乎看不到业务代码了，很清爽。但是问题又来了，业务代码是清爽，我们的工厂类还是不清爽啊，每次有一种新的消息类型，都要去工厂类里添加一段，不仅繁琐还不好找，万一新入职的员工找了半天都不知道在哪加怎么办呢？这设计模式用了还不如不用呢是不是？所以，我们再想一下，是不是可以根据一些特性，来动态得创建Handler呢？既然写道这里，那肯定是有办法的，这里我们以dotnet为例，通过特性+反射+IOC容器来实现动态创建。

首先我们创建一个TopicEndWith特性类，来给不同的消息进行分类：
```csharp
public class TopicEndWithAttribute : Attribute
{
    public TopicEndWithAttribute(string suffix)
    {
        Suffix = suffix;
    }

    public string Suffix { get; set; }
}

```

这里我们只创建了简单TopicEndWith特性，对于复杂一些的消息分类，我们也可以干脆正则表达式来实现，现在把特性给Handler装上：
```csharp
[TopicEndWith("heartbeat")]
public calss HeartbeatHandler: IMessageHandler
{
    public Task Handle(MqttMessage message){
        // 处理心跳包业务
    }
}
```

接着，我们再Program.cs文件里，把所有的Handler都注入到IOC容器中：
```csharp
 Assembly.GetAssembly(typeof(Program))
    .DefinedTypes
    .Where(x => x.IsClass && !x.IsAbstract &&
                x.GetInterfaces().Any(i => i == typeof(IMessageHandler)))
    .ForEach(type => { builder.Services.AddTransient(type); });
```

最后，我们再改一改工厂类：
```csharp
public class MqttMessageHandlerFactory 
{
    private readonly IServiceProvider _serviceProvider; // IOC容器工厂
    private readonly Dictionary<string, Type> _handlerTypes = new(); // 缓存Handler类型

    public MqttMessageHandlerFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
        var handlerTypeInfos = Assembly.GetAssembly(typeof(Program))
                                .DefinedTypes
                                .Where(x => x.IsClass && !x.IsAbstract && 
                                          x.GetInterfaces().Any(i => i == typeof(IMessageHandler)))
        foreach (var type in handlerTypeInfos)
        {
            var attr = type.GetCustomAttribute<TopicEndWithAttribute>();
            if (attr != null && !string.IsNullOrEmpty(attr.Suffix))
            {
                _handlerTypes.TryAdd(attr.Suffix.ToLower(), type);
            }
        }
    }

    public IMessageHandler CreateHandler(string topic)
    {
        var reg = new Regex(".*face/.+/.+");
        if (!reg.IsMatch(topic)) return null;
        var command = topic.Split('/').Last().ToLower();

        if (!_handlerTypes.TryGetValue(command, out var handlerType)) return null;

        var handler = _serviceProvider.GetRequiredService(handlerType) as IMessageHandler;
        return handler;
    }
}
```

到这里我们发现，我们已经把所有的if控制结构都拿掉了，如果我们后面有新的消息类型，只需要创建一个MessageHandler，并且把TopicEndWith特性装上，就可以专注于业务代码，完全不需要改动其他代码。同时，我们可以把所有的Handler放至同一个文件夹下，不仅代码清晰，也可以防止多个人改动同一个文件造成冲突改出Bug，甚至可以安排多个人员各自开发不同的消息业务，极大提升开发效率。

## 引入代理模式
然而工作并没有完成，我们上面的Handler里面提到一个AckHandler，它用来处理设备指令的统一回复，也就是说我们对设备下发的所有指令回复，都是通过这一个Handler来处理，这时我们需要解包消息内容，根据消息内容的特征(而非消息Topic的特征)来转交给其他Handler，所以我们又引出了代理模式：
```csharp
public interface IHandlerProxy{
    // 根据消息内容的分类创建对应的Handler并处理业务
    public Task DeliverHander(MqttMessage message);
}
```

在实现Handler代理的过程中，我们发现，之前的简单工厂模式又需要演变为抽象工厂模式。由于代码类似，后面就留给大家思考。


## 总结
通过以上的例子，我们依次使用了策略模式、工厂模式、控制反转、代理模式，实际上真正使用到设计模式的时候，不是单一的用到其中一个两个，而是多种模式互相配合。如果我们去研究dotnet的源码，去研究开源社区一些主流的框架，会发现都是这样的混合模式，因此可见，设计模式并不是完全没有用，当我们设计通用性框架的时候仍然必不可少。但设计模式也不是“屠龙术”，作为架构师，必然是行走在钢索上的，稍有不慎，便会引来争论。最后把看到的一首打油诗送给大家共勉。

我有一把新锤子，问题当成钉子看。
我刚学了屠龙术，猫狗当成龙来宰。
