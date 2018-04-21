---
title: 初识Rocketmq(二)
date: 2017-01-26 08:54:26
tags: [rocketmq]
categories: [消息队列]
description: 本文介绍Rocketmq消息消费的三种方式以及消息重试机制
---
# Rocketmq普通消息消费
这里是rocketmq-example的示例代码
```
/**
 * Producer，发送消息
 *
 */
public class Producer {
    public static void main(String[] args) throws MQClientException, InterruptedException {
        DefaultMQProducer producer = new DefaultMQProducer("common_message");
        producer.setNamesrvAddr("192.168.2.201:9876;192.168.2.202:9876");
        producer.start();

        for (int i = 0; i < 1000; i++) {
            try {
                Message msg = new Message("TopicTest",// topic
                    "TagA",// tag
                    ("Hello RocketMQ " + i).getBytes()// body
                        );
                SendResult sendResult = producer.send(msg);
                System.out.println(sendResult);
            }
            catch (Exception e) {
                e.printStackTrace();
                Thread.sleep(1000);
            }
        }

        producer.shutdown();
    }
}
```
```
/**
 * Consumer，订阅消息
 */
public class Consumer {

    public static void main(String[] args) throws InterruptedException, MQClientException {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("common_message");
        consumer.setNamesrvAddr("192.168.2.201:9876;192.168.2.202:9876");
        /**
         * 设置Consumer第一次启动是从队列头部开始消费还是队列尾部开始消费<br>
         * 如果非第一次启动，那么按照上次消费的位置继续消费
         */
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

        consumer.subscribe("TopicTest", "*");

        consumer.registerMessageListener(new MessageListenerConcurrently() {

            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                    ConsumeConcurrentlyContext context) {
                System.out.println(Thread.currentThread().getName() + " Receive New Messages: " + msgs);
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });

        consumer.start();

        System.out.println("Consumer Started.");
    }
}
```
这里没有过多的赘述，只是普通的消息生产与消费，需要指出的是，common_message是groupname，它的意义在于：作为生产者，当发送消息的一个节点宕机时，与它处于同一个group的其它节点可以发送消息；作为消费者，同样当消费者的一个节点宕机时，处于同一group的其它节点可以进行消费，实现了高可用的思想。
# Rocketmq顺序消息消费
首先同样给出代码示例
```
/**
 * Producer，发送顺序消息
 */
public class Producer {
    public static void main(String[] args) {
        try {
        	DefaultMQProducer producer = new DefaultMQProducer("order_message");
            producer.setNamesrvAddr("192.168.2.201:9876;192.168.2.202:9876");
            producer.start();

            String[] tags = new String[] { "TagA", "TagB", "TagC", "TagD", "TagE" };

            for (int i = 0; i < 100; i++) {
                // 订单ID相同的消息要有序
                int orderId = i % 10;
                Message msg =
                        new Message("topic_one", tags[i % tags.length], "KEY" + i,
                            ("Hello RocketMQ " + i).getBytes());

                SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
                    @Override
                    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                        Integer id = (Integer) arg;
                        int index = id % mqs.size();
                        return mqs.get(index);
                    }
                }, orderId);

                System.out.println(sendResult);
            }

            producer.shutdown();
        }
        catch (MQClientException e) {
            e.printStackTrace();
        }
        catch (RemotingException e) {
            e.printStackTrace();
        }
        catch (MQBrokerException e) {
            e.printStackTrace();
        }
        catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
```
/**
 * 顺序消息消费，带事务方式（应用可控制Offset什么时候提交）
 */
public class Consumer {

    public static void main(String[] args) throws MQClientException {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("order_message");

        consumer.setNamesrvAddr("192.168.2.201:9876;192.168.2.202:9876");
        /**
         * 设置Consumer第一次启动是从队列头部开始消费还是队列尾部开始消费<br>
         * 如果非第一次启动，那么按照上次消费的位置继续消费
         */
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

        consumer.setNamesrvAddr("192.168.2.201:9876;192.168.2.202:9876");

        consumer.subscribe("topic_one", "TagA || TagC || TagD");
        consumer.registerMessageListener(new MessageListenerOrderly() {
            AtomicLong consumeTimes = new AtomicLong(0);


            @Override
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {
                context.setAutoCommit(true);
                System.out.println(Thread.currentThread().getName() + " Receive New Messages: " + msgs);
                this.consumeTimes.incrementAndGet();
                if ((this.consumeTimes.get() % 2) == 0) {
                    return ConsumeOrderlyStatus.SUCCESS;
                }
                else if ((this.consumeTimes.get() % 3) == 0) {
                    return ConsumeOrderlyStatus.ROLLBACK;
                }
                else if ((this.consumeTimes.get() % 4) == 0) {
                    return ConsumeOrderlyStatus.COMMIT;
                }
                else if ((this.consumeTimes.get() % 5) == 0) {
                    context.setSuspendCurrentQueueTimeMillis(3000);
                    return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
                }

                return ConsumeOrderlyStatus.SUCCESS;
            }
        });

        consumer.start();

        System.out.println("Consumer Started.");
    }

}
```
这里需要注意以下几点：
1. 顺序消费的实现原理是生产者将需要有序消费的消息放在同一个队列中，并且该队列存储在MQ集群中的一个节点，不能分散存储，这样消费者只需要从该节点上拉取消费即可。
2. 消费者在消费时采用了多线程的方式，那么有心的你可能会有这样的疑问，即使消息可以按顺序到达，但是消费的快慢仍然无法保证有序。rocketmq本身实现了有序消费时线程间等待的功能，所以这个问题不用担心。
3. context.setAutoCommit(true);这句话的含义：一是删除msgTreeMapTemp里的消息，防止消息堆积，二是把拉消息的偏移量更新到本地，然后定时更新到broker。具体为什么会有这种设置，笔者暂不确定，不过设置为false时你会发现重复消费的情况。

# Rocektmq事务消息消费
以下是示例代码
```
/**
 * 发送事务消息例子
 *
 */
public class TransactionProducer {
    public static void main(String[] args) throws MQClientException, InterruptedException {

        TransactionCheckListener transactionCheckListener = new TransactionCheckListenerImpl();
        TransactionMQProducer producer = new TransactionMQProducer("trans_message");
        // 事务回查最小并发数
        producer.setCheckThreadPoolMinSize(2);
        // 事务回查最大并发数
        producer.setCheckThreadPoolMaxSize(2);
        // 队列数
        producer.setCheckRequestHoldMax(2000);
        producer.setTransactionCheckListener(transactionCheckListener);
        producer.setNamesrvAddr("192.168.2.201:9876;192.168.2.202:9876");
        producer.start();

        String[] tags = new String[] { "TagA", "TagB", "TagC", "TagD", "TagE" };
        TransactionExecuterImpl tranExecuter = new TransactionExecuterImpl();
        for (int i = 0; i < 1; i++) {
            try {
                Message msg =
                        new Message("TopicTest", tags[i % tags.length], "KEY" + i,
                            ("Hello RocketMQ " + i).getBytes());
                SendResult sendResult = producer.sendMessageInTransaction(msg, tranExecuter, null);
                System.out.println(sendResult);

                Thread.sleep(10);
            }
            catch (MQClientException e) {
                e.printStackTrace();
            }
        }

        for (int i = 0; i < 100000; i++) {
            Thread.sleep(1000);
        }

        producer.shutdown();

    }
}
```
```
/**
 * 执行本地事务
 */
public class TransactionExecuterImpl implements LocalTransactionExecuter {
    private AtomicInteger transactionIndex = new AtomicInteger(4);


    @Override
    public LocalTransactionState executeLocalTransactionBranch(final Message msg, final Object arg) {
        int value = transactionIndex.getAndIncrement();

        if (value == 0) {
            throw new RuntimeException("Could not find db");
        }
        else if ((value % 5) == 0) {
            return LocalTransactionState.ROLLBACK_MESSAGE;
        }
        else if ((value % 4) == 0) {
            return LocalTransactionState.COMMIT_MESSAGE;
        }

        return LocalTransactionState.UNKNOW;
    }
}
```
```
/**
 * 未决事务，服务器回查客户端
 */
public class TransactionCheckListenerImpl implements TransactionCheckListener {
    private AtomicInteger transactionIndex = new AtomicInteger(0);


    @Override
    public LocalTransactionState checkLocalTransactionState(MessageExt msg) {
        System.out.println("server checking TrMsg " + msg.toString());

        int value = transactionIndex.getAndIncrement();
        if ((value % 6) == 0) {
            throw new RuntimeException("Could not find db");
        }
        else if ((value % 5) == 0) {
            return LocalTransactionState.ROLLBACK_MESSAGE;
        }
        else if ((value % 4) == 0) {
            return LocalTransactionState.COMMIT_MESSAGE;
        }
//        return LocalTransactionState.UNKNOW;
        return LocalTransactionState.COMMIT_MESSAGE;
    }
}
```
这里举一个形象的例子
![](http://ok9lr2dej.bkt.clouddn.com/tansaction.png)
生产者首先向MQ发送消息，然后执行本地事务，当事务执行成功则再发送确认消息，否则会回滚本地事务。根据图示我们做以下几个说明：
1. 图中1操作：生产者向MQ发送一条消息，这条消息为TransactionPreparedType,被保存到commitlog中，并返回给生产者offset和messageid，但它不会保存在consumerqueue中，因此不会被消费，这里需要读者自行理解rocketmq的存储规律。
2. 当本地事务执行结束（即TransactionExecuterImpl）后，会根据返回结果确认本地事务是否执行成功。返回结果包括COMMIT_MESSAGE、ROLLBACK_MESSAGE、UNKNOWN。当结果为COMMIT_MESSAGE时会再向MQ发送一条消息，并存放在consumerQueue中提供消费者消费。当结果为ROLLBACK_MESSAGE时则同样发送一条消息但消息体为空。当结果为UNKONWN时则MQ会执行自动检测机制（即TransactionCheckListenerImpl），会回调生产者的方法根据该消息的状态做相应处理。

# Rocketmq消息重试机制
Rocketmq重试机制从三个方面来实现重试的策略。包括生产者向MQ发送消息失败的重试，消费者从MQ拉取消息失败的重试以及消费者处理消息异常时消息的重试。
![](http://ok9lr2dej.bkt.clouddn.com/Rocketmq2.png)
如图所示
1. 生产者的消息重试策略我们通常可以设置重试的最大次数以及重试的超时时间，也就是消息发送到MQ的超时时间。
2. 消费者获取消息的重试策略Rocketmq自身已经为用户实现了，默认是16次尝试，间隔时间为1s，30s,60s......
3. 消费者处理时的异常通过给MQ返回值来判定，ConsumeConcurrentlyStatus.CONSUME_SUCCESS表示处理成功，ConsumeConcurrentlyStatus.RECONSUME_LATER表示处理失败，如果失败则MQ重新发送该消息，直到该消息处理成功。
