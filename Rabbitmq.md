## Rabbit Message Queue

### 环境构建

------

1. 安装服务器

   ```bash
   brew install rabbitmq
   ```

   在.zshrc中添加

   ```bash
   alias rabbitmq-server="/usr/local/Cellar/rabbitmq/3.8.0/sbin/rabbitmq-server"
   ```

   在终端中输入```rabbitmq-server```来启动rabbitmq服务器

2. 安装pika

   ```bash
   conda install -n <env name> -c conda-forge pika
   ```

### What‘s RabbitMQ

------

#### 生产者(Producer)：

向消息堆发送消息。

#### 消息堆(Queue)：

储存消息（先进先出）。

#### 消费者(Consumer)：

从堆接收消息。

### Hello world

------

#### 启动RabbitMQ

Queue由rabbitmq的服务器来管理，因此需要在终端执行```rabbitmq-server```来启动rabbitmq服务器。

#### 生产者

1. 连接到服务器

```python
import pika

#连接到在本地运行的rabbitmq服务器（localhost）
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
```

1. 确保queue存在

```python
#确保queue存在（如果不存在则建立一个新的queue），如果向不存在的queue发送消息，rabbitmq会废弃该消息
channel.queue_declare(queue='hello')
```

1. 发送消息

```python
channel.basic_publish(exchange='', #默认交换机类型，交换机相关在之后会详细介绍
                      routing_key='hello', #在默认交换机类型中，即为queue的名字
                      body='Hello World!') #发送的消息内容
print(" [x] Sent 'Hello World!'")
```

1. 结束连接

```python
connection.close()
```

#### 消费者

1. 连接到服务器

```python
import pika

#连接到在本地运行的rabbitmq服务器（localhost）
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
```

2. 确保queue存在

```python
#确保queue存在（如果不存在则建立一个新的queue）
channel.queue_declare(queue='hello')
```

3. 定义callback

```python
#消费者在接收到消息时，执行callback函数，本程序的callback函数打印接收到的消息
def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)
```

4. 接收消息

```python
channel.basic_consume(queue='hello', #指定从hello queue里接收消息
                      auto_ack=True, #默认为False，当设为True时，消费者从queue中接收到消息后，自动返回acknowledgment(ack)以告诉queue我接收到消息了，queue在接收到ack后即会从queue中删除该消息。
                      on_message_callback=callback) #接收到消息后执行callback函数
```

5. 监听queue

```python
print(' [*] Waiting for messages. To exit press CTRL+C')
# 执行一个无限循环开始监听queue
channel.start_consuming()
```

### Work Queue

------

#### Situation

有一个生产者不断的发布任务（task）至Queue，并且有复数个消费者（worker）订阅Queue，当Queue中出现任务后消费者收到任务并开始工作。本程序中以```time.sleep()```作为任务，生产者指定sleep的时间。

```python
def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)
    time.sleep(body.count(b'.'))
    print(" [x] Done")
    ch.basic_ack(delivery_tag=method.delivery_tag)
```

#### Round-robin dispatching

每个消费者将按照任务的数量被平均分配任务。例如当有两个消费者，并且生产者发布了20个任务了，其中一个消费者将会被分配到第奇数(1,3,5,...,19)个任务，另一个消费则会被分配到第偶数(2,4,6,...,20)个任务。该分配方法的原理是，在生产者发布任务之后，Queue就会预先平均分配任务给消费者。

#### Fair dispatching

Round-robin dispatching的缺点是，当正好第奇数个任务所需要的时间很少，而第偶数个任务所需要的时间都很多事，第一个消费者将很快完成被分配的任务并开始进入空闲状态，而第二个消费者仍需要很长的时间去完成任务。对这个缺点的改善方案就是Fair dispatching，即每个消费者只有在完成当前任务之后才会被分配到新的任务。我们可以通过设置每个消费者可被预先分配的任务的数量为1来实现。

```python
channel.basic_qos(prefetch_count=1)
```

#### Message acknowledgment

在当前场景中，由于每个消费者在完成任务时需花费一定时间，如果我们像Hello World中一样继续auto_ack的话如果消费者在工作中死亡的话，当前任务就会消失，所以改成只有在完成任务之后才返回ack会是一个较好的解决方案。因此在本程序中我们将会删除```auto_ack=True```，并在完成任务后返回ack。

```python
ch.basic_ack(delivery_tag=method.delivery_tag)
```

#### Message durability

解决了消费者死亡会丢失任务的问题之后，我们还需要解决服务器意外停止而丢失任务的问题。为了解决这个问题，我们需要使任务（Message）更加持久，而任务的持久性需要由两个因素来确保。首先我们需要让我们的queue可以在服务器重启后仍旧存在，其次才是任务自身的持久性。我们可以通过以下代码来实现。

```python
channel.queue_declare(queue='task_queue', durable=True) #使Queue可持久

# coding...

channel.basic_publish(exchange='',
                      routing_key="task_queue",
                      body=message,
                      properties=pika.BasicProperties(
                         delivery_mode = 2, # 使message可持久
                      ))
```

### Publish/Subscribe (Broadcast)

------

###  Situation

同样是一个生产者复数个消费者的场景，但是现在我们需要所有消费者收到全部Message（类似广播）。本场景中我们会涉及到一个新的交换机类型（相较于前两个场景的默认交换机类型）。

#### 交换机

一方面从生产者接收消息，另一方面把消息推送到Queue以供消费者接收。当我们需要广播消息时，生产者将不再直接把信息发布到Queue，与此相对的是只将信息发布到交换机。交换机根据其对接收到的消息的处理（发送到一个queue还是多个...）被分成了四种类型（direct，topic，headers和fanout），本场景使用fanout类型。

```python
channel.exchange_declare(exchange='logs', #交换机的名字，可自由定义
                         exchange_type='fanout')
```

#### Temporary queues

本场景中我们不再需要也不能接收指定名字的Queue，因此Queue的名字不再重要，所以我们只需建立一个临时的Queue，并且由服务器为我们随机生成一个Queue的名字。

```python
result = channel.queue_declare(queue='', exclusive=True) #声明一个queue，由服务器命名。通过设置exlusive为True来使该queue为一个临时的queue。
```

#### Bindings

让交换机发送消息给Queue的操作。

```python
channel.queue_bind(exchange='logs',
                   queue=result.method.queue)
```

### Routing

------

### Situation

假设有一个生产者生产的消息可以被分成"ERROR"和"WARNING"，而我们的消费者只需接收"ERROR"的消息，或者有另一个消费者只想接收"WARNING"的消息，那么我们就需要使用路由(Routing)了。

### Bindings

在上一个场景中我们使用了Bindings，这里我们也需要使用Bindings。在Bindings中，还有另一个关键词"routing_key"，以使得我们的消费者可以只收取对应路由键的消息。但是该关键词依赖于交换机类型，比如在fanout类型的交换机会忽略该关键词。

```python
channel.queue_bind(exchange=exchange_name,
                   queue=queue_name,
                   routing_key='ERROR')
```

### Direct exchange

Direct类型的交换机和fanout类型交换机的差别在于，前者更加的灵活。它可以指定Queue接收到的消息route_key。

### Producer

我们需要创建一个Direct类型的交换机，并发送消息。

```python
channel.exchange_declare(exchange='direct_logs',
                         exchange_type='direct')
...
channel.basic_publish(exchange='direct_logs',
                      routing_key=severity,
                      body=message)
```

### Consumer

和上一个场景基本一样，只需在Bindings时指定route_key。

```python
result = channel.queue_declare(queue='', exclusive=True)
queue_name = result.method.queue

for severity in severities:
    channel.queue_bind(exchange='direct_logs',
                       queue=queue_name,
                       routing_key=severity)
```

