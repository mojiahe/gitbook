# 2.5 RPC源码解析（三）

---

上两节记录了RPC是客户端是如何通过动态代理建立socket连接、发送数据和接收响应数据。这一节记录服务端是如何建立服务、响应请求。

> 服务端是如何建立服务？

先上服务端的简单代码。

代码一：

```java
Configuration conf = new Configuration();
        RPC.Builder builder = new RPC.Builder(conf);
        builder.setInstance(new UserServiceImpl()).setPort(10000).setBindAddress("127.0.0.1").setProtocol(UserServiceInterface.class);
        RPC.Server server = builder.build();
        server.start();
```

这里就是通过创建一个RPC的Builder对象保存服务的地址和端口以及要发布的RPC协议，然后通过Builder对象来获取Server对象并启动监听服务。深入探究一下builder的过程到底是怎么回事。

代码二：

```java
/**
     * Build the RPC Server. 
     * @throws IOException on error
     * @throws HadoopIllegalArgumentException when mandatory fields are not set
     */
    public Server build() throws IOException, HadoopIllegalArgumentException {
      if (this.conf == null) {
        throw new HadoopIllegalArgumentException("conf is not set");
      }
      if (this.protocol == null) {
        throw new HadoopIllegalArgumentException("protocol is not set");
      }
      if (this.instance == null) {
        throw new HadoopIllegalArgumentException("instance is not set");
      }

      return getProtocolEngine(this.protocol, this.conf).getServer(
          this.protocol, this.instance, this.bindAddress, this.port,
          this.numHandlers, this.numReaders, this.queueSizePerHandler,
          this.verbose, this.conf, this.secretManager, this.portRangeConfig);
    }
  }

  --------------------------------------------------------------------------------------------
  /* Construct a server for a protocol implementation instance listening on a
   * port and address. */
  @Override
  public RPC.Server getServer(Class<?> protocolClass,
                      Object protocolImpl, String bindAddress, int port,
                      int numHandlers, int numReaders, int queueSizePerHandler,
                      boolean verbose, Configuration conf,
                      SecretManager<? extends TokenIdentifier> secretManager,
                      String portRangeConfig) 
    throws IOException {
    return new Server(protocolClass, protocolImpl, conf, bindAddress, port,
        numHandlers, numReaders, queueSizePerHandler, verbose, secretManager,
        portRangeConfig);
  }
```

build的过程的目的就是创建一个server对象，并且在创建Server这个对象时会把这个协议以及它的实现保存成RPC CALLS。

继续看代码一中是如何启动服务的。

代码三：

```java
  /** Starts the service.  Must be called before any calls will be handled. */
  public synchronized void start() {
    //响应线程，负责返回数据给客户端
    responder.start();
    //监听线程，负责监听socket并创建处理请求的对象
    listener.start();
    handlers = new Handler[handlerCount];

    for (int i = 0; i < handlerCount; i++) {
      handlers[i] = new Handler(i);
      handlers[i].start();
    }
  }
```

至此，服务器端完成了启动。

> 服务端是如何响应请求？

详细的细节需要分析上面代码的listener和responder两个线程。这其中涉及了JAVA NIO的使用，这里就留给日后一个完善的机会。

到此，Hadoop的RPC就分析完毕了。其中动态代理的使用感觉很牛逼，这里反射的使用相对动态代理来说太平常了。

