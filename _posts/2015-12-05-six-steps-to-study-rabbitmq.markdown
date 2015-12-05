---
layout:     post
title:      "六步学习RabbitMQ"
subtitle:   " Part 1  Hello World!"
date:       2015-12-05 17:00:00
author:     "Liuv"
header-img: "img/post-bg-2015-12-05.jpg"
tags:
    - AMQP
    - RabbitMQ
---

>  OpenStack架构中采用了异步消息模型来实现同一组件内不同服务之间的通信，默认使用RabbitMQ作为消息代理中间件。因此，掌握RabbitMQ是理解OpenStack工作原理的基础。

## 0x01 前言
<br>自从上篇博客发布后，嗖就一个月过去了，本来想弄的缓冲区溢出的专题也因为太懒而没弄了。想了想自己的主要方向还是OpenStack，缓冲区溢出纯属个人兴趣，于是决定关于缓冲区溢出的专题先放放，重心还是回到OpenStack上面来。
<p>
这几个月一直在调研OpenStack中的工作机制，打算自己做一个小项目。之前也看过keystone、nova中的一些源码，像用户认证流程、虚拟机的创建过程等都能理解。
但是，当到自己想做OpenStack的二次开发时，却发现之前理解的只是表面的过程，然而这是远远不够的。在各个组件的内部，还有着复杂的消息通信机制、线程并发控制、异常机制、包的发布机制、自动化测试过程等等。
今天，先把这段时间学习的有关消息通信的内容进行总结，与大家进行交流。
</p>
我的思路是这样的，今天会写一个RabbitMQ的专题，一共有六篇，内容由浅入深，主要是将<a href="http://www.rabbitmq.com/getstarted.html" target="_blank">RabbitMQ官网上的tutorials</a>进行了翻译，感兴趣的朋友也可以直接官网上去学习，毕竟有些概念不太好用中文翻译（水平有限哈）。

## 0x02 初窥RabbitMQ
<br>rabbitmq是一个消息代理中间件，功能是接收和转发消息。可以举一个邮局的例子：当你把邮件投到邮件箱时，你确定该邮件能够被邮递员准确的送到收件人那里。RabbitMQ就相当于邮箱，邮局和邮递员的合体，不同的是RabbitMQ传送的是数据块，能够自动将发送者的消息发送到指定的消息队列，然后对应的接受者会从消息队列中取出消息进行处理。
<br>为了下面能够更好的理解RabbitMQ的工作原理，先解释一下RabbitMQ中一些相关的术语:
<ul>
    <li>
    Producer：发送消息方，一般用P表示。
    </li>
    <li>
    Queue:消息队列。相当于邮箱，本质上是一个无大小限制的缓冲区。消息队列可以接受多个生产者的消息，多个消费者也可以共享一个消息队列。
    </li>
    <li>
    Consumer:消费者。一个等待接收消息的程序。一般用C表示。
    </li>
</ul>
<b><font color="red">注意：生产者、消费者和代理不必同驻在同一台物理机中。</font></b>
<br>接下来，开始上第一道菜了，老规矩，Hello world必须的。程序原理是这样的：Producer发送一个“Hello World!”字符串消息队列中，然后Consumer从消息队列中取出字符串并打印出来。示意图如下：
![img](/img/in-post/post3-hello-world.png)
首先是sending部分，发送一个消息到队列中。具体发送过程如下：
<ol type="i">
    <li>
    建立与rabbitmq服务器的connection
{% highlight python linenos %}
import pika
connection = pika.BlockingConnection(pika.ConnectionParameters(
            'localhost'))
channel = connection.channel()
{% endhighlight %}
    </li>
    <li>
    创建接受消息的队列“hello”。
{% highlight python linenos %}
channel.queue_declare(queue='hello')
{% endhighlight %}
    </li>
    <li>
    在rabbitmq中，一个消息不能直接发送到队列中，消息需要通过一个exchange。但是在这个例子中，我们只是使用一个默认的exchange，该exchange用空字符串命名，它允许我们制定一个特定的消息队列。
{% highlight python linenos %}
channel.basic_publish(exchange='',
                      routing_key='hello',
                      body='Hello World!')
print " [x] Sent 'Hello World!'"
{% endhighlight %}
    </li>
    <li>
    在退出程序之前，我们需要确定网络缓冲区被flushed并且消息已经被发送到rabbitmq中，我们可以关闭该connection来达到这个目的。
{% highlight python linenos %}
connection.close()
{% endhighlight %}
    </li>
</ol>
然后是receiving部分，主要功能是从hello队列中取出消息并打印到屏幕上，步骤如下：
<ol type="i">
    <li>
    建立与rabbitmq服务器的connection，代码和sending部分相同
    </li>
    <li>
    为了确保hello队列存在，我们需要再次创建一个名为hello的队列。事实上，我们可以多次创建该队列，而且最终只会创建一个。为什么需要重复声明消息队列呢？这是因为我们不确定那个程序会先运行，当然，如果你可以确定的话，当然能够只声明一次。
    </li>
    <li>
    创建一个回调函数。从消息队列中取消息的复杂度更大，它是通过为消息队列订阅一个回调函数来实现的。只要消息队列接收到消息，该回调函数就会被Pika库调用。
{% highlight python linenos %}
def callback(ch, method, properties, body):
    print " [x] Received %r" % (body,)
{% endhighlight %}
    </li>
    <li>
    将创建的回调函数与hello消息队列绑定，告诉rabbitmq该回调函数会从hello队列中接收消息。其中exchange表示交换器，能精确指定消息应该发送到哪个队列，routing_key设置为队列的名称，body就是发送的内容，具体发送细节暂时先不关注,后续的博客会详细讲解。
{% highlight python linenos %}
channel.basic_consume(callback,
                     queue='hello',
                      no_ack=True)
{% endhighlight %}
    </li>
    <li>
    让回调函数循环等待消息
{% highlight python linenos %}
print ' [*] Waiting for messages. To exit press CTRL+C'
channel.start_consuming()
{% endhighlight %}
    </li>
</ol>
好了，一个Hello World程序就这样完成，这个可以算是RabbitMQ中最简单的程序了，完整的代码我在第三部分贴出来。

## Ox03 总结
在第二部分，为了完整剖析程序的结构，将整个Hello World程序进行了分拆，现在再将它们合在一起，完整的Hello World代码如下：
首先是send.py
{% highlight python linenos %}
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()

channel.queue_declare(queue='hello')

channel.basic_publish(exchange='',
                      routing_key='hello',
                      body='Hello World!')
print " [x] Sent 'Hello World!'"
connection.close()
{% endhighlight %}

接下来是完整的receive.py
{% highlight python linenos %}
#!/usr/bin/env python
import pika
connection = pika.BlockingConnection(pika.ConnectionParameters(
        host='localhost'))
channel = connection.channel()
channel.queue_declare(queue='hello')
print ' [*] Waiting for messages. To exit press CTRL+C'
def callback(ch, method, properties, body):
    print " [x] Received %r" % (body,)
channel.basic_consume(callback,
                      queue='hello',
                      no_ack=True)
channel.start_consuming()
{% endhighlight %}


<br>好啦，总算写完了，顺便说一下，在64位系统中，传入函数的实参不再存储在栈里了，而是按规定存储在特定的寄存器中，但是基本原理都是一样的。下一篇博客就要开始正式介绍缓冲区溢出了。
<br>注：这篇博客是我在网上查阅了许多相关内容的博客后进行的总结，中间很多都是引用这些博客的内容，当然，我也加入了很多自己原创性的内容，欢迎拍砖。哈哈。







