---
layout: layout
title: Gradle多样化构建
date: 2017-01-04 09:51:15
tags:
---
# 概述
当我们在android studio新建android工程时,就会给我们生成默认的build.gradle文件,这个文件内容已经满足大多数情况,我们不需要添加新的东西.但是如果你需要在一次构建过程中打出差异化的包,那么就需要了解grale的多样化构建.
在gradle中variant是用来描述某一个构建包,每一个variant都有相对应的配置.在聊variant之前,我们得先了解构建类型和product flavor(我们常说的渠道)这两个概念.
<!--more-->
# 构建类型
构建类型是用来配置同一个渠道不同类型的构建,也就是说每一个渠道都会对应几种不同构建类型的包.最主要的应用就是打正式包和测试包.因为正式包需要去掉debugable,log,换正式服等等.当然你还可以自己定义其他的构建类型来高度的差异化.这些差异化可以通过构建类型很方便的实现.
# Product Flavor
product flavor就是我们常说的渠道,他还用来创建应用的不同版本.我们通常用这个打不同应用市场的包,以此来统计各个市场的下载量.
# Variant
variant是一个二维的概念.每一个variant是由Product Flavor和构建类型组成.假设我们的构建类型有debug和release两种.Product Flavor有免费版(free)和付费版(pay)两种.那么就可以构建四种不同的包
* freeDebug
* freeRelease
* payDebug
* payRelease

# 配置Variant
对于每一个Variant我们都可以做一些相应的配置以此来实现差异化.我们可以配置BuildConfig的字段,resValue以及替换manifest里面的metaData.

```
 buildTypes {
        debug {
            buildConfigField "String", "type", "\"debug\""
            resValue "string" , "test" , "hello world debug"
        }
        release {
            buildConfigField "String", "type", "\"release\""
            minifyEnabled false
            signingConfig signingConfigs.config
            proguardFiles getDefaultProguardFile('proguard-android.txt'),                     'proguard-rules.pro'
        }
        other.initWith(android.buildTypes.debug)
        other {
        }
    }
```

我们还可以在product flavor中配置这些值.


```
 productFlavors {
        free {
            buildConfigField "String" , "type" , "\"free\""
            resValue "string" , "test" , "hello world free"
        }
        pay {
            buildConfigField "String" , "type" , "\"pay\""
        }
    }
```

如果构建类型和渠道中存在相同的字段,那么优先使用构建类型中的字段.在上面的例子中,如果你打debug包.


```
 BuildConfig.type = "debug";
        R.string.text = "hello world debug";
```

替换manifest中metadata的值


```
 <meta-data
         android:name="UMENG_APPKEY"
         android:value="${umeng_app_key}"/>

      buildTypes {
        debug {
         manifestPlaceholders = [umeng_app_key: "你替代的内容"]
        }
        release {
     　　manifestPlaceholders = [umeng_app_key: "你替代的内容"]
        }
        develop {
    　　 manifestPlaceholders = [umeng_app_key: "你替代的内容"]
        }
    }
```

上述的定制只是轻量级的.如果我们要定制图片,布局等资源仅仅用上面的方法是办不到的.如果要做高度化的定制我们必须要引入一个新的概念–源集.

#源集
源集即源代码的集合.每个Variant都会对应一个源集(需要手动创建).


```
  -app
        |    -src
        |    |    -freeDebug
        |    |    |    -java
        |    |    |    |    -com.package
        |    |    |    |    |    -Constants.java
        |    |    |    -res
        |    |    |    |    -layout
        |    |    |    |    |    -activity_main.xml
        |    |    -freeRelease
        |    |    |    -java
        |    |    |    |    -com.package
        |    |    |    |    |    -Constants.java
        |    |    |    -res
        |    |    |    |    -layout
        |    |    |    |    |    -activity_main.xml
        |    |    -payDebug
        |    |    |    -java
        |    |    |    |    -com.package
        |    |    |    |    |    -Constants.java
        |    |    |    -res
        |    |    |    |    -layout
        |    |    |    |    |    -activity_main.xml
        |    |    -payRelease
        |    |    |    -java
        |    |    |    |    -com.package
        |    |    |    |    |    -Constants.java
        |    |    |    -res
        |    |    |    |    -layout
        |    |    |    |    |    -activity_main.xml
        |    |    -main
        |    |    |    -java
        |    |    |    |    -com.package
        |    |    |    |    |    -MainActivity
```

如上图所示,新建工程都会生成一个默认的源集目录main.如果你需要高度定制版本,就可以手动创建该版本的源集目录,然后开始定制你想要的java类或者资源文件.
**注意:主源集目录和其他版本的源集目录不能有相同的类.如果你需要定制一个java类,需要从主源集目录去掉,并在其他源集目录各新建一个.**
