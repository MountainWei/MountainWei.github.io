---
layout:     post
title:      "六步学习RabbitMQ（四）"
subtitle:   " Step 4  Routing"
date:       2015-12-09 13:00:00
author:     "Liuv"
header-img: "img/post-bg-2015-12-05.jpg"
tags:
    - AMQP
    - RabbitMQ
---

>  OpenStack架构中采用了异步消息模型来实现同一组件内不同服务之间的通信，默认使用RabbitMQ作为消息代理中间件。因此，掌握RabbitMQ是理解OpenStack工作原理的基础。

## 0x01 前言
在本节中，我们将会只订阅消息的子集，而不是订阅所有消息，fanout交换机不再适合，我们将选择direct交换机。direct交换机背后的算法也比较简单——消息将会分配给binding key和消息的routing keys相同的队列。在队列与交换机的绑定中，可以指定binding key。direct类型的声明方法如下：

```
channel.exchange_declare(exchange='direct_logs',
                         type='direct')
```

这样就声明了一个名为“direct_logs”的direct类型的exchange。

## 0x02 相关概念
`Routing key`:每个接收端的消息队列在绑定exchange的时候，可以设定相应的路由键。发送端通过exchange发送信息时，可以指定routing key，交换机会根据routing key把消息发送到预先绑定的相应的消息队列，这样接收端就能接收到消息了。绑定代码如下:

```
channel.queue_bind(exchange=exchange_name,
                   queue=queue_name,
                   routing_key='black')
```

`注意：Binding key的意义取决于具体exchaneg的类型，每个队列可以绑定多个binding_key。`
接下来，我们要实现一个对不同安全级的日志进行分别进行处理的日志处理程序。先来看一下程序的示意图：
![routing示意图](/img/in-post/post7-routing-1.png)
在上图的绑定中，队列一与exchange的binding_key为“error”，队列二则绑定“info”、“error”、“warning”。这样C1只会处理error安全级的日志，C2则会处理各种安全级的日志。

## 0x03 代码整理
发送端的程序Emit_log_direct.py:
{% highlight python linenos %}
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='direct_logs',
                         type='direct')

severity = sys.argv[1] if len(sys.argv) > 1 else 'info'
message = ' '.join(sys.argv[2:]) or 'Hello World!'
channel.basic_publish(exchange='direct_logs',
                      routing_key=severity,
                      body=message)
print " [x] Sent %r:%r" % (severity, message)
connection.close()
{% endhighlight %}
接收端的程序receive_logs_direct.py:
{% highlight python linenos %}
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='direct_logs',
                         type='direct')

result = channel.queue_declare(exclusive=True)
queue_name = result.method.queue

severities = sys.argv[1:]
if not severities:
    print >> sys.stderr, "Usage: %s [info] [warning] [error]" % \
                         (sys.argv[0],)
    sys.exit(1)

for severity in severities:
    channel.queue_bind(exchange='direct_logs',
                       queue=queue_name,
                       routing_key=severity)

print ' [*] Waiting for logs. To exit press CTRL+C'

def callback(ch, method, properties, body):
    print " [x] %r:%r" % (method.routing_key, body,)

channel.basic_consume(callback,
                      queue=queue_name,
                      no_ack=True)

channel.start_consuming()
{% endhighlight %}
这样，一个简单的Routing模式的RabbitMQ程序就实现了。在下一篇，我将会讲解Topic类型的exchange的用法，see you later!






