---
layout: post  
title: retrofit2封装okhttp时使用到的设计模型 
categories: [blog ]  
tags: [Tech, ]  
description: 「标准是好用且好看 」   
---
## retrofit2封装okhttp时使用到的设计模型－－代理模型和两次装饰者模型
retrofit2和okhttp的具体使用网上有很多，这里不具体阐述，只对retrofit2封装okhttp时使用到的几个设计模型学习一下。  
****
### okhttp基础知识
看下http最基本的使用方式：
![15:24:51.jpg](http://ww4.sinaimg.cn/large/006tNbRwjw1f5tgswca4fj30k40d2gmx.jpg)
先创建一个request，并利用该request，使用okhttpclient生成一个call。
call的enqueue会降该网络请求加入队列，进行数据的异步请求。我们只需要传入一个实现了的callback回调，即可对异步请求的结果进行处理。**问题1: 注意onResponse()方法也是在异步线程进行的，所以需要一个huandler调用post（runnable）方法返回到ui线程进行ui处理。**  
### retrofit2对该问题的封装
要了解retrofit2对该问题的封装，我们需要了解retrofit的大致工作情况：
#### retrofit2基础知识
我们需要先定义一个接口对象，演示一个最简单的get请求，接口如下所示：：

![15:44:27.jpg](http://ww3.sinaimg.cn/large/006tNbRwjw1f5thd6eht7j30ah0323yj.jpg)

可以看到有一个getUsers(）方法，通过@GET注解标识为get请求，@GET中所填写的这个“users”值和baseUrl组成完整的路径，baseUrl在构造retrofit对象时给出。
然后我们需要入如下步骤：
![15:42:50.jpg](http://ww1.sinaimg.cn/large/006tNbRwjw1f5thbhncqbj30ma0bfabg.jpg)
#### 代理模式
通过

	IUserBiz userBiz = retrofit.create(IUserBiz.class);
来得到上面接口的实例，然后当调用该接口的方法时，使用了**代理模式**的逻辑在内部会生成一个完整url的request，并使用内部的okhttpclient生成一个call。为什么当调用该接口的方法时，能够生成一个含有完整url的request的一个call呢？我们看下retrofit.create（）代码：
![17:42:27.jpg](http://ww2.sinaimg.cn/large/006tNbRwjw1f5tkryb1fhj30n4046aal.jpg)
然后，当调用生成的接口实例方法时，会调用上述代码的invoke（）方法，通过该invoke方法，我们可以得到之前写的接口的名称，注释和参数。结合baseurl，我们就可以得到一个完整url的request的一个call。具体是，通过这些信息得到request，然后利用okhttp生成它的call，然后retrofit2通过两次封装得到它对应的call。
前面说了，这个call不是okhttp中的那个call，而是对其进行了封装了的ExecutorCallbackCall。第一次封装的目的就是主要为了解决1。
#### 如何进行的封装？
ExecutorCallbackCall对okhttp的call的封装主要是继承那个call，并重写了enqueue方法：
该ExecutorCallbackCall的成员变量为：  
![16:15:28.jpg](http://ww2.sinaimg.cn/large/006tNbRwjw1f5ti9g44bqj307t0173yg.jpg)  
其中变量delegate我们**可以认为**是okhttp的那个call。  
我们看下重写的enqueue方法，我们发现还是调用的delegate中的enqueue方法，也就是说这里使用了**装饰者模式**，在delegate中的enqueue方法传入实现了“huandler调用post（runnable）方法返回到ui线程进行ui处理”逻辑的callback。
![16:30:28.jpg](http://ww3.sinaimg.cn/large/006tNbRwjw1f5tip1vfzkj30ml0gbgo3.jpg)
如果是Android平台，callbackExecutor为一个Executor对象，并且利用Looper.getMainLooper()实例化一个handler对象，在Executor内部通过handler.post(runnable)来执行callbackExecutor.execute（）。这样一来，ExecutorCallbackCall的作用就很明显了：**使用装饰者模式将okhttp中的那个call的异步回调转发至UI线程**，因此我们调用ExecutorCallbackCall.enqueue()传入的callback，只需要写ui的逻辑即可。
#### okhttp的那个call的二次封装
上面有一句“变量delegate我们**可以认为**是okhttp的那个call”，实际上变量delegate并不是okhttp的那个call，而是使用了**装饰者模式**对其进行了二次封装。为什么要进行二次封装呢？因为retrofit2生成的call相对于okhttp来说，不仅仅做了将异步回调转发至UI线程这一个工作，因为retrofit2生成的call还带有范性，所以调用改call方法的enqueue（）时，希望异步获取的response数据能直接转型为call所带的范性，这正是第二次封装的目的，利用**装饰者模式**实现该逻辑，思路和上一个一样。看下delegate这个类中enqueue方法，其在内部调用了okhttp的call的enqueue方法：  
![17:20:58.jpg](http://ww1.sinaimg.cn/large/006tNbRwjw1f5tk5lhqxuj30hp0gaq41.jpg)  
通过parseResponse（）方法将得到转型后的response。


 



