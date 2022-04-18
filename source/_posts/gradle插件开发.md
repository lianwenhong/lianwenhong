---
title: gradle插件开发
date: 2022-04-11 21:50:37
tags:
categories:
-----------

### gradle插件开发的方式

```
apply plugin: 'com.android.application'
apply plugin: 'com.android.library'

```

| 方式             | 说明                                                                        |
| ------------------ | ----------------------------------------------------------------------------- |
| Build script脚本 | 把插件写在build.gradle文件中，一般用于简单逻辑，作用范围为build.gradle文件  |
| buildSrc目录     | 将插件源代码放在buildSrc/src/main/中，作用范围为该项目                      |
| 独立项目         | 一个独立的java项目/模块,可将文件发布到仓库(jcenter)，使其他项目可以方便引入 |
---
任何可以运行在jvm中的语言都能用来开发gradle插件，比如groovy最终编译产物也是.class文件

```
1、aapt 打包资源文件 阶段
2、aidl 转 java 文件 阶段
3、Java 编译（Compilers）生成.class 文件 阶段
4、dex（生成 dex 文件）阶段
5、apkbuilder（生成未签名 apk）阶段
6、Jarsigner（签名）阶段
7、zipalign（对齐） 阶段
[https://juejin.cn/post/6844903850453762055#heading-6](https://juejin.cn/post/6844903850453762055#heading-6)
```
