title: .NET性能之遍历List<T>
date: 2015-02-14 17:15:15
tags:
---


最近对.NET项目进行性能优化，在开启编译器优化的前提下，对List<T>对象遍历的不同方式进行了简单的研究和对比，以此记录。
首先我们创建两个不同类型的List对象，各自塞了5000000个简单元素：
```c#
var listInt = new List<int>();
var listString = new List<String>();
for (var i = 0; i < 5000000; i++)
{
	listInt.Add(1);
	listString.Add("1");
}
```
在此我们分别通过foreach, for-loop, 和List<T>.ForEach方法对他们分别进行遍历:
```c#
CodeTimer.Time("foreach int", 10, () =>
{
    foreach (var item in listInt)
    {
    }
});

CodeTimer.Time("List.Foreach int", 10, () =>
{
    listInt.ForEach(s =>
    {
    });
});

CodeTimer.Time("for loop int", 10, () =>
{
    for (int i = 0; i < listInt.Count; i++)
    {
    }
});

CodeTimer.Time("foreach string", 10, () =>
{
    foreach (var item in listString)
    {
    }
});

CodeTimer.Time("List.Foreach string", 10, () =>
{
    listString.ForEach(s =>
    {
    });
});

CodeTimer.Time("for loop string", 10, () =>
{
    for (int i = 0; i < listString.Count; i++)
    {
    }
});

```
运行结果如图：
![foreach](/img/foreach.png)

从结果中很容易得出**结论**，三种遍历，无论是对值类型还是应用类型，都是for-loop更高效。

## 原因

至于原因，我们先来看看List<T>对foreach迭代的实现：
> * 调用GetEnumerator并在托管堆上创建一个Enumerator实例，将会引起垃圾回收。
> * 每次迭代时调用Enimerator的MoveNext方法
> * 每次迭代时给Current成员赋值

接着我们看看List<T>.ForEach的源码：
```c#
public void ForEach(Action<T> action) {
    if( action == null) {
        ThrowHelper.ThrowArgumentNullException(ExceptionArgument.match);
    }
    Contract.EndContractBlock();

    int version = _version;

    for(int i = 0 ; i < _size; i++) {
        if (version != _version && BinaryCompatibility.TargetsAtLeast_Desktop_V4_5) {
            break;
        }
        action(_items[i]);
    }

    if (version != _version && BinaryCompatibility.TargetsAtLeast_Desktop_V4_5)
        ThrowHelper.ThrowInvalidOperationException(ExceptionResource.InvalidOperation_EnumFailedVersion);
}
```
不难发现ForEach方法也是对for-loop的封装，只不过每次都要执行一次Action委托，比for-loop慢是肯定的。

最后，如果大家追求性能的话，建议尽量使用for-loop对List<T>遍历。

