---
title: RabbitMQ的基本概念和模型
date: 2016-08-07 16:54:21
tags: tech   web
---
   
RabbitMQ是一种消息队列，用于程序间的通信。形象地说，MQ就像一个邮局，发送者将消息写入MQ，MQ负责把消息发送给接收者。RabbitMQ可支持Java, PHP, Python, Go, JavaScript, Ruby等多种语言。然并卵，人生苦短，我还在用Java...      
本文主要介绍RabbitMQ的基本概念和模型。

## 几个基本概念

![图1 rabbit_model](http://obj8insuu.bkt.clouddn.com/rabbit_model2.png)   

RabbitMQ的基本模型如图1所示。先介绍一些术语。   
__生产者(producer)__   
在图中为P，表示消息的发送者。  
__交换机(exchanges)__   
在图中为X, 生产者发过来的消息需要经过交换机，交换机将决定将消息放到哪些队列当中。   
__队列（queue）__   
队列在图1中由红色矩形阵列表示，负责保存消息和发放消息。   
__消费者（consumer）__   
在图中为C，代表等待接收消息的程序。

## 信息流   
消息是怎么从生产者传递到消费者的呢？   
首先，生产者发送消息到交换机，同时发送一个key，通过这个key，交换机就知道该把消息发到哪个队列。随后交换机把消息发送到相应的队列中。由队列将消息发送给消费者。   
消费者监听某些队列，当有消息过来时，就立即处理消息。   
那么，接下来就有两个问题：   
* 交换机是如何根据key来分配消息到队列？   
* 队列怎样将消息发送给消费者？    

## 交换机类型
这个部分将回答第一个问题。交换机如何根据key来分配消息到队列?    
RabbitMQ的交换机有四种类型：direct, topic, headers, fanout   

__fanout__   
fanout 交换机就跟广播一样，对消息不作选择地发给所有绑定的队列。以图1为例，两个队列都将收到消息。   

__direct__   
![图2 direct](http://obj8insuu.bkt.clouddn.com/rabbit_direct.png)   
在direct模式里，交换机和队列之间绑定了一个key，只有消息的key与绑定了的key相同时，交换机才会把消息发给该队列。如图2所示，消息的key为orange时，消息将进入队列Q1;key为black或者green时，消息将进入队列Q2。若消息的key是其他字符串，被交换机直接遗弃。   
![图3 多重绑定](http://obj8insuu.bkt.clouddn.com/rabbit_multi5.png)   
同时，交换机支持多重绑定，多个队列可以以相同的key与交换机绑定。如图3所示，当消息的key为black时，消息将进入Q1和Q2.   

__topic__    
topic模式可以理解为主题模式，当key包涵某个主题时，即可进入该主题的队列。topic模式的key必须具有固定的格式：以"."作为间隔的一串单词；比如：“quick.orange.rabbit”，key最多不能超过255byte。    
交换机和队列的key可以以类似正则表达式的方式存在，有两种语法：   
1. "*"可以替代一个单词   
2. "#"可以替代0个或多个单词    

![图4 topic](http://obj8insuu.bkt.clouddn.com/rabbit_topit.png)   
说话比较抽象，看图4。图中，Q1与交换机绑定的kye为：“\*.orange.*”，故当消息的key为三个单词，且中间的单词为orange时，消息将进入Q1。Q2与exchange绑定的key为"rabbit.#"，当消息的key以rabbit开头时，消息将进入Q2。   
其实，图中的key描述的是动物的属性，第一个单词表示反应速度，第二个单词表示颜色，第三个单词表示名称。绑定的key定义了它们的筛选规则，所有橘色的动物都将进入Q1,所有懒洋洋的动物都将进入Q2,老鼠也将进入Q2(老鼠并不懒呀，不过人家就这么定的也没办法)。
下面给出更具体的例子帮助理解：   
key --> 进入的队列   
lazy.orange.elephant --> Q1和Q2   
quick.orange.fox --> Q1   
lazy.pink.rabbit --> Q2   
quick.brown.fox --> 被直接无视   

__headers__   
官网没介绍这个模式呀，大概不常用吧，有缘再见［微笑挥手脸］。
## 队列分发消息的方式
这一部分将回答第二个问题。队列怎样将消息发送给消费者？   
![图5 Round-robin dispatching](http://obj8insuu.bkt.clouddn.com/rabbit_round2.png)   

__循环发放（Round-robin dispatching）__   
队列分发消息给消费者的方式采用循环发放。举例来说，若队列里有四个消息w,x,y,z，则C1将得到消息z和x,C2将得到消息y和w。即每个消费者按顺序每人发一个消息。   
注意，在这种分配方式下，消息其实在刚进入队列的时候就已经内定好将要被分发的消费者。即z,x一定是给C1,y,w一定是给C2。   
这种方式存在一些隐患，如果z和x都是耗时的命令、y,z都是简单的命令，C1将不停地工作，而C2就比较空闲，造成资源浪费。    
公平发放解决了上述问题。   

__公平发放（fair dispatching）__   
这种方式下，队列只会把消息给空闲的消费者。如果它看到某个消费者正忙，就查找下一个空闲消费者。   

```
java中，公平发放的实现：
使用basicQos()方法将 prefetchCount 设为1。

int prefetchCount = 1;  
channel.basicQos(prefetchCount);

```
__消息的确认（Message acknowledgment）__   
若没有特别设定，消息一旦被队列分发给消费者，就被rabbitmq从内存中删除。   
在这种情况下，如果将一个正在处理消息的消费者强行关闭，那么，消息将未被完全处理，且RabbitMQ完全不知情。   
为了解决上述问题，可以配置使得消息处理完后，向RabbitMQ返回一个acknowledgment。RabbitMQ直到收到acknowledgment后，才将消息删除。   
当消费者死亡时（its channel is closed, connection is closed, or TCP connection is lost），RabbitMQ会知道这个消费者发生问题了，将重新发送消息给空闲的消费者。    
消息没有timeout，即使消费者处理很长很长时间，乃至无穷无尽，RabbmitMQ也认为消费者正在处理。（真有耐心）   
其实，消息的确认是默认开启的，不需要特地设置。

以上   
如有错误，还请指教

__更正：__   

*  交换机类型－topic章节中，示例有两个重复的key:

```
原文：

key --> 进入的队列   
lazy.orange.elephant --> Q1和Q2   
quick.orange.fox --> Q1   
quick.orange.fox --> Q2    //key名字打错了
quick.brown.fox --> 被直接无视 
```

* 已更正为：

```
key --> 进入的队列   
lazy.orange.elephant --> Q1和Q2   
quick.orange.fox --> Q1   
lazy.pink.rabbit  --> Q2   //此处为正确的key名
quick.brown.fox --> 被直接无视
```
   
__参考：__   
<a href=https://www.rabbitmq.com/tutorials/tutorial-five-java.html>RabbitMQ官网－RabbitMQ Tutorials</a>

  