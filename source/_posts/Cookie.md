title: Cookie
date: 2012-03-03 15:20:56
tags: [Android, Cookie]
categories: [Android]
---

## 1. 什么是cookie?

### 什么是Cookie?
Cookie是存在内存中或硬盘中的一些数据，用来和服务器打交道。
http://en.wikipedia.org/wiki/HTTP_cookie
http://zh.wikipedia.org/wiki/Cookie
http://baike.baidu.com/view/835.htm
<!--more-->

### 用途
 - 记录用户登录信息（authentication），下次访问时可以不需要输入自己的用户名、密码
 - 记录用户喜好，使网站个人化，如某个用户在CNN 网站，不想查看任何商务新闻，用户可以将之设置，下次访问时就不会再出现商务新闻。
 - 网上购物，比如购买N种物品和统一结账时，从cookies中提取购买列表。

### 存放位置
**Chrome for windows：** C:\Users\UserName\AppData\Local\Google\Chrome\UserData\Default\Cookies文件，该文件是SqliteDB文件
**IE：** %appdata%\Microsoft\Windows\Cookies，该文件夹被隐藏，通过命令行可以看到 
**Android中webbrowser：** Android中为安全的考虑，每个应用程序都会有独立的cookie，各个应用程序不能共享cookie

## 2. 原理
比如说我们访问某个网站：http://www.example.org/index.html
首先，浏览器第一次通过HTTP访问该网站。该网站收到请求后，通过Set-Cookie返回给browser，然后，browser在进行第二次访问时，将这些cookie顺便带过去，提供一些必要的信息。
![cookie-request-response](/image/cookie/cookie-request-response.png)

## 3. Android中如何保存session cookies?
在Android中，Session cookie的特点是当关闭掉web browser，该cookies就会被清除掉。
所以，如果你想通过cookies保存用户登陆信息，做到在一段时间内不需要二次登陆，就需要自己保存好cookies，然后下次登陆时再设置一遍就可以了。
**解决办法：**在第一次login后，保存login cookie到share preference，下次登陆，先要remove掉过期的session cookies，然后再重新设置登陆cookies.
### 1. save
```java
String loginCookies = CookieManager.getInstance().getCookie(NetworkConstant.SERVER_URL);
```

### 2. re-set
```java
CookieManager.getInstance().removeSessionCookie();            
CookieManager.getInstance().setCookie(url, loginCookies);
```

