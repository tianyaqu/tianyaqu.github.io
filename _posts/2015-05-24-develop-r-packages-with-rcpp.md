---
layout: post
title: "Rcpp开发R包"
date: 2015-05-24 11:43
comments: true
categories: [编程思想]
tags: [Rcpp,R]
---
最近一直在忙着做一个金融数据库系统的设计开发工作（当然不是database implementation了）。用mongodb搭建，能自动化导入数据并给不同语言提供查询接口。所以这个月俨然成为一个胶水匠，不停的给各个语言写wrapper。敌方阵营派出python、matlab、R与我交锋，史称“三英战tianyaqu”，奈何某技高一筹，均被我一一杀退。关于R开发的网上资料(中文)着实太少，统计之都上面给出的都是难度与官方timesTwo旗鼓相当的花拳绣腿。下面给出一些个人的浅薄经验，希望能给后人一些思路。

这当然不能作为一个开发R包的系统描述，Dirk Eddelbuettel大神给出的Rcpp [tutorial A](http://cran.r-project.org/web/packages/Rcpp/vignettes/Rcpp-extending.pdf) 和 [tutorial B](http://cran.r-project.org/web/packages/Rcpp/vignettes/Rcpp-package.pdf) 早已彪炳千秋，当然如果你想知道更详细、全面的底层实现，官方的[Writing R Extensions](http://cran.r-project.org/doc/manuals/r-release/R-exts.html) 也是个不错的参考。

<!--more-->

在R上扩展，目的无外乎这：提高运算速度，功能依赖(第三方库) ，代码复用。我算是第三种，虽然我反对这个做法，奈何做不了决定。所以就把alternative[开源](https://github.com/tianyaqu/mongodb-in-financial-market)了。:) 

### 数据交换
R提供的扩展实现了一种机制，所有的对象都可以用SEXP来传递。但这仍然有点繁琐，从SEXP到C/C++的对象需要频繁使用PROTECT避免变量被R的垃圾回收机制收割，然后使用诸如REAL(), INTEGER()来访问裸数据。在使用RCPP后，生活就变得非常简单，将接口函数用EXPORT出去，就可以被RCPP无缝包装，内部实现就可以完全使用C++的语法搭房子。如何要使用第三方库，直接裹上一层包装，在其中调用第三方库与普通的用法没什么两样。而这最终就被自动翻译成R提供的SEXP体系去，用户就免去了关注SEXP系统的劳苦。

下面的代码片段是调用第三方的例子，它实现wrapper的功能。可以看到我们的wrapper接口直接使用的是std::string格式，没有使用SEXP。与此相反，需要从c++回传给R的时候，Rcpp::CharacterVector,Rcpp::NumericVector,Rcpp::DataFrame 可以帮我们传递字符串向量、数值向量甚至是data.frame。指针(reference-like)也可以通过NumericVector实现，只不过是只操作vector的首元素罢了。


```
//' @param so many... 
//' @export

// [[Rcpp::export]]
DataFrame R_GetTicks(std::string market,std::string contractId, unsigned start, unsigned end, int limit)
{
    struct Tick* result_buffer = 0;
    unsigned len = 0,i = 0;
    //调用第三方library,或者functions
    int ret = m_GetTicks(&result_buffer,&len,market.c_str(),contractId.c_str(), start, end, limit);
    
    ....
    //manipulate result buffer to create dataframe
    ....
    return df
}
```

由于提供rcpp提供c++的环境，就可以完全利用C++的遗产，呼风唤雨都不是梦想。数据序列化反序列化、网络连接什么的都是小菜一叠，没什么不可能，只有发挥你的想象力。Rcpp最终利用Call机制给R提供接口。 mongoquery_R_GetTicks是Rcpp做的Name mangling，是wrapper被翻译后的名字。

```
R_GetTicks <- function(market, contractId, start, end, limit) {
    .Call('mongoquery_R_GetTicks', PACKAGE = 'mongoquery', market, contractId, start, end, limit)
}
```

### 第三方库

由于R只接受GCC系的工具链，在linux下面不成什么问题，在win成了大问题，这意味着项目依赖的所有第三方库都需要用rtools(其实它也是mingw工具套装)编译链接。所幸的是，多数的优秀第三方库都可以在mingw编译通过(比如boost,MLPACK)。只消用Makevars.win将依赖库(mingw compiled)加入即可。

    #BOOST_INC为设置的系统变量，可以使用诸如-IE:/src/boost/inc 方式导入头文件依赖，
    #lib path同理
    PKG_CPPFLAGS= -I$(BOOST_INC)
    PKG_LIBS =  -L../lib -lboost_xxx 
    
最终注意package发布时候要将使用的库一块发布。

### 工具

#### 1.rstudio

rstudio已经是标配了吧，支持舒心地r package开发。直接使用Rcpp.package.skeleton也无伤大雅，不过注意namespace文件需要手动更改。rstudio的不足在于写cpp代码时候不顺心，老迈的像微软官方记事本。

#### 2.qtcreator    

qtcreator 支持建立标准c/c++项目，协助c++部分的开发与调试。它支持自主选择工具链，选择rtools的工具链(或者将qtcreator的工具链作为rstudio的工具链)，选好工具链，可以编译第三方依赖库丢给Rcpp链接。

这次意外的收获是发现mingw对win的支持如此之好。项目用到了lzma压缩算法，7zip lzma sdk只提供了msvc的编译工程，翻了下代码，其压缩、解压缩使用了win的多线程，原以为路子就此断了，摩拳擦掌正准备将pylzma(基于7zip lzma sdk) 移植到标准过来，做一件利国利民的好事。却发现pylzma的多线程模块作者根本没作改动，而pylzma 支持mingw作为编译核心。于是依样画瓢，获得lzma库+1。

稍微dig下，原来mingw 使用原生w32api提供运行环境(Crgwin 做的是POSIX emulation,基于X Server的program唯一选择)
>As MinGW uses the standard Microsoft C runtime library which comes with Windows，you should be careful and use the correct function to generate a new thread.

