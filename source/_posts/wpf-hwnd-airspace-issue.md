title: wpf经典空域问题
date: 2017-06-16 15:25:43
tags:
- WPF
categories:
- .NET
---

我们开发WPF应用时，有时不可避免得需要在其中承载WinForm控件。那么这时会出现一些问题，由于WPF与WinForm界面的渲染方式截然不同（我在[上一篇文章](http://neilq.github.io/2017/06/16/hosting-winform-control-in-wpf/)中有具体阐述），所以呈现方式也必然是分开的，也就造成WinForm控件必然在所有WPF控件上方置顶。假如我们这时使用了诸如flyout、modal、popup之类需要置顶的WPF控件，就会出现一种奇怪的现象，WinForm控件会将这类popup的部分界面遮住，非常尴尬。想想也是合理的，渲染方式不同，当然不可能"夹汉堡包"。

问题抛出来了，怎么解决呢？既然WinForm控件必然在当前WPF窗口中置顶，那么我们在当前窗口之上，再盖上一层透明的窗口，将popup之类的控件转移到上层窗口中，不就行了吗。原理说起来就是这么简单，具体实施还是挺复杂的，需要定位窗口位置、处理resize、适应dpi缩放等等，幸运的是轮子早有了，我这里直接列出来，需要的同学请自行google吧。

* [Microsoft.DwayneNeed](https://microsoftdwayneneed.codeplex.com/)
* [AirspaceFixer](https://github.com/chris84948/AirspaceFixer)