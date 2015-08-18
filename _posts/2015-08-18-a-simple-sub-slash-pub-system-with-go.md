---
layout: post
title: 一个简单的sub/pub系统 (基于 golang channel机制)
date: 2015-08-18 20:23
comments: true
categories: [编程思想]
tags: [协程,go,channel] 
---

go的channel机制是语言内置的通信方式，可以轻松实现各个不同模块(协程)之间的同步、通信。本文利用channel实现一个简单的sub/pub系统原型。

大致流程为：server建立后，开启一个协程监听控制命令，server内部会维护一个注册表，记录订阅话题的用户，当要发布消息时候，将订阅该话题的用户逐个发送消息即可。具体实现的时候，订阅的用户被当作一个channel，控制命令也是一个channel，控制命令的channel负载用户的控制信息，其中包括用户的channel信息，通过穿针引线的方式，将主题与订阅的用户联系起来。命令channel达到了channel复用的效果，同时肩负着控制命令接收与分发的任务。
<!--more-->
如下是具体实现，code explains itself well

    package main

    import "fmt"

    const (
        sub int = iota
        pub
        unsub
    )

    type Pubsub struct{
        //command channel,multiplexing
        cmdChan chan cmd
        capacity int
    }

    type cmd struct {
        op     int
        topic string
        ch     chan interface{}
        msg    interface{}
    }

    func PubServer(capacity int) *Pubsub {
        ps := &Pubsub{make(chan cmd),capacity}
        go ps.start()
        return ps
    }

    func (ps *Pubsub) Sub(topic string) chan interface{}{
        ch := make(chan interface{},ps.capacity)
        ps.cmdChan <- cmd{op:sub,topic:topic,ch:ch}
        return ch
    }

    func (ps *Pubsub) Pub(msg interface{},topic string){
        ps.cmdChan <- cmd{op: pub, topic: topic, msg: msg}
    }

    //topic ->sublist
    type registry struct{
        topics map[string][]chan interface{}
    }

    //bind sub channels to topics
    func (reg *registry) add(topic string,ch chan interface{}){
        if reg.topics[topic] == nil {
            reg.topics[topic] = make([]chan interface{},0,5)
        }
        reg.topics[topic] = append(reg.topics[topic],ch)
    }

    //kick the ball
    func (reg *registry) send(topic string,msg interface{}){
        for _,ch := range reg.topics[topic]{
            ch <- msg
        }
    }

    //worker,ready to dispatch
    func (ps *Pubsub) start(){
        reg := registry{
            topics:make(map[string][]chan interface{}),
        }
    loop:
        for cmd := range ps.cmdChan{
            if cmd.topic == ""{
                continue loop
            }
            switch cmd.op {
                case sub:
                    reg.add(cmd.topic, cmd.ch)
                case pub:
                    reg.send(cmd.topic, cmd.msg)
                case unsub:
                    ; //skipped,for it's no easy to remove an element given a slice
            }
        }
        
    }

    func main() {
        s := PubServer(1)
        ch1 := s.Sub("english")
        ch2 := s.Sub("french")
        ch3 := s.Sub("chinese")
        
        s.Pub("hello english", "english")
        s.Pub("hello french", "french")
        s.Pub(1.222222, "chinese")
        
        fmt.Printf("%v ","hello english" == <-ch1)
        //shall be false
        fmt.Printf("%v ","hello frenchxxx" == <-ch2)
        fmt.Printf("%v ",1.222222 == <-ch3)
    }
