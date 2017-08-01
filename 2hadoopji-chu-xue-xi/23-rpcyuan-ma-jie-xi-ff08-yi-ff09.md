# 2.3 RPC源码解析（一）

---

在上一节的记录中，明确了RPC的主要流程分为三步。这一节来探究一下客户端是如何生成proxy动态对象。

注意：hadoop的RPC封装全部位于org.apache.hadoop.ipc这个package下。

在客户端生成proxy动态对象这个过程中，其实是还没有建立socket连接，只是把要调用的接口类名放到了一个代理对象里，拿到了这个代理对象proxy之后就可以去调用接口类的方法。在RPC的过程中，服务器与客户端约定了，接口类称为Protocol（协议）。那么这个接口类名是怎么与socket关联上的呢？关联上之后调用接口类的方法时，方法调用后就开始了socket通信，这个关联里做了什么手脚？带着这两个问题，开始...

> 接口类怎么与socket关联？

现在模拟一下RPC的过程，简单的代码跟踪如下：

代码 一：

```java
UserServiceInterface userServiceImpl = RPC.getProxy(UserServiceInterface.class, 1L, new InetSocketAddress("127.0.0.1", 10000), conf);

---------------------------------------深入getProxy----------------------------------------------------------------------------------------

public static <T> T getProxy(Class<T> protocol, long clientVersion, InetSocketAddress addr, Configuration conf) throws IOException {
        return getProtocolProxy(protocol, clientVersion, addr, conf).getProxy();
    }

---------------------------------------getProxy----------------------------------------------------------------------------------------

public T getProxy() {
        return this.proxy;
    }
```

从这段可以看出，getProtocolProxy\(\)这个方法才是真正的处理核心，getProxy\(\)只是简单获取一个proxy对象。继续深入getProtocolProxy\(protocol, clientVersion, addr, conf\)。

代码二：

```java
   public static <T> ProtocolProxy<T> getProtocolProxy(Class<T> protocol, long clientVersion, InetSocketAddress addr, Configuration conf) throws IOException {
        return getProtocolProxy(protocol, clientVersion, addr, conf, NetUtils.getDefaultSocketFactory(conf));
    }
---------------------------------------getDefaultSocketFactory----------------------------------------------------------------------------------------
    public static SocketFactory getDefaultSocketFactory(Configuration conf) {
        String propValue = conf.get("hadoop.rpc.socket.factory.class.default", "org.apache.hadoop.net.StandardSocketFactory");
        return propValue != null && propValue.length() != 0?getSocketFactoryFromProperty(conf, propValue):SocketFactory.getDefault();
    }
---------------------------------------深入getProtocolProxy----------------------------------------------------------------------------------------
    public static <T> ProtocolProxy<T> getProtocolProxy(Class<T> protocol, long clientVersion, InetSocketAddress addr, Configuration conf, SocketFactory factory) throws IOException {
        UserGroupInformation ugi = UserGroupInformation.getCurrentUser();
        return getProtocolProxy(protocol, clientVersion, addr, ugi, conf, factory);
    }
---------------------------------------深入getProtocolProxy----------------------------------------------------------------------------------------
    public static <T> ProtocolProxy<T> getProtocolProxy(Class<T> protocol, long clientVersion, InetSocketAddress addr, UserGroupInformation ticket, Configuration conf, SocketFactory factory) throws IOException {
        return getProtocolProxy(protocol, clientVersion, addr, ticket, conf, factory, getRpcTimeout(conf), (RetryPolicy)null);
    }
---------------------------------------深入getProtocolProxy----------------------------------------------------------------------------------------
    public static <T> ProtocolProxy<T> getProtocolProxy(Class<T> protocol, long clientVersion, InetSocketAddress addr, UserGroupInformation ticket, Configuration conf, SocketFactory factory, int rpcTimeout, RetryPolicy connectionRetryPolicy) throws IOException {
        return getProtocolProxy(protocol, clientVersion, addr, ticket, conf, factory, rpcTimeout, connectionRetryPolicy, (AtomicBoolean)null);
    }
---------------------------------------深入getProtocolProxy----------------------------------------------------------------------------------------
    public static <T> ProtocolProxy<T> getProtocolProxy(Class<T> protocol, long clientVersion, InetSocketAddress addr, UserGroupInformation ticket, Configuration conf, SocketFactory factory, int rpcTimeout, RetryPolicy connectionRetryPolicy, AtomicBoolean fallbackToSimpleAuth) throws IOException {
        if(UserGroupInformation.isSecurityEnabled()) {
            SaslRpcServer.init(conf);
        }

        return getProtocolEngine(protocol, conf).getProxy(protocol, clientVersion, addr, ticket, conf, factory, rpcTimeout, connectionRetryPolicy, fallbackToSimpleAuth);
    }
```

这里看到有一个getProtocolEngine\(\)的方法，它返回的是一个RpcEngine对象。

代码三：

```java
    static synchronized RpcEngine getProtocolEngine(Class<?> protocol, Configuration conf) {
        RpcEngine engine = (RpcEngine)PROTOCOL_ENGINES.get(protocol);
        if(engine == null) {
            Class impl = conf.getClass("rpc.engine." + protocol.getName(), WritableRpcEngine.class);
            engine = (RpcEngine)ReflectionUtils.newInstance(impl, conf);
            PROTOCOL_ENGINES.put(protocol, engine);
        }

        return engine;
    }
```

这段看到有一个PROTOCOL\_ENGINES，它是一个Map，用以保存已经存在的协议，也就是充当一个协议池的角色。

然后Class impl = conf.getClass\("rpc.engine." + protocol.getName\(\), WritableRpcEngine.class\);根据当前的配置项获取这个协议对应处理类的类名，默认协议处理的类都是org.apache.hadoop.ipc.WritableRpcEngine。最后就返回这个WritableRpcEngine。

接着看代码二，调用的是getProxy\(\);看看WritableRpcEngine的getProxy\(\)方法：

代码四：

```java
  /** Construct a client-side proxy object that implements the named protocol,
   * talking to a server at the named address. 
   * @param <T>*/
  @Override
  public <T> ProtocolProxy<T> getProxy(Class<T> protocol, long clientVersion,
                         InetSocketAddress addr, UserGroupInformation ticket,
                         Configuration conf, SocketFactory factory,
                         int rpcTimeout, RetryPolicy connectionRetryPolicy)
    throws IOException {
    return getProxy(protocol, clientVersion, addr, ticket, conf, factory,
      rpcTimeout, connectionRetryPolicy, null);
  }

-------------------------------------------------------------------------------------------------
/** Construct a client-side proxy object that implements the named protocol,
   * talking to a server at the named address. 
   * @param <T>*/
  @Override
  @SuppressWarnings("unchecked")
  public <T> ProtocolProxy<T> getProxy(Class<T> protocol, long clientVersion,
                         InetSocketAddress addr, UserGroupInformation ticket,
                         Configuration conf, SocketFactory factory,
                         int rpcTimeout, RetryPolicy connectionRetryPolicy,
                         AtomicBoolean fallbackToSimpleAuth)
    throws IOException {    

    if (connectionRetryPolicy != null) {
      throw new UnsupportedOperationException(
          "Not supported: connectionRetryPolicy=" + connectionRetryPolicy);
    }

    T proxy = (T) Proxy.newProxyInstance(protocol.getClassLoader(),
        new Class[] { protocol }, new Invoker(protocol, addr, ticket, conf,
            factory, rpcTimeout, fallbackToSimpleAuth));
    return new ProtocolProxy<T>(protocol, proxy, true);
  }
```





