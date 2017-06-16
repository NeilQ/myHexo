title: 解决获取WPF控件句柄问题：在WPF中承载WinForm控件
date: 2017-06-16 14:57:21
tags:
- WPF
categories:
- .NET
---

基于需求，我需要获取WPF的某控件的句柄。

搜索过后得到一些关于`HwndSource`的代码示例，然而通过尝试发现，此类代码获取的只是Window的句柄，而我需要的是UserControl的句柄。事实上，**获取WPF UserControl的句柄是根本不可能的。**大家应该都知道，要获取WinForm控件的句柄相当简单，是因为WinForm是通过GDI+来渲染界面，每个控件都拥有自己的句柄，但是WPF的界面是由DirectX渲染，除了Window本身，其他部分都没有句柄。那么既然Winform控件有句柄，WPF控件没有句柄，那么我们能不能在WPF中承载WinForm控件呢？

显然是可以的。很简单，首先引用`WindowsFormsIntegration`、`System.Windows.Forms`两个程序集，接着在Window元素中添加命名空间`xmlns:wf="clr-namespace:System.Windows.Formsassembly=System.Windows.Forms"`，最后就可以在xmal中布局winform控件了

```xml
<Grid>
  <WindowsFormsHost>
    <wf:TextBox x:Name="tbName" Text="Hello"/>
  </WindowsFormsHost>
</Grid>
```


