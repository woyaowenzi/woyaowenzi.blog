title: 高效使用Eclipse
date: 2014-03-14 23:41:56
tags: [Android, Eclipse，效率, tools]
categories: [Android, Eclipse, Tools]
---

#工欲善其事，必先利其器
作为一个Android程序员，Eclipse应该是你第一个上手的IDE，当然你也可以选择传说中的神器：**`Android Studio`** 或 **`IntelliJ IDEA`**。这里暂时不讨论Eclipse和这些新晋升的神器的区别（传送门：[Android Studio比Eclipse好用在哪里？][1]），我们单纯的只从Eclipse出发，如何提高我们的工作效率。
<!--more-->

##设置你的File Encoding
对于很多童鞋来说，没有设置`File Encoding`的习惯，而其注释里面经常是中文注释，为了我们的注释能够在别人的机器正常显示，防止乱码滋生，请修改文本编码方式为 UTF-8。
**如何设置**：Window → Preferences → General → Workspace → Text file encoding，设置成 **UTF-8**

##字体选择
每一位程序员都有一套自己喜爱的编程字体，对于在感官上追求完美的人来说，一套好字体，通常能最小限度的分散你编程的注意力。通常，Eclipse默认字体是 **`Courier New`**，10号字体，看起来显示得还不错，不过，我们有更好的字体可供选择，无论是显示英文还是中文，都比较完美，即 **`Consolas`**，字号为11。  
**如何设置**：Window → Preferences → General → Appearence Colors And Fonts，在右边的对话框里选择Java - Java Editor Text Font，设置编程字体，当然，对话框之类的字体也是可以在此设置的。
***更多完美的编程字体请查看***：[10个效果最佳的编程字体][2]
最后，再推荐一款由Adobe公司开源出来的等宽字体：[Source Code Pro][3]，非常之漂亮。
![Source Code Pro](/image/eclipse/source-code-pro.png)

##Theme
如果你厌倦了改来改去调整字体、文本配色等繁琐的事情，我们还有统一的解决方案，就是**`Eclipse Color Theme`**插件，它能提供给你很多非常漂亮的主题，如Sublime Text 2、Obsidian、Notepad++ Like等几十个主题，足以适应各类人群的需求，它就是强大的：[Eclipse Color Themes][4]  
**如何安装**：
Step 1：安装`Marketplace`：http://download.eclipse.org/mpc/indigo/ 
Step 2：通过`Marketplace`安装该插件
当然，你也可以直接通过在Eclipse中，点击 Help → Install New Software 来安装这个插件，详情请Google之。
![Eclipse Color Themes](/image/eclipse/eclipse-color-themes.jpg)

##智能感知
### JAVA
智能感知无疑是加速编程最直接的方式。默认情况下，我们编写JAVA程序时，只有遇到英文句号`.`时，或者手动按`Alt+/`，Eclipse才会将代码提示显示出来，极不方便，如何在输入任意字符时，让智能提示框显示出来呢？
**如何设置**：Window → Preferences → Java → Editor → Content Assist，设置Auto activation triggers for java为：  
`.abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVW_(`，另外，为了加速智能提示框触发时间，我们将Auto activation delay (ms)设置成`20`
设置后效果图：
![JAVA智能感知](/image/eclipse/eclipse-java-intellisense.png)

### XML
对于Android程序员来说，经常要写XML布局文件，但有件很头疼的事情，就是XML里面没有智能感之，只有在出现`<=:`时，才会有智能提示框，如何解决？
其实解决办法如同设置JAVA的智能感知一样。
**如何设置**：Window → Preferences → XML → XML files → Editor → Content Assist → Auto Activation，设置Prompt when thiese characters are inserted：`.abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ_(<=:`，并将Auto activation delay (ms)设置成`20`
设置后效果图：
![XML智能感知](/image/eclipse/eclipse-xml-intellisense.png)

##\#与@
大部分懒人与不喜欢写注释的人，都会抱怨注释难写，太容易分散精力，作为一个程序员，如果连注释都懒得写的话，我个人认为这是个极不负责的程序员，因为软件的软件的生命期80%都是花费在维护上，不写注释的后果是比较严重的，它极大的提高的软件维护的难度，额...写个注释真的很难写吗？至于如何写出专业的注释，我不做过多的解释，只是提供两个简单的技巧。
在代码中添加注释的技巧：使用`@#`激活`JavaDoc`
**@：添加文档中的关键字**
```java```
/**
 * 该类是整个 DEMO 程序的入口。
 * 
 * @author XXX
 * @date 2013-09-29
 */
public class DemoMainActivity extends Activity {
```

![JavaDoc-@](/image/eclipse/javadoc-keyword-link.png)
**#：添加代码链接**
```java```
/**
 * Calls {@link android.view.Window#getCurrentFocus} on the
 * Window of this Activity to return the currently focused view.
 * 
 * @return View The current View with focus or null.
 * 
 * @see #getWindow
 * @see android.view.Window#getCurrentFocus
 */
public View getCurrentFocus() {
    return mWindow != null ? mWindow.getCurrentFocus() : null;
}
```
![JavaDoc-#](/image/eclipse/javadoc-code-link.png)

##快捷键
快捷键的重要性就不必多说了，为了加速编码、阅读代码的效率，我认为每个人都应该掌握下面常用的快捷键。如果你忘记了快捷键，可通过`Ctrl-Shift-L`打开快捷键窗口查看。  
![XML智能感知](/image/eclipse/eclipse-shortcut.png)
[研读代码必须掌握的Eclipse快捷键][5]
[Eclipse快捷键 10个最有用的快捷键][6]

##集成SVN与Beyond Compare
将SVN集成到Eclipse是一个很好的方式，可以让我们不用从Windows文件夹中切来去。早期的时候，我一直不太信任这个SVN插件，原因在于感觉直接使用`TortoiseSVN + Beyond Compare`会更加爽一些，SVN插件自带的比较工具太差了，比较起代码来极度不爽。
### SVN插件的选择
目前市面上有两个很好的插件：
- Subversion（Eclipse团队官方出品）
- Subclipse（SVN团队官方出品）

它们的之间的优劣可以看看这篇文章：[Subclipse vs Subversive][7]

我选择是`Subversion`，原因在于它和`Beyond Compare 3`配置起来更加简单，`Subclipse`我配置了很长时间，未果。
**如何安装Subversion插件**：
- 直接从Eclipse的`Marketplace`上安装
- 通过`Help → Install New Software`安装该插件  
http://community.polarion.com/projects/subversive/download/eclipse/3.0/juno-site/

另外，使用Subversion时，在选择`SVN Connector`时，选择`JavaHL`（请注意版本的选择）

### 在SVN中配置Beyond Compare 3
如果你习惯了BeyondCompare比较文件的风格，再去使用SVN原生的比较风格，你会觉得痛苦无比，因此，我们有必要让BeyondCompare能够配置到SVN插件里面去。
![Beyond Compare比较文件](/image/eclipse/beyond-compare.png)

**如何配置**：
Step 1：Window → Preferences → Team → SVN → Diff Viewer  
Step 2：点击`Add`，设置文件类型为任意文件，即在`Extension or mime-type`中添加`*`  
Step 3：选择Beyond Compare的文件路径（请注意将路径加上双引号），  
在Diff中设置参数：`"${base}" "${mine}"`  
在Merge中设置参数：
`
"${theirs}" "${mine}" "${base}" "${merged}"  
/lefttitle="Incoming (${theirs})"  
/centertitle="Base (${base})"  
/righttitle="Local (${mine})"  
/outputtitle="Merged (${merged})"  
`
如图所示：
![Beyond Compareg与SVN插件整合](/image/eclipse/svn-beyondcompare-integration.png)

##Java Code Style各种配置
Java Code Style的各种配置，主要是指配置我们代码的编码规范，如换行，空格，注释等。  
可以在Window → Preferences → Java → Code Style 下，对`Clean Up`，`Code Templates`，`Formatter`进行配置，一般情况下，一个团队，为了统一编码风格，我们需要有一些统一风格模板，以下三个链接是目前我做的一个模板，仅供参考。
- [Clean Up][8]
- [Code Template][9]
- [Formatter][10]

**如何使用**：保存上面的三个文件，以**Formatter**举例，在`Formatter`界面，点击`Import`导入即可，其它两个类似。

[1]: http://www.zhihu.com/question/21534929
[2]: http://www.iteye.com/news/11102-10-great-programming-font
[3]: http://www.iplaysoft.com/source-code-pro-font.html
[4]: http://eclipsecolorthemes.org/
[5]: http://www.cnblogs.com/yanyansha/archive/2011/08/30/2159265.html
[6]: http://www.open-open.com/bbs/view/1320934157953
[7]: https://www.akii.org/eclipse-svn-plugins-subclipse-vs-subversive.html
[8]: /resource/codestyle/CodeStyle-CleanUp.xml
[9]: /resource/codestyle/CodeStyle-CodeTemplates.xml
[10]: /resource/codestyle/CodeStyle-Formatter.xml
[11]: http://ips.chotee.com/wp-content/uploads/2012/f55126f99c1a_A3EB/source-code-pro.png