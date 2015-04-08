title: Python Web框架选择小记
date: 2015-04-08 20:44:09
tags:
---

前段时间用python flask写了个Markdown blog，初始选择平台及框架的时候，着实纠结了一番。今偶有所感，特此记录。

写这个blog程序初始目的是为了练手，当然，我必然是希望自己的作品能够实际部署的。由于不想折腾不成熟的mono，首先排除了专长技术Asp.net, 另外本人对Python十分喜爱，便把目光瞄向了一些Python的主流web框架：Bottle, Django, Flask, Tornado等，为此找了好多关于这些框架对比的资料，最终敲定性能快，路由优雅，轻量的Flask。

定好目标，便是一场“不分昼夜的战役”（夸张了）。在代码敲到一半的时候，渐渐对此产生了怀疑。

首先在“轻量”这个特性上，轻量也意味着功能的缺乏，很多东西需要自己去实现，对于我这个业余时间不是很多的情况下，便有些拖累了。好在Flask社区有很多非常棒的插件，如WTForm，flask-restful, flask-sqlarchemy。但问题来了，相较于到处寻找需要的插件，我为什么当初不直接选择集成内容丰富的Django呢？它不需要我到处google各种插件及其使用方法，它的文档也及其完整，不像Flask插件那般只有寥寥术语，很多时候都需要去查看底层api或者针对性得google。Flask确实可定制性高，而Django在这一方面的诟病世人皆知，但问题在于，Django自己所带的内容，已经完全能满足我这个小博客了。

“Flask的性能特别棒”，这是当初我选择它的另一个理由。可是它就算比Django快了好几倍又怎么样呢？对于这么一个blog程序，20毫秒与100毫秒真的差别不大，况且就算今后部署使用，我的一些好友能经常访问就谢天谢地了，肯本不存在负载问题。相比之下，我还不如把时间花在优化前端页面加载上呢。

有个同事曾说过，2-3年经验的程序员，大多都想把一切都做完美。由此想来，这句话说得是极好的。之前选择框架便是如此，要性能，要轻量，要扩展，却是走上了形而上学之路，脱离了需求本身，总之，吃一堑长一智。