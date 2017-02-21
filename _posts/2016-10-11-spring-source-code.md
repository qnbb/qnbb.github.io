---
layout: post
title: IntelliJ IDEA 导入 Spring 源码
categories: [Spring, OpenSource]
description: IntelliJ IDEA 导入 Spring 源码
keywords: Spring, Open Source
---

### 1. 下载intellij idea15

网下载连接：https://www.jetbrains.com/idea/download/
建议下载Ultimate版本，这样对web开发的支持组件比较多。破解参考：http://idea.lanyus.com/。

### 2. 下载jdk8、Gradle，配置环境

###3. clone spring源码

spring源码github地址：https://github.com/spring-projects/spring-framework

###4. 打开，首先阅读源码中的import-into-idea.md文件，文件内容如下，按着一步步操作即可

Within your locally cloned spring-framework working directory:
1. Precompile `spring-oxm` with `./gradlew cleanIdea :spring-oxm:compileTestJava`
2. Import into IntelliJ (File -> New -> Project from Existing Sources -> Navigate to directory -> Select build.gradle)
3. When prompted exclude the `spring-aspects` module (or after the import via File-> Project Structure -> Modules)
4. Code away

-----------------------
### 注意事项：
不同分支 import-into-idea.md 内容不同，操作不做不同。下载源码后，选择一个分支，根据指示部署。如果选择A分支，用B分支的操作步骤，容易导致导入的包冲突。
