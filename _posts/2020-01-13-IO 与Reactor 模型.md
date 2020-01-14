---
layout:     post
title:      Linux I/O 与 Java I/O
subtitle:   学习笔记
date:       2020-01-12
author:     sincosmos
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - I/O, Reactor
---  

## 什么是 I/O 
用户线程无法直接操作 I/O 设备（例如磁盘、网卡等），而是通过系统调用请求 kernel 来协助完成 I/O 操作。内核会为每个 I/O 设备维护一个缓冲区。**读操作**即从 I/O 设备获取的数据（等待数据到达），由内核将其读入到内核缓冲区，然后再拷贝到用户线程内存空间供线程使用。**写操作**用户线程写入 I/O 设备的数据（等待 I/O 设备就绪），先从线程的内存拷贝到内核缓冲区，然后再有内核负责将数据写入到 I/O 设备。 
 
1. 同步/异步 I/O  
	描述的是用户线程与内核的交互方式。**同步**是指用户线程发起 I/O 请求后，需要等待内核或者轮询内核 I/O 操作完成后才能继续执行；**异步**是指用户线程发起 I/O 请求后，无需等待内核的 I/O 操作，而是继续执行当前线程，而当内核 I/O 操作完成后会通知用户线程，或者调用用户线程注册的回调函数处理 I/O 事件。
2. 阻塞/非阻塞 I/O  
	描述的是用户线程调用内核 I/O 操作的方式。**阻塞**是指 I/O 操作需要彻底完成后才返回到用户空间；**非阻塞**是指 I/O 操作被调用后立即返回给用户一个状态值，无需等到 I/O 操作彻底完成，但是，用户线程需要重复检查（例如使用 select, poll, epoll 等） I/O 状态，找到 ready 的 I/O 处理相应的事件。
	
## 流与通道
Java BIO 是面向流的，数据转移的形态如下图所示。
![](https://static.javatpoint.com/core/javanio/images/nio-tutorial6.png)
Java NIO 是面向 buffer 的，数据会先被读到 jvm 内存中的数据缓冲区，然后线程通过 channel 处理数据。
![](https://static.javatpoint.com/core/javanio/images/nio-tutorial7.png)

stream 和 channel 的主要区别是：stream 的数据传输是单向的（读入数据使用 InputStream，写数据使用 OutputStream），channel 可以双向传输数据。

## I/O 模型
下面以网络 I/O 读请求为例来说明各种 I/O 模型。
1. 同步阻塞 I/O  
用户线程发起 I/O 读的系统调用后，持续等待内核从 I/O 设备获得数据，即使 I/O 设备并没有数据读入；内核缓冲区 Ready 后，用户线程阻塞等待内核将缓冲区数据复制到线程的内存空间，直到此时，用户线程才解除阻塞，开始处理线程内存区数据。另外，如果发生错误（如信号中断），也会使用户线程解除阻塞。  
Java Blocking I/O 就是基于同步阻塞 I/O 的实现的。例如下面 ServerSocket，如果单线程处理所有的客户端连接，那么多个客户端连接需要排队等待；如果采用多线程处理并发的客户端连接，那么需要注意使用线程池来限制并发数量，同时当并发较高时也会面临客户端连接排队等待。传统的 Web 应用多数采用同步阻塞 I/O 配合线程池来处理用户并发请求。  
```
try {
	ServerSocket serverSocket = new ServerSocket(port);
    while(listening){
        /* Wait for the client to make a connection (blocks) and when it does,
        create a new socket to handle the request */
        Socket socket = serverSocket.accept();

        //1. 服务器单线程处理客户端连接
        /*try{
            PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
            
            BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));

            String request, response;
            
            while((request=in.readLine()) != null){
                response = processRequest(request);
                out.println(response);
                if("Done".equals(request)) break;
            }
        } catch (IOException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
        	  //必须关闭当前连接才能开启下一个
            socket.close();
        }*/

        //2. Handle each connection in a new thread to manage concurrent users
        new Thread(()->{
            try{
                PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
                BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));

                String request, response;
                while((request=in.readLine()) != null){
                    response = processRequest(request);
                    out.println(response);
                    if("Done".equals(request)) break;
                }
                socket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }).start();
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

2. 同步非阻塞 I/O  
用户线程发起 I/O 读的系统调用后，如果内核缓冲区数据不是处于就绪状态，则立即返回；如果内核缓冲区数据处于就绪状态，则会触发内核将内核缓冲区数据拷贝到用户内存空间的，用户线程将阻塞等待，直到拷贝完成后用户线程解除阻塞，开始处理相应的数据。同步非阻塞 I/O 下，用户线程可能通常需要循环调用 recvfrom 系统方法，不断询问内核数据是否 ready。  
Java NIO 就是基于同步非阻塞 I/O 的实现的。例如下面 serverSocketChannel，用户线程持续询问是否有 ready 的连接，如果有 ready 的连接，就处理其数据。示例中是在当先线程中进行处理的，当然也可以类似上面例子中，使用线程池技术并发处理多个用户连接，但实际上，使用 Java NIO 一般会使用 I/O 多路复用技术实现。  
```
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
//非阻塞
serverSocketChannel.configureBlocking(false);
serverSocketChannel.bind(new InetSocketAddress(port));
while(true){
    System.out.println("waiting for connections...");
    // return inmediately because blocking is set to fasle
    // if no incoming connection had arrived, null would be returned
    // 轮询查询是否有 ready 的连接，空转消耗 CPU 资源
    SocketChannel sc = server.accept();
    //如果有 ready 的连接
    if(sc != null){
        System.out.println("Incoming connection from: "
                + sc.socket().getRemoteSocketAddress());
        buffer.clear();
        if(sc.read(buffer) > 0){
            buffer.flip();
            while(buffer.hasRemaining())
                System.out.print((char) buffer.get());
        }
        sc.close();
    }
}
```
3. I/O 多路复用  
I/O 复用，也称为事件驱动 I/O，Reactor 模式，就是在单个线程中同时监控多个 Socket，使用系统方法 select, poll 或者 epoll 监控这些 Socket 是否有数据到达，如果某（些）个 socket 有数据到达（内核缓冲区），用户线程开始阻塞等待数据从内核缓冲区拷贝到用户线程内存区，拷贝完成后用户线程解除阻塞，并开始处理数据。  
![I/O 多路分离](https://images0.cnblogs.com/blog/405877/201411/142332187256396.png)  
与同步阻塞 I/O 对比：  
同步阻塞 I/O 一个线程只能处理一个 socket，单线程下如果要处理下一个连接，只能等待前一个 socket 关闭，前一个连接的阻塞期会导致后一个连接等待时间增加；I/O 多路复用可以单线程处理多个连接，前一个线程的阻塞期不会影响下一个连接的等待时间。当然，在同步阻塞 I/O 中，可以使用多线程来避免各个连接阻塞期的叠加，但线程池中线程数量有限，线程本身开销较大，在极大并发的情况下性能较差。这也是 Redis I/O 多路复用单线程性能优越的原因。  
与同步非阻塞 I/O 对比：  
在没有 I/O 就绪的情况下，select 操作会阻塞，从而避免 CPU 空转，同时，epoll 技术可以避免遍历查询所有被 Selector 监控的连接，而只找出就绪的连接即可，节省计算资源。

可以把循环调用 select 的操作从用户业务逻辑中抽离出来，专门交给一个模块去做，我们称这个模块为 Reactor。而用户线程向 Reactor 进行注册它感兴趣的 I/O 事件，当 Reactor 发现用户线程感兴趣的 I/O 事件时，再去调用相应线程注册的 EventHandler 处理相应的事件。  
![Reactor 模式](https://images0.cnblogs.com/blog/405877/201411/142333254136604.png)  

```
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
//非阻塞
serverSocketChannel.configureBlocking(false);
serverSocketChannel.bind(new InetSocketAddress(port));
    
// serverSocketChannel is ready for connection
serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
System.out.println("starting server on port >> " + port);
while(true){
    // wait for ready events
    // ready for connection, ready for reading, ready for writing
    // if no event is ready, select blocks, cpu 资源被节省下来
    // 同时，如果多个线程同时进行 I/O 操作，各个线程都不会被阻塞，只有 select 线程会被阻塞
    if(selector.select() == 0) continue;
    Set<SelectionKey> selectKeys = selector.selectedKeys();
    Iterator<SelectionKey> iter = selectKeys.iterator();
    while(iter.hasNext()){
        SelectionKey selectionKey = iter.next();
        if(selectionKey.isAcceptable()){
            // previous registered serverSocketChannel is ready for connection
            ServerSocketChannel server = (ServerSocketChannel) selectionKey.channel();
            // after a client is connected, a new SocketChannel is returned
            SocketChannel client = server.accept();
            client.configureBlocking(false);
            System.out.println("Connected by: " + client.socket().getRemoteSocketAddress());
            // register the socketChannel between server and client to the selector
            client.register(selector, SelectionKey.OP_READ);
        }else if(selectionKey.isReadable()){
            SocketChannel client = (SocketChannel) selectionKey.channel();
            //System.out.println("channel ready for read >>" + client.socket().getRemoteSocketAddress());
            ByteBuffer buffer = ByteBuffer.allocate(8);
            // 当客户端不再传来数据（客户端关闭）时，必须关闭这里的 channel，否则每次 select 后，这个 channel 都会是可读状态
            // channel.close() 内部会调用 selectionKey 的 cancel() 方法，将其从 selector 中取消注册
            if(client.read(buffer) < 0) client.close();
            buffer.flip();
            while(buffer.hasRemaining()){
                System.out.print((char)buffer.get());
            }
        }else if(selectionKey.isWritable()){
            SocketChannel client = (SocketChannel) selectionKey.channel();
            client.write(ByteBuffer.wrap("Server sending message to client...\n".getBytes()));
        }
        // we have to remove the processed ready event from the "ready table"
        // the ready table which returned by calling of selector.selectedKeys() method will not remove
        // the "ready channel" returned by previous call of selector.selectedKeys() method
        iter.remove();
    }
}
```
4. 信号驱动 I/O   
现在用的较少。  
5. 异步 I/O  
即 Proactor 模式，需要操作系统底层支持，Linux 内核 2.5 版本中首次出现，2.6 版本才开始成为标准的内核功能。其核心工作流是用户线程发起一个 I/O 操作后，不阻塞，继续执行线程。内核将线程的 I/O 请求处理完成后，通过回调函数通知用户线程结果，如果是读操作，用户线程可以直接在线程指定的缓冲区获得 I/O 数据。  
Java 1.7 版本在 NIO 包中加入了对异步 I/O 的支持，被称为 JAVA AIO，主要包括 AsynchronousSocketChannel，AsynchronousServerSocketChannel 和 AsynchronousFileChannel 这三个类。Java AIO 提供了两种使用方式，分别是返回 Futrue 实例或传入回调函数（java.nio.channels.CompletionHandler）。
  





