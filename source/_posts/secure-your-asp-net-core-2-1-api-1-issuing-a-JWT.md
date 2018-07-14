title: ASP.NET core 2.1 API 认证与授权(1 - 使用JWT)
date: 2018-07-14 14:06:14
tags:
- dotnetcore
categories:
- dotnetcore
---

最近在为API重构认证与授权机制。之前在做api认证时，使用的是自己定义的Bearer token，不是很规范，因此决定使用JWT进行重构，记录一下相关知识点。

## Authentication和Authorization

首先，我们需要理清一下认证（Authentication）与授权（Authorization）的关系，否则在理解与查找asp.net core认证授权资料时会非常困惑，下面我会统一使用英文Authentication与Authorization，便于理解代码。

* Authentication（认证）是指知道用户的标识。例如佩奇使用她的用户名与密码登录，或者佩奇访问API每次都带着一个加密的token字符串，里面包含着她的个人信息，验证这个信息的过程就叫做Authentication。
* Authorization（授权）是指确定是否允许用户执行某个操作。例如佩奇可能允许查询某个数据，但不能更改这个数据，验证这个权限的过程叫做Authorization。

在asp.net core 2.1 中，Authentication发生在`Middleware`层，而Authorization发生在`Filter`层（可以获得Controller的信息，这是配置权限的关键）。

Authentication包括以下几种：Basic，Bearer，Digest等等，这里不详细介绍，而我们要讨论的就是Api常用的Bearer认证，默认的，ASP.NET core将会验证http的头信息Authorization，如果值以`Bearer`开头，那么就是Bearer认证了。

## JWT
`JWT(JSON Web Token)`是一种用来传输JSON对象的安全标准，例如
```json
{ 
    "name": "Jon",
    "middle_name": "Who's asking?!" 
}
```

这里不详细介绍JWT标准，有兴趣的朋友自行Google。


`Claims`是包裹了用户信息的声明，包含了你的用户标识，这些标识决定了你是否允许访问Api，具有哪些操作权限。在上面的例子中，`name`与`middle_name`就是Claims声明。

那么，我们是怎么利用JWT来做API认证的呢？分为以下步骤：

- 用户佩奇访问客户端站点，跳转到登录界面
- 佩奇提交了她的用户名与密码作为凭证
- 客户端站点将凭证提交到Api
- Api验证用户名与密码，给客户端返回一个JWT token

一旦客户端获得了token，它每次访问Api时都将在头信息里附带token，当Api接受到请求时，验证token是否合法，如果不合法则返回`401`，客户端收到后跳转到登录页面，如果合法则返回相应的数据。

## 如何生成JWT

如果你使用了`Identity Server 4`，那么你可以很简单得生成token，但这里我们要介绍的是不使用第三方解决方案时，怎么去操作。

首先，我们需要一个controller action来发放token：
```csharp
[AllowAnonymous]
[HttpPost]
public IActionResult RequestToken([FromBody] TokenRequest request)
{
    if (request.Username == "Jon" && request.Password == "Not for production use, DEMO ONLY!")
    {
        var claims = new[]
        {
            new Claim(ClaimTypes.Name, request.Username)
        };

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_configuration["SecurityKey"]));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: "yourdomain.com",
            audience: "yourdomain.com",
            claims: claims,
            expires: DateTime.Now.AddMinutes(30),
            signingCredentials: creds);

        return Ok(new
        {
            token = new JwtSecurityTokenHandler().WriteToken(token)
        });
    }

    return BadRequest("Could not verify username and password");
}
```
代码很简单，通过用户名与密码获得用户信息，然后指定密钥与加密方式，通过内置的方法生成token，其中
- `issuer`表示token 的发放者。
- `audience`表示接收者，如果你有多个客户端访问Api，可以利用这个字段区分不同的客户端。
- `expires`表示token的过期时间，防止token被盗后被恶意使用，过期后需要重新申请。
- `claims`包含了用户的标识

## 你有了JWT，接下来呢？

现在你的客户端已经有了token，并且每次访问都会包含在头信息里，`Authorization: Bearer myToken`。

但你的Api仍然是面向所有人的，我们需要做一些工作，来使Api收到请求后自动验证token。


### 为Api controller添加认证需求

我们需要用`[Authorize]`属性使得api要求用户认证：
```csharp
[Authorize]
[Route("api")]
public class ApiController : Controller
{
    [HttpGet("Test")]
    public IActionResult Test()
    {
        return Ok("Super secret content, I hope you've got clearance for this...");
    }

    // rest of controller goes here
}
```

加上这个属性后，每次api收到请求都会进行用户认证，如果没有token，就会返回401。

### JWT认证配置

接着，我们需要在Startup中启用JWT认证，并配置一些信息。

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }

    app.UseAuthentication();
	
    app.UseMvc();
    app.UseStaticFiles();
	
}
```

以上代码表示我们已经启用了用户认证，但没有指定如何认证，需要将JWT认证模式加进去：


```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
        .AddJwtBearer(options =>
        {
            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuer = true,
                ValidateAudience = true,
                ValidateLifetime = true,
                ValidateIssuerSigningKey = true,
                ValidIssuer = "yourdomain.com",
                ValidAudience = "yourdomain.com",
                IssuerSigningKey = new SymmetricSecurityKey(
                    Encoding.UTF8.GetBytes(_configuration["SecurityKey"]))
            };
        });

    services.AddMvc();
}
```

至此，我们已经完成的JWT认证的启用。

### JWT认证过程原理

可能有些同学会想知道ASP.NET core是如何进行认证过程的，为什么配置完这些内容就完成认证了，有兴趣同学可以阅读一下[Microsofr.AspNetCore.Authentication的源码](https://github.com/aspnet/Security/tree/master/src)。

通过代码我们可以知道，这里有一个`JwtBearerHandler`类，用来专门处理JWT认证，它将提取Request的Authorization头信息，验证值是否以`Bearer`开头，然后获得具体的token，对token进行解密，最后验证是否合法，如果合法，将提取的用户信息塞到HttpContext.User中，我们可以注入`HttpContextAccessor`，这样在任何地方就可以通过依赖获得HttpContext.User，得到用户的一切信息。

那么，ASP.NET core是什么时候将认证交由`JwtBearerHandler`处理的呢？文章的开头我们提到Authentication是在`Middleware`层进行处理，这里就有一个内置的`AuthenticationMiddleware`。当收到请求后，它找出你所配置的所有认证方式，如果你的`[Authorize]`属性指定了具体的方式，如`[Authorize(AuthenticationSchema='Bearer')]`，它会找到指定的`AuthenticationHandler`，交由它处理，如果你没有指定，它将交由默认的`AuthenticationHandler`处理，以上的例子，我们已经配置了默认的认证方式，` services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)`，其中`JwtBearerDefaults.AuthenticationScheme`就是我们配置的默认认证方式，它的值就是`Bearer`。

## 总结
现在，我们已经知道了如何生成以及配置JWT认证使其工作，那么如果我们的Api需要多种认证方式该怎么办呢？下一篇文章我们将具体讨论如果自定义`AuthenticationHandler`以及如何混合多种认证方式。