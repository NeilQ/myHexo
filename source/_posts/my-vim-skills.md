title: "我的vim黑科技"
date: 2015-12-17 13:58:16
tags:
- vim
---

用以记录我的vim技巧。

### 回到屏幕中间
NORMAL模式下按zz，可以使当前光标所在行处于屏幕中间。

### ctrl-o, zz
在INSERT模式下，按ctrl-o将会回到NORMAL模式，此时将可以执行一次NORMAL命令，执行完毕后回到INSERT模式，可与zz配合使用

    
### 将外部命令执行结果放到当前文本中来
试试
```
:.!ls
```
```
:.!dir
```

### 执行python代码
在任何格式的文本中，想插入一些数据，只需写一小段python代码，然后v选中这段代码，输入“!python”,这段代码的执行结果就被插入在了这段代码所在的位置。


### dib
删除一对小括号里面的内容

### ]p
调整缩进粘贴

### 浏览之前的打开过的文件目录
:bro ol (browser old)

### Align 插件
选中文本后，执行“:Align”+符号，可将文本按照该符号对其，如
```
var a = 1
var foo = 2
var foobar = 3
```
选中后输入”:Align=“ + ENTER, 结果将变为
```
var a      = 1
var foo    = 2
var foobar = 3
```

### INSERT模式下的光标移动
.vimc中加入
```
" Ctrl + K 插入模式下光标向上移动
imap <c-k> <Up>

" Ctrl + J 插入模式下光标向下移动
imap <c-j> <Down>

" Ctrl + H 插入模式下光标向左移动
imap <c-h> <Left>

" Ctrl + L 插入模式下光标向右移动
imap <c-l> <Right>
```

### ESC替代键
如果你嫌ESC距离手指太远，可以选择其他替代键
ctrl+[ 与esc功能相同
ctrl+c 与esc功能相同

或者自定义配置键盘映射，如
```
:imap jj <ESC>
```
将jj映射为ESC键，但必须快速按下.

### 修改文件保存类型
使用unix文件格式重新保存
```
:w ++ff=unix
```
