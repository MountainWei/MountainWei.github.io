---
layout:     post
title:      "六步学习RabbitMQ（二）"
subtitle:   " Step 2  Work Queues"
date:       2015-12-07 19:00:00
author:     "Liuv"
header-img: "img/post-bg-2015-12-05.jpg"
tags:
    - AMQP
    - RabbitMQ
---

>  OpenStack架构中采用了异步消息模型来实现同一组件内不同服务之间的通信，默认使用RabbitMQ作为消息代理中间件。因此，掌握RabbitMQ是理解OpenStack工作原理的基础。

## 0x01 前言
<br>在上一篇博客中，我们实现了一个简单的Hello World程序，对RabbitMQ中消息的发送和接收有了基本的了解。接下来，我们更进一步学习RabbitMQ，实现一个Work Queue模型。
<br>Work queues的主要思想是避免马上处理队列中的任务，这样就必须等待一段时间直到该任务执行完后才能继续执行下一个任务。相反的，可以先对任务进行调度，然后再执行。也就是说可以在后端跑多个worker，当任务到达时，先对任务进行调度，分配到指定的worker，然后再执行。这在web应用中很重要，因为在一个短的HTTP请求窗口期间不可能处理一个复杂的任务。

## 0x02 Work Queue实现
<br>首先需要进行一些准备，为了模仿一个复杂的任务，我们使用time.sleep（）函数来控制一个任务的执行时间，发送的字符串中有多少个小数点就要执行time.sleep（）多少次，这样就可以模仿那些耗时长的复杂任务。
<br>大部分代码还是重用上一篇文章中的代码，为了便于理解，在发送端将send.py文件名改为net_task.py,并支持用户输入字符串。代码如下：
{% highlight python %}
import sys

message = ' '.join(sys.argv[1:]) or "Hello World!"
channel.basic_publish(exchange='',
                      routing_key='hello',
                      body=message)
print " [x] Sent %r" % (message,)
{% endhighlight %}
在接收端，receive.py文件名改为worker.py，在callback函数中解析接收到的字符串，有多少个小数点就sleep多少秒。代码如下：
{% highlight python %}
import time

def callback(ch, method, properties, body):
    print " [x] Received %r" % (body,)
    time.sleep( body.count('.') )
    print " [x] Done"
{% endhighlight %}
使用消息队列的优点就是方便并行处理，当任务数量过大产生积压时，可以通过增加worker数量来增加弹性。在本次实验中，打开3个窗口，一个P,两个C，先打开C1,C2：
shell1$ python worker.py
 [*] Waiting for messages. To exit press CTRL+C

shell2$ python worker.py
 [*] Waiting for messages. To exit press CTRL+C
再打开P：
shell3$ python new_task.py First message.
shell3$ python new_task.py Second message..
shell3$ python new_task.py Third message...
shell3$ python new_task.py Fourth message....
shell3$ python new_task.py Fifth message.....

C1和C2的结果:
shell1$ python worker.py
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'First message.'
 [x] Received 'Third message...'
 [x] Received 'Fifth message.....'

shell2$ python worker.py
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'Second message..'
 [x] Received 'Fourth message....'
<br>可以看到，rabbitmq默认的是轮询调度，会将消息按顺序的发送到下一个consumer。平均每个consumer会得到相同数量的消息。</br>

## 0x03 改进——消息回复
在当前的代码中，当worker死掉后，正在处理的所有消息将会丢失，那些已经被分到该worker中但还没来得及处理的消息也是一样。好的方案是当一个worker死了后，我们可以将任务分配到另一个worker中。为了实现这个方案，RabbitMQ支持消息回复。当一个消息被处理完后consumer会返回给rabbitmq一个ack，告诉rabbitmq可以释放并删除该消息。如果consumer死亡，没有发送ack，rabbitmq将会认为该消息没有被处理完，并会重新将消息分配给另一个consumer。这样，即使consumer死掉也能保证消息不会丢失。该机制不会带来任何消息延迟。重分配只发生在worker的连接丢失。消息回复机制默认是打开的，我们可以通过no_ack=True来手动的关闭。Worker.py修改如下：
{% highlight python %}
def callback(ch, method, properties, body):
    print " [x] Received %r" % (body,)
    time.sleep( body.count('.') )
    print " [x] Done"
    ch.basic_ack(delivery_tag = method.delivery_tag)

channel.basic_consume(callback,
                      queue='hello')
{% endhighlight %}
<font color="red">注意：不要忘记添加basic_ack函数，这个错误很容易犯，而且后果很严重。它会导致当客户端退出时消息重现分配，这样rabbitmq将会在内存中保留这些信息从而占用大量内存，并且不会释放任何没有回复的消息。</font>

## Ox04 改进——消息持久化
我们已经可以保证当consumer死亡消息也不会丢失，但是如果rabbitmq服务器停止后我们的消息还是会丢失。当rabbitmq退出或宕机时，它将会丢失所有的队列和消息。为了保证消息不会丢失，我们需要做两件事情：将对列和消息标记为durable
先声明队列为durable:
{% highlight python %}
channel.queue_declare(queue='task_queue', durable=True)
{% endhighlight %}
此时队列名词为“task_queue”而不是‘hello'，因为我们已经定义了一个undurable的“hello”队列，然而rabbitmq不允许重新设置一个已存在的队列的属性，所以必须重现声明一个durable的队列。
<br>接着，声明消息为persistent,通过定义delivery_mode为2来实现:
{% highlight python %}
channel.basic_publish(exchange='',
                      routing_key="task_queue",
                      body=message,
                      properties=pika.BasicProperties(
                         delivery_mode = 2, # make message persistent
                      ))
{% endhighlight %}
<font color="red">注意：消息声明为persistence后并不能百分百保证不会丢失，因为rabbitmq接受消息有一个短暂的时间窗口，在该窗口内消息是没有保存的。此外，rabbitmq并不是对每个消息进行磁盘同步的，它们首先会被保存在内存中，实际上并未马上写到磁盘中。</font>

## 0x05 改进——公平调度
轮询调度并不能保证每个每个任务被合理的分配。例如，在有2个worker的条件下，所有的奇数号任务都很大，而所有的偶数号任务都很小，那么采用轮询调度就会导致一个woker总是处于busy状态，而另一个任务则会很空闲。
<br>为了避免这种情况发生，我们可以使用basic.qos方法，将prefetch_count赋值为1。它会告诉rabbitmq不要一次给一个worker分配超过一个的任务，只有当任务完成后再分配下一个任务。声明语句如下：
{% highlight python %}
channel.basic_qos(prefetch_count=1)
{% endhighlight %}

## 0x06 代码整理
整理后的代码如下：
完整的New_task.py如下：
{% highlight python %}
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.queue_declare(queue='task_queue', durable=True)

message = ' '.join(sys.argv[1:]) or "Hello World!"
channel.basic_publish(exchange='',
                      routing_key='task_queue',
                      body=message,
                      properties=pika.BasicProperties(
                         delivery_mode = 2, # make message persistent
                      ))
print " [x] Sent %r" % (message,)
connection.close()
{% endhighlight %}

接下来是完整的Worker.py：
{% highlight python linenos %}
#!/usr/bin/env python
import pika
import time

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.queue_declare(queue='task_queue', durable=True)
print ' [*] Waiting for messages. To exit press CTRL+C'

def callback(ch, method, properties, body):
    print " [x] Received %r" % (body,)
    time.sleep( body.count('.') )
    print " [x] Done"
    ch.basic_ack(delivery_tag = method.delivery_tag)

channel.basic_qos(prefetch_count=1)
channel.basic_consume(callback,
                      queue='task_queue')

channel.start_consuming()
{% endhighlight %}
<br>RabbitMQ的Work Queue现在已经完全实现了，在下一篇，我将会讲解publish-subscribe模型的实现，see you later!






