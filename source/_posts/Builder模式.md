title: Builder模式
date: 2015-02-27 15:20:56
tags: [设计模式，Builder]
categories: [设计模式]
---

一般我们自己要定义一个实体类，我们大多有两种方式来定义其构造函数：
假设有如下的User类：
<!--more-->
```java
public class User {
    private final String firstName;  //required
    private final String lastName;   //required
    private final int age;           //optional
    private final String phone;      //optional
    private final String address;    //optional
    ...
}
```

构造函数我们可以这么写：
```java
public User(String firstName, String lastName) {
    this(firstName, lastName, 0);
}

public User(String firstName, String lastName, int age) {
    this(firstName, lastName, age, "");
}

public User(String firstName, String lastName, int age, String phone) {
    this(firstName, lastName, age, phone, "");
}

public User(String firstName, String lastName, int age, String phone, String address) {
    this.firstName = firstName;
    this.lastName = lastName;
    this.age = age;
    this.phone = phone;
    this.address = address;
}
```

然后，再给出一堆get/set方法
```java
public String getFirstName() {
    return firstName;
}

public void setFirstName(String firstName) {
    this.firstName = firstName;
}

public String getLastName() {
    return lastName;
}

public void setLastName(String lastName) {
    this.lastName = lastName;
}

public int getAge() {
    return age;
}

public void setAge(int age) {
    this.age = age;
}

public String getPhone() {
    return phone;
}

public void setPhone(String phone) {
    this.phone = phone;
}

public String getAddress() {
    return address;
}

public void setAddress(String address) {
	this.address = address;
}
```

OK，一个实体类差不多就这样构建完毕了，看上去非常完美，有鼻子有眼。
可是写多了这种Model类，你不觉得很这样写很啰嗦么？那么问题来了，上面这个类有啥问题？
问题列表：
1. 构造函数
	1.1 参数多了，怎么办？
	再加几个构造函数，形如：
	public User(String firstName, String lastName, int age, String phone, String address, String xxxx, ...)
	1.2 在调用构造方法时，这么多构造函数，我该选哪个？
	由于有一部分参数是可选的，可选的参数必须得给个默认值，那对于使用者来说，默认值是多少呢？会不会导致数据体不正确呢？
	这给使用者造成很多困扰，当然，你也可以把函数对应的javaDoc写得很清楚来规避这个问题，但总感觉不美。
2. set方法
	1.1 在未完全调用setX方法前，这个User对象是不完整的

Builder模式
```java
public class User {
    private final String firstName; // required
    private final String lastName;  // required
    private final int age;          // optional
    private final String phone;     // optional
    private final String address;   // optional

    private User(UserBuilder builder) {
        this.firstName = builder.firstName;
        this.lastName = builder.lastName;
        this.age = builder.age;
        this.phone = builder.phone;
        this.address = builder.address;
    }

    public String getFirstName() {
        return firstName;
    }

    public String getLastName() {
        return lastName;
    }

    public int getAge() {
        return age;
    }

    public String getPhone() {
        return phone;
    }

    public String getAddress() {
        return address;
    }

    public static class UserBuilder {
        private final String firstName;
        private final String lastName;
        private int age;
        private String phone;
        private String address;

        public UserBuilder(String firstName, String lastName) {
            this.firstName = firstName;
            this.lastName = lastName;
        }

        public UserBuilder age(int age) {
            this.age = age;
            return this;
        }

        public UserBuilder phone(String phone) {
            this.phone = phone;
            return this;
        }

        public UserBuilder address(String address) {
            this.address = address;
            return this;
        }

        public User build() {
            return new User(this);
        }
    }
}
```

使用：
```java
public User getUser() {
    return new User.UserBuilder("Jhon", "Doe")
            .age(30)
            .phone("1234567")
            .address("Fake address 1234")
            .build();
}
```

这种风格的设计模式提供了哪些福利？
1. User的构造器是私有的，这就意味着客户端不能直接创建实例。
2. 这个类是不可变的。所有属性都是final类型并且他们由构造器设置值。此外，我们只提供getter操作。
3. 建造者使用流式接口习语（链式表达）来让客户端代码更易读。
4. 建造者的构造器只接受两个必须的参数，并且这两个属性是仅有的被设置为final类型的，这样就能保证这些属性在构造器中是被赋值的。

适用场景&示例
1. 并不是每个类都适合这样搞，很明显，要写个Activity类肯定是不合适的，只有在以下情况，才可能需要考虑这么写
	- 成员较多
	- 有部分想成为可选成员
	- 有部分成员想保持final（只允许被初始化一次）
	- 比较纯粹的Model类
2. Android中示例
	1. Notification.Builder
```java
Notification notification = new Notification.Builder(context)
    .setSmallIcon(R.drawable.ic_stat_player)
    .addAction(R.drawable.ic_prev, "Previous", prevPendingIntent)
    .addAction(R.drawable.ic_pause, "Pause", pausePendingIntent)
    .addAction(R.drawable.ic_next, "Next", nextPendingIntent)
    .setStyle(new Notification.MediaStyle()
    .setMediaSession(mMediaSession.getSessionToken())
    .setContentTitle("Wonderful music")
    .setContentText("My Awesome Band")
    .setLargeIcon(albumArtBitmap)
    .build();
```

	2. AlertDialog.Builder
```java
public Dialog showDialog() {
    // Use the Builder class for convenient dialog construction
    new AlertDialog.Builder(getActivity())
        .setTitle(R.string.security_warning)
        .setIcon(android.R.drawable.ic_dialog_alert)
        .setMessage(R.string.dialog_fire_missiles)
        .setPositiveButton(R.string.fire, new DialogInterface.OnClickListener() {
            public void onClick(DialogInterface dialog, int id) {
               // TODO
           }
        })
        .setNegativeButton(R.string.cancel, new DialogInterface.OnClickListener() {
            public void onClick(DialogInterface dialog, int id) {
                // TODO
           }
        })
        .show();
    }
```

参考：[如何使用建造者模式(Builder Pattern)创建不可变类][1]
[1]: http://www.importnew.com/7860.html