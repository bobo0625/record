### 记一次kafka事故的排查

准生产环境测试同学提bug小应用开通，app页面未展示小应用，于是开始紧急排查

1，查看展示app页面的findOwnApp方法（user服务)，没有返回值，执行日志里面的sql,结果确实没有刚开通的app相关信息

2，查看appStore的日志，各种insert,update都比较正常,且日志中有kafka产生消息发送的打印

3，返回再查看user服务的日志，发现缺少几步更新的操作，查看代码发现这几步是kafka回调函数中进行的操作

4，进入kafka服务器，执行./bin/kafka-topics.sh  --zookeeper url --list查看当前所有的主题topic

5，执行./bin/kafka-topics.sh  --zookeeper url --topic topicname --describe查看topic的详情，可以看到partion位置，以及当前kafka集群的leader，副本replicas与ISR情况 ，可以看到当前主副本都正常运行

6，无奈查看当前的group,执行./bin/kafka-consumer-groups.sh --zookeeper url --group groupname --describe，详情列出了当前gruop的各项参数，如下

| group           | topic         | partion | current offset | log end offset | lag  | owner                |
| --------------- | ------------- | ------- | -------------- | -------------- | ---- | -------------------- |
| SSP-Bis-Company | corpAppUpdate | 0       | 350            | 425            | 75   | SSP-BisCompany_win.. |

* 核心参数参数为生产者生产消息数（log end offset）与消费者消费消息数（350），可以看到他们的偏差为75，也就是说有75条消息未被消费

7，查了下关于消息未被消费的情况，可能原因：a,磁盘满了未发送成功;b,同个topic被多台机器消费者监听，已经被其他人消费了；c,某次消费失败导致线程被占用一直阻塞，无法消费其他消息；d,consumer监听的不对或者说是consumer的参数配置不对

> 排查ab：非生产环境，且是晚上，于是将本地的kafka集群暂停改为一台，删除topic（注意要彻底删除，先delete删除topic，再删除zk节点 rmr /brokers/topics/XX，再ls查看是否彻底删除），重启之后依然消息无法被消费
>
> 排查d：zk节点下查看get /brokers/ids/，可以看到各项参数如endpoints,version,port等参数正常
>
> 排查c：服务重新启动，排除线程问题

8，无奈请教架构师大佬，说可能跟consumer的group有关，让我删除指定group （rmr /consumers/groups/XX）,然后重启服务试试，服务启动的会新建consumer，尝试发送一条消息，竟然看到lag变为0了，于是一看日志，果然已经被消费掉了，狂喜哈哈哈

总结：

1，遇到问题不能着急，按序排查，这次还好是在准生产环境，且要多请教公司大佬，调用资源，解决问题会更快。

2，kafka的原理要多理解，要深入理解kafka生产消费的本质，对于关键参数的配置也要心里有数