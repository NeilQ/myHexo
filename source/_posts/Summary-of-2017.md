title: 2017年码上那些事
date: 2017-12-30 14:18:38
tags:
categories:
- 随笔
---

2017年将要过去，截至元旦，最后一位90后也已经成年了，而我，也由一个翩翩少年，逐渐成为更加成熟的青年，就酱。伤感完毕，还得是一年一度的年终总结，行己勤劝须自省。

# 新的工作
今年年初，我以一份新的工作开始新的一年，现在回顾，感觉非常满意。工作内容上，得到了一定的决策权，包括进度管理、项目管理、技术框架的抉择，也通过与其他各部门的工作交流，在沟通能力与技巧上得到了一定的提升。待遇上，老板对我很宽容，不打卡，偶尔请假不扣工资，在家工作只需打个招呼，这也使我对公司感恩，对工作充满热情、工作积极性很高。我认为这是一种良性循环，当然，作为打工者，这也是一份可遇不可求的机会，可以说是比较完美的状态了。

# 工作成果
今年独自完成了一个停车场管理系统，包括前端，Api后台，与一个桌面收费客户端。其间还开发了一个相机的代理服务，但由于必要性不足，以及过高的系统复杂性分散了我的精力，暂时将其挂起，计划18年重新启动，用来优化系统的实时性。

系统技术栈使用Angular5 + Dotnet core 2.0 + Posgresql + WPF。其开发过程中，也与摄像机、显示屏等硬件设备进行交互，使我对嵌入式开发及硬件行业的现状也有了更深入的了解。

# 最满意的代码
工作中，也产生了一些令自己满意的代码，每次看到，都能让我暗自咋舌，感叹良久，真是破费科特啊。作为一名普通开发者，写不出神经网络算法，写不出火热的人工智能学习算法，能有一些自己的小幸福，也是挺不错的。

## 计费算法
停车产的计费算法，算是今年碰到的最复杂的业务算法。4种计费模式，使用了教科书般的策略模式，很多人都怀疑设计模式的必要性，但在这种情况下，我无法想象不使用设计模式将会如何糟糕。如果用函数式编程的思想来实现，我想一定是个上千行又臭又长的裹脚布，各种各样的业务规则混杂在一起，阅读和维护都将是一种灾难。

## 资源调度算法
在开发桌面客户端时，碰到了一些资源调度上的问题。起初是为了节省系统资源开销，减少对象创建，提高一些视图的复杂用户控件的可重用性，利用队列数据结构将一些控件进行循环使用。

接着，在另一个需求中，遇到了相似但更复杂的情况。显示屏在空闲状态下循环显示内容播放，但遇到事件触发，需要将循环任务挂起，优先显示事件内容，一段时间后再恢复循环任务。一个相对简单的任务调度，当我发现它具有多次重用的价值后，将其封装成可重用的组件，自己比较满意。

# 新技术的学习

## Dotnet core 2.0 
今年8月份，微软发布了dotnet core 2.0正式版，本来我对此一直持观望态度，借此机会，毅然将原本基于framework的asp.net web api框架迁移了过来，并将原本用于前端的nodejs web项目也迁移了过来，出乎想象得顺利，花费了一个月不到的时间。

在使用过程中发现，asp.net core在思想上与开源界主流的框架保持了高度的一致性，都属于组件化的轻量级web框架，与nodejs express，python flask，以及golang revel等在使用上都有很高的相似性。但同时，也保持了面向对象的优势，比如内置的依赖注入。

## Docker
双十一期间，公司入了阿里云服务，此时正好刚刚迁移了dotnet core，我也大胆得尝试利用docker来进行系统部署，过程也比较顺利。同时发现目前docker的生态也比较完善，简单的配置即可部署集群，对未来的业务扩大也有很大的便捷性。18年在稳定系统之后，计划集成CI，实现系统一键部署更新。

## 操作系统基础知识的丰富
在开发桌面客户端的过程中，也使我对操作系统资源管理有了新的认识，也踩过不少雷区，比如传说中的内存泄露。之前工作接触的一直是web开发，web框架对系统资源做了封装，任何线程、内存管理都不需要亲自处理，而在桌面开发中，这些都需要注意。在开发过程，遇到了后台线程没有终止导致内存无法释放，释放对象时没有释放非托管资源，造成内存泄露等等问题，给我造成了很大困扰，也使我得到了极大的提升。

# 工作之外
17年发生了很多事，让我对社会、对工作、对生活有了新的理解，令我恍然大悟的是，作为历史进程中的一股卑微的小水流，没有任何事比身体健康、身心快乐更重要，今后，任何违背这一原则的事物我都将坚决抵抗，做佛系青年中的武僧。很多生活的感悟简单到一句话，但背后都是血的教训，有时候，我宁愿保持单纯，也不愿意去获得这样一份成熟。

今年，我养成了阅读的习惯，截至今日，看完了史书《全球通史》，小说《白鹿原》，《黑客与画家》，杂文《年少荒唐》，哲学书《毛泽东选集卷一》，小半本《心理学与生活》，感悟颇多。3月份，我开始了系统的健身，完成了一次半程马拉松，如今，体重增长了6kg，肌肉、脂肪参半，100斤的大米可以说是随手就拎走了。参加运动，保持健康，能量充沛，充满自信，是我收益最高的一笔投资。

# 总结
总体回顾，我对2017年还是比较满意，技术上得到了持续提升，工作待遇令我能体面得生活，兴趣爱好也使我得到了很大的心里满足。对18年充满干劲，希望大家共勉。