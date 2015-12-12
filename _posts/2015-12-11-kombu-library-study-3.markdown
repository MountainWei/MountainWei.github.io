---
layout:     post
title:      "Kombu学习笔记（三）"
subtitle:   " Part III  重要对象介绍续和Work Queue的实现"
date:       2015-12-11 17:00:00
author:     "Liuv"
header-img: "img/post-bg-2015-12-05.jpg"
tags:
    - AMQP
    - Kombu
---

>  Kombu is a messaging library for Python.The aim of Kombu is to make messaging in Python as easy as possible by providing an idiomatic high-level interface for the AMQ protocol, and also provide proven and tested solutions to common messaging problems.——摘自[kombu官网](http://kombu.readthedocs.org/en/latest/introduction.html)

## 0x01 前言
在[上一篇：Kombu学习笔记（二）](/2015/12/11/kombu-library-study-2/)中，我介绍了Kombu中Connection、Producer、Consumer这三个重要的对象的使用方法。接下来，在本文中，我们继续来学习另外几个重要的对象，并在最后展示一个Work Queue的例子。

## 0x02 重要对象介绍

#### Connection Pools
Kombu提供了两个全局的资源池——connection pool 和 producer pool。先来看看connection pool吧，直接上代码：
![建立连接池](/img/in-post/post11-kombu-2.png)
从上图中可以看出，我们使用<b>kombu.pool.connections</b>这个对象来新建一个connection pool，需要指定一个connection实例。`值得注意的是，相同的connection实例创建的connection pool是同一个，`这个可以从上图中可以看出。

当然，创建connection pool的更优雅的一种方法是使用with语句：

```
from kombu import Connection
from kombu.pools import connections

connection = Connection('redis://localhost:6379')

with connections[connection].acquire(block=True) as conn:
    print('Got connection: %r' % (connection.as_uri(), ))
```
“block=True”意味着当pool中没有可用的connection时，所有新的获取connection请求都会被阻塞，直到有可用的connection。如果代码中没有严格的处理connection的获取和释放，则会造成死锁，这时就需要设置timeout参数来防止死锁的发生。另外，如果block参数设置为False，那么当池中没有可用的connection时，新的获取connection请求会抛出<b>kombu.exceptions.ConnectionLimitExceeded</b>异常。

当然，你可以一次连接多个broker，方法如下：

```
from kombu import Connection
from kombu.pools import connections

c1 = Connection('amqp://')
c2 = Connection('redis://')

with connections[c1].acquire(block=True) as conn1:
    with connections[c2].acquire(block=True) as conn2:
        # ....
```

#### Producer Pool
Producer Pool类似于Connection Pool，不过它用来管理Producer实例。下面的程序使用producer pool发布一个message到名为news的exchange中：

```
from kombu import Connection, Exchange
from kombu.pools import producers, connections

# The exchange we send our news articles to.
news_exchange = Exchange('news')

# The article we want to send
article = {'title': 'No cellular coverage on the tube for 2012',
           'ingress': 'yadda yadda yadda'}

# The broker where our exchange is.
connection = Connection('amqp://guest:guest@localhost:5672//')

with connections[connection].acquire[block=True) as conn:
	with producers[conn].acquire(block=True) as producer:
    	producer.publish(
        	article,
        	exchange=new_exchange,
        	routing_key='domestic',
        	declare=[news_exchange],
        	serializer='json',
        	compression='zlib')
```
默认情况下，每个connection pool支持的最大connection数为200，可以使用kombu.pools.set_limit()来改变最大connection数的限制。在runtime期间，我们可以增加pool的大小，但是无法缩小，所以要尽可能早的设置pool的大小。

```
>>> from kombu import pools
>>> pools.set_limit()
```
Kombu还支持开发者自定义Connection and Producer Pools。方法如下;

```
from kombu import pools
from kombu import Connection

connections = pools.Connections(limit=100)
producers = pools.Producers(limit=connections.limit)

connection = Connection('amqp://guest:guest@localhost:5672//')

with connections[connection].acquire(block=True):
    # ...
```

#### Serializers
众所周知，message传输前后需要被序列化和反序列化。Kombu默认使用JSON格式对message进行编码，因此Python中的字典对象和list对象都可以直接作为message进行发送和接收。此外，还支持YAML,msgpack和Python内建的pickle模块。开发者还可以自己注册任何自定义的序列化格式。

默认情况下，Kombu只识别JSON格式的message，如果想使用其他序列化的格式，则必须显式的在consumer中使用accept参数来声明。

```
Consumer(conn, [queue], accept=['json', 'pickle', 'msgpack'])
```
每种序列化格式的优缺点如下：

- json—JSON现在已经被大多数编程语言所支持，已成为Python标准中的一部分，使用cjson和simplejson这些Python库，可以达到相当快的解码速度。缺点是只支持有限的集中数据类型：strings, Unicode, floats, boolean, dictionaries,lists。尤其是不支持Decimals和dates类型的数据。
- pickle—如果你只需要Python就可以支持所有业务，那么建议使用pickle。pickle可以支持Python所有内建的数据类型（class instance除外）,编码后的数据量更小，而且比JSON的处理速度有了一些提高。pickle的缺点就是人们常说的安全问题。一个精心构造的pickle payload几乎可以实现普通Python程序所能实现的所有功能，所以在使用pickle时必须进行严格的访问控制，防止恶意的第三方向broker发送message。
- yaml—YAML类似于JSON，支持跨语言，而且比JSON支持更多的数据类型（dates, recursive references等）。缺点是Python中支持YAML的库的处理速度比JSON慢。

在Kombu中，设置message的serializer方式有两种方法，分别如下：

1. 在创建Producer时指定serializer：

```
>>> producer = Producer(channel,
...                     exchange=exchange,
...                     serializer="yaml")
```
2.在发送message时指定serializer:

```
>>> producer.publish(message, routing_key=rkey,
...                  serializer="pickle")
```

## 0x03 Work Queue实现
在最后，我们实现一个简单的Work Queue消息模型来作为对Kombu学习的总结。在前面的[六步学习RabbitMQ（三）](/2015/12/09/six-steps-to-study-rabbitmq-3/)中我们使用RabbitMQ实现了Work Queue模型，我们可以边学习边与前面的进行对比，这样可以对Kombu有一个更加感性的认识。

在下面实现的Work Queue模型中，设置了三个Queue绑定到一个direct类型的exchange上，然后consumer监听所有的队列。消息来了后就轮询调用consumer进行处理。

先是queues.py，主要内容是声明exchange和Qeueue，并将Queue与exchange进行绑定：

```
from kombu import Exchange, Queue

task_exchange = Exchange('tasks', type='direct')
task_queues = [Queue('hipri', task_exchange, routing_key='hipri'),
               Queue('midpri', task_exchange, routing_key='midpri'),
               Queue('lopri', task_exchange, routing_key='lopri')]
```
接下来是worker.py，主要内容是声明一个consumer，并监听上面声明的task_queues:

```
from kombu.mixins import ConsumerMixin
from kombu.log import get_logger
from kombu.utils import kwdict, reprcall

from .queues import task_queues

logger = get_logger(__name__)


class Worker(ConsumerMixin):

    def __init__(self, connection):
        self.connection = connection

    def get_consumers(self, Consumer, channel):
        return [Consumer(queues=task_queues,
                         accept=['pickle', 'json'],
                         callbacks=[self.process_task])]

    def process_task(self, body, message):
        fun = body['fun']
        args = body['args']
        kwargs = body['kwargs']
        logger.info('Got task: %s', reprcall(fun.__name__, args, kwargs))
        try:
            fun(*args, **kwdict(kwargs))
        except Exception as exc:
            logger.error('task raised exception: %r', exc)
        message.ack()

if __name__ == '__main__':
    from kombu import Connection
    from kombu.utils.debug import setup_logging
    # setup root logger
    setup_logging(loglevel='INFO', loggers=[''])

    with Connection('amqp://guest:guest@localhost:5672//') as conn:
        try:
            worker = Worker(conn)
            worker.run()
        except KeyboardInterrupt:
            print('bye bye')
```
然后是tasks.py，consumer收到消息后会调用它里面的方法来处理:

```
def hello_task(who="world"):
    print("Hello %s" % (who, ))
```
最后是消息发送端client.py:

```
from kombu.pools import producers

from .queues import task_exchange

priority_to_routing_key = {'high': 'hipri',
                           'mid': 'midpri',
                           'low': 'lopri'}


def send_as_task(connection, fun, args=(), kwargs={}, priority='mid'):
    payload = {'fun': fun, 'args': args, 'kwargs': kwargs}
    routing_key = priority_to_routing_key[priority]

    with producers[connection].acquire(block=True) as producer:
        producer.publish(payload,
                         serializer='pickle',
                         compression='bzip2',
                         exchange=task_exchange,
                         declare=[task_exchange],
                         routing_key=routing_key)

if __name__ == '__main__':
    from kombu import Connection
    from .tasks import hello_task

    connection = Connection('amqp://guest:guest@localhost:5672//')
    send_as_task(connection, fun=hello_task, args=('Kombu', ), kwargs={},
                 priority='high')
```
我在本机上对该程序进行了运行，打开了3个worker和以一个client，client向exchange发送了4次message，每个worker的运行结果如下;

- worker1处理了2个message：
![worker1](/img/in-post/post12-kombu-1.png)
- worker2处理了1个message：
![worker2](/img/in-post/post12-kombu-2.png)
- worker3处理了1个message：
![worker3](/img/in-post/post12-kombu-3.png)

## 0x04 总结
有关Kombu的学习就到这里了。可以体会出，通过Kombu可以使我们的程序在消息处理方面具有更高的灵活性和扩展性，而这一特性主要是因为Kombu为我们提供了一个高层抽象——transport。我以前一直以为每个transport对应一种AMQP broker，但是在写博客的过程中，我发现这种理解大错特错了。transport的本质是一个链接库，为用户连接不同的broker提供了统一的接口和统一的异常处理方法等。每个transport都可以根据用户指定的URL来连接底层的broker，不同的transport之间的区别在于连接效率、对特定broker的支持性上。

这段时间连着发了十来篇博客，整的都有点累了，所以打算先放一下。下一篇博客就是对oslo.messaging的分析了，对于这一块我其实也不是太熟悉，所以得先下些功夫理解。see you later！
