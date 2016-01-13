title: "web api 自定义返回对象的字段"
date: 2016-01-13 09:56:31
tags:
- .NET
- Web Api
categories:
- .NET

---

由于前端的需求，我们需要自定义返回对象的字段，以减少不必要的网络开销。

假设我们有这样一个类
```csharp
public class Foo
{
    public string A { get; set; }
    public int B { get; set; }
    public bool C { get; set; }
    public Bar Bar { get; set; }
}

public class Bar
{
    public string F1 { get; set; }
    public string F2 { get; set; }
}
```
通常情况下，我们在get foo api中将会返回整个Foo对象，但有时前端的需求可能只需要返回Foo对象的A、B两个对象。针对此类需求，我们不可能为所有组合都写个api，因此动态化地返回对象字段成了必然要求。

那么，我们怎样去实现这一功能呢？诶对了，[ExpendoObject](https://msdn.microsoft.com/en-us/library/system.dynamic.expandoobject.aspx)和反射。

###代码实现
实现思路很简单，首先，我们需要调用者传来一个query string，比如"fields=a,b"，通过逗号分割字符串提取出自定义的字段。然后我们将其与Foo中反射得到的属性匹配，组成ExpandoObject对象，将其返回。下面展示一个实现代码：
```csharp
public class ExpandoMapper : IExpandoMapper
{
    public ExpandoObject Map(string fields, Type t, object entity)
    {
        var fieldHash = Utils.SplitCommaHash(fields);
        if (fieldHash == null || fieldHash.Count == 0)
        {
            return null;
        }

        var props = t.GetProperties();
        var comparable = new IgnoreCaseStringCompare();

        var expando = new ExpandoObject();
        var dictionary = (IDictionary<string, object>)expando;

        for (var i = 0; i < props.Length; i++)
        {
            if (fieldHash.Contains(props[i].Name, comparable))
            {
                dictionary.Add(props[i].Name, props[i].GetValue(entity));
            }
        }
        return expando;
    }

    public List<ExpandoObject> Map(string fields, Type t, IList<object> entity)
    {
        var fieldHash = Utils.SplitCommaHash(fields);
        if (fieldHash == null || fieldHash.Count == 0)
        {
            return null;
        }

        var props = t.GetProperties();
        var comparable = new IgnoreCaseStringCompare();

        var list = new List<ExpandoObject>();
        for (var i = 0; i < entity.Count; i++)
        {
            var expando = new ExpandoObject();
            var dictionary = (IDictionary<string, object>)expando;

            for (var j = 0; j < props.Length; j++)
            {
                if (fieldHash.Contains(props[i].Name, comparable))
                {
                    dictionary.Add(props[i].Name, props[i].GetValue(entity));
                }
            }
            list.Add(expando);
        }

        return list;
    }
}
```

由此，api我们可以这样写：
```csharp
public IHttpActionResult Get(string fields = "")
{
    var value = new Foo
    {
        A = "aaa",
        B = 2,
        C = true,
        Bar = new Bar
        {
            F1 = "f1",
            F2 = "f2"
        }
    };

    var mapper = new ExpandoMapper();
    var obj = mapper.Map(fields, typeof(Foo), value);

    return Ok(obj);
}
```
效果是这样的：
![custom fields](/img/custom_fields.png)


另外要注意的是，这里我们在api中直接声明**ExpandoMapper**对象，但我更建议大家将其接口化，注入到**controller**中，方便后期优化。

###性能优化想法
从上面的代码中可以看出，我们在每次映射对象时都对源类进行了属性反射，当请求量达到一定级别，必然会影响到性能。因此后期打算将类反射后的属性缓存起来，这样只需在启动时反射一次，将性能开销降到最低。
细心的朋友会发现，以上想法在实体对象不多的情况下是可行的。当项目中实体对象量大时，过多的缓存又会提高内存占用。假如达到某个数量级别，我的想法是使用[LRU cache(最近最少使用缓存)策略](https://en.wikipedia.org/wiki/Cache_algorithms)来优化。
当然，**抛开剂量谈伤害都是耍流氓**，今天暂且研究到这里，把时间放在更重要的地方，假如后期有这方面的代码实现，我再来更新。
