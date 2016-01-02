---
layout:     post
title:      "Kombu学习笔记（一）"
subtitle:   " Part I  Kombu概述"
date:       2015-12-10 15:00:00
author:     "Liuv"
header-img: "img/post-bg-2015-12-05.jpg"
tags:
    - AMQP
    - Kombu
---

>  Kombu is a messaging library for Python.The aim of Kombu is to make messaging in Python as easy as possible by providing an idiomatic high-level interface for the AMQ protocol, and also provide proven and tested solutions to common messaging problems.——摘自[kombu官网](http://kombu.readthedocs.org/en/latest/introduction.html)

## 0x01 前言
从上面官方对Kombu的描述中可以看到，Kombu为AMQP协议提供了一个通用的高层接口，也即是说Kombu不负责具体的消息传输服务，而是在消息中间件层之上作了一层封装，并为性能、可靠性和异常捕获等常见的问题提供了统一的解决方案，这样开发人员就可以聚焦于业务代码的编写，而不用考虑消息如何发送和接收。

本专题是对[RabbitMQ系列](/2015/12/06/six-steps-to-study-rabbitmq-1/)的扩展延伸，因此对RabbitMQ还不了解的朋友，请先看一下前面的有关内容，否则会对一些概念无法清楚的了解。之所以要学习Kombu，其实也是为后面分析oslo.messaging做铺垫。社区为了适应OpenStack中对消息传输的一些特殊要求，在Kombu的基础上进行扩展从而形成了oslo.messaging，新增了许多新特性。因此，Kombu的学习也是掌握oslo.messaging必不可少的一环。

说句题外话，Kombu的意思是海带！！1难道开发这个库的程序员喜欢吃海带？话说想找个海带的唯美背景图还真是难~~

好啦，接下来，就开始正式学习Kombu，本篇博客主要内容是对Kombu做一个整体上的概述，为后面的深入了解打下基础。事先声明，本系列的内容都是在Kombu官方文档的基础上进行整理形成，部分参考了[网友bingotree](http://bingotree.cn/?p=204)的博客，在这里向他表示感谢。

## 0x02 概述

#### 重要概念
在Kombu中，有一些重要的概念需要事先了解，有的与RabbitMQ相同，也有的是RabbitMQ中没有的，下面来具体看一下。

- Producers: 发送消息给exchange
- Exchanges: 用于路由消息（消息发给exchange，exchange发给对应的queue）。路由就是比较routing-key（这个message提供）和binding-key（这个queue注册到exchange的时候提供）。使用时，需要指定exchange的名称和类型（direct，topic和fanout）。可以发现，和RabbitMQ中的exchange概念是一样的。
- Consumers: consumer需要声明一个queue，并将queue与指定的exchange绑定，然后从queue里面接收消息。
- Queues: 接收exchange发过来的消息
- Routing keys: 每个消息在发送时都会声明一个routing_key。routing_key的含义依赖于exchange的类型。一般说来，在AMQP标准里定义了四种默认的exchange类型，此外，vendor还可以自定义exchange的类型。但是，我们下面只关注AMQP 0.8版本中定义的三种默认exchange类型，也是最常用的三类exchange。

	- Direct exchange: 如果message的routing_key和某个consumer中的routing_key相同，就会把消息发送给这个consumer监听的queue中。
	- Fan-out exchange: 广播模式。exchange将收到的message发送到所有与之绑定的queue中。
	- Topic exchange: 该类型exchange会将message发送到与之routing_key类型相匹配的queue中。routing_key由一系列“.”隔开的word组成，“*”代表匹配任何word，“#”代表匹配0个或多个word，类似于正则表达式。

可以看到，除了有在AMQP中定义的exchange、queue等概念，Kombu还对消息的发送方Producers和接收方Consumer作了抽象，这在RabbitMQ是没有的。不知道大家还记不记得RabbitMQ中消息的发送和接收的方式。在RabbitMQ中，发送是将指定完exchange、routing_key以及其他一些属性的message用channel.basic_publish方法来实现的，接收则是声明一个queue然后与exchange绑定，并指定callback函数。而在Kombu中消息的发送和接收则会更加简单和形象，更加的面向对象，相信大家在后面的例子中会强烈的感受到。

#### 主要特点
接下来，我们从整体上来了解一下Kombu的主要特点。

- 支持将不同的消息中间件以插件的方式进行灵活配置。Kombu中使用transport这个术语来表示一个具体的消息中间件（后续均用broker指代）。
	
	- transport使用py-amqp、librabbitmq、qpid-python等链接库来实现与RabbitMQ、Qpid等broker的连接。
	- 用C语言实现了一个高性能的rabbitmq链接库——librabbitmq
	- 引入transport这个抽象概念可以使得后续添加对non-AMQP的transport非常简单。当前Kombu中build-in支持有Redis、Beanstalk、Amazon SQS、CouchDB,、MongoDB,、ZeroMQ,、ZooKeeper、SoftLayer MQ和Pyro。
	- 同样，可以使用SQLAlchemy或Django ORM transport来将数据库当作broker使用
	- In-memory transport for unit testing（没理解= =）
- 支持对message的编码、序列化和压缩
- 对各种transport提供一致的异常处理
- 对connection和channle中的错误提供优雅的处理方案
最重要的就是对所有的broker进行了抽象——transport，为不同的broker提供了一致的解决方案。通过Kombu，开发者可以根据实际需求灵活的选择或更换broker。下表对当前主流broker之间进行了一些对比。
![Transport Comparison](/img/in-post/post10-kombu-1.png)

## 0x03 牛刀小试——Hello World
下面我们以一个Hello World程序作为本篇博客的结尾，看看Kombu是怎么帮助我们实现消息通信的。一些细节看不懂没关系，在后续我们会对里面的每个对象进行深入的学习。

首先是消息发送端hello_publisher.py：
{% highlight python linenos %}
from kombu import Connection
import datetime

# "amqp://guest:guest@localhost:5672//"中的amqp就是上文所提到的transport，
# 后面的部分是连接具体transport所需的参数，具体含义下篇博客中会讲到
with Connection('amqp://guest:guest@localhost:5672//') as conn:
    simple_queue = conn.SimpleQueue('simple_queue')
    message = 'helloword, sent at %s' % datetime.datetime.today()
    simple_queue.put(message)
    print('Sent: %s' % message)
    simple_queue.close()
{% endhighlight %}
然后是消息接收端hello_consumer.py：
{% highlight python linenos %}
from kombu import Connection

with Connection('amqp://guest:guest@localhost:5672//') as conn:
    simple_queue = conn.SimpleQueue('simple_queue')
    message = simple_queue.get(block=True, timeout=1)
    print("Received: %s" % message.payload)
    message.ack()
    simple_queue.close()
{% endhighlight%}
 从这个简单的程序中就可以体会到，Kombu的这个Hello World程序比[RabbitMQ的Hello World](/2015/12/05/six-steps-to-study-rabbitmq-1/)在易读性上要好很多，代码也更加整洁。
 
## 0x04 总结
有关Kombu的概述就到这里，想必看到这里，大家对Kombu这个python库有了一个大致的了解吧。在后续的博客中我将会对Kombu中重要的对象做出详尽的讲解，see you later!




