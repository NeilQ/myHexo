title: cs文件推送到git-server中文乱码问题
date: 2016-08-18 13:34:12
tags:
categories:
- .NET
---

昨天碰到一个问题，某个cs文件push到git server后，里面的所有中文（包括注释和字符串）全都变成了乱码，猜测是编码问题导致，于是将其编码格式从'uft8'改成了'utf8-bom'，再push上去就正常了。BOM（Byte-Order Mark，字节序标记）是Unicode码点`U+FEFF`。详情参考[典型乱码](http://jimliu.net/2015/03/07/something-about-encoding-extra/)

至于怎么改编码格式，如果是用vscode，点击右下角编码格式信息就能切换。如果使用vim，那么：
```
set encoding=utf8
set bomb
```
如果要去掉bom:
```
set encoding=utf8
set nobomb
```
