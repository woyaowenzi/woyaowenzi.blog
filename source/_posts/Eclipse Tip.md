title: Eclipse Tip
date: 2014-07-15 23:55:56
tags: [Android, Eclipse，效率, tools]
categories: [Eclipse]
---

**Q：**通过`Ctrl+Shift+T`找对应的类时，类明明存在，并且也在编译路径下，但就是查找不到，原因应该是Eclipse为类建立的索引出了问题。
**A：**直接删除`workspace/.metadata/.plugins/org.eclipse.jdt.core`这个目录，重启Eclipse即可

<!--more-->

**Q：**Eclipse的环境设置导出？
**A：**File → Export（导出），在窗口中展开 General（常规） → Perferences（首选项） → Export all（全部导出），点击Next选择保存配置文件路径即可。

**Q：**eclipse.ini中各参数含义
**A：**