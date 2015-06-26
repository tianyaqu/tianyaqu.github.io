---
layout: post
title: "模仿一个问答网站"
date: 2015-06-26 22:10
categories: [编程思想]
tags: [mongodb,tornado,python,web]
comments: true
---
为了适应互联网发展的浪潮，捣鼓出一个问答网站出来。集成问答、timeline、好友、话题、问题关注，以及简陋的热门功能。
采用tornado + mongodb实现，timeline使用push的方式，其中一些数据比如timeline没有使用redis存储，每次都是查询数据库获得，
比较简陋，算是学习web开发的一个实验品吧。设计的思路来自[这个](https://github.com/renxing/quora-python) 以及 ruby-china。

<!--more-->
项目地址[在此](https://github.com/tianyaqu/quora-python)

{% img /images/a-zhihu-fork-home.png home page %}


{% img /images/a-zhihu-fork-discovery.png discovery page %}


{% img /images/a-zhihu-fork-topics.png topics page %}


{% img /images/a-zhihu-fork-profile.png profile page %}
