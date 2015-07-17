title: web api 2 - 自定义help page的返回类型 
date: 2015-07-17 11:16:42
tags:
- .NET
- Web Api
- Help Pages
categories:
- .NET
---

当我们用HttpMessageResponse或者IHttpActionResult作为返回api的返回类型时，帮助页面的返回类型相应得也变成了HttpMessageResponse或者IHttpActionResult,而这不是我们所期望的Product、User等实际的返回实体。幸运的是，这是可自定义的, 简单得我都不好意思讲。

## 方法一（推荐）
为方法添加一个属性[ResponseType(typeof(ProductDto))]就OK了。

## 方法二
在HelpPageConfig.Register方法里添加config.SetActualResponseType(typeof (ProductDto), "Products", "Get");
但这种方法只能改变返回类型的示例（Response Format），而其详细描述（Response Description）并没有改变。

这两种方法都是局部的改变，也就是针对每个controller、action都要配置，实际上我们也可以全局动态地去控制它，但时间原因，暂且只记下这俩方法，有兴趣的同学可以自行研究。
