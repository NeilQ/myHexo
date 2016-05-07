title: "如何写好单元测试"
date: 2016-05-07 15:34:40
tags:
- Unit Test
catagories:
- Test

---
我们都知道为模块编写单元测试有很多好处，比如保证软件健壮，建立团队信心，快速定位bug等等，但是许多程序员并不知道怎样写出一个规范的单元测试。事实上，一个无效的单元测试并不能带来如上所属优点。

这里，我想跟大家讨论下怎么写好一个单元测试。我们不扯如何测试，大家可以找一些书看；也不聊具体的测试技术，比如mock和stub，我们只讨论在代码层面上，应该写些什么东西，才能成为一个有效的单元测试。

首先我们从来看一个案例，假设我们有个**User**类，定义了**GetUser(int id)**方法。现在我们来通过单元测试来验证这个方法的正确性，程序员小A是这样写的：
```csharp
public void TestGetUser()
{
    var userService = new UserService();
    var user  = userService.GetUser(1);
}
```
为了跑通这个测试方法，小A需要在数据库创建一个id为1的User，然后进入调试状态，断点打在最后一行，然后看看user变量的值是否是所期望的。这种方式导致这个测试方法成了半自动的，脱离了人工干预，这个测试根本跑不起来，无法重复运行、回归测试，也就无法保证代码的健壮性。那么，我们在测试方法体中到底应该怎么写？

目前，我们的单元测试基本都是遵照AAA规则：Arrange, Act, Assert，即准备，行动，断言。

### Arrange
在进行上述用例测试时，首先得有个id为1的用户，我们必须准备好这个数据用于测试，而这一准备的动作也必须写在代码中。

### Act
接着，才是执行我们需要测试的方法，即**GetUser**。

### Assert
GetUser执行完成后，测试并没有结束，我们需要验证得到的user是否是我们所期望的，数据是否正确，所有的字段是否都得到了，等等。

综上，我们对**GetUser**这个方法的测试体应该这样写：
```csharp
public void TestGetUser()
{
    // arrange
    var userService = new UserService();
    var id = userService.Add(new User{Name="bob", Age=18});
    
    // act
    var user  = userService.GetUser(id);
    
    // assert
    Assert.IsNotNull(user);
    Assert.AreEqual("bob", user.Name);
    Assert.AreEqual(18, user.Age);
}
```
实际上，很多程序员在写单元测试时，会进行前两步，却根本没有断言验证，这是需要注意的地方。

完。
