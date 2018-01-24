# my_thrift_learn
thrift学习笔记
## 1. thrift API里三个重要组成部分

>> Protocol，Transport，Server。

### (1) Protocol

```
Protocol定义了消息如何序列化。常见的是TBinaryProtocol，TJSONProtocol，TCompactProtocol。

```

### (2) Transport

```

Transport定义了消息在客户端和服务端如何通信。常见的是TSocket，TFramedTransport，TNonblockingTransport等。

```

### (3) Server

```
Server从transport端接收序列化后的消息，根据protocal反序列化回来，然后调用用户实现的消息handler(接口实现类)，

最后把返回的数据序列化后再传回给客户端。常见的TServer为TSimpleServer，THsHaServer，TThreadPoolServer，

TNonBlockingServer，TThreadedSelectorServer。下面会具体介绍各个Server的特点，开发者需要选择适合自己场景的一套 Server+对应的Transport+对应的Protocol。

```
## 2. TServer说明

```
Thrift实现的几种不同的TServer。对于Java

```
### (1) TSimpleServer

```
TSimpleServer在sever端只有一个I/O阻塞的单线程，每次只接受并服务一个客户端，适合测试使用，不能用于线上服务。
```

### (2) TNonblockingServer

```

TNonblockingServer修改了TSimpleServer里阻塞的缺点，借助NIO里的Selector实现非阻塞I/O，允许多个客户端连接并且客户端可以使用select()选择。

但是处理消息和select()的是同一个线程，当有大量客户端连接的时候，性能是不理想的。

```
### (3) THsHaServer

```
THsHaServer(半同步半异步server)在以上基础上，使用一个单独线程来处理网络I/O，一个worker线程池来处理消息。

好处是只要有空闲worker线程，消息可以被及时、并行处理，吞吐量会大一些。

```

### (4) TThreadedSelectorServer
```
这个用得最多

TThreadedSelectorServer，与THsHaServer的区别是处理网络I/O也是多线程了，它维护两个线程池，一个负责网络I/O，一个负责数据处理。
优点是当网络I/O是瓶颈的情况下，性能比THsHaServer更好。
```
### (5) TThreadPoolServer

```

TThreadPoolServer有一个专用的线程来接收connections，连接被建立后，会从ThreadPoolExecutor里取一个工作线程来负责这次连接

直到连接断开后线程回到线程池里，且线程池大小可配。也就是说，并发性的大小可根据服务器设定，如果不介意开很多线程的话，

TThreadPoolServer是个还不错的选择。

```

## 3. 协议介绍

```
Thrift 可以让用户选择客户端与服务端之间传输通信协议的类别，在传输协议上总体划分为文本 (text) 和二进制 (binary) 传输协议，

为节约带宽，提高传输效率，一般情况下使用二进制类型的传输协议为多数，有时还会使用基于文本类型的协议

这需要根据项目 / 产品中的实际需求。常用协议有以下几种：
```

### (1) TBinaryProtocol二进制编码格式进行数据传输

```java
 
 /* 服务端 */
 
 import org.apache.thrift.TProcessor; 
 import org.apache.thrift.protocol.TBinaryProtocol; 
 import org.apache.thrift.protocol.TBinaryProtocol.Factory; 
 import org.apache.thrift.server.TServer; 
 import org.apache.thrift.server.TThreadPoolServer; 
 import org.apache.thrift.transport.TServerSocket; 
 import org.apache.thrift.transport.TTransportException; 
 import service.demo.Hello; 
 import service.demo.HelloServiceImpl; 

 public class HelloServiceServer { 
    /** 
     * 启动 Thrift 服务器
     * @param args 
     */ 
    public static void main(String[] args) { 
        try { 
            // 设置服务端口为 7911 
            TServerSocket serverTransport = new TServerSocket(7911); 
            // 设置协议工厂为 TBinaryProtocol.Factory 
            Factory proFactory = new TBinaryProtocol.Factory(); 
            // 关联处理器与 Hello 服务的实现
            TProcessor processor = new Hello.Processor(new HelloServiceImpl()); 
            TServer server = new TThreadPoolServer(processor, serverTransport, 
                    proFactory); 
            System.out.println("Start server on port 7911..."); 
            server.serve(); 
        } catch (TTransportException e) { 
            e.printStackTrace(); 
        } 
    } 
 } 
 
 /* 客户端 */
 package service.client; 
 import org.apache.thrift.TException; 
 import org.apache.thrift.protocol.TBinaryProtocol; 
 import org.apache.thrift.protocol.TProtocol; 
 import org.apache.thrift.transport.TSocket; 
 import org.apache.thrift.transport.TTransport; 
 import org.apache.thrift.transport.TTransportException; 
 import service.demo.Hello; 

 public class HelloServiceClient { 
 /** 
     * 调用 Hello 服务
     * @param args 
     */ 
    public static void main(String[] args) { 
        try { 
            // 设置调用的服务地址为本地，端口为 7911 
            TTransport transport = new TSocket("localhost", 7911); 
            transport.open(); 
            // 设置传输协议为 TBinaryProtocol 
            TProtocol protocol = new TBinaryProtocol(transport); 
            Hello.Client client = new Hello.Client(protocol); 
            // 调用服务的 helloVoid 方法
            client.helloVoid(); 
            transport.close(); 
        } catch (TTransportException e) { 
            e.printStackTrace(); 
        } catch (TException e) { 
            e.printStackTrace(); 
        } 
    } 
 } 

```

### (2) TCompactProtocol 高效率的、密集的二进制编码格式进行数据传输

```java
/* 服务端替换Factory proFactory = new TBinaryProtocol.Factory();  */
TCompactProtocol.Factory proFactory = new TCompactProtocol.Factory(); 

/* 客户端替换Factory proFactory = new TBinaryProtocol.Factory();  */
 TCompactProtocol protocol = new TCompactProtocol(transport); 


```

### (3) TJSONProtocol —— 使用 JSON 的数据编码协议进行数据传输

```java
/* 服务端替换Factory proFactory = new TBinaryProtocol.Factory();  */
TJSONProtocol.Factory proFactory = new TJSONProtocol.Factory(); 

/* 客户端替换Factory proFactory = new TBinaryProtocol.Factory();  */
 TJSONProtocol protocol = new TJSONProtocol(transport); 

```

### (4) TSimpleJSONProtocol —— 只提供 JSON 只写的协议，适用于通过脚本语言解析


## 4. 传输层介绍

### (1) TSocket —— 使用阻塞式 I/O 进行传输，是最常见的模式

```
 /* 服务端 */
 
 import org.apache.thrift.TProcessor; 
 import org.apache.thrift.protocol.TBinaryProtocol; 
 import org.apache.thrift.protocol.TBinaryProtocol.Factory; 
 import org.apache.thrift.server.TServer; 
 import org.apache.thrift.server.TThreadPoolServer; 
 import org.apache.thrift.transport.TServerSocket; 
 import org.apache.thrift.transport.TTransportException; 
 import service.demo.Hello; 
 import service.demo.HelloServiceImpl; 

 public class HelloServiceServer { 
    /** 
     * 启动 Thrift 服务器
     * @param args 
     */ 
    public static void main(String[] args) { 
        try { 
            // 设置服务端口为 7911 
            TServerSocket serverTransport = new TServerSocket(7911); 
            // 设置协议工厂为 TBinaryProtocol.Factory 
            Factory proFactory = new TBinaryProtocol.Factory(); 
            // 关联处理器与 Hello 服务的实现
            TProcessor processor = new Hello.Processor(new HelloServiceImpl()); 
            TServer server = new TThreadPoolServer(processor, serverTransport, 
                    proFactory); 
            System.out.println("Start server on port 7911..."); 
            server.serve(); 
        } catch (TTransportException e) { 
            e.printStackTrace(); 
        } 
    } 
 } 
 
 /* 客户端 */
 package service.client; 
 import org.apache.thrift.TException; 
 import org.apache.thrift.protocol.TBinaryProtocol; 
 import org.apache.thrift.protocol.TProtocol; 
 import org.apache.thrift.transport.TSocket; 
 import org.apache.thrift.transport.TTransport; 
 import org.apache.thrift.transport.TTransportException; 
 import service.demo.Hello; 

 public class HelloServiceClient { 
 /** 
     * 调用 Hello 服务
     * @param args 
     */ 
    public static void main(String[] args) { 
        try { 
            // 设置调用的服务地址为本地，端口为 7911 
            TTransport transport = new TSocket("localhost", 7911); 
            transport.open(); 
            // 设置传输协议为 TBinaryProtocol 
            TProtocol protocol = new TBinaryProtocol(transport); 
            Hello.Client client = new Hello.Client(protocol); 
            // 调用服务的 helloVoid 方法
            client.helloVoid(); 
            transport.close(); 
        } catch (TTransportException e) { 
            e.printStackTrace(); 
        } catch (TException e) { 
            e.printStackTrace(); 
        } 
    } 
 } 
```

### (2) TFramedTransport 使用非阻塞方式，按块的大小进行传输

```
若使用 TFramedTransport 传输层，其服务器必须修改为非阻塞的服务类型,
```

```java
 /*服务端*/
 TNonblockingServerTransport serverTransport; 
 serverTransport = new TNonblockingServerSocket(10005); 
 Hello.Processor processor = new Hello.Processor(new HelloServiceImpl()); 
 TServer server = new TNonblockingServer(processor, serverTransport); 
 System.out.println("Start server on port 10005 ..."); 
 server.serve(); 
 
 /*客户端*/
 TTransport transport = new TFramedTransport(new TSocket("localhost", 10005)); 
 
```
>>完整列子

```java
/*服务端*/
	public static void main(String[] args) {
		// TODO Auto-generated method stub
		ExecutorService executor = Executors.newFixedThreadPool(100);
		int port = 8083;
        TNonblockingServerSocket serverTransport;
		try {
			serverTransport = new TNonblockingServerSocket(port);
	        TThreadedSelectorServer.Args tArgs = new TThreadedSelectorServer.Args(serverTransport);
	        tArgs.processor(new HelloService.Processor<HelloService.Iface>(new HelloServiceImpl()));
	        tArgs.transportFactory(new TFramedTransport.Factory());
	        // 二进制协议
	        tArgs.protocolFactory(new TBinaryProtocol.Factory());
	        tArgs.executorService(executor);
	        TServer server = new TThreadedSelectorServer(tArgs);
	        //executor.execute(server::serve);
	        System.out.println("Hello TThreadedSelectorServer....");
	       // server.serve(); // 启动服务
	        executor.execute(server::serve);
	        
		} catch (TTransportException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}

	}

	
/*客户端*/
TTransport transport = null;
transport = new TFramedTransport(new TSocket(SERVER_IP, SERVER_PORT, TIMEOUT));
// 协议要和服务端一致
TProtocol protocol = new TBinaryProtocol(transport);
// TProtocol protocol = new TCompactProtocol(transport);
// TProtocol protocol = new TJSONProtocol(transport);
Hello.Client client = new Hello.Client(protocol); 
transport.open();
// 调用服务的 helloVoid 方法
client.helloVoid(); 
transport.close();

					
```
