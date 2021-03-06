#REDIS源码8-Stream

##基本用途

Stream是Redis5.0新发布的数据结构，是一个支持多播的可持久化消息队列，作者自己也承认Stream的设计借鉴了Kafka的设计。

Stream的基本结构是一个链表，每个链表都有一个唯一的key，每个Stream都可以挂多个消费族，每个消费族有一个id，这个id就代表当前消费族消费到哪条消息，每个消费组在Stream内也有一个唯一的名称，消费族不会自动创建，需要单独的指令来创建。

同时每个消费组的状态也都是独立的，每个消费族有一个挂多个消费消费者，这些消费者会相互竞争的消费消息。大致结构如下

```
a|b|c|d|...|x|y|z|...
```

这些字母就代表着一个个消息，然后有多个consumer_group对象，每个consumer_group内有多个consumer，和一个消息id，比如consumer_group1指向x，那么就代表consumer_group1内所有的消费者消费到x这条消息，后面的consumer会继续消费后面的消息。

**消息id序列：**消息id就是时间戳-序列，这样的消息天然递增的（也必须要递增）。

**链表实现：**同时Stream的实现由于是一个链表，所以肯定就如Java线程池的LinkedBlockingQueue一样，有链表太长导致内存溢出的问题，所以Redis的Stream可以提供一个定长的功能，如果达到了最大长度，那么就把老的消息丢弃掉。

**消息消费后的确认：**Redis读取到了新消息之后，就会把新消息放到PEL（正在处理的消息）这个结构里面，如果Redis收到了客户端返回的ACK，就把这个PEL里面的消息删除。同样的，如果客户端消费了消息没有ACK，那么PEL这个数组里面的待确认消息就越来越多。同时PEL还有防止消息丢失的功能，client可能消费到了消息但是挂了，所以没有发送ACK，那么就可以根据PEL的内容来消费之前的消息（所以应该说确保了消息至少消费一次）。

**高可用：**Stream的高可用时建立在主从复制的基础上的。不过由于Redis是异步主从复制，所以有可能会丢失一小部分数据

**分区：**Redis没有原生支持分区，如果想要支持分区，就需要分配多个Stream，然后根据不同的策略略来生产消息到不同的Stream。

由于Stream的数据结构比较复杂，用到了新的数据结构listpack（类似ziplist）以及rax（相当于TreeMap），所以就拆成两个部分，先分析listpack，然后分析rax，然后再回过头来分析Stream。