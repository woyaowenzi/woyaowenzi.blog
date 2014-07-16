title: SublimeText-Markdown-Preview
date: 2014-04-13 23:41:56
tags: [SublimeText，效率, Tools]
categories: [SublimeText]
---

使用 [Sublime Text][1] 神器编辑`markdown`文本后，无法预览是一件很蛋疼的事情，幸运的是现在有一款`Markdown Preview`插件，能够满足我们的需求。
<!--more-->
**项目地址：**[Markdown Preview][2]
**快速安装：**
`Command+Shift+P`调出命令行，输入`Install Package`，然后搜索 `markdown`，找到 `Markdown Preview` 插件，安装即可。
功能列表：
- 以Python/Github Markdown显示在浏览器中
- 导出Python/Github Markdown到HTML文本中
- 将Markdown以HTML的格式拷贝到剪贴版

**快捷键绑定：**
打开`Preference -> 按键绑定 - 用户`，设置快捷键。
以**Python样式**的Markdown预览：
```python
[
	{ "keys": ["alt+m"], "command": "markdown_preview", "args": {"target": "browser", "parser":"markdown"} }, 
]
```
以**Github样式**的Markdown预览：
```python
[
	{ "keys": ["alt+m"], "command": "markdown_preview", "args": {"target": "browser", "parser":"github"} }, 
]
```

[1]: http://www.sublimetext.com/
[2]: https://github.com/revolunet/sublimetext-markdown-preview