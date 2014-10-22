title: 在WCF Uri中使用特殊字符
date: 2014-10-22 19:48:56
tags:
---

当我们在iis中部署wcf时，我们会因为各种原因遇到无法在url的schema中使用一些特殊字符：
%,&,*,:,<,>,+,#, /, ?,\

即使通过uri encode, 这些特殊字符仍无法被当作普通的string，如果我们想通过url传递这些特殊字符，需要在web.config中进行如下配置：

``` xml
<httpRuntime requestPathInvalidCharacters="" requestValidationMode="2.0"/> 
<pages validateRequest="false"/>
```

通过上述配置，我们能通过现在能通过url传递（%,&,*,:,<,>）这几个特殊字符了。当然，我们也可以只允许其在某些字符，只要将其填入 “requestPathInvalidCharacters”即可。
剩下的+, ？, #，我们需要进行如下配置：

``` xml
<system.webServer> 
  <security> 
    <requestFiltering allowDoubleEscaping="true" /> 
  </security> 
</system.webServer>
```

这时我们就能在url中传递除\,/以外的特殊字符了，对于正反斜杠，目前还没有找到好的办法支持。
另外，支持这些特殊字符是一件危险的事，可能会引发一些安全问题，比如url注入等等，具体没有深入研究过。

值得注意的是，在iis8或7.5中，我们需要对url中的特殊字符encode 2 次，而在iis8.5中只需要encode一次，这是由于iis8.5会对自己对url encode一次，省去我们自己的一步encode。但我们可以在web.config中加入如下配置，让iis8.5向前兼容：

``` xml
<appSettings>
<add key="aspnet:UseLegacyRequestUrlGeneration" value="true" />
</appSettings>
```