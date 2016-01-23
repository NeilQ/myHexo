title: "利用Unity进行AOP编程: 策略注入（三)"
date: 2016-01-23 16:09:17
tags:
- AOP
- Unity
categories:
- .NET
---

#前言
[上一篇文章里](http://neilq.github.io/2015/12/21/AOP-programming-with-Unity-2/)，给大家介绍的利用Unity拦截进行编码实战，那么本编文章将给大家演示一些更高级灵活的使用。

#策略注入
首先我们先展示一段利用策略添加拦截行为的代码，然后我们再来分析。
```csharp
container.RegisterType<ITenantStore, TenantStore>(
  new InterceptionBehavior<PolicyInjectionBehavior>(),
  new Interceptor<InterfaceInterceptor>());

container.RegisterType<ISurveyStore, SurveyStore>(
  new InterceptionBehavior<PolicyInjectionBehavior>(),
  new Interceptor<InterfaceInterceptor>());

var first = new InjectionProperty("Order", 1);
var second = new InjectionProperty("Order", 2);

container.Configure<Interception>()
  .AddPolicy("logging")
  .AddMatchingRule<AssemblyMatchingRule>(
    new InjectionConstructor(
    new InjectionParameter("Tailspin.Web.Survey.Shared")))
  .AddCallHandler<LoggingCallHandler>(
    new ContainerControlledLifetimeManager(),
    new InjectionConstructor(),
    first);

container.Configure<Interception>()
  .AddPolicy("caching")
  .AddMatchingRule<MemberNameMatchingRule>(
    new InjectionConstructor(new [] {"Get*", "Save*"}, true))
  .AddMatchingRule<NamespaceMatchingRule>(
    new InjectionConstructor(
      "Tailspin.Web.Survey.Shared.Stores", true))
  .AddCallHandler<CachingCallHandler>(
    new ContainerControlledLifetimeManager(),
    new InjectionConstructor(),
    second);
```
我们可以看到示例中通过**PolicyInjectionBehavior**类型来注册了两个Sotre类。这种行为利用了策略定义的方式，将处理类插入到应用程序管道中，这两个处理类分别是**LoggingCallHandler**和**CachingCallHandler**,每种策略都有一个名称作唯一标识，以及有一到两个的匹配规则来定义何时得以应用。

这种策略注入程序块包含的内置匹配规则有：
* Assembly名称
* 命名空间
* 类型
* Tag特性 
* 自定义特性
* 成员名称
* 方法签名
* 参数类型
* 属性
* 返回类型
当然，我们仍然可以自定义匹配规则。这里我们不作详细描述，有兴趣的朋友请自行研究。

我们来看上述代码的缓存策略，它使用了两种并集匹配规则：当满足方法名以**Get**或者**Save**(**true**参数表示它不忽略大小写)，并且所调用的对象在**Tailspin.Web.Survey.Shared.Stores**命名空间时，缓存处理类将被触发。
**first**和**second**参数用来控制容器对同一个满足规则的实例调用处理类的顺序，在示例中，容器将首先调用logging handler以确保日志不丢失。

需要注意的是，这两个策略都使用**ContainerControlledLifetimeManager**类型来确保处理类是单例。

接下来我们展示两个处理类的具体实现：
```csharp
class LoggingCallHandler : ICallHandler
{
  public IMethodReturn Invoke(IMethodInvocation input,
    GetNextHandlerDelegate getNext)
  {
    // Before invoking the method on the original target
    WriteLog(String.Format("Invoking method {0} at {1}",
      input.MethodBase, DateTime.Now.ToLongTimeString()));

    // Invoke the next handler in the chain
    var result = getNext().Invoke(input, getNext);

    // After invoking the method on the original target
    if (result.Exception != null)
    {
      WriteLog(String.Format("Method {0} threw exception {1} at {2}",
        input.MethodBase, result.Exception.Message,
        DateTime.Now.ToLongTimeString()));
    }
    else
    {
      WriteLog(String.Format("Method {0} returned {1} at {2}",
        input.MethodBase, result.ReturnValue,
        DateTime.Now.ToLongTimeString()));
    }

    return result;
  }

  public int Order
  {
    get;
    set;
  }

  private void WriteLog(string message)
  {
    ...
  }
}
```
```csharp
public class CachingCallHandler :ICallHandler

{
  public IMethodReturn Invoke(IMethodInvocation input,
    GetNextHandlerDelegate getNext)
  {
    //Before invoking the method on the original target
    if (input.MethodBase.Name == "GetTenant")
    {
      var tenantName = input.Arguments["tenant"].ToString();
      if (IsInCache(tenantName))
      {
        return input.CreateMethodReturn(FetchFromCache(tenantName));
      }
    }

    IMethodReturn result = getNext()(input, getNext);

    //After invoking the method on the original target
    if (input.MethodBase.Name == "SaveTenant")
    {
      AddToCache(input.Arguments["tenant"]);
    }

    return result;
  }

  public int Order
  {
    get;
    set;
  }

  private bool IsInCache(string key)
  {
    ...
  }

  private object FetchFromCache(string key)
  {
    ...
  }

  private void AddToCache(object item)
  {
    ...
  }
}
```

在多数情况下，策略注入处理类可能比上篇文章中的拦截行为类更容易定位到横切面。这是因为我们可以利用策略规则来控制容器是否调用处理类，而不用像拦截行为类那样注入到每个符合需求的类型。

#策略注入和特性
利用策略匹配规则意味着我们可以直接在容器中全局管理策略，我们可以将错略定义在容器代码中或者配置文件中。

然而，我们还有另一种方式，利用特性在业务代码中标识是否需要容器调用处理类。在这种情况下，写业务类的码猴需要确保所有的横切关系都被正确定位到。

我们展示一下代码：
```csharp
container.RegisterType<ITenantStore, TenantStore>(
  new InterceptionBehavior<PolicyInjectionBehavior>(),
  new Interceptor<InterfaceInterceptor>());

container.RegisterType<ISurveyStore, SurveyStore>(
  new InterceptionBehavior<PolicyInjectionBehavior>(),
  new Interceptor<InterfaceInterceptor>());
```
当我们利用特性时，就不需要配置策略了，而只要定义相应的特性，以下代码展示了日志特性。缓存特性几乎是与之一样的，因此不再展示。
```csharp
using Microsoft.Practices.Unity;
using Microsoft.Practices.Unity.InterceptionExtension;

class LoggingCallHandlerAttribute : HandlerAttribute
{
  private readonly int order;

  public LoggingCallHandlerAttribute(int order)
  {
    this.order = order;
  }

  public override ICallHandler CreateHandler(IUnityContainer container)
  {
    return new LoggingCallHandler() { Order = order };
  }
}
```
**CreateHandler**方法创建了一个**LoggingCallHandler**实例，并设置了调用顺序。然后我们就能用这个特性来装饰我们的业务类了。
```csharp
class MyTenantStoreWithAttributes : ITenantStore, ITenantLogoStore
{
  ...

  [LoggingCallHandler(1)]
  public void Initialize()
  {
    ...
  }

  [LoggingCallHandler(1)]
  [CachingCallHandler(2)]
  public Tenant GetTenant(string tenant)
  {
    ...
  }

  [LoggingCallHandler(1)]
  public IEnumerable<string> GetTenantNames()
  {
    ...
  }

  [LoggingCallHandler(1)]
  [CachingCallHandler(2)]
  public void SaveTenant(Tenant tenant)
  {
    ...
  }

  [LoggingCallHandler(1)]
  public virtual void UploadLogo(string tenant, byte[] logo)
  {
    ...
  }
}
```

#总结
这篇文章向大家介绍了如何利用策略规则和特性来灵活得进行切面拦截，系列到这里已经结束了，但其中仍有很多细节值得研究，如有机会希望和大家一起交流。

这是我第一个完整的系列博客，最初打算写依赖注入系列的文章，但考虑到这一技术在国内已经较为普遍，中文资料也较多，因此跳过，直接介绍Unity的拦截器。在这之后，如有时间，我将向大家介绍单元测试中的moq技术，其实这在国外也是非常普遍的技术，奈何国内软件开发从来就不重视测试，借此吐槽。
