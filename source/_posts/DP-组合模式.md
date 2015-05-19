title: 组合模式
date: 2015-02-24 15:20:56
tags: [设计模式，组合]
categories: [设计模式]
---

##整理中…
<!--more-->
组合模式，用于构建树状(tree)模型。
我认为最经典的就是文件和文件夹模型。如下结构：
Android中，View和ViewGroup的关系，也是非常经典的树状模型，如下的ViewTree的构建。
http://developer.android.com/intl/zh-cn/guide/topics/ui/overview.html

通过Hierarchy Viewer来观察
http://developer.android.com/intl/zh-cn/tools/debugging/debugging-ui.html

经典的模型如下：
View和ViewGroup模型如下：

什么时候应该使用这种设计模式？
1. 你想要构建树状结构的模型
2. 需求中体现整体-部分的层次结构，而且希望忽略组合对象和单个对象的不同，即统一的使用对外接口。

---
### 组合模式
![组合模式](/image/design-pattern/组合模式-composite.png)

---
### 组合模式在android中的体现
![组合模式在android中的体现](/image/design-pattern/组合模式-composite-android.png)