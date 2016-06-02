title: "利用Unity进行AOP编程: 拦截器实战（二)"
date: 2015-12-21 15:49:07
tags:
- AOP
- Unity
categories:
- .NET
---

# 前言
[上一篇文章里](http://neilq.github.io/2015/10/31/AOP-programming-with-Unity-1/)，我们介绍了Unity拦截器的基本概念，本编文章我们将利用它来进行实战。

# 在Unity Container中配置
默认情况下，Unity Container不支持拦截器，因此我们得显式地将其加入到项目中，当然，首先得通过NuGet安装扩展包。
```csharp
using Microsoft.Practices.Unity.InterceptionExtension;
...

IUnityContainer container = new UnityContainer();
container.AddNewExtension<Interception>();
```

# 定义拦截器
上篇文章，我们用装饰器模式实现了logging和caching两种横切关系，而在这里，我们通过实现**IInterceptionBehavior**接口来实现拦截。
```csharp
using Microsoft.Practices.Unity.InterceptionExtension;

class LoggingInterceptionBehavior : IInterceptionBehavior
{
  public IMethodReturn Invoke(IMethodInvocation input,
    GetNextInterceptionBehaviorDelegate getNext)
  {
    // Before invoking the method on the original target.
    WriteLog(String.Format(
      "Invoking method {0} at {1}",
      input.MethodBase, DateTime.Now.ToLongTimeString()));

    // Invoke the next behavior in the chain.
    var result = getNext()(input, getNext);

    // After invoking the method on the original target.
    if (result.Exception != null)
    {
      WriteLog(String.Format(
        "Method {0} threw exception {1} at {2}",
        input.MethodBase, result.Exception.Message,
        DateTime.Now.ToLongTimeString()));
    }
    else
    {
      WriteLog(String.Format(
        "Method {0} returned {1} at {2}",
        input.MethodBase, result.ReturnValue,
        DateTime.Now.ToLongTimeString()));
    }
    
    return result;
  }

  public IEnumerable<Type> GetRequiredInterfaces()
  {
    return Type.EmptyTypes;
  }

  public bool WillExecute
  {
    get { return true; }
  }

  private void WriteLog(string message)
  {
    ...
  }
}
```

**IInterceptionBehavior**接口定义了三个方法：**WillExecute**, **GetRequiredInterface**以及**Invoke**。在大多数场景下，我们可以使用**WillExecute**与**GetRequiredInterface**的默认实现。**WillExecute**属性允许我们通过定义该行为是否应该执行来优化行为链; 在示例代码中，这个属性永远返回**true**，因此这个拦截行为将始终执行。**GetRequiredInterface**方法允许我们指定关联到改拦截行为的接口类型。在示例中，我们将在拦截器注册时指定接口类型，因此这个方法返回**Type.EmptyTypes**。

**Invoke**方法接受2个参数：**input**包含了调用的客户对象的方法名称和参数值的信息，**getNext**是一个委托，定义了拦截管道（或者代理对象）中要调用的下一个拦截行为。在示例中，我们可以看到该拦截行为如何在下一个拦截行为之前获取到客户对象调用的方法名，然后记录方法调用的详细信息。接着它又调用下一个拦截行为，并且记录其是否调用成功。最终，它将调用结果返回给管道中的前一个拦截行为。

以下的代码展示了更复杂的caching拦截行为。
```csharp
using Microsoft.Practices.Unity.InterceptionExtension;

class CachingInterceptionBehavior : IInterceptionBehavior
{
  public IMethodReturn Invoke(IMethodInvocation input,
    GetNextInterceptionBehaviorDelegate getNext)
  {
    
    // Before invoking the method on the original target.
    if (input.MethodBase.Name == "GetTenant")
    {
      var tenantName = input.Arguments["tenant"].ToString();
      if (IsInCache(tenantName))
      {
        return input.CreateMethodReturn(
          FetchFromCache(tenantName));
      }
    }

    IMethodReturn result = getNext()(input, getNext);

    // After invoking the method on the original target.
    if (input.MethodBase.Name == "SaveTenant")
    {
      AddToCache(input.Arguments["tenant"]);
    }

    return result;

  }

  public IEnumerable<Type> GetRequiredInterfaces()
  {
    return Type.EmptyTypes;
  }

  public bool WillExecute
  {
    get { return true; }
  }

  private bool IsInCache(string key) {...}

  private object FetchFromCache(string key) {...}

  private void AddToCache(object item) {...}
}
```

代码中可以看出，这个拦截行为过滤出名为**GetTenant**的方法，接着尝试从缓存中获取相关的tenant。假如它在缓存中找到对应的tenant，就没必要继续调用**GetTenant**这个方法来获取tenant，而是直接使用**CreateMethodReturn**这个方法从缓存中将tenant返回给管道中的前一个拦截行为。

同样地，该行为在调用了下一个拦截行为之后过滤出**SaveTenant**方法 ，并把tenant对象的副本放入了缓存中。
>这里我们简单地过滤出方法名称，后面我们将展示如何利用策略配置来聪明地做到这一点。

# 注册拦截器
在完成了**CachingIntercptionBehavior**，**LoggingInterceptionBehavior**之后，我们需要将其注册到Unity容器：
```csharp
using Microsoft.Practices.Unity.InterceptionExtension;

...

container.AddNewExtension<Interception>();
container.RegisterType<ITenantStore, TenantStore>(
  new Interceptor<InterfaceInterceptor>(),
  new InterceptionBehavior<LoggingInterceptionBehavior>(),
  new InterceptionBehavior<CachingInterceptionBehavior>());
```

**RegisterType**方法的第一个参数——Interceptor对象，定义了拦截器的类型，另外两个参数则注册了**TenantStore**的两个拦截行为。
>**InterfaceInterception**定义了这个拦截器基于代理对象。我们同样可以使用**TransparentProxyInterceptor**和**VirtualMethodInterception**这两个拦截器类型，之后再作详细介绍。

拦截行为的注册顺序将决定它们在管道中的执行顺序。在该示例中，我们一定不能把顺序弄错，因为当caching拦截行为在缓存中找到tenant后，就不把客户对象的调用请求传递给下一个拦截行为了，假如搞错了顺序且在缓存中找到了tenant，你就不会记录日志了。

# 使用拦截器
最后一步就是如何在运行时使用拦截器了，很简单，我们只需要调用相关方法就行了，logging和caching拦截将会自动附加上去:
```csharp
var tenantStore = container.Resolve<ITenantStore>();
tenantStore.SaveTenant(tenant);
```
要注意的是，上述**tanantStore**变量并非是**TenantStore**类型的，而是一个动态创建、实现了**ITenantStore**接口的**代理对象**，这个代理对象包含了**ITenantStore**接口定义的方法、属性和事件。上一篇文章的**实例拦截**介绍了这一点。

# 总结
这篇文章向大家介绍了Unity拦截器在项目中基础的实例拦截实战，下篇文章将给大家介绍其他更加灵活的拦截方式。
