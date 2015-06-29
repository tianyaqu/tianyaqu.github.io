---
layout: post
title: "matlab 闲话"
date: 2015-05-14 23:04
comments: true
categories: [编程思想]
tags: [matlab,mongodb,dll,datenum]
---
matlab 调用c dll时,最可恨的是dll与matlab之间数据的交互，在dll中中获取的数据，如果内容比较大。那么回传数据的时候，是由matlab准备空间呢，
还是dll中申请。经验是在dll中申请，并向matlab提供释放接口。

这样matlab使用者就不必担心内存申请、释放问题（实际情况是matlab的使用者已经完全不知内存为何物），matlab的获取的空间是通过blank,zeros这种矩阵方式获得的，用起来比较别扭，还不容易凑够心仪大小buffer。
<!--more-->

##1. matlab与dll

具体实现，可以采用Multilevel Pointers方式hack。 matlab的官方样例shrlibsample有提到这么个实例，但语焉不详。这里贴出详细步骤，以
[mongodb-in-financial-market](https://github.com/tianyaqu/mongodb-in-financial-market)中的例子来说一下。

首先dll的接口要通过__declspec(dllexport)修饰，这里将其定义为EXPORT宏,这样在定义时候用EXPORT修饰，看起来自然的多。

    #define EXPORT __declspec(dllexport)

    EXPORT int decompress(const char* market,unsigned char *compressed, struct TickData** result, unsigned int* out_len);
    EXPORT void free_mem(void* ptr);

看到decompress的第三个参数result，为回传给matlab的buffer块首地址，二级指针是让使用者不必担心内存申请问题，只准备好一个指针大小空间，
dll中会将它挂在malloc获得的空间上。最后一个参数提供了buffer长度out_len，用户只要注意不超出out_len就不会越界。

而调用者只需要准备好数据，二级指针和长度，然后利用回传的buffer和长度，根据offset遍历即可。

    %dx为输入数据,对应compressed.
    compress_ptr = libpointer('uint8Ptr',dx);
    tickdDta_ptr = libpointer('TickData');
    len = libpointer('uint32Ptr',0);
    ret = calllib('decompress','decompress',db,compress_ptr,tickdDta_ptr,len);
    if ret == 0
        for i = 1:len.Value
        offset = i-1;
        ptr = get(tickdDta_ptr + offset);
        day = ptr.Value.ActionDay;
        updatetime = ptr.Value.UpdateTime;
        mill = ptr.Value.UpdateMillisec;
        ...
    end

##2. datenum与毫秒

题外话。datenum处理毫秒问题。datenum不提供直接毫秒用毫秒构造时间，如：datenum(year,month,day,hour,minute,sec)，毫秒直接报错。
不过可以将毫秒折算成单位天时间，补到秒级的datenum上去，这样得到的datenum实际上存储的即是毫秒级数据。

    dt = datenum(year,month,day,hour,minute,sec) + millsecond/(1000*60*60*24);
    
直接用datestr不能够显示毫秒数据，需要通过格式字符FFF来精确显示到毫秒

    datestr(dt,'yyyy-mm-dd HH:MM:SS:FFF')
