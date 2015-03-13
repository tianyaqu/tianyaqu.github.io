---
layout: post
title: "from c10k to coroutine"
date: 2015-02-24 15:36
comments: true
categories: c++
tags: [Reactor,muduo,Eventloop,coroutine]
---
[c10k](http://www.kegel.com/c10k.html)问题：当网络的用户访问量达到一定级别后(一万的并发量)，经典的编程模型便不能满足数以万计的连接请求。最近在看陈硕的多线程编程书，阅读muduo代码，真如醍醐灌顶。一方面解释了自己一直模糊的概念（异步），同时也开了眼界（多线程、Eventloop），这都是平常书上难读到（业界却常用）的‘常识’。另加上自己最近事情的一些体会，记下来当作读书笔记吧。
<!--more-->
先理一下常见模型。从最常见的网络编程教程的玩具程序开始讲起。服务端绑定、监听后，客户端连接过来，Alice与Bob就开始光天化日的眉来眼去。这只能是玩具，因为只能同时服务一个用户，在当前客户端断开前，其他客户的请求是看不到的。

然后就有了fork-per-client（正是linux环境编程里举的例子），以及它的变形thread-per-client。即起一个进程或者线程处理请求，这两种是换汤不换药的做法。进程和线程都是需要消耗系统资源的，拿线程来讲，一个线程默认栈空间是10m,在32位系统上只能起300左右线程。离10k相差甚远。

然后使用select/poll，epoll这一套先进生产力工具。select/poll同时服务的客户端增大了，但仍然不够。epoll每次得到活跃fd即可。根据fd号，将请求分发到accept或者handler上，这就是鼎鼎大名的Reactor模式。已经能出色应付得了多数网站需求了。可是，对于耗费cpu的操作，其他客户的请求也不能满足了，所以引入了多线程，多招几个能干活的伙计，把干活的与接客的分开。

线程池就是这些伙计们，用Eventloop把干活的与接客的分开，如果活重，辛好还有别的伙计能顶替，同时每个伙计可也做好几件事情(Eventloop)，但不会把接客的任务给耽误了。这个就是陈硕最终提倡的模型：Reactor + Threadpool + one Eventloop per Thread.陈硕的书讲了很多，除开一些经验之谈（怎么做才较好，智能指针实现RAII手法读来让人心旷神怡），Eventloop与Threadpool当是核心了。

多线程一出现，考虑问题脑子就要多转个弯。不得不注意异步，事件的顺序也不能想当然了。前段时间刚好给我埋了个小坑。要作的事情很简单，是在app启动时候，往redis写点关键数据进去。由于网络环境问题，有时候没写成功，所以要加入重试功能，失败了最多再试三次。用的client刚好是个异步接口实现，这下懵了，结果是什么、什么时候出来不知道，sleep原地等待肯定是行不通的。client的callback倒是可以看到结果，可是执行都是在其他线程的，通知本进程只能用ipc了，有点杀鸡用牛刀的嫌疑，并且为了等结果本进程还是得阻塞才行。

后来问了神牛，说这种用coroutine实现最好了，这当然是不可能的（先天无力）。最后只能用笨方法，callback里面继续使用client查询,为了存储尝试次数，不得不裹了一层。

    callback(context* r,void* para)
    {
        if(r->result != "ok")
        {
            if(para->time-- > 0)
            {
                asyncClient client
                client.query(callback,para)
            }
        }
    }

    app()
    {
        Para para
        para.time = 3
        asyncClient client
        client.query(callback,&para)
        do_other_stuff()
    }

coroutine在tornado里面见过，作为一个装饰器出现。从作用上来看，是把异步调用同步化，把单线程当多线程用，处理上面的问题刚好是顺心趁手的，可惜没有巨人焉能成为牛顿。在python中有yield这种语言级支持，可也轻松实现，在c/c++中，看到两种实现。一种是基于上下文切换的，使用Setcontext族函数保存上下文跳转，比如[云风的实现](https://github.com/cloudwu/coroutine)。当然还有一种逆天的实现，使用类似goto语句的跳转(比如[这里](http://www.chiark.greenend.org.uk/~sgtatham/coroutines.html))，例如包裹switch case的宏，著名的实现是 boost::asio::coroutine。


