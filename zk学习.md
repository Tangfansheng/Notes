### zk 学习-分析

#### zookeeper是什么

ZooKeeper 是一个典型的分布式数据一致性解决方案，分布式应用程序可以基于 ZooKeeper 实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master 选举、分布式锁和分布式队列等功能。

##### 概念

1.Session 指的是 ZooKeeper 服务器与客户端会话。在 ZooKeeper 中，一个客户端连接是指客户端和服务器之间的一个 TCP 长连接。

2.**Znode** zooleeper的节点有两类：第一类是机器节点，第二类是数据模型中的数据单元，称为ZNode。ZooKeeper 将所有数据存储在内存中，数据模型是一棵树（Znode Tree)，由斜杠（/）的进行分割的路径，就是一个 Znode，例如/foo/path1。每个上都会保存自己的数据内容，同时还会保存一系列属性信息。

3.**Watcher** 事件监听器，ZooKeeper 允许用户在指定节点上注册一些 Watcher，并且在一些特定事件触发的时候，ZooKeeper 服务端会将事件通知到感兴趣的客户端上去，该机制是 ZooKeeper 实现分布式协调服务的重要特性。

4.**ZooKeeper & ZAB 协议 & Paxos 算法**  在 ZooKeeper 中，主要依赖 ZAB 协议来实现分布式数据一致性

具体使用：https://zookeeper.apache.org/doc/r3.6.2/zookeeperProgrammers.html

##### 秒杀项目中的Zookeeper应用

 zookeeper配置类 主要是配置ip，超时时间，将zkClient加入到容器中。

用了一个Concurrent包里面的功能，CountDownLatch

```java
final CountDownLatch countDownLatch = new CountDownLatch(1);
zooKeeper = new ZooKeeper(connectString, timeout, new Watcher() {
    @Override
    public void process(WatchedEvent event) {
        if (Event.KeeperState.SyncConnected == event.getState()) {
            System.out.println("【初始化ZooKeeper连接成功....】");
            //如果收到了服务端的响应事件,连接成功
            countDownLatch.countDown();
        }
    }
});
countDownLatch.await();
```

当这个zookeeper完成连接之前，会一直控制线程等待。

##### zookeeper原理

1.当前线程在获取分布式锁的时候，会在ZNode节点(ZNode节点是Zookeeper的指定节点)下创建临时顺序节点，释放锁的时候将删除该临时节点。

2.客户端/服务 调用createNode方法在 ZNode节点 下创建临时顺序节点，然后调用getChildren(“ZNode”)来获取ZNode下面的所有子节点，注意此时不用设置任何Watcher。

3.客户端/服务获取到所有的子节点path之后，如果发现自己创建的***\*子节点序号最小\****，那么就认为该客户端获取到了锁，即当前线程获取到了分布式锁。

4.如果发现自己创建的节点并非ZNode所有子节点中最小的，说明自己还没有获取到锁，此时客户端需要找到比自己小的那个节点，然后对其调用exist()方法，同时对其注册事件监听器。



##### zookeeper实验

官方推荐使用Curator Framework来操作Zookeeper，以下是获取curator实例

```java
CuratorFramework client = CuratorFrameworkFactory.builder().namespace("MyApp") ... build();
```

中间的省略号是其他的配置项，以下是本地测试代码

```java
client = CuratorFrameworkFactory.builder()
    .connectString("127.0.0.1:2181")
    .namespace("kill")
    .retryPolicy(new RetryNTimes(5,1000))
    .build();
client.start();
```

namespace的设置需要注意，不同的应用不能使用相同的命名空间，做应用的隔离。

使用的步骤很简单，创建一个mutex操作实例，在临界代码前面加aquire，使用完之后release就行了。

```java
  public void killItem() {
        //获取分布式锁的操作组件实例
        //mutex就是互斥锁，集群中用相同的path的请求会争夺一个临界资源
        InterProcessMutex mutex = new InterProcessMutex(client, path);
        //若干线程模拟抢购
        for(int i=0; i<10; i++){
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        mutex.acquire(10L,TimeUnit.SECONDS);
                        buy();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }finally {
                        try {
                            mutex.release();
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                }
            }).start();
        }
    }
    
    //临界代码
    public void buy(){
        System.out.println("----------------" + Thread.currentThread().getName()+ "开始抢购-----------------");
        //获取库存
        int currentStock = Product.getNumber();
        if(currentStock == 0){
            System.out.println("商品已经售空");
        }else{
            //获取锁
            System.out.println("购买成功 库存减一 剩下" + (currentStock-1)+"件库存");
            Product.setNumber(--currentStock);
            //延时便于观察
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
```


####  Zookeeper Watcher机制

`Zookeeper Watcher流程`：客户端向服务端的某个节点路径上注册一个watcher，客户端同时会在本地watcherManager中存储特定的watcher，当发生节点数据或者节点子节点变化时，服务端会通知客户端节点变化信息，然后客户端收到通知后，会调用回调函数。

##### watcher注册

可以在创建zk户端的时候注册watcher

```
ZooKeeper(String connectString, int sessionTimeout, Watcher watcher)
```

还有 **getData()**, **getChildren()**, **exists()**方法都可以设定watcher

watcher机制是一次性的，在client收到watcher回调后，设定的watcher就失效了。如果向保持监听，则需要在回调函数中重新设定watcher
