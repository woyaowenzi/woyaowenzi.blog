title: 如何设计一个SDK
date: 2014-08-03 15:20:56
tags: [SDK，设计]
categories: [设计]
---

##整理中…
<!--more-->

接口设计：确定到底是使用全静态还是使用单例

如何设计一个好的SDK？

从2013年9月到2013年12月，我主要重构了微博SDK，期间基本上全部重构了微博SDK的所有代码，N月过去了，感觉应该留下点什么，
今天重新翻阅了下以前的代码，总结一些SDK的设计理念。

理念：用起来爽！就这么简单
无论在设计SDK，还是大到架构，小到类、函数的量级，都要遵循这个规则。

接口名、创建方式有待考虑
统一客户端与SDK之间通讯、SDK内部的ErrorCode、ErrorMessage、ErrorDesc
WeiboException的设计
Web授权时，第三方可定制WebView的Container 未指派
网络模块框架+重构 未指派
安全性问题：HTTPS API/包名签名
文档、视频：要保证从你出手的代码和文档是清晰的，对于任何想要接手你的人来说，都是能快速接受的。

[1] https://tower.im/projects/85e9de8a064e4cde97bc8a878b3a8533/messages/81c5fa31fd634f629abd620613c61250/