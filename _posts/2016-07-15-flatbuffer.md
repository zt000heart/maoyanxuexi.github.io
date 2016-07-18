---
layout: post  
title: 是时候来一看一下flatbuffers了
categories: [blog ]  
tags: [Tech, ]  
description: 「」   
---
注：注。

Ps

null


* 1、flatbuffers简介
* 2、flatbuffers VS JSON
* 3、flatbuffers 使用


## **1、flatbuffers简介**

flatbuffer是google的一个跨平台串行化库，开发这个最初是用在游戏项目中，github项目地址[flatbuffers](https://github.com/google/flatbuffers)FlatBuffer提供了详细的使用文档，可以参考Google.github.io主页上的教程。

## **2、flatbuffers VS JSON**

对于Json我们使用了这么长时间，目前几乎所有的数据传输格式都是Json，我们都知道Json是当今全服务平台的轻量级数据传输格式，Json量级轻，并且可读性强，使用友好却不过时，Json是语言独立的数据格式，但Json转换的时候却耗费较多的时间和内存，Facebook尝试过将所有的App中Json传输都换成使用Flatbuffers，而最终的效果可以参考一下这篇文章[Improving Facebook's performance on Android with FlatBuffers](https://code.facebook.com/posts/872547912839369/improving-facebook-s-performance-on-android-with-flatbuffers/)，看起来这个确实很有前途。正如facebook展示的那样，遵循Android快速响应界面的16ms原则。

如果你想把项目中所有的Json都替换为flatbuffers，首先要确认项目中真的需要这个，很多时候对性能的影响是潜移默化的，相比而言数据安全更加重要。

下面介绍三个数据序列化的候选方案：

Protocal Buffers：强大，灵活，但是对内存的消耗会比较大，并不是移动终端上的最佳选择。
Nano-Proto-Buffers：基于Protocal，为移动终端做了特殊的优化，代码执行效率更高，内存使用效率更佳。
FlatBuffers：这个开源库最开始是由Google研发的，专注于提供更优秀的性能。

上面这些方案在性能方面的数据对比如下图所示：

![Paste_Image.png](http://img.blog.csdn.net/20160112165416290)
![Paste_Image.png](http://img.blog.csdn.net/20160112165503594)


为什么flatbuffers这么高效？

1.序列化数据访问不经过转换，即使用了分层数据。这样我们就不需要初始化解析器（没有复杂的字段映射）并且转换这些数据仍然需要时间。

2.flatbuffers不需要申请更多的空间，不需要分配额外的对象。

## **3、flatbuffers 使用**
 
 **（1）生成flatc**


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

定义Schema：

类型支持：基本的数据类型，只有标量值可以有默认值，非标量（string/vector/table）字段默认不存在时为null。如果指定会报

	error: default values currently only supported for scalars

的错误，如果想要指定String类型的默认值可以修改生成的Java文件，虽然不建议这么做。
默认值写法：

age : int = 4 ;
isMe : bool = false;

支持数组写法：

array : [string];

支持自定义数据类型

people ： People；

但使用的数据类型必须在同一文件中定义。


 **（3）终于回到Android Stadio部分**
 在gradle中添加
 
     compile 'com.github.davidmoten:flatbuffers-java:1.3.0.1'
 
 然后把刚才生成的People的Java问价添加到项目中。
 准备活动完成，使用起来就比较方便了。
 使用过程中，用到最多的是FlatBufferBuilder。
 
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
        
 生成FlatBufferBuilder, new的方式传入的参数是内部缓冲区的初始大小。使用的非标量类型需要提前生成offset，offset时缓冲区中已编码的字符串开始的偏移量。添加属性前需要先调用类型的start，变量赋值是以add的方式，赋值完成后调用类型的end方法，然后builder的finish，最后转换成字节buffer用于传输。
 
 取值时通过类型的getRootAs...(buffer)直接拿到对象。
 
—End—

## **迭代**

---

### **【乐趣在于挑战】**

不上高山，体会不到一览众山小的乐趣；不下大海，享受不到战胜风浪的自豪。

----