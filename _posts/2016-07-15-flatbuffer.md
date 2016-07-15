---
layout: post
title: 是时候来一看一下flatbuffers了
date: 2016-09-15
categories: blog
tags: [tags]
description: 一篇Android上使用flatbuffer的详细介绍

---

注：注。

Ps

null


* 1、flatbuffers简介
* 2、flatbuffers VS JSON
* 3、flatbuffers 使用


## **1、flatbuffers简介**


## **2、flatbuffers VS JSON**


## **3、flatbuffers 使用**
 
 **（1）生成flatc**
 
github项目地址[flatbuffers](https://github.com/google/flatbuffers)

在使用前我们需要先生成flatc，flatc用来将我们编写的fbs文件转换成Java文件，也可以转换成其他文件，但我们在这里并不关心。
编译FlatBuffers生成flatc，我使用的是CMake工具，我们要先安装cmake。
首先要查看我们的系统中是否安装了gcc和g++编译工具。通过下面两条指令查看。默认情况下是安装了的。

	gcc -v
	g++ -v

如果没有安装通过下面两条指令安装

	yum install gcc
	yum install gcc-c++
	
接下来就需要安装CMake，下载CMake源码进行编译安装

	wget https://cmake.org/files/v3.5/cmake-3.5.2.tar.gz
	tar -zxvf cmake-3.5.2.tar.gz
	cd cmake-3.5.2/
	sh bootstrap
	make
	make install
	
cd到从git下载下来的文件夹，编译flatc程序

	cd flatbuffers/
	cmake -G "Unix Makefiles"
	make
	
这样就生成了flatc，和一些其他的可执行文件。

 **（2）定义Schema，生成Java文件**
 
 flatBuffers使用Schema定义数据结构，下面是一个简单的示例。语法较为简单
 
	namespace org.sample;
	table People {
	  name:string;
	  age:int;
	}
	root_type People;

将文件命名为sample.fbs，然后通过flatc来编译这个Schema。

	./flatc --java sample.fbs

会在目录下生成相关的java文件。

 **（3）终于回到Android Stadio部分**
 在gradle中添加
 
     compile 'com.github.davidmoten:flatbuffers-java:1.3.0.1'
 
 然后把刚才生成的People的Java问价添加到项目中。
 准备活动完成，使用起来就比较方便了。
 我们可以看一下flatbuffers，jar包里的文件，Table对应于类，Struct是结构体主要是一个ByteBuffer，Constants存放了三个常量，SIZEOF_SHORT，SIZEOF_INT和FILE_IDENTIFIER_LENGTH，用到最多的是FlatBufferBuilder。
 
        FlatBufferBuilder builder = new FlatBufferBuilder(0);
        int sun = builder.createString("Sun");
        //下面向People中填充属性
        People.startPeople(builder);
        People.addName(builder, sun);
        People.addAge(builder, 18);
        int tom = People.endPeople(builder);
        builder.finish(tom);
        ByteBuffer buffer = builder.dataBuffer();

        People people = People.getRootAsPeople(buffer);
        textView.setText(people.name());


—End—

## **迭代**

---

### **【乐趣在于挑战】**

不上高山，体会不到一览众山小的乐趣；不下大海，享受不到战胜风浪的自豪。

----