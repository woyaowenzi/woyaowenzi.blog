---
title: URI Scheme
date: 2012-03-03 13:53:29
tags: [Network, URI Scheme]
---

###什么是URI Scheme?
一般情况下，遇到这种概念不清的问题，最好的第一手资料就是wiki，实在看不懂，再看百度百科，但前者给出的资料一般都是更加准确一些。
以下为维基百科和百度百科关于这个问题的连接：
* [URI Scheme][1]
* [URI][2]

<!--more-->

从维基百科上的定义我们得知，URI Scheme 是统一资源标识符（Uniform Resource Identifier ）的命名结构，说白了，就是定义一个资源的(暂且我们先认为 URI Scheme 与 URI 的概念是一样的）。但是，这个资源是一个宽泛的概念，并不一定是我们所说的 web 资源，它可以是你本机的一个文件，也可以是网络上的视频，等等。因此，我们有必要从一些常规的误区分辨出来，以加深我们的理解：
* **认为 URI 就是 URL**。
实际上，从其名称上来看，URI（Uniform Resource Identifier）和 URL（Uniform Resource Locator）两者名称都不一样，所以必然有区别，前者是统一资源标识符，后者是统一资源定位符，后者是网络上用于定位互联网上Web资源的，如 HTML 文档、图像、视频片段、程序等。

* **它是一个通用定义**，不是 "protocols"，也不是 URI protocols 或者 URL protocols。
* **它经常用于设计特殊的协议。**如http scheme（HTTP协议），XML namespaces，文件标示等等。从上面的一些结论来看，URI Scheme实际上一个概念性的东西，是一个规范，所以符合它的规范的都可以称之为 URI Scheme，当然，我们也可以设计我们自己的 scheme，用来实现我们特殊的目的。

它一般具有如下的形式：
```java
<scheme name> : <hierarchical part> [ ? <query> ] [ # <fragment> ]
```
举几个常见的例子：
```java
http://write.blog.csdn.net/postedit/7313543,  
file:///c:/WINDOWS/clock.avi
git://github.com/user/project-name.git
ftp://user1:1234@地址
ed2k://|file|%5BMAC%E7%89%88%E6%9E%81%E5%93%81%E9%A3%9E%E8%BD%A69%EF%BC%9A%E6%9C%80%E9%AB%98%E9%80%9A%E7%BC%89%5D.%5BMACGAME%5DNeed.For.Speed.Most.Wanted.dmg|4096933888|2c55f0ad2cb7f6b296db94090b63e88e|h=ltcxuvnp24ufx25h2x7ugfaxfchjkwxa|/
```
这些都是一个URI Scheme。
其中：
`<scheme name>`：很明显，这是scheme的名称，对于上面五个scheme，它们scheme名分别是http, file, git, ftp, ed2k（电驴协议），实际上，它们也代表着协议名称。
`<hierarchical part>`：实际上，一般情况，它包含 authority 和 path。 
`<query>`：可选项目，一般使用；隔开或&隔开的键值对`<key>=<value>`
`<fragmentg>` ：可选项目包，其它额外的标识信息

从wiki上抄来一个更加具体的例子可能更好的说明问题，假设有如下两个URI Scheme，
foo://username:password@example.com:8042/over/there/index.dtb?type=animal&name=narwhal#nose
urn:example:animal:ferret:nose
它们的分析结果如下：

如果你看懂了这个图，你肯定也明白URI Scheme到底是什么玩意了。
![URI Scheme](/image/uri/uri-scheme.png)

---

在android开发中，我们可能会需要分析webbrowser上web页面的一些特定URI Scheme，比如说点击网页上某个下载按钮，该按钮被点击后发出一个scheme，内里包含了需要下载的文件信息。我们监听这个点击事件，获取并分析URI Scheme，然后去下载对应的文件。实际上，就是要分析一些链接，然后再操作，那如何获取路径这个scheme？
假设我们的scheme是电驴协议的URI，如：
`ed2k://|file|%5BMAC%E7%89%88%E6%9E%81%E5%93%81%E9%A3%9E%E8%BD%A69%EF%BC%9A%E6%9C%80%E9%AB%98%E9%80%9A%E7%BC%89%5D.%5BMAC-GAME%5DNeed.For.Speed.Most.Wanted.dmg|4096933888|2c55f0ad2cb7f6b296db94090b63e88e|h=ltcxuvnp24ufx25h2x7ugfaxfchjkwxa|/`
我们需要做的事就是通过webview重写`shouldOverrideUrlLoading`这个方法，获取指定的URI Scheme，并解析它，最后下载对应文件。
该函数的作用是：在新的页面进行跳转前，给应用程序一次接管新的url的机会，当应用程序处理了该url，返回true，如果想让当然的webview自行处理，返回false.
以下是一个简单的代码片段。
```java
webview.setWebViewClient(new WebViewClient() {  
    @Override  
    public boolean shouldOverrideUrlLoading(WebView wv, String url) {
        if (null != url && url.startsWith("ed2k:")) {  
            // Parse the url and download file async...  
            return true;  
        } else {  
            return super.shouldOverrideUrlLoading(wv, url);  
        }  
    }  
}); 
```

[1]:http://en.wikipedia.org/wiki/URI_scheme
[2]:http://baike.baidu.com/view/160675.htm
[3]:http://hi.csdn.net/attachment/201203/3/16423_13307557171Ru7.png