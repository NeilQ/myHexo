title: 如何让测试脱离vs跑自动化测试 
date: 2015-07-07 14:32:50
tags:
- 测试
- 自动化测试
categories:
- 测试
---

为什么要脱离vs跑自动化测试？这本就是个矛盾的问题，负责自动化测试的Test必须要有阅读、审查、修改代码的能力，才能保证测试结果的有效性。但实际情况中测试人员普遍没有这种能力，有代码能力的测试要么在大公司，要么都转开发去了（吐槽）。在这种情况下，让测试去跑自动化，就如同一个不识字的少年捡到一本九阴真经秘籍，却因不识字而毫无用处。那么我们需要做的就是让秘籍变成图谱。

Visual Statio是开发工具的集合，很多功能都是调用了不同的组件，包括它的Unit Test Runner。vs2010使用了"MSTest.exe"工具，而vs2012以上版本使用的是"VSTest.Console.exe"工具作为更高效的替代。这里我使用了MSTest，因为它的参数更易用，并且有更好的输出定制。

到这里，用Console Application还是Winform做壳已经不重要了，甚至你可以写个批处理。不管用什么，我们只需要去调用MSTest，输出结果，展示给我们的测试就行了。关于具体调用参数，请参考[MSTest.exe命令行选项](https://msdn.microsoft.com/zh-cn/cn/library/ms182489.aspx)。这里贴一份调用MSTest的C#代码

```csharp
   public static class MSTestRunner
    {
        public static string Execute(string dosCommand)
        {
            return Execute(dosCommand, 10);
        }

        public static string Execute(string command, int seconds)
        {
            var sb = new StringBuilder();
            if (command != null && !command.Equals(""))
            {
                // 创建MSTest.exe进程
                var process = new Process();
                var startInfo = new ProcessStartInfo
                {
                    FileName = "MStest.exe",
                    Arguments = command,
                    UseShellExecute = false,
                    RedirectStandardInput = true,
                    RedirectStandardOutput = true,
                    CreateNoWindow = false
                };
                process.StartInfo = startInfo;
                try
                {
                    //开始进程
                    if (process.Start())
                    {
                        process.OutputDataReceived += (s, e) =>
                        {
                            //将输出保存到StringBuilder,并输出到Console
                            sb.AppendLine(e.Data);
                            Console.WriteLine(e.Data);
                        };
                        process.BeginOutputReadLine();

                        if (seconds == 0)
                        {
                            //无限等待进程结束
                            process.WaitForExit();
                        }
                        else
                        {
                            //等待进程结束,等待时间为指定的毫秒
                            process.WaitForExit(seconds); 
                        }
                    }
                }
                finally
                {
                    process.Close();
                }
            }
            return sb.ToString();
        }
    }
```

工作到这里并没有结束，我们找到了跑Unit Test的方法，并且写了个简单的UI，但是我们的输出结果并不能给测试带来信任感，因为测试不看代码，即使我们输出类似：“”通过：15，失败：1 ”的最终结果，也不一定能让人信服，说不定你的TestMethod所验证的内容就与测试所期望的不符，也就是需求理解差异。因此我们需要在TestMethod里输出详细的行为日志，并保存日志。
这里我们关注几个MSTest的命令选项：
- /detail:errormessage 输出错误信息 
- /detail:description 输出TestMethod DescriptionAttribute
- /detail:errorstacktrace 输出错误堆栈
- /detail:stdout 输出标准output，在这里我们可以用Console.WriteLine(string text)来自定义输出内容

分享下我的调用代码以及运行截图:
```csharp
        static void Main(string[] args)
        {
            string command = ConfigurationManager.AppSettings["args"];
            // command is "/testcontainer:KD.Automation.dll /category:FunctionTest /detail:errormessage /detail:description /detail:errorstacktrace /detail:stdout"

            var output = MSTestRunner.Execute(command, 0);

            Console.WriteLine("Done.");

            // 保存日志文件 
            if (!Directory.Exists("./TestLogs"))
            {
                Directory.CreateDirectory("./TestLogs");
            }

            var filename = DateTime.Now.ToString("yyyy-MM-dd HH_mm_ss");
            File.WriteAllText(String.Format(@".\TestLogs\{0}.txt",filename), output);
            Console.WriteLine("按任意键退出...");
            Console.ReadKey();
        }
```

![unittestrunner](/img/unittestrunner.png)

