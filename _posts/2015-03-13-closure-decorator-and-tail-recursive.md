---
layout: post
title: "闭包，装饰器与尾递归"
date: 2015-03-13 19:33
comments: true
categories: 编程思想
tags: [Scala,python,装饰器,闭包,尾递归]
---
最近一周忙里偷闲，看着pdf学习Scala语言，算是初次接触函数式编程吧(如果不算python的foreach、迭代器与lamda表达式的话)。基本数据类型都是老生常谈了，函数式编程强调抛弃中间变量的做法在python中也算提前打了预防针，不过漫天遍地的新语法真让人开眼，比如古怪的占位符用法、不知所云的偏应用函数(partially applied function，书中的例子看起来好像在演示默认参数的用法)，以及一些奇技淫巧，比如String* 与 _*的不同妙用。总的感觉上是跟传统的语言(c/c++,java)在语法表达上都相差很远，读完不免有种囫囵吞枣的味道。

这里梳理下几个“高大上”的概念，结合一部分python语言的设计，聊一下闭包、尾递归，以及python的装饰器，如有理解错误欢迎指正。
<!--more-->
##1. 闭包与装饰器


闭包的定义：“闭包是由函数和与其相关的引用环境组合而成的实体”。好吧，说人话就是闭包是函数与函数环境组成的那一坨东西。Scala中函数是第一类值，不仅可以定义和调用，也能把他们像值那样传递。

	var increase = (x: Int) => x + 1
	def inc(base:Int) =  (step: Int) => base + step
	
前者得到的是函数类型的increase: Int => Int = \< function1 \>,increase是函数变量，在Scala shell中输入increase可以查看其内容。inc函数的是结果是返回一个函数类型，它具有一个可配置参数x。闭包的作用就发挥出来了，比如，可以获得以不同base的inc函数。这样base 与 step都是可配置的。

    var inc_base_10 = inc(10)
    var result = inc_base_10(5) //step = 5

在python中，有了语法糖的帮助，闭包的一个常见用法是使用装饰器，来实现切面编程。

    import time
    import functools
     
    def timeit(func):
        @functools.wraps(func)
        def wrapper(para):
            start = time.clock()
            func(para)
            end =time.clock()
            print 'used:', end - start
        return wrapper
        
    @timeit
    def work(para):
        print para
        #do something
        pass

被timeit装饰的work任务可以方便的使用timeit提供的计算时间功能，达到重用代码的目的，任何想使用计时功能的模块只要加上timeit装饰器就行了。

##2. 尾递归


递归算是众所周知的概念。它不停的调用自身，在边界条件下返回结果并退出。计算阶乘和裴波那契数列使用递归已经是老生常谈。由于递归要保持每次调用的状态，期间使用的栈空间不释放，当调用次数较多时候，对内存压力很大。对处理速度要求高的场合都不建议使用递归，换由其他方式实现。

那么尾递归呢？它是抓住了递归的痛点。它在本轮计算结束后，把本轮的结果也一并传递给了下轮，这样进入下轮计算时候，上轮的结果就没必要保持了。以计算阶乘为例：
    factorialTailrec(5, 1)  //初始为1,传递给下轮1 * 5 = 5
    factorialTailrec(4, 5)  // 传递给下轮4 * 5 = 20
    factorialTailrec(3, 20) 
    factorialTailrec(3, 60) 
    factorialTailrec(2, 120) 
    factorialTailrec(1, 120) 
    120
    
Scala可以自动实现尾递归的优化，Scala 编译器检测到尾递归就用新值更新函数参数，并把它替换成一个回到函数开头的跳转，所以不必担心使用尾递归的开销。不过Scala中尾递归的使用局限依然很大，JVM 指令集使实现更加先进的尾递归形式变得很困难。以下的两种情况，Scala不能优化:

1.递归是间接的，两个函数相互调用
    def isEven(x: Int): Boolean =
        if (x == 0) true else isOdd(x - 1)
    def isOdd(x: Int): Boolean =
        if (x == 0) false else isEven(x - 1)
        
2.函数调用是一个函数值
    val funValue = nestedFun _
    def nestedFun(x: Int) {
        if (x != 0) { println(x); funValue(x - 1) }
    }
    
虽然funValue的确是代表了nestedFun，但是很抱歉，这种优化Scala编译器不会认为它是尾递归，它只能优化形式上非常严格的尾递归。
