---
title: Netty之构建Netty应用程序
date: 2018-06-14 21:53:00  
tags: [Netty]    
categories: Netty  
toc: true
---

### 一、编写Netty服务器
所有的Netty服务器都需要以下两个部分：  
- 至少一个ChannelHandler——用于处理服务器从客户端接收的数据  
- 引导Bootstrap——这是配置服务器的启动代码，会将服务器绑定到它要监听连接请求的端口上  
<!-- more -->

#### ChanneleHandler和业务逻辑
Netty服务器会响应传入的消息，所以它需要实现ChannelInboundHandler接口，用来定义响应入站事件的方法。简单的应用程序继承ChannelInboundHandlerAdapter类就可以了，它提供了ChannelInboundHandler的默认实现。  
其中，我们感兴趣的方法是：
- channelRead()——对于每个传入的消息都要调用
- channelReadComplete()——通知handler最后一次对channelRead()的调用是当前批量读取中的最后一条消息
- exceptionCaught()——在读取操作期间，有异常抛出时会调用

下面来编写这个EchoServerHandler：

```
//表示ChannelHandler可以被多个Channel安全的共享
@Sharable
public class EchoServerHandler extends ChannelInboundHandlerAdapter{
    
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg){
        ByteBuf in = (ByteBuf) msg;
        System.out.println("Server received: " + in.toString(CharasetUtil.UTF_8));
        ctx.write(in);
    }
    
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx){
        ctx.writeAndFlush(Unpooled.EMPTY_BUFFER)
        .addListener(ChannelFutureListener.CLOSE);
    }
    
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause){
        cause.printStackTrace();
        ctx.close();
    }
}

```
每个Channel都拥有与之相关联的ChannelPipeline，其持有一个ChannelHandler的实例链。如果没有捕获异常，那么所接收的异常会被传递到ChannelPipeline的尾端并被记录。  

#### 引导服务器
下面编写EchoServer类：

```
public clasee EchoServer{
    private final int port;
    
    public EchoServer(int port){
        this.port = port;
    }
    
    public static void main(String[] args) throws Exception{
        int port = Integer.parseInt(args[0]);
        new EchoServer(port).start();
    }
    
    public void start() throws Exception{
        final EchoServerHandler serverHandler = new EchoServerHandler();
        EventLoopGroup group = new NioEventLoopGroup();
        try{
            ServerBootstrap b = new ServerBootstrap();
            b.group(group)
            .channel(NioServerSocketChannel.class)
            .localAddress(new InetSocketAddress(port))
            //当一个新的连接接受时，就会创建一个新的子Channel，
            //ChannelInitializer会把serverHandler实例添加到该Channel的ChannelPipeline中
            .childHandler(new ChannelInitializer<SocketChannel>(){ 
                @Override
                public void initChannel(SocketChannel ch) throws Exception{
                    ch.pipeline().addLast(serverHandler);
                }
            });
            //异步绑定服务器，调用sync()阻塞等待直到绑定完成
            ChannelFuture f = b.bind().sync();
            f.channel().closeFuture().sync();
        }finally{
            //关闭EventLoopGroup，释放所有资源
            group.shutdownGracefully().sync();
        }
    }
    
}
```
### 二、编写客户端
客户端主要完成以下工作：  
- 连接到服务器
- 发送一个或多个消息
- 对于每个消息，等待并接收从服务器发回的相同的消息
- 关闭连接

#### 通过ChannelHandler实现客户端逻辑
跟服务器类似，客户端需要一个用来处理数据的ChannelInboundHandler，在此场景，我们扩展SimpleChannelInboundHandler类来处理所有必须的任务，我们要重写下面几个方法：  
- channelActive()——在跟服务器的连接已经建立之后会被调用
- channelRead0()——从服务器接收到一条消息时会被调用
- exceptionCaught()——在处理过程中引发异常时调用  

客户端EchoClientHandler如下：

```
@Sharable
public class EchoClientHandler extends SimpleChannelInboundHandler<ByteBuf>{
    
    //在一个连接被建立时调用
    @Override
    public void channelActive(ChannelHandlerContext ctx){
        ctx.writeAndFlush(Unpooled.copiedBuffer("Netty works!", CharasetUtil.UTF_8));
    }
    
    //每当接收数据时，都会调用这个方法
    @Override
    public void channelRead0(ChannelHandlerContext ctx, ByteBuf in){
        System.out.print("Client received :" + in.toString(CharasetUtil.UTF_8));
    }
    
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause){
        cause.printStackTrace();
        ctx.close();
    }
}
```
##### SimpleChannelInboundHandler与ChannelInboundHandler的区别
在客户端，当channelRead0()方法完成后，客户端已经有了传入消息，并且已经处理完成了，在这个方法返回时，SimpleChannelInboundHandler负责释放指向保存该消息的ByteBuf的内存引用；  

而在服务端中，仍然需要将传入的消息回送给发送者。而ctx.write(in)是异步的，直到channelRead()方法返回后可能仍然没有完成，所以serverHandler扩展了ChannelInboundHandler，它不会再这个时间点上释放消息的ByteBuf。

#### 引导客户端
与引导服务端类似，知识客户端需要用主机和端口参数连接远程地址。  
EchoClient类如下：

```
public class EchoClient{
    private final String host;
    private final int port;
    
    public EchoClient(String host, int port){
        this.host = host;
        this.port = port;
    }
    
    public void start() throws Exception{
        EventLoopGroup group = new NioEventLoopGroup();
        try{
            Bootstrap b = new Bootstrap();
            b.group(group)
            .channel(NioSocketChannel.class)
            .remoteAddress(new InetSocketAddress(host, port))
            .handler(new ChannelIntializer<SocketChannel>(){
                @Override
                public void initChannel(SocketChannel ch) throws Exception{
                    ch.pipeline().addLast(new EchoClientHandler());
                }
            });
            //阻塞等待连接完成
            ChannelFuture d = b.connect().sync();
            //阻塞直到关闭
            f.channel().closeFuture().sync();
        }finally{
            group.shutdownGracefully().sync();
        }
        
    }
    
    public static void main(String[] args){
        String host = args[0];
        int port = Integer.parseInt(args[1]);
        new EchoClient(host, port).start();
    }
}
```
运行结果：
```
Server received: Netty works!
```

这样，我们的Netty应用程序就编写好了。
