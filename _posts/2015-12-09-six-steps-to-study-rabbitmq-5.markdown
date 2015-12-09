---
layout:     post
title:      "六步学习RabbitMQ（五）"
subtitle:   " Step 5  Topics"
date:       2015-12-09 14:00:00
author:     "Liuv"
header-img: "img/post-bg-2015-12-05.jpg"
tags:
    - AMQP
    - RabbitMQ
---

>  OpenStack架构中采用了异步消息模型来实现同一组件内不同服务之间的通信，默认使用RabbitMQ作为消息代理中间件。因此，掌握RabbitMQ是理解OpenStack工作原理的基础。

## 0x01 前言
本节将要对[上篇：Routing模式](/2015/12/09/six-steps-to-study-rabbitmq-4/)实现的日志系统进行改进，不仅要基于安全等级来订阅日志，还要基于产生日志的来源来订阅日志。例如我们想要即监听来自于“cron”的error信息，也要监听来自于“kern”的所有日志信息，为了实现这样的日志系统，我们将使用topic类型的exchange。和direct exchange的声明类似，topic exchange的声明只需将type属性指定为topic。

```
channel.exchange_declare(exchange='topic_logs',
                         type='topic')
```

这样就声明了一个名为“topic_logs”的topic类型的exchange。下面我们将详细了解topic exchange的用法。

## 0x02 相关概念——topic exchange
发往topic exchange的消息不能随意的设置选择键（routing_key），必须是由点隔开的一系列的标识符组成。标识符可以是任何东西，但是一般都与消息的某些特性相关。一些合法的选择键的例子："stock.usd.nyse", "nyse.vmw","quick.orange.rabbit".你可以定义任何数量的标识符，上限为255个字节。

绑定键（binding_key）和选择键的形式一样。Topic exchange背后的逻辑和direct exchange很类似：一个附带特定的routing key的消息将会被转发到与之匹配的binding key对应的队列中。需要注意的是：关于绑定键有两种特殊的情况:
<ul>
	<li>
		`*`:可以匹配一个标识符。
	</li>
	<li>
		`#`:可以匹配0个或多个标识符。
	</li>
</ul>
为了便于理解，先来看一个发送动物消息的例子。现在，我们准备发送关于动物的消息。消息会附加一个包含3个标识符（两个点隔开）的routing_key。第一个标识符描述动物的速度，第二个标识符描述动物的颜色，第三个标识符描述动物的物种,形式如下：

```
<speed>.<color>.<species>
```

我们创建2个消息队列Q1和Q2，Q1用"*.orange.*"绑定topic exchange，Q2用"*.*.rabbit"和"lazy.#"绑定topic exchange，整体结构如下：
![动物消息的AMQP模型](/img/in-post/post8-topic-1.png)
可以简单的认为:Q1对所有的橙色动物感兴趣。Q2想要知道关于兔子的一切以及关于懒洋洋的动物的一切。

一个附带quick.orange.rabbit的routing_key的消息将会被转发到两个队列。附带lazy.orange.elephant的消息也会被转发到两个队列。另一方面quick.orange.fox只会被转发到Q1，lazy.brown.fox将会被转发到Q2。lazy.pink.rabbit虽然与两个binding_key匹配，但是也只会被转发到Q2。quick.brown.fox不能与任何绑定键匹配，所以会被丢弃。

如果我们违反我们的约定，发送一个或者四个标识符的选择键，如orange，quick.orange.male.rabbit这些选择键，不能与任何绑定键匹配，所以消息将会被丢弃。另一方面，lazy.orange.male.rabbit虽然是四个标识符，然而可以与lazy.#匹配，因此会被转发至Q2。

#### `注意`：
<ol type="1">
	<li>
		当一个队列与绑定键#绑定，将会收到所有的消息，类似fanout类型exchange。
	</li>
	<li>
		当绑定键中不包含任何#与*时，类似direct类型exchange。
	</li>
</ol>

## 0x03 代码整理
发送端的程序emit_log_topic.py:
{% highlight python linenos %}
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='topic_logs',
                         type='topic')

routing_key = sys.argv[1] if len(sys.argv) > 1 else 'anonymous.info'
message = ' '.join(sys.argv[2:]) or 'Hello World!'
channel.basic_publish(exchange='topic_logs',
                      routing_key=routing_key,
                      body=message)
print " [x] Sent %r:%r" % (routing_key, message)
connection.close()
{% endhighlight %}
接收端的程序receive_logs_topic.py:
{% highlight python linenos %}
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='topic_logs',
                         type='topic')

result = channel.queue_declare(exclusive=True)
queue_name = result.method.queue

binding_keys = sys.argv[1:]
if not binding_keys:
    print >> sys.stderr, "Usage: %s [binding_key]..." % (sys.argv[0],)
    sys.exit(1)

for binding_key in binding_keys:
    channel.queue_bind(exchange='topic_logs',
                       queue=queue_name,
                       routing_key=binding_key)

print ' [*] Waiting for logs. To exit press CTRL+C'

def callback(ch, method, properties, body):
    print " [x] %r:%r" % (method.routing_key, body,)

channel.basic_consume(callback,
                      queue=queue_name,
                      no_ack=True)

channel.start_consuming()
{% endhighlight %}
相比于fanout和direct类型的exchange，topic exchange可以实现更加灵活的消息路由。在下一篇，也是RabbitMQ系列的最后一篇，我将会用RabbitMQ实现一个RPC，里面会涉及到前面讲过的大部分内容，see you later!






