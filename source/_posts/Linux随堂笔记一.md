---
layout: '[post]'
title: Linux随堂笔记一
date: 2019-10-30 00:32:28
tags: linux
categories: linux随笔
---
# linux 学习笔记  --- java环境配置
<!--more-->
1. 下载JDK
    ```
        https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html
    ```
    ![logo](Linxu随堂笔记一/JDK下载链接.png)
    ![logo](Linxu随堂笔记一/JDK下载页面.png)

2. linux下载jdk
    ```
        wegt   https://download.oracle.com/otn/java/jdk/8u221-b11/230deb18db3e4014bb8e3e8324f81b43/jdk-8u221-linux-x64.tar.gz?AuthParam=1569677105_d884aebf4704904ce34b1296f5f1fcac
    ```
    ![logo](Linxu随堂笔记一/linux下载jdk.png)
3.  解压jdk
    ```
        tar -zxvf jdk_1.8.0（以下载后的文件名为准）
        按 ls 查看目录下的文件得知
    ```
4.  配置环境变量
    ``` 
        vim /etc/profile 进入配置 然后按 e 
        按i进行编辑在末尾添加以下代码
        export JAVA_HOME=/root/sdcxy/jdk_1.8.0
        export JRE_HOME=$JAVA_HOME/jre
        export PATH=$JAVA_HOME/bin:$PATH
        配置完成之后 按esc 然后按ctrl + c 在按 shift + z z （记得按两次）保存并退出
    ```
    ![logo](Linxu随堂笔记一/JDK环境配置.png)
5.  测试环境是否配置成功
    ```
       输入 source /etc/profile 刷新环境变量
       java -version 查看如下图则配置成功
    ```
    ![logo](Linxu随堂笔记一/查询Java版本.png)

