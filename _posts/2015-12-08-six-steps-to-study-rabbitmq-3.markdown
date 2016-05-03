---
layout:     post
title:      "六步学习RabbitMQ（三）"
subtitle:   " Step 3  Work Queues"
date:       2015-12-08 17:00:00
author:     "Liuv"
header-img: "img/post-bg-2015-12-05.jpg"
tags:
    - AMQP
    - RabbitMQ
---

>  OpenStack架构中采用了异步消息模型来实现同一组件内不同服务之间的通信，默认使用RabbitMQ作为消息代理中间件。因此，掌握RabbitMQ是理解OpenStack工作原理的基础。

## 0x01 前言
在前面的两种模式中，每个消息对应一个worker，如果要想一个消息发给多个consumer该怎么办？那就需要使用`publish/subscribe`模式。
为了更好的说明该模式，我们将编写一个简单的日记系统，它包括两个程序，一个提交日志信息，另一个接收日志信息并打印。在该日志系统中，每个后台运行的consumer都会收到日志信息，其中一个consumer直接将日志信息写到磁盘中，另一个consumer则将日志信息打印屏幕上。本质上，发布的日志信息会广播给所有的consumer。

## 0x02 相关概念
在介绍模式前，首先需要引入一个新的概念——交换机（exchange）。rabbitmq的核心思想是:producer不直接将消息发到队列，事实上，producer甚至不知道消息将会发到哪个消息队列中。相反的，producer只需要将消息发送给交换机。一方面，交换机接受来自producer的消息，另一方面，交换机将消息push到对应的消息队列中，那么交换机该将信息发送到哪个消息队列中呢？这得由交换机的类型来决定，在rabbitmq中，交换机一共有`direct`、`topic`、`headers`和`fanout`这四种类型。在这个例子中，我们使用fanout类型的交换机。声明如下:

```
channel.exchange_declare(exchange='logs',type='fanout')
channel.basic_publish(exchange='logs',
                      routing_key='',
                      body=message)
```

fanout交换机很简单，它会把接收到的消息广播到所有它知道的队列中。
在本日志系统中，我们不关心消息队列的名字，虽然它很重要，此时我们可以使用临时队列——让rabbitmq自动生成名字的消息队列，声明如下：

```
result = channel.queue_declare(exclusive=True)
```

可以用result.method.queue变量来得到随机生成的队列名。Exclusive=True保证consumer连接中断后对应的消息队列被删除。
声明完exchange和消息队列后，一定要对消息队列和exchange进行绑定（bind），这样exchange才会将收到的消息按照特定的方式转发到与之绑定的消息队列中。绑定语句如下:

```
channel.queue_bind(exchange='logs',queue=result.method.queue)
```

## 0x03 代码整理
发送端的程序Emit_log.py:
{% highlight python %}
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='logs',
                         type='fanout')

message = ' '.join(sys.argv[1:]) or "info: Hello World!"
channel.basic_publish(exchange='logs',
                      routing_key='',
                      body=message)
print " [x] Sent %r" % (message,)
connection.close()
{% endhighlight %}
接收端的程序Receive_logs.py:
{% highlight python %}
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='logs',
                         type='fanout')

result = channel.queue_declare(exclusive=True)
queue_name = result.method.queue

channel.queue_bind(exchange='logs',
                   queue=queue_name)

print ' [*] Waiting for logs. To exit press CTRL+C'

def callback(ch, method, properties, body):
    print " [x] %r" % (body,)

channel.basic_consume(callback,
                      queue=queue_name,
                      no_ack=True)

channel.start_consuming()
{% endhighlight %}
RabbitMQ的publish/subscribe模式现在已经完全实现了，在下一篇，我将会讲解Routing模型的实现，see you later!






