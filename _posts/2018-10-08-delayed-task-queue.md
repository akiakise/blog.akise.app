---
title: 聊聊延时任务队列
copyright: true
date: 2018-10-08 14:38:01 +0800
categories: [Technology]
tags: [queue, redis, rabbitmq]
---

延时队列，顾名思义，是为了让一些任务不立即执行，放到队列里面等到特定时间后再执行。常用的场景有：

- 订单一直处于未支付状态，需要及时关闭订单，并退还库存
- 用户通过遥控设备控制智能设备在指定时间进行工作
- eFuture 中未来邮件需要在用户指定的时间点发送

<!-- more -->

[eFuture](https://github.com/xlui/eFuture/) 是我最近刚完成的一个项目，主要目的是提供“未来邮件”的服务。其中，邮件需要被存储并且需要在特定时间点发送。于是我就需要一个延时队列来保存发信任务，并在指定时间由程序消费任务，从而发送邮件。

对于 eFuture 来说，有两件事是最重要的：

1. 未来邮件在延时任务队列中保存，并且不会因为系统故障（如突发停机）而丢失
1. 未来邮件需要在指定日期被取出并消费掉（发送出）

这两点需求意味着，提供延时任务的组件需要具备容灾能力，所以常用的 Java 内部的实现就不行了（虽说我也没用 Java 来写这个项目）。一番搜索之后，发现常用的实现有两个符合我的要求：RabbitMQ 与 Redis。

## RabbitMQ

RabbitMQ 本身并没有直接支持延时队列功能，但是我们可以通过 RabbitMQ 队列的特性模拟出延时队列的功能。

RabbitMQ 在创建 Queue 的时候可以指定一个 `x-dead-letter-exchange` 的选项，指定该选项后， Queue 中过期的消息都将自动转发到相应的 Exchange。我们只需要在对应的 Exchange 上绑定接收过期任务的队列即可。

![Dead Letter eXchange](/assets/img/dlx.png "引用一张相关博客的图片")

Python 代码：

```python
import logging
import threading

import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(
    host='localhost',
    port=5672,
    virtual_host='/',
    credentials=pika.PlainCredentials('user', 'user')
))
channel = connection.channel()
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    datefmt='%H:%M:%S'
)

channel.exchange_delete('msg.exchange')
channel.queue_delete('msg.queue.dead')
channel.queue_delete('msg.queue.task')

channel.exchange_declare('msg.exchange')
channel.queue_declare('msg.queue.dead', arguments={'x-dead-letter-exchange': 'msg.exchange'})
channel.queue_declare('msg.queue.task')
channel.queue_bind(queue='msg.queue.task', exchange='msg.exchange', routing_key='msg.queue.dead')
```

上面这段代码主要做了：

- 利用 [pika](https://pika.readthedocs.io/en/stable/) 连接到 RabbitMQ
- 配置 logging
- 清除已有的同名 Exchange 与 Queue
- 创建 Exchange 与 Queue 并对其中一个队列设置 `dead-letter-exchange`。

下面开始测试：

```python
def push():
    for i in range(1, 6):
        logging.info('push {}'.format(i))
        channel.publish(
            exchange='', 
            routing_key='msg.queue.dead', 
            body="hello, {}".format(i),
            properties=pika.spec.BasicProperties(content_type="text/plain", expiration=str(1000 * i))
        )


def pop():
    for method, properties, body in channel.consume('msg.queue.task'):
        logging.info('receive {}'.format(body))


if __name__ == '__main__':
    tPush = threading.Thread(target=push)
    tPop = threading.Thread(target=pop)
    logging.info('start push...')
    tPush.start()
    logging.info('start pop...')
    tPop.start()
```

- `push` 依次向 RabbitMQ 的 `msg.queue.dead` 中发布了 `hello, 1`, `hello, 2`, `hello, 3`, `hello, 4`, `hello, 5`，其中 `hello, i` 的过期时间为 `i` 秒。
- `pop` 监听 RabbitMQ 的 `msg.queue.task`，一旦其中有数据就将其取出进行消费。

运行一下：

```bash
15:00:50 [INFO] start push...
15:00:50 [INFO] push 1
15:00:50 [INFO] start pop...
15:00:50 [INFO] push 2
15:00:50 [INFO] push 3
15:00:50 [INFO] push 4
15:00:50 [INFO] push 5
15:00:51 [INFO] receive b'hello, 1'
15:00:52 [INFO] receive b'hello, 2'
15:00:53 [INFO] receive b'hello, 3'
15:00:54 [INFO] receive b'hello, 4'
15:00:55 [INFO] receive b'hello, 5'
```

看起来我们似乎成功实现了延迟任务队列，在上面的测试用例下这个队列也工作的很好。但是，上面的实现中却有一个致命缺陷，我们把 `push` 函数稍稍修改一下：

```py
def push():
    # for i in range(1, 6):
    for i in range(5, 0, -1):
        logging.info('push {}'.format(i))
        channel.publish(
            exchange='',
            routing_key='msg.queue.dead',
            body="hello, {}".format(i),
            properties=pika.spec.BasicProperties(content_type="text/plain", expiration=str(1000 * i))
        )
```

再次运行：

```bash
15:02:47 [INFO] start push...
15:02:47 [INFO] push 5
15:02:47 [INFO] start pop...
15:02:47 [INFO] push 4
15:02:47 [INFO] push 3
15:02:47 [INFO] push 2
15:02:47 [INFO] push 1
15:02:52 [INFO] receive b'hello, 5'
15:02:52 [INFO] receive b'hello, 4'
15:02:52 [INFO] receive b'hello, 3'
15:02:52 [INFO] receive b'hello, 2'
15:02:52 [INFO] receive b'hello, 1'
```

可以看到，我们只是把插入消息的顺序调整了一下，队列就失去了作用。在新的 `push` 函数中，我们把过期时间长的任务先插入队列 `msg.queue.dead`，可以看到，只有在第一个消息过期并被转发到 `msg.queue.task` 后，后面的过期的消息才会被处理。

换言之，如果在 `msg.queue.dead` 中存在先后两个 task，第一个 task 过期时间 10s，第二个过期时间 1s，两个 task 是同时插入队列的。那么只有等第一个 task 过期从队列里移出后，第二个 task 才会被处理，**尽管它在第一个 task 过期前就已经过期了**。从中我们也可以看出 RabbitMQ 对于声明了 `x-dead-letter-exchange` 的队列中过期消息的处理策略：只检测最前面一个消息是否过期，如果过期就转发到指定的 Exchange，如果没有过期就不作任何处理。

RabbitMQ 的这种特性显然与我们的需求是相悖的，我们需要的是队列头元素始终是最接近过期的，所以我又将目光转向了 Redis。

## Redis

Redis 内部也没有对延时队列做支持，我们需要使用 Redis 的 `zset` 来手动实现。ZSet 是 Redis 内置的数据结构之一，其特性是值依据对应的 score 排序，内部使用 HashMap 与 SkipList 来存储数据并保证有序，HashMap 中存放的是值到 score 的映射，SkipList 中存放的是所有值，排序依据是 score。

利用 Redis 实现延时队列主要是就利用 ZSet，将值设置为 task，而 score 设置为任务执行时间点的 timestamp。然后 **通过比较当前时间戳与 zset 中第一个元素的 score 来判断其是否过期**。

Python 代码：

```python
import logging
import datetime

import redis

QUEUE_KEY = 'delayed task queue'
pool = redis.ConnectionPool(host='localhost', port=6379, db=0, password='')
connection = redis.Redis(connection_pool=pool)
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    datefmt='%H:%M:%S'
)


def push(message: str, date: datetime.datetime):
    connection.zadd(QUEUE_KEY, message, date.timestamp())


def pop():
    task = connection.zrange(QUEUE_KEY, 0, 0)
    if not task:
        return False, ''
    message = task[0]
    timestamp = connection.zscore(QUEUE_KEY, message)
    now = datetime.datetime.now().timestamp()
    if timestamp < now or abs(timestamp - now) <= 1e-6:
        connection.zrem(QUEUE_KEY, message)
        return True, message
    return False, ''


if __name__ == '__main__':
    now = datetime.datetime.now()
    logging.info('push hello 1')
    # 3 秒后过期
    push('hello 1', now + datetime.timedelta(seconds=3))
    logging.info('push hello 2')
    # 7 秒后过期
    push('hello 2', now + datetime.timedelta(seconds=7))
    while True:
        boolean, message = pop()
        if boolean:
            logging.info(message)

```

代码意思应该很明了，我们运行上述代码：

```bash
15:30:17 [INFO] push hello 1
15:30:17 [INFO] push hello 2
15:30:20 [INFO] b'hello 1'
15:30:24 [INFO] b'hello 2'
```

修改两个 task 的过期时间：

```py
now = datetime.datetime.now()
logging.info('push hello 1')
# 7 秒后过期
push('hello 1', now + datetime.timedelta(seconds=7))
logging.info('push hello 2')
# 3 秒后过期
push('hello 2', now + datetime.timedelta(seconds=3))
```

然后再次运行：

```bash
15:33:12 [INFO] push hello 1
15:33:12 [INFO] push hello 2
15:33:15 [INFO] b'hello 2'
15:33:19 [INFO] b'hello 1'
```

可以看到，如我们预期正确的执行了。

在代码中 `while True` 无限循环尝试取出元素是对 CPU 资源的一种浪费，同时也给 Redis 产生了不小的压力，我们可以放缓速度，比如每秒检测一次：

```py
while True:
    time.sleep(1)
    boolean, message = pop()
```

## 总结

除了文中提到的两种实现方式，延时队列还可以使用数据库定期轮询、DelayQueue、Timer、时间轮、Quartz 等方式实现，本文中没有讨论这些，等将来碰到相关使用场景后，或许会填这个坑 :)

使用 Redis 作为延时任务队列的实现，在具体应用的时候我们需要开启 Redis 的持久化，最好两种方式同时开启，同时定期备份 Redis 持久化文件，最大程度避免任务丢失。