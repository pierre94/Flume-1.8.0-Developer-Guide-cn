# Flume-1.8.0-Developer-Guide-cn
Flume 1.8.0 Developer Guide中文版

flume 1.8.0 开发者指引(Flume 1.8.0 Developer Guide)

原文:http://flume.apache.org/FlumeDeveloperGuide.html
# 介绍
## 概述
Apache Flume是一个用于高效地从大量异构数据源收集、聚合、传输到一个集中式数据存储的分布式、高可靠、高可用的系统.
Apache Flume是Apache基金会的顶级项目.现在有两个代码版本线可以获取:0.9.x和1.x.本文档对应的是1.x版本.

## 数据流模型
Event是流经flume agent的最小数据单元.一个Event(由Event接口实现)从source流向channel，再到sink.Event包含了一个payload(byte array)和可选的header(string attributes).一个flume agent就是一个jvm下的进程：控制着Events从一个外部的源头到一个外部的目的地.

Source消费着具有特殊格式的Events(这些Event传递到Source通过像Web server这样外在的数据源).例如AvroSource可以被用于接收Avro的Events，从本客户端或者其他运行中的flume客户端.当一个Source接收到一个Event，它会把它插入到一个或者多个Channel里.Channel会被动地存储这些Event直到它们被一个Sink消费到.Flume中一种Channel是FileChannel,其使用文件系统来作为后端存储.Sink需要负责任地将一个Event从Channel中移除，并将其放入像hdfs一样的外部存储系统(例如HDFSEventSink)，或者转发到传输中下一个节点的source中.Source和Sink在agent中异步地交互Channel中的Event.

## 可靠性
Event是存储在Flume agent的Channel里.Sink的责任就是传输Event到下一个agent或者最终的存储系统(像hdfs).Sink只有当Event写入下一个agent的Channel 或者 存储到最终的系统时才会从channel里面删掉Event.这就是Flume如何在单跳消息传输中提供端到端的可靠性.Flume提供了一个事务性的方法来修复可靠传输中的Event.Source和Sink包含了Event的存储和重试(通过由channel提供的事务).

# 构建Flume
## 获取源码
通过git

## 编译/测试 Flume
Flume使用maven来build.你可以通过标准的maven命令行来编译Flume.
1. 仅编译:mvn clean compile
2. 编译且运行单元测试:mvn clean test
3. 运行独立的测试:mvn clean test -Dtest=<Test1>,<Test2>,... -DfailIfNoTests=false
4. 打包:mvn clean install
5. 打包(忽略单元测试):mvn clean install -DskipTests

注意：Flume build需要在path中有Google Protocol Buffers编译器.

## 更新Protocol Buffer版本
File channel依赖Protocol Buffer.当你想更新Protocol Buffer版本时，你需要如下更新使用到Protocol Buffer的data access类：
1. 本机安装你想要的PB版本
2. 更新pom.xml中PB的版本
3. 生成flume中新的PB data access类：cd flume-ng-channels/flume-file-channel; mvn -P compile-proto clean package -DskipTests
4. 在所有生成文件中加上Apache license(如果缺了的话)
5. rebuild及测试Flume:cd ../..; mvn clean install

# 开发自定义部分
## client
Client在Event产生时运转，并将他们传递到Flume的agent.Client通常运行在应用消费数据的进程空间中.Flume目前支持Avro, log4j, syslog, 以及 Http POST (with a JSON body)方式从外部数据源传输数据.同时ExecSource支持将本地进程的输出作为Flume的输入.

可能已有的方案是不够的.本案例中你可以使用自定义的方法来向flume发送数据.这里有两种方法来实现.第一：写一个自定义的客户端来和flume已有的source交互，像AvroSource 或者 SyslogTcpSource.此时Client需要将数据转换成这些Source能理解的message.另外一个方案:写一个自定义的Flume Source，通过IPC或者RPC，直接地和已有的client应用通信(需要将client的数据转换成Flume的Event).注意这些存储在flume agent channel中的事件，必须以Flume Event形式存在.

## Client SDK
尽管Flume包含了一系列内置的，用于接收数据的方法(即Source)，人们常常想直接地通过flume和自定义的程序进行通信.Flume SDK 就是这样一个lib，它可以通过RPC直接地连接到Flume，并且发送到Flume的数据流.

## RPC客户端接口
一个RPC客户端接口的实现，包含了支持Flume的RPC方法.用户的程序可以简单地调用Flume SDK客户端的append(Event)或者appendBatch(List<Event>)接口来发送数据，而不用考虑消息交互的细节.用户可以通过使用诸如SimpleEvent类，或者使用EventBuilder的 静态helper方法withBody()，便捷地实现直接提供事件接口所需的事件ARG.

## RPC clients - Avro and Thrift
Flume 1.4.0时，Avro成为默认的RPC协议.NettyAvroRpcClient和ThriftRpcClient实现了RpcClient的接口.客户端需要建立一个包含目的Flume agent host和port信息的对象.使用RpcClient来发送数据到客户端.下例展示了如何在程序中使用flume client sdk api.
```java
import org.apache.flume.Event;
import org.apache.flume.EventDeliveryException;
import org.apache.flume.api.RpcClient;
import org.apache.flume.api.RpcClientFactory;
import org.apache.flume.event.EventBuilder;
import java.nio.charset.Charset;

public class MyApp {
  public static void main(String[] args) {
    MyRpcClientFacade client = new MyRpcClientFacade();
    // Initialize client with the remote Flume agent's host and port
    client.init("host.example.org", 41414);

    // Send 10 events to the remote Flume agent. That agent should be
    // configured to listen with an AvroSource.
    String sampleData = "Hello Flume!";
    for (int i = 0; i < 10; i++) {
      client.sendDataToFlume(sampleData);
    }

    client.cleanUp();
  }
}

class MyRpcClientFacade {
  private RpcClient client;
  private String hostname;
  private int port;

  public void init(String hostname, int port) {
    // Setup the RPC connection
    this.hostname = hostname;
    this.port = port;
    this.client = RpcClientFactory.getDefaultInstance(hostname, port);
    // Use the following method to create a thrift client (instead of the above line):
    // this.client = RpcClientFactory.getThriftInstance(hostname, port);
  }

  public void sendDataToFlume(String data) {
    // Create a Flume Event object that encapsulates the sample data
    Event event = EventBuilder.withBody(data, Charset.forName("UTF-8"));

    // Send the event
    try {
      client.append(event);
    } catch (EventDeliveryException e) {
      // clean up and recreate the client
      client.close();
      client = null;
      client = RpcClientFactory.getDefaultInstance(hostname, port);
      // Use the following method to create a thrift client (instead of the above line):
      // this.client = RpcClientFactory.getThriftInstance(hostname, port);
    }
  }

  public void cleanUp() {
    // Close the RPC connection
    client.close();
  }

}
```
远端的flume agent需要AvroSource在监听相关端口(或者是ThriftSource).下面是一个正在等待MyApp连接的flume agent示例配置文件.
```bash
a1.channels = c1 
a1.sources = r1 
a1.sinks = k1
a1.channels.c1.type = memory
a1.sources.r1.channels = c1 
a1.sources.r1.type = avro

# For using a thrift source set the following instead of the above line. # a1.source.r1.type = thrift
a1.sources.r1.bind = 0.0.0.0
a1.sources.r1.port = 41414
a1.sinks.k1.channel = c1 
a1.sinks.k1.type = logger
```
更灵活一点，Flume client实现((NettyAvroRpcClient and ThriftRpcClient) )可以如下配置:
```bash
client.type = default (for avro) or thrift (for thrift)
hosts = h1     # default client accepts only 1 host
                # (additional hosts will be ignored)
hosts.h1 = host1.example.org:41414
                # host and port must both be specified
                # (neither has a default)
batch-size = 100        # Must be >=1 (default: 100)
connect-timeout = 20000 # Must be >=1000 (default: 20000)
request-timeout = 20000 # Must be >=1000 (default: 20000)
```

## Secure RPC client - Thrift
Flume 1.6.0时，Thrift source和sink支持基于kerberos的认证.客户端需要使用SecureRpcClientFactory的getThriftInstance方法来实现SecureThriftRpcClient.当你使用SecureRpcClientFactory时，kerberos认证模块需要放在classpath的flume-ng-auth路径下.客户端的主体和密钥表通过properties以参数形式传入.同时目的服务端的Thrift source也需要使用这样处理.用户的数据程序可以参考如下例子来使用SecureRpcClientFactory：
```java
import org.apache.flume.Event;
import org.apache.flume.EventDeliveryException;
import org.apache.flume.event.EventBuilder;
import org.apache.flume.api.SecureRpcClientFactory;
import org.apache.flume.api.RpcClientConfigurationConstants;
import org.apache.flume.api.RpcClient;
import java.nio.charset.Charset;
import java.util.Properties;

public class MyApp {
  public static void main(String[] args) {
    MySecureRpcClientFacade client = new MySecureRpcClientFacade();
    // Initialize client with the remote Flume agent's host, port
    Properties props = new Properties();
    props.setProperty(RpcClientConfigurationConstants.CONFIG_CLIENT_TYPE, "thrift");
    props.setProperty("hosts", "h1");
    props.setProperty("hosts.h1", "client.example.org"+":"+ String.valueOf(41414));

    // Initialize client with the kerberos authentication related properties
    props.setProperty("kerberos", "true");
    props.setProperty("client-principal", "flumeclient/client.example.org@EXAMPLE.ORG");
    props.setProperty("client-keytab", "/tmp/flumeclient.keytab");
    props.setProperty("server-principal", "flume/server.example.org@EXAMPLE.ORG");
    client.init(props);

    // Send 10 events to the remote Flume agent. That agent should be
    // configured to listen with an AvroSource.
    String sampleData = "Hello Flume!";
    for (int i = 0; i < 10; i++) {
      client.sendDataToFlume(sampleData);
    }

    client.cleanUp();
  }
}

class MySecureRpcClientFacade {
  private RpcClient client;
  private Properties properties;

  public void init(Properties properties) {
    // Setup the RPC connection
    this.properties = properties;
    // Create the ThriftSecureRpcClient instance by using SecureRpcClientFactory
    this.client = SecureRpcClientFactory.getThriftInstance(properties);
  }

  public void sendDataToFlume(String data) {
    // Create a Flume Event object that encapsulates the sample data
    Event event = EventBuilder.withBody(data, Charset.forName("UTF-8"));

    // Send the event
    try {
      client.append(event);
    } catch (EventDeliveryException e) {
      // clean up and recreate the client
      client.close();
      client = null;
      client = SecureRpcClientFactory.getThriftInstance(properties);
    }
  }

  public void cleanUp() {
    // Close the RPC connection
    client.close();
  }
}
```
远端的ThriftSource需要以kerberos模式启动.下面这个示例Flume agent配置文件用于等待MyApp的连接：
```bash
a1.channels = c1
a1.sources = r1
a1.sinks = k1

a1.channels.c1.type = memory

a1.sources.r1.channels = c1
a1.sources.r1.type = thrift
a1.sources.r1.bind = 0.0.0.0
a1.sources.r1.port = 41414
a1.sources.r1.kerberos = true
a1.sources.r1.agent-principal = flume/server.example.org@EXAMPLE.ORG
a1.sources.r1.agent-keytab = /tmp/flume.keytab


a1.sinks.k1.channel = c1
a1.sinks.k1.type = logger
```

## failover client
这个class包含一个默认的Avro RPC客户端，来提供客户端的故障切换处理能力.它使用了一个以空格分隔的list(<host>:<port>代表flume agent)作为一个故障切换组.目前failoverRPC客户端还不支持thrift.如果与选中的host agent通信出现错误，那么failover client会自动的选取列表中下一个host.例如：
```java
// Setup properties for the failover
Properties props = new Properties();
props.put("client.type", "default_failover");

// List of hosts (space-separated list of user-chosen host aliases)
props.put("hosts", "h1 h2 h3");

// host/port pair for each host alias
String host1 = "host1.example.org:41414";
String host2 = "host2.example.org:41414";
String host3 = "host3.example.org:41414";
props.put("hosts.h1", host1);
props.put("hosts.h2", host2);
props.put("hosts.h3", host3);

// create the client with failover properties
RpcClient client = RpcClientFactory.getInstance(props);
```
FailoverRpcClient可以以下列参数更灵活的配置:
```bash
client.type = default_failover

hosts = h1 h2 h3                     # at least one is required, but 2 or
                                     # more makes better sense

hosts.h1 = host1.example.org:41414

hosts.h2 = host2.example.org:41414

hosts.h3 = host3.example.org:41414

max-attempts = 3                     # Must be >=0 (default: number of hosts
                                     # specified, 3 in this case). A '0'
                                     # value doesn't make much sense because
                                     # it will just cause an append call to
                                     # immmediately fail. A '1' value means
                                     # that the failover client will try only
                                     # once to send the Event, and if it
                                     # fails then there will be no failover
                                     # to a second client, so this value
                                     # causes the failover client to
                                     # degenerate into just a default client.
                                     # It makes sense to set this value to at
                                     # least the number of hosts that you
                                     # specified.

batch-size = 100                     # Must be >=1 (default: 100)

connect-timeout = 20000              # Must be >=1000 (default: 20000)

request-timeout = 20000              # Must be >=1000 (default: 20000)
```

## 负载均衡的RPC client
Flume Client SDK同样支持RPC的负载均衡.它使用了一个以空格分隔的list(<host>:<port>代表flume agent)作为一个负载均衡组.同时支持随机和轮询两种负载均衡策略.你同样可以通过implement loadBalancingRpcClient$HostSelector接口在你自定义的class中实现选取算法.这种情况下，你需要在host-selector属性中填上你自定义类的FQCN(Full Qualified Class Name).目前负载均衡的RPC client同样没有支持thrift.

如果启用了backoff，则客户端将暂时将失败的主机列入黑名单，从而在给定的timeout时间内将它们排除为故障转移的host.当timeout时间到了，如果host仍然没有响应，那么这被认为是顺序故障，并且timeout时间会以指数方式增加，以避免在无响应的主机上长时间等待时卡住.

可以通过设置maxBackoff(以毫秒为单位)来配置最大backoff时间. maxBackoff默认值为30秒(在OrderSelector类中指定，它是两个负载平衡策略的超类). 退避超时将随着连续故障呈指数级增长，直至最大可能的退避超时(最大可能的退避限制为65536秒(约18.2小时)).
```java
// Setup properties for the load balancing
Properties props = new Properties();
props.put("client.type", "default_loadbalance");

// List of hosts (space-separated list of user-chosen host aliases)
props.put("hosts", "h1 h2 h3");

// host/port pair for each host alias
String host1 = "host1.example.org:41414";
String host2 = "host2.example.org:41414";
String host3 = "host3.example.org:41414";
props.put("hosts.h1", host1);
props.put("hosts.h2", host2);
props.put("hosts.h3", host3);

props.put("host-selector", "random"); // For random host selection
// props.put("host-selector", "round_robin"); // For round-robin host
//                                            // selection
props.put("backoff", "true"); // Disabled by default.

props.put("maxBackoff", "10000"); // Defaults 0, which effectively
                                  // becomes 30000 ms

// Create the client with load balancing properties
RpcClient client = RpcClientFactory.getInstance(props);
```
LoadBalancingRpcClient可以以下列参数更灵活的配置:
```bash
client.type = default_loadbalance

hosts = h1 h2 h3                     # At least 2 hosts are required

hosts.h1 = host1.example.org:41414

hosts.h2 = host2.example.org:41414

hosts.h3 = host3.example.org:41414

backoff = false                      # Specifies whether the client should
                                     # back-off from (i.e. temporarily
                                     # blacklist) a failed host
                                     # (default: false).

maxBackoff = 0                       # Max timeout in millis that a will
                                     # remain inactive due to a previous
                                     # failure with that host (default: 0,
                                     # which effectively becomes 30000)

host-selector = round_robin          # The host selection strategy used
                                     # when load-balancing among hosts
                                     # (default: round_robin).
                                     # Other values are include "random"
                                     # or the FQCN of a custom class
                                     # that implements
                                     # LoadBalancingRpcClient$HostSelector

batch-size = 100                     # Must be >=1 (default: 100)

connect-timeout = 20000              # Must be >=1000 (default: 20000)

request-timeout = 20000              # Must be >=1000 (default: 20000)
```

## 嵌入式的agent
Flume有一个嵌入式式的api，允许用户在他们的应用程序中嵌入.当不使用全部的source、sink、channel时，agent就会很轻量.具体而言，使用的source是一个特殊的嵌入式的source，事件通过EmbeddedAgent对象的put、putall方法来发送数据到source.Avro是唯一支持的sink，而channel只允许是File和Memory Channel.嵌入式的agent同样支持Interceptors. 

注意:嵌入式agent依赖hadoop-core.jar.

嵌入式agent的配置类似于完整agent的配置.以下是一份详尽的可选配置列表：
加粗的为必选项.

属性名|默认|描述
---|---  |---
source.type	|embedded|只能选embedded source.
**channel.type**|-|memory或者file
channel.*|-|详见MemoryChannel和FileChannel的用户指引
**sinks**|-|sink名的列表
**sink.type**|-|必须为avro
sink.*	|-|参考AvroSink的用户指引
**processor.type**|-|failover or load_balance
processor.*	|-|参考failover or load_balance的用户指引
source.interceptors|-|空格分隔的interceptors列表
source.interceptors.*|-|每个独立source.interceptors配置

使用案例:
```java
Map<String, String> properties = new HashMap<String, String>();
properties.put("channel.type", "memory");
properties.put("channel.capacity", "200");
properties.put("sinks", "sink1 sink2");
properties.put("sink1.type", "avro");
properties.put("sink2.type", "avro");
properties.put("sink1.hostname", "collector1.apache.org");
properties.put("sink1.port", "5564");
properties.put("sink2.hostname", "collector2.apache.org");
properties.put("sink2.port",  "5565");
properties.put("processor.type", "load_balance");
properties.put("source.interceptors", "i1");
properties.put("source.interceptors.i1.type", "static");
properties.put("source.interceptors.i1.key", "key1");
properties.put("source.interceptors.i1.value", "value1");

EmbeddedAgent agent = new EmbeddedAgent("myagent");

agent.configure(properties);
agent.start();

List<Event> events = Lists.newArrayList();

events.add(event);
events.add(event);
events.add(event);
events.add(event);

agent.putAll(events);

...

agent.stop();
```

## Transaction(事务)接口
Transaction接口是Flume可靠性的基础.所有主要组件(即source，sink和channel)必须使用Flume Transaction.
Transaction在channel的实现中实现.每个source和sink连接到channel时必须要得到一个channnel的对象.Source使用channnelprocessor来管理transaction.sink明确地通过他们配置的channel来管理transaction.存储一个事件(把他们放入channnel中)或者抽取一个事件(从channnel中取出)在一个激活的transaction中完成.例如:
```java
Channel ch = new MemoryChannel();
Transaction txn = ch.getTransaction();
txn.begin();
try {
  // This try clause includes whatever Channel operations you want to do

  Event eventToStage = EventBuilder.withBody("Hello Flume!",
                       Charset.forName("UTF-8"));
  ch.put(eventToStage);
  // Event takenEvent = ch.take();
  // ...
  txn.commit();
} catch (Throwable t) {
  txn.rollback();

  // Log exception, handle individual exceptions as needed

  // re-throw all Errors
  if (t instanceof Error) {
    throw (Error)t;
  }
} finally {
  txn.close();
}
```
在这里，我们从channel获取transaction.在begin()返回后，Transaction现在处于活动/打开状态，然后将Event放入Channel中.如果put成功，则提交并关闭Transaction.

## Sink
Sink的目的就是从Channel中提取事件并将其转发到传输中的下一个Flume Agent或将它们存储在外部存储库中.根据Flume属性文件中的配置，接收器只与一个通道关联.每个已配置的Sink都有一个SinkRunner实例，当Flume框架调用SinkRunner.start()时，会创建一个新线程来驱动Sink(使用SinkRunner.PollingRunner作为线程的Runnable)，该线程管理Sink的生命周期.Sink需要实现start()和stop()方法作为LifecycleAware接口的一部分.

- Sink.start()方法应初始化Sink并将其置于可将事件转发到其下一个目标的状态.
- Sink.process()应该执行从Channel提取Event并转发它的核心处理过程.
- Sink.stop()方法应该进行必要的清理(例如释放资源).

Sink实现还需要实现Configurable接口来处理自己的配置设置.例如：
```java
public class MySink extends AbstractSink implements Configurable {
  private String myProp;

  @Override
  public void configure(Context context) {
    String myProp = context.getString("myProp", "defaultValue");

    // Process the myProp value (e.g. validation)

    // Store myProp for later retrieval by process() method
    this.myProp = myProp;
  }

  @Override
  public void start() {
    // Initialize the connection to the external repository (e.g. HDFS) that
    // this Sink will forward Events to ..
  }

  @Override
  public void stop () {
    // Disconnect from the external respository and do any
    // additional cleanup (e.g. releasing resources or nulling-out
    // field values) ..
  }

  @Override
  public Status process() throws EventDeliveryException {
    Status status = null;

    // Start transaction
    Channel ch = getChannel();
    Transaction txn = ch.getTransaction();
    txn.begin();
    try {
      // This try clause includes whatever Channel operations you want to do

      Event event = ch.take();

      // Send the Event to the external repository.
      // storeSomeData(e);

      txn.commit();
      status = Status.READY;
    } catch (Throwable t) {
      txn.rollback();

      // Log exception, handle individual exceptions as needed

      status = Status.BACKOFF;

      // re-throw all Errors
      if (t instanceof Error) {
        throw (Error)t;
      }
    }
    return status;
  }
}
```

## Source
Source的目的是从外部客户端接收数据并将其存储到已配置的Channels中.Source可以获取其自己的ChannelProcessor的实例来处理在Channel本地事务中提交的串行事件.在exception的情况下,需要Channels传播异常，则所有Channels将回滚其事务，但先前在其他Channel上处理的事件将保持提交.

与SinkRunner.PollingRunner Runnable类似，有一个PollingRunner Runnable，它在Flume框架调用PollableSourceRunner.start()时创建的线程上执行.每个配置的PollableSource都与自己运行PollingRunner的线程相关联.该线程管理PollableSource的生命周期，例如启动和停止.

- PollableSource必须实现LifecycleAware接口中声明的start()和stop()方法.
- PollableSource的运行器调用Source的process()方法. process()方法应检查新数据并将其作为Flume事件存储到Channel中.

注意，实际上有两种类型的Source:已经提到过PollableSource，另一个是EventDrivenSource.与PollableSource不同，EventDrivenSource必须有自己的回调机制，捕获新数据并将其存储到Channel中.EventDrivenSources并不像PollableSources那样由它们自己的线程驱动.下面是一个自定义PollableSource的示例：

```java
public class MySource extends AbstractSource implements Configurable, PollableSource {
  private String myProp;

  @Override
  public void configure(Context context) {
    String myProp = context.getString("myProp", "defaultValue");

    // Process the myProp value (e.g. validation, convert to another type, ...)

    // Store myProp for later retrieval by process() method
    this.myProp = myProp;
  }

  @Override
  public void start() {
    // Initialize the connection to the external client
  }

  @Override
  public void stop () {
    // Disconnect from external client and do any additional cleanup
    // (e.g. releasing resources or nulling-out field values) ..
  }

  @Override
  public Status process() throws EventDeliveryException {
    Status status = null;

    try {
      // This try clause includes whatever Channel/Event operations you want to do

      // Receive new data
      Event e = getSomeData();

      // Store the Event into this Source's associated Channel(s)
      getChannelProcessor().processEvent(e);

      status = Status.READY;
    } catch (Throwable t) {
      // Log exception, handle individual exceptions as needed

      status = Status.BACKOFF;

      // re-throw all Errors
      if (t instanceof Error) {
        throw (Error)t;
      }
    } finally {
      txn.close();
    }
    return status;
  }
}
```
## Channel
TBD(To Be Determined 待决定)


