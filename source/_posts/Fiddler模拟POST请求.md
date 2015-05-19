title: Fiddler模拟POST请求
date: 2015-05-19 21:00:00
tags: [tools, 抓包]
categories: [tools]

---

### 1. 打开Fiddler，切换到Composer模式，如下图:
 <!--more-->
![Fiddler POST](/image/tools/fiddler/post_simulate.png)

### 2. 需要修改的内容有如下：
 - 请求类型为：POST
 - 请求的URL（粘贴即可）：http://xxxx.com/xxxxx/xxxx.json
 - 请求的头域：（粘贴并修改Authorization）
```java
User-Agent: Fiddler
Host: i.api.weibo.com
Content-Length: 46
Authorization: Basic **d295YW93ZW56aV9hYmNAc2luYS5jb206YWJjZGVmZzEyMzQ**=
Content-Type: application/x-www-form-urlencoded
```

其中：加粗的字符串为 你的**用户名:密码** 进行Base64编码后的字符串。
如：test_abc@sina.com:abcdefg1234  -> dGVzdF9hYmNAc2luYS5jb206YWJjZGVmZzEyMzQ=

![Base64编码](/image/tools/fiddler/text_wizard.png)

 - POST请求的BODY为（粘贴即可）：id=1234567&item_id=data&value=2
 
### 3. 点击右上角 Execute 按钮执行一次请求

