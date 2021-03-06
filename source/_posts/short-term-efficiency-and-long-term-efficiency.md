title: 开发之短期效率与长期效率
date: 2017-03-09 16:28:36
tags:
categories:
- 随笔
---

工作中接到需求并着手开发时，我们通常会有两种选择，我将这两种方式分别称为追求短期效率与追求长期效率。

## 短期效率
我们假设这样一个场景，当你接到一个开发任务时，上司会交代：三天内必须完成，更有甚者：今天必须完成。可事实上，所有人都清楚，这个任务根本不可能在本周内完成。需求不明确，按照某某某的样子抄；这个地方逻辑可能会有特殊情况，先不管碰到再说；这部分代码很乱，可能性能还很差，不管了先这样。终于，在你吭哧吭哧加班之后，功能终于完成了，交付测试。我将这种现象称为追求短期效率。

然而，不幸的是，你或者你的小组，在接下的几周或数月，都要不断为这个早就“完成”的功能上修复bug、调整需求、优化性能。当然，在这段时间里，你不会仅仅只是修补这个功能，你会不断得用一天、三天、一周来完成一个个功能，同时穿插着修补一个个不断积累下来的技术债务。长期以往，上下级便会无法互相信任。你的领导会想：“这小子能力不行，每件事都做得一塌糊涂。算了，现在难招人，先用着吧。”而你会想：“这sb经理屁事不懂，跟他说也不听，自以为是，爱咋咋的，每个单位都会有或多或少的坑，没办法。”哪怕互相之间已经早已不合，仍然磕磕碰碰前进，狼狈为奸。

## 长期效率
让我们再看另一种情况，当你接到一个开发任务时，上司会组织需求方与开发开一个短会，详尽得阐述整个功能的流程、表现，并让你回去写一份开发方案文档，根据功能复杂度，通常整个小组会花一天到一周的时间来商讨、改进这个方案，使用test-driven模式的团队还会同时设计测试用例，使其足够健壮、可扩展，紧接着才进行一周或者一至三个月的开发，在整个周期完成后，几乎不需要再为这个功能操心，它已经真正完成了！我将这种现象称为追求长期效率。

在这种情形下，整个团队会极其和睦，balance work。经理对手下充满信心：这些家伙能力不错，积极性还高，年底给他们多加点薪。开发者也觉得经理不错，值得跟随。

## 如何应对
大家应该已经从我的文字中看出我的态度了，事实上，通过我的观察，两中方式的开发周期基本是一致的。假如你处在追求长期效率的团队，相信你会异常舒适。那么假如你处在追求短期效率的团队，你应该怎么办，怎么去改变？

如果你只是一个普通的程序员，你将毫无办法，是的，毫无办法。要么长期忍受，要么弃坑跳槽，及时止损。而假如你凭借自己的技术、能力得到他人的信任，有一定的话语权，那么我建议你试一试将这种情况反转过来，**不积极去改变，你和咸鱼有什么区别**。

在我过去的两年多时间里，我恰巧经历了这两种公司。

第一家公司追求长期效率，开发舒适，团队和睦，每什么好说的。

第二家公司初期则是追求短期效率，在初期阶段，某总监在国庆前夕让我三天完成某一功能。第一天，我便意识到这个功能根本不可能在如此短的时间内完成，三天加班中，我完成了骨架。紧接着，国庆后我与另一位开发花了将近一个月才完成这项功能。那时我只是一个普通程序员，并没有任何办法改变这件事。

几个月后，由于一些不可描述的原因，上层管理换了。这时，我的技术能力已经得到了团队的认可，具备了一定的话语权，在我的不断安利与上司的支持下，我们小组率先作为试点，组内成员都积极得去尝试敏捷开发，开发初期做更多思考，终于取得了一定的成效，追求更长远的效率。

然而，几个月后，由于一些不可描述的原因，上层管理又换了，嗯。由于管理理念的差异，渐渐地，迫于压力，团队又进入了追求短期效率的混乱中:)。此时我已升为组长，具有一定的小范围管理权，在这种大环境下，我仍然在开发中坚持稳健，绝不冒进。当我接到一项任务，并且指示一天、三天要完成时，我不会立即含糊指派下去，相反，我都会仔细思考，拆解任务，并将具体设计与关键要素都整理成文档，解释清楚，最终安排各成员去做。显然，这些任务仍然不可能一天或三天完成，但通过显式的文档，说服大家相信，这项任务花这么长时间，值，没有人会指责我。当然，假如我消极处理，把任务往下压，还是得花这么多时间。

**既然我们可以漂亮得活着，为何要蓬头垢面？**