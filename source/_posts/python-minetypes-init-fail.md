title: " python在windows中mimetypes初始化失败问题解决"
date: 2013-12-24 22:27:43
tags:
categories:
- Python
---

很多人反馈python2.7在windows中经常会出先如下错误信息：
```
File "C:\Python27\lib\SimpleHTTPServer.py", line 208, in SimpleHTTPRequestHand
ler
    mimetypes.init() # try to read system mime.types
  File "C:\Python27\lib\mimetypes.py", line 358, in init
    db.read_windows_registry()
  File "C:\Python27\lib\mimetypes.py", line 258, in read_windows_registry
    for subkeyname in enum_types(hkcr):
  File "C:\Python27\lib\mimetypes.py", line 249, in enum_types
    ctype = ctype.encode(default_encoding) # omit in 3.x!
UnicodeDecodeError: 'ascii' codec can't decode byte 0xd7 in position 2: ordinalnot in range(128)
```

今天在django admin页加载css文件时也碰倒，经查询是python的一个bug，具体见[http://bugs.python.org/issue9291](http://bugs.python.org/issue9291)
官方废除的解决方案是修改mimetype.py文件，见[http://bugs.python.org/file19332/9291a.patch](http://bugs.python.org/file19332/9291a.patch),改完即ok。

该问题在进行http请求操作时可能会出现，是由url编码方式未能正确转换引起的。



