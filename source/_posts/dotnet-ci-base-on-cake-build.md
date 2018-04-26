title: 基于cake-build的dotnet自动化发布
date: 2018-04-18 17:22:54
tags:
- cake-build
- dotnetcore
- CI
categories:
- .NET
---

最近公司项目开发完成，准备进行给客户部署试用，因此为项目的发布、打包耗费了心神。我们的系统包含三大项目：Restful Api、web管理站点以及一个桌面客户端，那么在发布新版本时，必须手动release三个项目，并且更改3个配置文件，然后复制到某个目录中。操作简单，但是一旦发生频繁的发布，不可避免得会出现疲劳、抗拒等等负面情绪...因此自动化这一工作成了刚需。

由于公司的资源限制以及团队规模较小，没有部署gitlab等支持ci的系统，哪怕有轻量级的系统，我也不可能在现阶段花一天两天去做这件事，我需要的是最快的脚本化的任务处理，因此我找到了cake-build。

这里我唠叨下是怎么找到它的。最近一段时间我在阅读github上各种dotnet项目源码时，发现很多项目中都会有build.ps1或者build.sh文件，深究之下，发现它来自cake-build，并逐渐发现它的美。熟悉我的人都知道，我不止一次向周围人建议多去读开源项目源码，不管是面对面，朋友圈，还是博客。正是由于我从中不止一次得到各种各样惊喜，我才会这样去做。正如cake-build，随着dotnetcore的深入发展，社区越来越活跃，这一构建工具使用得越来越频繁，如果你不去自己发现，而是等一篇又一篇国人的翻译文章过来，再去炒冷饭，你就落后了。

回到主题，cake-build是一个由dotnet实现实现的自动化构建脚本工具，可以利用它来进行项目的自动化编译、测试、发布、部署等工作。熟悉前端工程的同学一定知道gulp与grunt，那么cake-build就是类似于gulp的脚本构建工具，但是它更适应于dotnet平台。同时，它对常用的构建操作进行了大量的封装，使我们在调用git，msbuild，io操作等都更加便捷，更加轻量。甚至不仅于此，它还提供了npm、android、db、email等等其他平台一眼望不到头的大量插件。

# 场景实践
对cake-build，我就介绍这么多，关于一些安装、使用方式，请大家自行搜索，官方文档已经非常完整了。下面通过我的一个简单工作场景，来实践它的使用价值。

如开头所描述，我将使用它来解决我的自动化编译、发布问题。我设定了如下的任务流程：

- EnsureRepos: 将git服务器的项目代码拉下来，保存到git-repos文件夹中。
- clean: 清空release文件夹。
- publish：编译发布，并将一些配置文件替换掉

三个流程去掉项目配置文件，总计2个文件，其中一个是生成的，另一个包含130多行代码，耗时两小时，替代了无数个手动完成的十几分钟的工作。Code here:

```csharp
var target = Argument("target", "pack");
#addin nuget:?package=Cake.Git

Task("ensureRepos")
  .Does(()=>
  {
    Information("Downloading source codes...");
    // check repo folders
    EnsureDirectoryExists("git-repos");
    var username = "gituser";
    var password = "gitpassword";

    if(!DirectoryExists("git-repos/admin"))
    {
      CreateDirectory("git-repos/admin");
      Information("Clone admin...");
      GitClone("http://git.domain.com/xxx/admin.git", 
                "git-repos/admin", 
                username, 
                password);
    }
    else
    {
      GitPull("git-repos/admin","cake-ci","email",username,password,"origin");
    }

    if(!DirectoryExists("git-repos/api"))
    {
      CreateDirectory("git-repos/api");
      GitClone("http://git.domain.com/xxx/api.git", 
                "git-repos/api", 
                username, 
                password);
      Information("Clone api...");
    }
    else
    {
      GitPull("git-repos/api","cake-ci","email",username,password,"origin");
    }

    if(!DirectoryExists("git-repos/watch"))
    {
      CreateDirectory("git-repos/watch");
      GitClone("http://git.domain.com/xxx/watch.git", 
                "git-repos/watch", 
                username, 
                password);
      Information("Clone watch...");
    }
    else
    {
      GitPull("git-repos/watch","cake-ci","email",username,password,"origin");
    }
  });

Task("clean")
  .Does(()=>
  {
    if(DirectoryExists("release"))
    {
      CleanDirectories("release");
    }
  });

Task("publish-api")
 .Does(()=>
  {
    EnsureDirectoryExists("release");
    var settingsApi = new DotNetCorePublishSettings
    {
        Framework = "netcoreapp2.0",
        Configuration = "Release",
        OutputDirectory = "./release/api"
    };
    DotNetCorePublish("./git-repos/api/Namespace.Api",settingsApi);
  });

Task("publish-admin")
 .Does(()=>
  {
    EnsureDirectoryExists("release");
    var settings = new DotNetCorePublishSettings
    {
        Framework = "netcoreapp2.0",
        Configuration = "Release",
        OutputDirectory = "./release/admin"
    };
    DotNetCorePublish("./git-repos/admin/backend/Namespace.Admin",settings);
  });

Task("publish-watch")
  .Does(()=>
  {
    EnsureDirectoryExists("release");

    var solutions = GetFiles("./git-repos/watch/*.sln");
    foreach(var solution in solutions)
    {
        Information("Restoring {0}", solution);
        NuGetRestore(solution);
    }

    MSBuild("./git-repos/watch/Namespace.WatchClient", new MSBuildSettings {
    Verbosity = Verbosity.Minimal,
    ToolVersion = MSBuildToolVersion.VS2017,
    Configuration = "Release",
    ArgumentCustomization = args=>args.Append("/p:PreBuildEvent= /p:PostBuildEvent="),
    PlatformTarget = PlatformTarget.MSIL
    });
    CopyDirectory("./git-repos/watch/CameraSDK", "./git-repos/watch/Namespace.WatchClient/bin/Release/CameraSDK");
    CopyDirectory("./git-repos/watch/Namespace.WatchClient/bin/Release", "./release/client");
  });

Task("publish")
  .IsDependentOn("publish-api")
  .IsDependentOn("publish-admin")
  .IsDependentOn("publish-watch")
  .Does(()=>
  {
     CopyDirectory("./replace/admin", "./release/admin");
     CopyDirectory("./replace/api", "./release/api");
     CopyDirectory("./replace/client", "./release/client");
  });


Task("pack")
  .IsDependentOn("ensureRepos")
  .IsDependentOn("clean")
  .IsDependentOn("publish")
  .Does(() =>
  {

  });

RunTarget(target);

```

最后，有同学要问了，同样的工作，我写一个控制台程序也可以做啊。是的，可以实现，但是你得花更多的时间。写一个app，msbuild的命令你得去研究研究吧？git相关的package你得去找找吧？如果你引入了测试，环境变量设置等等一些其他操作，一个简单的工作你就可以当项目去做了。脚本化的工具库，其优势就是利用最少的代码，最少的时间去完成尽可能多的工作，这也是Python能成为人工智能热门语言的最大原因。

# 后期构想
目前我实现了3个项目的编译、发布和配置替换，今后打算按需针对自动化测试、版本号更改、打包、部署等再进行扩展，如果有条件，可以无缝接入gitlab ci等系统。
