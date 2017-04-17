title: GRPC琐碎
date: 2017-04-17 14:57:18
tags:
categories:
- .NET
---

记录一些grpc使用过程中的体验、坑及技巧，Server及Client语言都为C#。

## 体验：
- 搭建简单，一次尝试即成功。
- 分布式好伙伴

## 技巧
- server意外掉线后重新上线，client将会自动重连，不需要自己处理。
- 利用`streamimg`功能，可快速实现推送服务器，即观察者模式。之前有做过利用`rabbitmq`消息队列中间件实现消息推送，或许可以用grpc streaming替代，无论性能、复杂度、稳定性都将提升。

## 坑
- 暂无