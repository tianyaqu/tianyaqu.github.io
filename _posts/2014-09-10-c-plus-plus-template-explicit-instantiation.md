---
layout: post
title: c++ template explicit instantiation
date: 2014-09-10 17:54:40
categories: c++
comments: true
tags: [template,explicit instantiation]
---
在组织c/c++工程的代码时候，有一个重要原则便是“接口与实现的分离”，这样对于调用者屏蔽了实现细节，只曝露最小的外部功能使用接口，调用者只关注自己的业务逻辑，由实现者保证功能实现的有效性。对于已经发布的产品，一些功能升级也只需要替换掉某些相关库而已，也有利于保护核心代码。同时头文件的代码也好看许多，好处种种不一。
当c++模版要达到这个效果，就比较悲剧了。扫一眼书本案例，照着以往的经验兴冲冲开始一阵噼里啪啦，编译出来的结果确是“undefined reference”，令人打消继续学习的念头。
<!--more-->
这个问题的原因在于，template具现化（instantiated）的时机是在编译期，而编译期间不同文件的细节是彼此不知道的，除非include过去（include就把两个文件合并了，仍然是同一个文件）。template可以理解为一个“模式”，其声明部分说明①“模式是什么”，实现部分说明②"怎么做",而调用部分说明③“哪一个模式”，只有编译器同时知道了两者，才能正确的具现化。
template具现化的时机是找到被调用的地方，这样找到了③，通过头文件顺藤摸瓜获得了①,如果②与①不再一个文件,那么"编译期间不能跨文件"的魔力就发挥出来了,导致具现化失败。
那么有办法做到分离么？肯定有。下面介绍三种方法。

##1. 通过include引用
本质上是把声明与实现放到同一个文件。foo.h是声明部分，foo.cpp是实现部分，main.cpp是调用部分，在foo.h末尾include "foo.cpp",main.cpp使用时候include "foo.h"即可。include关键字的作用是在将当前行替换为include文件的内容。这也就避免了编译时期跨文件的问题。这种方法被称为置入式模型（inclusion model）。

##2. explicit instantiation
这个是本文主要提到的内容。除了置入式模型之外，c++提供另外一种方式，称之为“显式具现化”（explicit instantiation）。它要求用户主动明确地书写出模版的适用类型，
    
    template Class<int>;
    
这样代码的结构变成了:foo.h声明，foodef.h实现部分，foo_inst.cpp显式具现化，然后由main.cpp调用。

    //foo.h
    template <typename T>
    class Interface
    {
    public:
        Interface(T const&);
        void show(T);
    private:
        T data;
    };

    //foodef.h
    #include "foo.h"
    #include <iostream>
    template <typename T>
    Interface<T>::Interface(T const&t)
    {
        data = t;
    }

    template<typename T>
    void Interface<T>::show(T x)
    {
        std::cout<<x<<std::endl;
        std::cout<<data<<std::endl;
    }


    //foo_inst.cpp
    #include "foodef.h"

    template class Interface<int>;
    template class Interface<float>;

    //main.cpp
    #include "foo.h"
    Interface<float> fl(2.4);
    fl.show(4.5);
    Interface<int> fi(5);
    fi.show(2);

这种方法甚至可以将模版的实现作成lib被外部使用，将foo_inst.cpp作成lib。

    g++ -c foo_inst.cpp -o foo_inst.o
    ar rcs libfoo.a foo_inst.o
    
在项目中链接该库即可。(有人说这样的话，要把foo.h中使用extern将显式的template声明出来，我这里编译器gcc 4.6.3，没有使用extern关键字不提示错误或者警告。原因不明，有知道原因的请告诉我）。

##~~3. export关键字~~
该方法的使用比较简单，需要在在申明与实现的部分在template前加上export关键字，好假啊，为什么不早点告诉我？
编译器支持太有限了！如《c++ template》中讲到，02年时候只有一家公司(Edison Design Group,Inc)实现了export，gcc,vs,clang都不支持。用户使用起来简单了，而内部实现却复杂了。比如
    
    template Class<int>
    
变为
    
    template Class<float>
    
所有依赖的代码都要重新编译。而常用的项目组织工具如make,会根据文件修改时间以及文件依赖次序来决定是否重新编译，在这种情况下(template代码没变)可能不按照大家的意图来重新编译。只有靠编译器来一一记住这些依赖关系来决定编译与否，这也不会有太多编译时间上的优势。
还有一个现实是，在c++0x标准中，export被列为过时（obsolete）的！而推用extern了。坑爹那！
