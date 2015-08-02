---
layout: post
title: "greenDAO使用入门"
date: 2015-07-26 11:25:29 +0800
comments: true
categories: Android
---

ORM（Object Relational Mapping）可以帮我们减轻开发工作量，提高工作效率。之前做Java Web项目用过Hibernate，现在想在Android项目中用一下ORM。  

之前上网查了一下目前主流的Android ORM，有如下几种：  
* ORMLite  
* SugarORM  
* GreenDAO  
* Active Android  
* Realm  

当这些库都没有用过的时候，如何进行技术选型呢，我用了个很简单的办法，看下Github上的star数就好了，greenDAO是其中最多的，并且初步横向比较了下，greenDAO的性能不错，下边是网上一张greeDAO和ORMLite的性能对比图

![](http://i3.tietuku.com/b07d9bec3ea47b7d.png)

greenDAO的使用比较简单，下边就开始介绍下如何使用greenDAO吧。
<!--more-->
##1. DAOGenerator工程和Android App工程
![](http://i3.tietuku.com/4e32221ef721f18c.png)

上边这张图说明了greenDAO的作用，greenDAO用来帮助我们的工程生成Java Objects和访问数据库的DAO类，其中Java Objects中的属性字段和SQLite Database里的表的列属性一一对应。生成Java Objects和DAO类后，应用程序Android App可以通过使用DAO类来进行增删查改的数据库操作，操作方法全都是基于Java Objects的，不用写sql语句。

这里涉及了两个工程，DAOGenerator工程和Android App工程。前者用于生成Java Objects和DAO类，后者是开发者的应用工程，使用前者生成的类。

下边就以我做的项目来举例，两个工程的目录结构如下所示

|--PmdCampus  
&nbsp;&nbsp;&nbsp;&nbsp;|--src    
&nbsp;&nbsp;&nbsp;&nbsp;|--src-gen          
|--PmdCampusDAOGenerator  

PmdCampusDAOGenerator是生成DAO类的独立Java工程，PmdCampus是应用开发者的Android App工程。

如果是用gradle来编译的话，PmdCampusDAOGenerator工程的build.gradle文件如下
```java
apply plugin: 'java'
apply plugin: 'application'

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile 'de.greenrobot:greendao-generator:1.3.1'
}
```
其中`compile 'de.greenrobot:greendao-generator:1.3.1'`这行是引入greenDAO的库。

再来看下PmdCampus工程的build.gradle，由于太多其它内容，这里只贴出关键的部分。需要引入以下库
```java
compile 'de.greenrobot:greendao:1.3.7'
```

大家注意PmdCampus下除了常规的src目录外，还有一个手工创建的src-gen目录，这个目录就是存放PmdCampusDAOGenerator工程生成的Java Objects和DAO类的。所以，PmdCampus的build.gradle文件配置一下，编译src-gen里的java文件，需要加入以下的配置：
```java
sourceSets {
    main {
        java {
            srcDir 'src'
            srcDir 'src-gen'
        }
    }
}
```

##2. 在DAOGenerate工程创建DAOGenerator类
DAOGenerate工程是一个普通的Java工程，DAOGenerator类是用来生成DAO文件的类，有main函数入口，直接从main函数运行就可以了。DAOGenerator类的代码如下：
```java
package com.example;

import de.greenrobot.daogenerator.DaoGenerator;
import de.greenrobot.daogenerator.Entity;
import de.greenrobot.daogenerator.Schema;

public class DAOGenerator {
    public static void main(String args[]) throws Exception {
        Schema schema = new Schema(1, "com.tencent.PmdCampus.module.message.dao");
        addChat(schema);
        new DaoGenerator().generateAll(schema, "./PmdCampus/src-gen");
    }

    private static void addChat(Schema schema){
        Entity chat = schema.addEntity("Chat");
//        chat.setTableName("ChatAnotherTableName");		//可以设置sqlite中的table name，默认跟Entity的名字相同
        chat.addIdProperty().primaryKey().autoincrement();  // 消息唯一ID
        chat.addIntProperty("conversationId").notNull();   // 聊天会话ID
        chat.addIntProperty("type").notNull();              // 消息类型，1文字，2图片
        chat.addStringProperty("senderId").notNull();        // 发送人唯一标识
        chat.addStringProperty("receiverId").notNull();      // 接收人唯一标识
        chat.addStringProperty("content").notNull();        // 消息内容，1文字内容，2图片url
        chat.addLongProperty("send_time").notNull();        // 消息发送的时间
        chat.addIntProperty("status").notNull();            // 消息是否发送成功（0表示成功或已读， 1表示失败, 2表示正在发送, 3表示未读）
        chat.addLongProperty("seq").notNull();              // 服务端有序的序号
    }
}

```
在main方法中
```java
Schema schema = new Schema(1, "com.tencent.PmdCampus.module.message.dao");
```
该方法第一个参数用来更新数据库版本号，第二个参数为要生成的DAO类所在包路径。
```java
addChat(schema);
new DaoGenerator().generateAll(schema, "./PmdCampus/src-gen");
```
进行建表和设置要生成DAO文件的目标工程的项目路径。addChat方法里定义了Chat对象的属性，数据库中的字段也会一一对应。

之后运行main函数，就会在PmdCampus/src-gen目录下生成Java Objects和DAO类了。
![](http://i3.tietuku.com/d85c5423440290ca.png)

##3. 在Android App工程里创建DBHelper类
生成了DAO类之后，在自己的Android App工程就可以使用了，为了更加方便，这里封装了一个DBHelper类，代码如下
```java
package com.tencent.PmdCampus.module.message.storage.db;

import android.content.Context;
import android.util.Log;

import com.tencent.PmdCampus.module.message.dao.Chat;
import com.tencent.PmdCampus.module.message.dao.ChatDao;
import com.tencent.PmdCampus.module.message.dao.DaoMaster;
import com.tencent.PmdCampus.module.message.dao.DaoSession;

import java.util.List;

import de.greenrobot.dao.query.DeleteQuery;
import de.greenrobot.dao.query.QueryBuilder;

/**
 * Created by junshao on 15/7/26.
 */
public class ChatDBHelper {
    private static Context mContext;
    private static ChatDBHelper instance;

    private static ChatDao chatDao;

    public static ChatDBHelper getInstance(Context context)
    {
        if (instance == null)
        {
            instance = new ChatDBHelper();
            if (mContext == null)
            {
                mContext = context;
            }

            DaoMaster.OpenHelper helper = new DaoMaster.DevOpenHelper(context, "message.db", null);
            DaoMaster daoMaster = new DaoMaster(helper.getWritableDatabase());
            DaoSession daoSession = daoMaster.newSession();
            chatDao = daoSession.getChatDao();
        }
        return instance;
    }

    public void insertChat(Chat chat){
        chatDao.insert(chat);
    }

    public void delChat(long id){
        chatDao.deleteByKey(id);
    }

    public void delAllChat(){
        chatDao.deleteAll();
    }

    public List<Chat> queryChatList(long seq){
        QueryBuilder qb = chatDao.queryBuilder();
        qb.where(ChatDao.Properties.Seq.le(seq));
        qb.orderDesc(ChatDao.Properties.Seq);
        return qb.list();
    }
}

```

ChatDBHelper是一个单例，主要负责初始化ChatDao对象，然后利用ChatDao就可以用操作对象的方式直接操作DB了。queryChatList里用到了QueryBuilder，QueryBuilder可以设置诸多属性，这里用到了where和orderDesc，相当于对应的sql语句。关于API更多的使用说明可以参考[greenDAO官方网站](http://greendao-orm.com/)。

##4. 在业务逻辑里使用DBHelper类
现在可以愉快方便的使用DBHelpler类了，生成一个Chat对象并插入，然后查询的代码如下:
```java
ChatDBHelper chatDBHelper = ChatDBHelper.getInstance(this);
Chat chat = new Chat();
chat.setConversationId(0);
chat.setType(0);
chat.setSenderId("111");
chat.setReceiverId("111");
chat.setContent("content");
chat.setStatus(1);
chat.setSeq(111);
chatDBHelper.insertChat(chat);
List<Chat> chatList = chatDBHelper.queryChatList(111);
```

这篇文章只是简单的介绍了greenDAO的入门用法，更多高级内容请参考官方网站。

***

参考资料：  
*  [greenDAO – Android ORM for SQLite](http://greendao-orm.com/)   
*  [Android开源：数据库ORM框架GreenDao学习心得及使用总结](http://glblong.blog.51cto.com/3058613/1354953)