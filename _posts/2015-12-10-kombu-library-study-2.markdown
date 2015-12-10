---
layout:     post
title:      "Kombu学习笔记（二）"
subtitle:   " Part II  Connections，Producers和Consumers对象"
date:       2015-12-10 23:00:00
author:     "Liuv"
header-img: "img/post-bg-2015-12-05.jpg"
tags:
    - AMQP
    - Kombu
---

>  Kombu is a messaging library for Python.The aim of Kombu is to make messaging in Python as easy as possible by providing an idiomatic high-level interface for the AMQ protocol, and also provide proven and tested solutions to common messaging problems.——摘自[kombu官网](http://kombu.readthedocs.org/en/latest/introduction.html)

## 0x01 前言
在本文中，我们会讲解Kombu中三个重要的对象：Connections、Producers和Consumers。它们的概念在[上一篇概述](/2015/12/10/kombu-library-study-1/)中已经介绍过了，简单说来，Connections是对broker的一个连接，Producers用来发送消息，Consumer用来接收并处理消息。废话不多说，直奔主题吧。

## 0x02 具体用法

#### Connections
在发送和接收消息之前，首先得需要指定一个具体的transport，并用该transport与broker建立connection。在Kombu中，用的最多的transport是amqp、librabbitmq、redis、qpid和in-memory。当然，你也可以自定义一个transport，Kombu默认的transport是amqp。下面是使用默认的amqp transport创建一个connection对象的方法：

```
from kombu import Connection
connection = Connection('amqp://guest:guest@localhost:5672//')
```
此时，与broker的连接<b>并没有</b>建立，事实上connection只会在需要的时候建立。你可以调用connect()方法来显式的建立与broker的连接

```
connection.connect()
```
可以查看connected属性来判断连接是否建立,使用完后要记得调用close()方法来关闭连接哦。下图是connection建立与关闭的演示：
[connection建立与关闭](/img/in-post/post11-kombu-1.png)
然而，关闭connection的最佳方式是调用release()方法，当connection是从connection pool中获得的，该方法会释放这个connection所占用的资源；如果connection是通过上面的方式直接建立的，那么该方法则会关闭该connection。

```
connection.release()
```
当然，总有人会忘记关闭connection，所以一个更好的创建connection的方式是使用python的with语句。

```
with Connection() as connection:
	#work with connection
```
下面我们来仔细分析一下声明Connection时传入的那一坨字符串，Kombu中把这称为URLs，格式如下：

```
transport://userid:password@hostname:port/virtual_host
```
根据这个格式，我们就很容易分析出上文使用的URL：'amqp://guest:guest@localhost:5672//'的含义了。它的意思是使用amqp这个默认的transport，也即是RabbitMQ。由于RabbitMQ在安装后，默认会创建一个user_id和password都为guest的用户，并监听5672这个端口，所以整个URL就为这个样子。

另外，官方也给出了一些合法URLs的示例：

```
# Specifies using the amqp transport only, default values
# are taken from the keyword arguments.
amqp://

# Using Redis
redis://localhost:6379/

# Using Redis over a Unix socket
redis+socket:///tmp/redis.sock

# Using Qpid
qpid://localhost/

# Using virtual host '/foo'
amqp://localhost//foo

# Using virtual host 'foo'
amqp://localhost/foo
```
URL的查询字段可以用来设置一些其他参数，例如下面的URL设置Connection使用ssl来传输消息。

```
amqp://localhost/myvhost?ssl=1
```
Connection类支持的其他参数列表如下,我就不一一翻译了：

|      keyword      |        interpretion       |
| :----------: | :---------------- |
|  hostname   | Default host name if not provided in the URL. |
|  userid   | Default user name if not provided in the URL.        |
|  password   | Default password if not provided in the URL.        |
| virtual_host | Default virtual host if not provided in the URL.        |
| port | Default port if not provided in the URL.        |
|  transport  | Default transport if not provided in the URL. Can be a string specifying the path to the class. (e.g. kombu.transport.pyamqp:Transport), or one of the aliases: pyamqp, librabbitmq, redis, qpid, memory, and so on.        |
|  ssl  | Use SSL to connect to the server. Default is False. Only supported by the amqp and qpid transports.           |
| insist | Insist on connecting to a server. No longer supported, relic from AMQP 0.8        |
| connect_timeout | Timeout in seconds for connecting to the server. May not be supported by the specified transport.        |
| transport_options | A dict of additional connection arguments to pass to alternate kombu channel implementations. Consult the transport documentation for available options.        |

接着来说说AMQP的transports。Kombu为我们提供了4个与AMQP相关的transport，分别是pyamqp 、librabbitmq 、amqp 、qpid 。

1. pyamqp：使用纯python库——amqp来与AMQP broker进行连接，随着Kombu一起被安装
2. librabbitmq:用C写的一个高性能连接amqp broker的链接库，可以看作pyamqp的高新能版
3. amqp：首先会尝试用librabbitmq去连接rabbitmq，如果失败则再使用pyamqp去连接
4. qpid：使用纯python库——qpid.messaging来与broker进行连接，随着Kombu一起被安装。Qpid库虽然也使用的是amqp，但是针对Qpid broker的一些特点作了扩展。

#### Producers
Producers是消息的发送者，其初始化方法为：

```
 class kombu.Producer(channel, exchange=None, routing_key=None, serializer=None, auto_declare=None, compression=None, on_return=None)
```
构造函数中的每个参数含义如下：

- channel——上面建立的connection，或者是比较轻量级的channel
- exchange——默认发送到的exchange，指定默认值后每次发送消息时就不用再制定exchange了
- routing_key——默认的routing key，会的被附加到message上
- serializer——默认的serializer。默认是json格式（就是用来指明message的格式）
- compression——默认（针对所有消息而言）的消息的压缩方法。默认（针对该参数而言）是不压缩
- auto_declare——是否在建立的时候声明exchange（也就是说，如果exchange还没有被建立，那么把这个设置为true就能建立一个）.默认值为True
- on_return——消息发送失败时的回调函数。

在Producer类中有下面几个重要的方法：

- declare()：用于声明exchange
- publish(body, routing_key=None, delivery_mode=None, mandatory=False, immediate=False, priority=0, content_type=None, content_encoding=None, serializer=None, headers=None, compression=None, exchange=None, retry=False, retry_policy=None, declare=, []**properties)：发送一个消息，几个重要形参的含义如下：
    - body-消息的内容
    - routing_key-消息的routing key，exchange在对该消息进行路由时需要用到
    - priority-消息的优先级，0-9之间的数字
    - content-type-消息的内容类型。默认是auto-detect
    - contente_encoding-消息内容编码方式。默认是auto-detect
    - serializer-所使用的序列化方法。默认是auto-detect
    - compression-所使用的压缩方法。默认不压缩
    - exchange-消息发送到的exchange，该值会覆盖声明Producer时传入的exchange
    - retry-连接失败后重试的最大次数
    - retry-policy-连接失败重试相关的配置，由ensure()方法支持
    - expiration-每个消息TTL的失效期（单位为秒）。默认永不失效
    - **properties-额外的消息属性
- revive(channel):连接丢失后重启producer

#### Consumers



## 0x03 牛刀小试——Hello World
下面我们以一个Hello World程序作为本篇博客的结尾，看看Kombu是怎么帮助我们实现消息通信的。一些细节看不懂没关系，在后续我们会对里面的每个对象进行深入的学习。

首先是消息发送端hello_publisher.py：
{% highlight python linenos %}
from kombu import Connection
import datetime

# "amqp://guest:guest@localhost:5672//"就是上文所提到的transport，
# 在这里代表的是使用RabbitMQ作为broker
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
 从这个简单的程序中就可以体会到，Kombu的这个Hello World程序比[RabbitMQ的Hello World](/2015/12/06/six-steps-to-study-rabbitmq-1/)在易读性上要好很多，代码也更加整洁。
 
## 0x04 总结
有关Kombu的概述就到这里，想必看到这里，大家对Kombu这个python库有了一个大致的了解吧。在后续的博客中我将会对Kombu中重要的对象做出详尽的讲解，see you later!




