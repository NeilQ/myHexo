title: Asp.net core 2 如何设置SnakeCase形式的QueryString参数
date: 2019-01-11 22:47:57
tags:
- dotnetcore
categories:
- dotnetcore
---

我在项目中设计Web Api规范时，决定将QueryString参数命名使用SnakeCase的形式，这就带来一个问题，SnakeCase与C#的通用参数命名格式有所冲突，举个例子：

```csharp
public class SearchQuery
{
   public int Value_type { get; set;}
}

public class ValueController: Controller
{
    public IActionResult Get([FromQuery("value_type")] int valueType)
    {
        return Ok();
    }

    public IActionResult GetAnother([FromQuery] int value_type)
    {
        return Ok();
    }

    public IActionResult GetNext(SearchQuery query)
    {
        return Ok();
    }
}
```

如上所示，通常我们的Property命名使用PascalCase格式，方法参数命名使用CamelCase格式，如果你也有代码洁癖，在Property中间加一个下划线就显得有些怪异。今天我们就聊一聊如何从全局上解决这种冲突。

首先我们确定目标，要解决这个问题，要改变两个地方，一是Controller的ModelBinding需要将字段自动转变为SnakeCase，二是我们的Swagger文档生成Api描述时，也要将字段转变为SnakeCase。

# ModelBinding

通过官方文档，我们发现ModelBinding获取参数值时，有一个可扩展的适配器：`IValueProvider`，比如默认的`QueryStringValueProvider`，我们要做的就是创建一个自定义的适配器来自动适配SnakeCase。

```csharp
    public class SnakeCaseQueryValueProvider : QueryStringValueProvider
    {
        public SnakeCaseQueryValueProvider(
            BindingSource bindingSource,
            IQueryCollection values,
            CultureInfo culture)
            : base(bindingSource, values, culture)
        {
        }

        public override bool ContainsPrefix(string prefix)
        {
            return base.ContainsPrefix(prefix.ToSnakeCase());
        }

        public override ValueProviderResult GetValue(string key)
        {
            return base.GetValue(key.ToSnakeCase());
        }
    }

    public class SnakeCaseQueryValueProviderFactory : IValueProviderFactory
    {
        public Task CreateValueProviderAsync(ValueProviderFactoryContext context)
        {
            if (context == null)
            {
                throw new ArgumentNullException(nameof(context));
            }

            var valueProvider = new SnakeCaseQueryValueProvider(
                BindingSource.Query,
                context.ActionContext.HttpContext.Request.Query,
                CultureInfo.CurrentCulture);

            context.ValueProviders.Add(valueProvider);

            return Task.CompletedTask;
        }
    }
```

如上所示，我们在获取QueryString参数值时，将参数类中的字段名转换成SnakeCase格式，非常简单。其中`ToSnakeCase()`方法大家随便Google一下就有一大推，这里不再赘述了。接着我们将其这个自定义适配器加入到MVC的适配器列表中：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc(ops =>
        {
            ops.ValueProviderFactories.Add(new SnakeCaseQueryValueProviderFactory());
            //ops.ValueProviderFactories.Insert(0, new SnakeCaseQueryValueProviderFactory());
        })
        .SetCompatibilityVersion(CompatibilityVersion.Version_2_1);
}
```

通过研究Asp.net core的源码，将会发现ModelBinding上下文有一个`IValueProvider`列表，每次进行模型绑定获取参数值时，都会遍历其中的是适配器列表，与具体的参数名称进行对比从而获取参数值。因此如果大家想要提升一下自定义适配器的优先级，可以使用`ops.ValueProviderFactories.Insert(0, new SnakeCaseQueryValueProviderFactory())`方法，将其放到列表的第一个。实际上框架中自带一些默认的`IValueProvider`，如`QueryStringValueProvider`，`FormValueProvider`、`RouteValueProviderFactory`、`JQueryFormValueProviderFactory`，如果大家觉得作为一个RestfulApi应用，像`JQueryFormValueProviderFactory`, `FormValueProvider`这种东西不需要，我们也可以移除掉略微提升一些性能，当然，如果QPS达不到一定量几乎感觉不到性能提升。

# Swagger文档

此时我们已经改变了QuerString值检索的行为，但是应用的Swagger文档描述的参数名仍然是PascalCase的，我们还需要改一下Api描述相关的适配器，因此我们找到了`IApiDescriptionProvider`扩展。同样的，创建一个自定义适配器，接着将其注入到IOC容器中：

```csharp
public class SnakeCaseQueryParametersApiDescriptionProvider : IApiDescriptionProvider
{
    public int Order => 1;

    public void OnProvidersExecuted(ApiDescriptionProviderContext context)
    {
    }

    public void OnProvidersExecuting(ApiDescriptionProviderContext context)
    {
        foreach (var parameter in context.Results.SelectMany(x => x.ParameterDescriptions).Where(x => x.Source.Id == "Query" || x.Source.Id == "ModelBinding"))
        {
            parameter.Name = parameter.Name.ToSnakeCase();
        }
    }
}

public void ConfigureServices(IServiceCollection services)
{
    services.TryAddEnumerable(ServiceDescriptor
         .Transient<IApiDescriptionProvider, SnakeCaseQueryParametersApiDescriptionProvider>());
}
```

逻辑很简单，Api描述上下文中的参数描述类中有个BindSource类型的字段，通过这个`Source`字段可以判断这个参数的类型，它包括`Body`、`Form`、`Query`、`ModelBinding`等等类型。这里有一点需要注意的是，官方文档中对`ModelBinding`的定义是：`A BindingSource for model binding. Includes form-data, query-string and route data from the request.`。在RestfulApi场景中，`form-data`肯定是用不到，因此只需注意一下`route data`就行了，这是另一个话题，这里就不深究了。

