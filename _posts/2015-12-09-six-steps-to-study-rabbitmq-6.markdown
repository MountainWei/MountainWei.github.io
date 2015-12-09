---
layout:     post
title:      "六步学习RabbitMQ（六）"
subtitle:   " Step 6  RPC"
date:       2015-12-09 16:00:00
author:     "Liuv"
header-img: "img/post-bg-2015-12-05.jpg"
tags:
    - AMQP
    - RabbitMQ
---

>  OpenStack架构中采用了异步消息模型来实现同一组件内不同服务之间的通信，默认使用RabbitMQ作为消息代理中间件。因此，掌握RabbitMQ是理解OpenStack工作原理的基础。

## 0x01 前言
在[Rabbit系列第二篇](/2015/12/07/six-steps-to-study-rabbitmq-2/)中，我们了解到使用work queue在多个worker中分发耗时任务。但是，如果我们想要调用远程计算机中的函数并等待返回结果（也就是我们常说的Remote Procedure Call，简称RPC），这时用RabbitMQ该如何实现呢？okay，在这篇博客中，我们就来用RabbitMQ实现一个RPC系统，该系统由一个Client和可弹性扩展的RPC server组成，为了简单起见，RPC服务是一个返回斐波那契数的程序。

## 0x02 相关概念

#### Client interface
为了更好的展示一个RPC服务是如何调用的，我们新增了一个简单的client class。在该class中定义一个call方法，用来发送RPC请求并等待结果返回，需要注意的是，整个过程是同步的，也即client在结果返回前，一直处于阻塞状态。client的使用方式如下：

```
fibonacci_rpc = FibonacciRpcClient()
result = fibonacci_rpc.call(4)
print "fib(4) is %r" % (result,)
```

<b>关于RPC的一些想法:</b>尽管RPC在现在的计算机系统中使用广泛，但是它也常常引来批评。特别的，当你无法判断某个函数调用是否是本地调用，抑或RPC调用过程时间很长时，在这种情况下使用RPC会导致系统的运行状态的不确定性，并会为调试带来不必要的复杂度。因此，关于RPC的使用我有一下建议：
<ul>
	<li>
		明确函数调用是本地调用还是远程调用
	</li>
	<li>
		为系统编制详细的文档说明，确保组件之间依赖关系的清晰性。
	</li>
	<li>
		做好异常处理。例如当RPC server长时间崩溃时client该采取何种措施。
	</li>
</ul>

#### Callback queue
在前面讲述的五种通信模式中，Producer只负责发送消息，所做的工作仅仅是连上RabbitMQ服务器，将消息的routing_key设置好后发送到指定的exchange中，剩下的工作就与Producer无关了。然而，在RPC模式中，client发送完消息后，还需要等待接收response，因此client需要建立callback queue来接收response。client发送消息的过程如下：
{% highlight python linenos %}
result = channel.queue_declare(exclusive=True)
callback_queue = result.method.queue

channel.basic_publish(exchange='',
                      routing_key='rpc_queue',
                      properties=pika.BasicProperties(
                            reply_to = callback_queue,
                            ),
                      body=request)

# ... and some code to read a response message from the callback_queue ...
{% endhighlight %}
可以看到，我们在发送message时使用了“reply_to”属性，该属性的值就是client用来接受response消息的callback queue名。AMQP协议为message预定义了14个属性，然而大部分属性都很少被使用，常用的只有一下4个：

-  delivery_mode:当值为2时，说明该message是一个持久消息；若是其他值，则是临时消息（不知道怎么翻译，transient message）。我们在[Rabbit系列第二篇](/2015/12/07/six-steps-to-study-rabbitmq-2/)中使用这个属性来持久化一个消息，这样可以保证消息在worker出现异常时不会丢失。
-  content_type:用于描述编码的mime-type。例如，当content_type为application/json时，就是在传输消息时使用JSON格式。
-  reply_to:用于命名一个callback queue
-  correlation_id:用来将RPC response与对应的request绑定，下面会详细介绍它的用法。

#### Correlation id
按照上面的方式发送消息时，每个RPC request都会创建一个callback queue，这会造成整个RPC系统的效率会很低。我们可以用一个更合适的方法——为每个client建立一个唯一的callback queue。

但是，这样会引入一个新的问题，那就是当client从callback queue收到一个response后，无法判断该response对应的是哪一个request。这时候，correlation_id属性就派上用场了。

我们为每一个request赋予唯一的correlation_id，当client收到一个response时，通过response中的correlation_id值就可以找到与之对应的request。如果收到了一个未知的correlation_id值的response，说明该response不属于client发出的任何请求，就可以直接丢弃。

大家对于未知的response的处理方式可能会有疑问？为什么是直接丢弃而不是报错呢？这样处理主要是考虑到RPC server端可能会出现的一种情况：server刚刚向callback queue发送完response，还没来得及发送ack消息就down掉了，这样当RPC server重启后会重新发送response。因此，丢弃未知的response是一种更加优雅的做法。

#### summary
如下图，整个RPC系统的工作流程如下：

1. Client初始化时，会创建一个匿名的exclusive回调队列
2. 对于一个RPC请求，Client发送带有两个属性的消息：
	1.reply_to		: 指定回调队列的名字
	2.correlation_id：指定每个request的唯一值，当client在callback queue
					中接收到response后，会查询消息的correlation_id属性，
					从而能够找到与之相匹配的request。
3. request会送到rpc_queue队列中
4. server循环等待rpc_queue队列中的request,当有request到达时，它会调用相应的函数进行处理，并将处理结果用消息发回Client，此时使用reply_to指定的消息队列进行接收
5. client等待来自callback queue的response。当有消息到达时，client会检查该消息的correlation_id属性，如果与request中的值匹配，那么则会返回response中的结果。
![RPC系统示意图](/img/in-post/post9-rpc-1.png)

## 0x03 代码整理
RPC client端的程序rpc_client.py:
{% highlight python linenos %}
#!/usr/bin/env python
import pika
import uuid

class FibonacciRpcClient(object):
    def __init__(self):
        self.connection = pika.BlockingConnection(pika.ConnectionParameters(
                host='localhost'))

        self.channel = self.connection.channel()

        result = self.channel.queue_declare(exclusive=True)
        self.callback_queue = result.method.queue

        self.channel.basic_consume(self.on_response, no_ack=True,
                                   queue=self.callback_queue)

    def on_response(self, ch, method, props, body):
        if self.corr_id == props.correlation_id:
            self.response = body

    def call(self, n):
        self.response = None
        self.corr_id = str(uuid.uuid4())
        self.channel.basic_publish(exchange='',
                                   routing_key='rpc_queue',
                                   properties=pika.BasicProperties(
                                         reply_to = self.callback_queue,
                                         correlation_id = self.corr_id,
                                         ),
                                   body=str(n))
        while self.response is None:
            self.connection.process_data_events()
        return int(self.response)

fibonacci_rpc = FibonacciRpcClient()

print " [x] Requesting fib(30)"
response = fibonacci_rpc.call(30)
print " [.] Got %r" % (response,)
{% endhighlight %}
RPC server端的程序rpc_server.py:
{% highlight python linenos %}
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))

channel = connection.channel()

channel.queue_declare(queue='rpc_queue')

def fib(n):
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        return fib(n-1) + fib(n-2)

def on_request(ch, method, props, body):
    n = int(body)

    print " [.] fib(%s)"  % (n,)
    response = fib(n)

    ch.basic_publish(exchange='',
                     routing_key=props.reply_to,
                     properties=pika.BasicProperties(correlation_id = \
                                                     props.correlation_id),
                     body=str(response))
    ch.basic_ack(delivery_tag = method.delivery_tag)

channel.basic_qos(prefetch_count=1)
channel.basic_consume(on_request, queue='rpc_queue')

print " [x] Awaiting RPC requests"
channel.start_consuming()
{% endhighlight %}

## 0x04 总结
好的，关于RPC的学习就到这里，rabbitmq系列所有的博客也到此为止了，这系列的六篇博客弄的比较匆忙，一共只花了四天的时间，所以后续我还会不断完善，例如为每个模型的实现附上运行结果，对各种模型做一些概念上的说明等等。虽说OpenStack中默认采用rabbitmq进行不同服务之间的通信，但是为了提高可扩展性，OpenStack对RabbitMQ进行了多层封装，在后续的博客里我会将这些封装层层解开，尽我最大的能力将OpenStack中的消息通信机制给分析清楚，敬请期待~~





