title: "利用Unity进行AOP编程: 概念介绍（一)"
date: 2015-10-31 16:14:36
tags:
- AOP
- Unity
categories:
- .NET

---

#前言
在读这篇文章之前，建议大家要对Unity容器进行依赖注入有一定了解与实践，并且了解一些常用的设计模式。这里我们主要介绍如何利用Unity进行[AOP编程](https://zh.wikipedia.org/zh-cn/%E9%9D%A2%E5%90%91%E4%BE%A7%E9%9D%A2%E7%9A%84%E7%A8%8B%E5%BA%8F%E8%AE%BE%E8%AE%A1)。
面向 Aspect 的编程（AOP）是一种新的编程技术，它允许程序员对 横切关系（crosscutting concerns）（跨越典型职责界限的行为，例如日志记录）进行模块化。

#横切关注点(Crosscutting Concerns)
传统的程序经常表现出一些不能自然地适合单个程序模块或者几个紧密相关的程序模块的行为，因此面向 Aspect 的编程（AOP）应运而生。Aspect 的先驱将这种行为称为 横切，因为它跨越了给定编程模型中的典型职责界限。例如在系统中，我们可能需要在不同的模块区域记录日志。常见的横切关注点包括：日志记录、数据验证、异常处理、身份认证、缓存、性能监测、加密、压缩等。

可能系统中的不同类和组件都需要实现这些功能行为。然而，实现这些功能将会带来一些列的问题：
**维护一致性。**我们必须确保所有类和组件都以同一种方式实现这些功能。假如我们修改某个横切关系的实现方式，则必须确保所有地方都被一致修改。
**代码可维护性。**单一职责原则可以保证代码的可维护性，一个业务类必然不应该去实现例如记录日志的横切点。
**避免代码重复。**我们肯定不希望在系统各个角落到处写满重复的记录日志的代码。

在介绍如何利用Unity实现切面拦截之前，我们先看一下有哪些其他的实现方式。装饰者模式和AOP编程就是常见的拦截横切点的编程方式。

#装饰者模式
装饰者模式可以为我们的现有代码创建一个外壳，来处理横切关系。假设我们有一个**TanantStore**类负责tanant实体的数据存储，我们来演示下怎么利用装饰者模式，在不修改或扩展TanantStore代码的前提下，处理日志记录和缓存处理横切关系。
以下代码为TanantStore类和ITanantStore接口：
```csharp
public interface ITenantStore
{
void Initialize();
Tenant GetTenant(string tenant);
IEnumerable<string> GetTenantNames();
void SaveTenant(Tenant tenant);
void UploadLogo(string tenant, byte[] logo);
}

public class TenantStore : ITenantStore
{
...
public TenantStore()
{
...
}
...
}
```

以下代码展示了一个为**TanantStore**添加了日志记录功能的装饰器:
```csharp
class LoggingTenantStore : ITenantStore
{
    private readonly ITenantStore tenantStore;
    private readonly ILogger logger;

    public LoggingTenantStore(ITenantStore tenantstore, ILogger logger)
    {
        this.tenantStore = tenantstore;
        this.logger = logger;
    }

    public void Initialize()
    {
        tenantStore.Initialize();
    }

    public Tenant GetTenant(string tenant)
    {
        return tenantStore.GetTenant(tenant);
    }

    public IEnumerable<string> GetTenantNames()
    {
        return tenantStore.GetTenantNames();
    }

    public void SaveTenant(Tenant tenant)
    {
        tenantStore.SaveTenant(tenant);
        logger.WriteLogMessage("Saved tenant" + tenant.Name);
    }

    public void UploadLogo(string tenant, byte[] logo)
    {
        tenantStore.UploadLogo(tenant, logo);
        logger.WriteLogMessage("Uploaded logo for " + tenant);
    }
}
```

注意到这个装饰器类实现了**ITanantStore**接口，并在构造器中加了**ITenantStore**实例的参数。在每个方法体中，它在记录日志之前调用了原接口的方法。当然，我们可以灵活的改变记录日志的位置，不管是在调用原方法之前或者之后。
假如我们有另一个装饰器类**CachingTanantStore**实现了**ITenantStore**接口，我们就可以同时处理日志和缓存功能：
```csharp
var basicTenantStore = new TenantStore(tenantContainer, blobContainer);
var loggingTenantStore = new LoggingTenantStore(basicTenantStore, logger);
var cachingAndLoggingTenantStore = new CachingTenantStore(loggingTenantStore, cache);
```

假如我们调用cachingAndLoggingTenantStore**对象的**UploadLogo**方法，就会先调用**CachingTenantStore**类的**UploadLogo**方法并追溯到**LoggingTenantStore**类的**UploadLogo**方法，最终追溯到调用**TenantStore**类的**UploadLogo**方法。
![decorator](/img/decorator pattern at run time.png)

#用Unity连接装饰器链
当然我们可以用Unity容器来代替手动写装饰器对象：
```csharp
container.RegisterType<ILogger, Logger>();
container.RegisterType<ICacheManager, SimpleCache>();

container.RegisterType<ITenantStore, TenantStore>("BasicStore");
container.RegisterType<ITenantStore, LoggingTenantStore>(
"LoggingStore",
new InjectionConstructor(
new ResolvedParameter<ITenantStore>("BasicStore"),
new ResolvedParameter<ILogger>()));

// Default registration
container.RegisterType<ITenantStore, CachingTenantStore>(
new InjectionConstructor(
new ResolvedParameter<ITenantStore>("LoggingStore"),
new ResolvedParameter<ICacheManager>()));
```

#AOP编程
在AOP编程中，我们可以将业务类与处理横切关系的类分离。在上面的代码中，**TenantStore**类就只负责业务逻辑，而实现了**ILogger**和**ICacheManager**接口的类则负责处理横切关系。
上段示例演示了怎么利用装饰器显示连接负责横切关系的类，然而它需要我们分别为之写装饰器类，无论是显式实例化或利用依赖注入。
而AOP提供了一种简化的方式来连接横切关系类和业务类，它使我们在不写单独的装饰器类的情况下就可以处理横切。这种方式通过一种动态机制连接切面类来减少代码依赖。
关于C#的AOP框架示例，请自行参考[What is PostSharp?](http://doc.postsharp.net/)

#拦截
拦截是AOP实现动态链接的必要方式。我们可以通过拦截机制动态地将代码插入到切面，因此它也是AOP编程的基础机制。

拦截器可以将多个实现了横切关系的代码插入到调用代码与目标业务代码之间的管道中，它本质上也是使用与装饰者模式相似的方式来实现的。拦截器与装饰者模式之间的关键差异就是拦截器在运行过程中动态得创建了装饰器。这使得我们不需要手动写大量的装饰器代码，而只需要进行简单的配置。下面我们简单介绍下两种不同的拦截机制。

##实例拦截
通过实例拦截，Unity在客户端对象与目标对象之间动态地创建一个代理对象，这个代理对象负责传递客户端对象对目标对象的调用指令，并在其过程中实现了横切点的功能。下图展示了其机制：
![instance interception](/img/instance interception.png)

实例拦截是Unity中最常用是方式，我们可以利用Unity的实例拦截方式来拦截各个对象，无论是否使用Unity容器，无论是否是虚方法。**但是，我们不能将动态创建的代理类转换成目标对象的类型。**

##类型拦截
通过类型拦截，Unity动态创建一个继承目标对象类型、并包含处理横切关系功能的新类型，Unity容器将在运行时中实例化这个子类。下图展示了类型拦截机制。
![type interception](/img/type interception.png)

我们可以用Unity的类型拦截方式来拦截虚方法。**同时，由于新的类型从目标对象继承，我们可以把它当作目标对象来使用。**

#总结
在这篇文章中，我们通过装饰者模式引申出拦截方式如何在运行时中动态创建相关类型来处理横切关系。在下篇博客中，将给大家介绍我们如何利用Unity容器来具体实现拦截对象。

