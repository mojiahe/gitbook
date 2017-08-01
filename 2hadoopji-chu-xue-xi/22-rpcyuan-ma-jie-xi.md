# 2.3 RPC源码解析（二）

---

在上一节的记录中，明确了客户端是如何生成proxy动态对象，在动态对象生成过程中只是生成了一个代理对象并且保存了远端服务器端的地址、端口以及一些认证信息、协议的接口类到proxy对象中。现在来看看客户端是如何使用这些信息与服务器进行通信的。

带这三个主要问题来分析RPC客户端源码：

* 如何与服务器端建立RPC连接？
* 客户端如何发送数据？
* 客户端如何接收返回的相应数据？

> ###### 如何与服务器端建立RPC连接？

，其中有如下方法：

**代码一：**

```java
String user = userServiceImpl.login("test", "123456");
-------------------------------------------------------------
@Override
    public Object invoke(Object proxy, Method method, Object[] args)
      throws Throwable {
      long startTime = 0;
      if (LOG.isDebugEnabled()) {
        startTime = Time.now();
      }
      TraceScope traceScope = null;
      if (Trace.isTracing()) {
        traceScope = Trace.startSpan(RpcClientUtil.methodToTraceString(method));
      }
      ObjectWritable value;
      try {
        value = (ObjectWritable)
          client.call(RPC.RpcKind.RPC_WRITABLE, new Invocation(method, args),
            remoteId, fallbackToSimpleAuth);
      } finally {
        if (traceScope != null) traceScope.close();
      }
      if (LOG.isDebugEnabled()) {
        long callTime = Time.now() - startTime;
        LOG.debug("Call: " + method.getName() + " " + callTime);
      }
      return value.get();
    }
```



**代码二：**

```java
public Writable call(RPC.RpcKind rpcKind, Writable rpcRequest,
      ConnectionId remoteId, int serviceClass,
      AtomicBoolean fallbackToSimpleAuth) throws IOException {
    //创建一个call远程调用对象
    final Call call = createCall(rpcKind, rpcRequest);
    //RPC连接的建立
    Connection connection = getConnection(remoteId, call, serviceClass,
      fallbackToSimpleAuth);
    try {
    //发送数据
      connection.sendRpcRequest(call);                 // send the rpc request
    } catch (RejectedExecutionException e) {
      throw new IOException("connection has been closed", e);
    } catch (InterruptedException e) {
      Thread.currentThread().interrupt();
      LOG.warn("interrupted waiting to send rpc request to server", e);
      throw new IOException(e);
    }

//等待服务器端返回数据
    synchronized (call) {
      while (!call.done) {
        try {
          call.wait();                           // wait for the result
        } catch (InterruptedException ie) {
          Thread.currentThread().interrupt();
          throw new InterruptedIOException("Call interrupted");
        }
      }

      if (call.error != null) {
        if (call.error instanceof RemoteException) {
          call.error.fillInStackTrace();
          throw call.error;
        } else { // local exception
          InetSocketAddress address = connection.getRemoteAddress();
          throw NetUtils.wrapException(address.getHostName(),
                  address.getPort(),
                  NetUtils.getHostname(),
                  0,
                  call.error);
        }
      } else {
        return call.getRpcResponse();
      }
    }
  }
```

在上面的源码位于org.apache.hadoop.ipc.Client这个类，主要是做了三个事情：RPC连接的建立、发送请求数据、接收响应结果。

我们可以看到是首先创建了一个Call对象，这个Call对象在这里代表的就是一个远程调用对象，并且这个对象的实例在后面也相继使用，尤其是用来等待服务器返回响应数据。

紧接着，就是创建了一个collection对象，这里的collection对象官方解析是：读取响应并通知调用者的一个线程；每个collection对象都拥有连接到远程地址的socket。通过保存这个collection对象以达到复用此套接字的效果。

这里继续深入了解这个collection的建立过程。

**代码三：**

```java
/** Get a connection from the pool, or create a new one and add it to the
   * pool.  Connections to a given ConnectionId are reused. */
  private Connection getConnection(ConnectionId remoteId,
      Call call, int serviceClass, AtomicBoolean fallbackToSimpleAuth)
      throws IOException {
    if (!running.get()) {
      // the client is stopped
      throw new IOException("The client is stopped");
    }
    Connection connection;
    /* we could avoid this allocation for each RPC by having a  
     * connectionsId object and with set() method. We need to manage the
     * refs for keys in HashMap properly. For now its ok.
     */
    do {
      synchronized (connections) {
        connection = connections.get(remoteId);
        if (connection == null) {
          connection = new Connection(remoteId, serviceClass);
          connections.put(remoteId, connection);
        }
      }
    } while (!connection.addCall(call));

    //we don't invoke the method below inside "synchronized (connections)"
    //block above. The reason for that is if the server happens to be slow,
    //it will take longer to establish a connection and that will slow the
    //entire system down.
    connection.setupIOstreams(fallbackToSimpleAuth);
    return connection;
  }
```

这段代码的作用是从连接池collections中以remoteId来获取一个collection对象，如果存在则拿出来用，不存在就创建新的collection并放回连接池中去。最主要的是connection.setupIOstreams这个方法，继续深入。

**代码四：**

```java
 /** Connect to the server and set up the I/O streams. It then sends
     * a header to the server and starts
     * the connection thread that waits for responses.
     */
    private synchronized void setupIOstreams(
        AtomicBoolean fallbackToSimpleAuth) {
      if (socket != null || shouldCloseConnection.get()) {
        return;
      } 
      try {
        if (LOG.isDebugEnabled()) {
          LOG.debug("Connecting to "+server);
        }
        if (Trace.isTracing()) {
          Trace.addTimelineAnnotation("IPC client connecting to " + server);
        }
        short numRetries = 0;
        Random rand = null;
        while (true) {
          setupConnection();
          InputStream inStream = NetUtils.getInputStream(socket);
          OutputStream outStream = NetUtils.getOutputStream(socket);
          writeConnectionHeader(outStream);
          if (authProtocol == AuthProtocol.SASL) {
            final InputStream in2 = inStream;
            final OutputStream out2 = outStream;
            UserGroupInformation ticket = remoteId.getTicket();
            if (ticket.getRealUser() != null) {
              ticket = ticket.getRealUser();
            }
            try {
              authMethod = ticket
                  .doAs(new PrivilegedExceptionAction<AuthMethod>() {
                    @Override
                    public AuthMethod run()
                        throws IOException, InterruptedException {
                      return setupSaslConnection(in2, out2);
                    }
                  });
            } catch (Exception ex) {
              authMethod = saslRpcClient.getAuthMethod();
              if (rand == null) {
                rand = new Random();
              }
              handleSaslConnectionFailure(numRetries++, maxRetriesOnSasl, ex,
                  rand, ticket);
              continue;
            }
            if (authMethod != AuthMethod.SIMPLE) {
              // Sasl connect is successful. Let's set up Sasl i/o streams.
              inStream = saslRpcClient.getInputStream(inStream);
              outStream = saslRpcClient.getOutputStream(outStream);
              // for testing
              remoteId.saslQop =
                  (String)saslRpcClient.getNegotiatedProperty(Sasl.QOP);
              LOG.debug("Negotiated QOP is :" + remoteId.saslQop);
              if (fallbackToSimpleAuth != null) {
                fallbackToSimpleAuth.set(false);
              }
            } else if (UserGroupInformation.isSecurityEnabled()) {
              if (!fallbackAllowed) {
                throw new IOException("Server asks us to fall back to SIMPLE " +
                    "auth, but this client is configured to only allow secure " +
                    "connections.");
              }
              if (fallbackToSimpleAuth != null) {
                fallbackToSimpleAuth.set(true);
              }
            }
          }

          if (doPing) {
            inStream = new PingInputStream(inStream);
          }
          this.in = new DataInputStream(new BufferedInputStream(inStream));

          // SASL may have already buffered the stream
          if (!(outStream instanceof BufferedOutputStream)) {
            outStream = new BufferedOutputStream(outStream);
          }
          this.out = new DataOutputStream(outStream);

          writeConnectionContext(remoteId, authMethod);

          // update last activity time
          touch();

          if (Trace.isTracing()) {
            Trace.addTimelineAnnotation("IPC client connected to " + server);
          }

          // start the receiver thread after the socket connection has been set
          // up
          start();
          return;
        }
      } catch (Throwable t) {
        if (t instanceof IOException) {
          markClosed((IOException)t);
        } else {
          markClosed(new IOException("Couldn't set up IO streams", t));
        }
        close();
      }
    }
```

这个方法里做了好几个事情，从大方向来看，首先是通过setupConnection\(\)去建立一个socket连接，然后是通过start\(\)启动这个接收线程去监听socket返回的响应。

其中setupConnection\(\)方法如下：

**代码五：**

```java
private synchronized void setupConnection() throws IOException {
      short ioFailures = 0;
      short timeoutFailures = 0;
      while (true) {
        try {
          this.socket = socketFactory.createSocket();
          this.socket.setTcpNoDelay(tcpNoDelay);
          this.socket.setKeepAlive(true);

          /*
           * Bind the socket to the host specified in the principal name of the
           * client, to ensure Server matching address of the client connection
           * to host name in principal passed.
           */
          UserGroupInformation ticket = remoteId.getTicket();
          if (ticket != null && ticket.hasKerberosCredentials()) {
            KerberosInfo krbInfo = 
              remoteId.getProtocol().getAnnotation(KerberosInfo.class);
            if (krbInfo != null && krbInfo.clientPrincipal() != null) {
              String host = 
                SecurityUtil.getHostFromPrincipal(remoteId.getTicket().getUserName());

              // If host name is a valid local address then bind socket to it
              InetAddress localAddr = NetUtils.getLocalInetAddress(host);
              if (localAddr != null) {
                this.socket.bind(new InetSocketAddress(localAddr, 0));
              }
            }
          }

          NetUtils.connect(this.socket, server, connectionTimeout);
          if (rpcTimeout > 0) {
            pingInterval = rpcTimeout;  // rpcTimeout overwrites pingInterval
          }
          this.socket.setSoTimeout(pingInterval);
          return;
        } catch (ConnectTimeoutException toe) {
          /* Check for an address change and update the local reference.
           * Reset the failure counter if the address was changed
           */
          if (updateAddress()) {
            timeoutFailures = ioFailures = 0;
          }
          handleConnectionTimeout(timeoutFailures++,
              maxRetriesOnSocketTimeouts, toe);
        } catch (IOException ie) {
          if (updateAddress()) {
            timeoutFailures = ioFailures = 0;
          }
          handleConnectionFailure(ioFailures++, ie);
        }
      }
    }
```

核心 NetUtils.connect\(this.socket, server, connectionTimeout\);

**代码六：**

```java
/**
   * Like {@link NetUtils#connect(Socket, SocketAddress, int)} but
   * also takes a local address and port to bind the socket to. 
   * 
   * @param socket
   * @param endpoint the remote address
   * @param localAddr the local address to bind the socket to
   * @param timeout timeout in milliseconds
   */
  public static void connect(Socket socket, 
                             SocketAddress endpoint,
                             SocketAddress localAddr,
                             int timeout) throws IOException {
    if (socket == null || endpoint == null || timeout < 0) {
      throw new IllegalArgumentException("Illegal argument for connect()");
    }

    SocketChannel ch = socket.getChannel();

    if (localAddr != null) {
      Class localClass = localAddr.getClass();
      Class remoteClass = endpoint.getClass();
      Preconditions.checkArgument(localClass.equals(remoteClass),
          "Local address %s must be of same family as remote address %s.",
          localAddr, endpoint);
      socket.bind(localAddr);
    }

    try {
      if (ch == null) {
        // let the default implementation handle it.
        socket.connect(endpoint, timeout);
      } else {
        SocketIOWithTimeout.connect(ch, endpoint, timeout);
      }
    } catch (SocketTimeoutException ste) {
      throw new ConnectTimeoutException(ste.getMessage());
    }

    // There is a very rare case allowed by the TCP specification, such that
    // if we are trying to connect to an endpoint on the local machine,
    // and we end up choosing an ephemeral port equal to the destination port,
    // we will actually end up getting connected to ourself (ie any data we
    // send just comes right back). This is only possible if the target
    // daemon is down, so we'll treat it like connection refused.
    if (socket.getLocalPort() == socket.getPort() &&
        socket.getLocalAddress().equals(socket.getInetAddress())) {
      LOG.info("Detected a loopback TCP socket, disconnecting it");
      socket.close();
      throw new ConnectException(
        "Localhost targeted connection resulted in a loopback. " +
        "No daemon is listening on the target port.");
    }
  }
```

到了这里，socket真正建立了，就是通过java的网络编程来监听一个固定端口，终于找到当初在教科书上写得简单例子了。呵呵！

> ###### 客户端如何发送数据？

我们回到代码二，发送数据在这个方法connection.sendRpcRequest\(call\);

**代码七：**

```java
/** Initiates a rpc call by sending the rpc request to the remote server.
     * Note: this is not called from the Connection thread, but by other
     * threads.
     * @param call - the rpc request
     */
    public void sendRpcRequest(final Call call)
        throws InterruptedException, IOException {
      if (shouldCloseConnection.get()) {
        return;
      }

      // Serialize the call to be sent. This is done from the actual
      // caller thread, rather than the sendParamsExecutor thread,

      // so that if the serialization throws an error, it is reported
      // properly. This also parallelizes the serialization.
      //
      // Format of a call on the wire:
      // 0) Length of rest below (1 + 2)
      // 1) RpcRequestHeader  - is serialized Delimited hence contains length
      // 2) RpcRequest
      //
      // Items '1' and '2' are prepared here. 
      final DataOutputBuffer d = new DataOutputBuffer();
      RpcRequestHeaderProto header = ProtoUtil.makeRpcRequestHeader(
          call.rpcKind, OperationProto.RPC_FINAL_PACKET, call.id, call.retry,
          clientId);
      header.writeDelimitedTo(d);
      call.rpcRequest.write(d);

      synchronized (sendRpcRequestLock) {
        Future<?> senderFuture = sendParamsExecutor.submit(new Runnable() {
          @Override
          public void run() {
            try {
              synchronized (Connection.this.out) {
                if (shouldCloseConnection.get()) {
                  return;
                }

                if (LOG.isDebugEnabled())
                  LOG.debug(getName() + " sending #" + call.id);

                byte[] data = d.getData();
                int totalLength = d.getLength();
                out.writeInt(totalLength); // Total Length
                out.write(data, 0, totalLength);// RpcRequestHeader + RpcRequest
                out.flush();
              }
            } catch (IOException e) {
              // exception at this point would leave the connection in an
              // unrecoverable state (eg half a call left on the wire).
              // So, close the connection, killing any outstanding calls
              markClosed(e);
            } finally {
              //the buffer is just an in-memory buffer, but it is still polite to
              // close early
              IOUtils.closeStream(d);
            }
          }
        });

        try {
          senderFuture.get();
        } catch (ExecutionException e) {
          Throwable cause = e.getCause();

          // cause should only be a RuntimeException as the Runnable above
          // catches IOException
          if (cause instanceof RuntimeException) {
            throw (RuntimeException) cause;
          } else {
            throw new RuntimeException("unexpected checked exception", cause);
          }
        }
      }
    }
```

这里就是使用collection的DataOutputStream去将请求数据发送的服务器端。

> ###### 客户端如何接收返回的相应数据？

接收服务器端返回的数据在代码二的一段：

```java
    synchronized (call) {
      while (!call.done) {
        try {
          call.wait();                           // wait for the result
        } catch (InterruptedException ie) {
          Thread.currentThread().interrupt();
          throw new InterruptedIOException("Call interrupted");
        }
      }
```

这里直接调用wait\(\)；方法阻塞等待返回的结果。

这里有一个问题就是wait方法需要notify方法或者notifyAll方法来唤醒当前线程，那是哪里做到的呢？唤醒之后，又是怎么知道是这个调用等待的结果呢？

还记得代码四吗？connection建立之后，调用了start，而connection就是一个thread，所以start之后就是调用run方法，我们来分析run方法。

**代码八：**

```java
@Override
    public void run() {
      if (LOG.isDebugEnabled())
        LOG.debug(getName() + ": starting, having connections " 
            + connections.size());

      try {
        while (waitForWork()) {//wait here for work - read or close connection
          receiveRpcResponse();
        }
      } catch (Throwable t) {
        // This truly is unexpected, since we catch IOException in receiveResponse
        // -- this is only to be really sure that we don't leave a client hanging
        // forever.
        LOG.warn("Unexpected error reading responses on connection " + this, t);
        markClosed(new IOException("Error reading responses", t));
      }

      close();

      if (LOG.isDebugEnabled())
        LOG.debug(getName() + ": stopped, remaining connections "
            + connections.size());
    }

    --------------------------------------------------------------------------------------------------------------
     /* Receive a response.
     * Because only one receiver, so no synchronization on in.
     */
    private void receiveRpcResponse() {
      if (shouldCloseConnection.get()) {
        return;
      }
      touch();

      try {
        int totalLen = in.readInt();
        RpcResponseHeaderProto header = 
            RpcResponseHeaderProto.parseDelimitedFrom(in);
        checkResponse(header);

        int headerLen = header.getSerializedSize();
        headerLen += CodedOutputStream.computeRawVarint32Size(headerLen);

        int callId = header.getCallId();
        if (LOG.isDebugEnabled())
          LOG.debug(getName() + " got value #" + callId);

        Call call = calls.get(callId);
        RpcStatusProto status = header.getStatus();
        if (status == RpcStatusProto.SUCCESS) {
          Writable value = ReflectionUtils.newInstance(valueClass, conf);
          value.readFields(in);                 // read value
          calls.remove(callId);
          call.setRpcResponse(value);

          // verify that length was correct
          // only for ProtobufEngine where len can be verified easily
          if (call.getRpcResponse() instanceof ProtobufRpcEngine.RpcWrapper) {
            ProtobufRpcEngine.RpcWrapper resWrapper = 
                (ProtobufRpcEngine.RpcWrapper) call.getRpcResponse();
            if (totalLen != headerLen + resWrapper.getLength()) { 
              throw new RpcClientException(
                  "RPC response length mismatch on rpc success");
            }
          }
        } else { // Rpc Request failed
          // Verify that length was correct
          if (totalLen != headerLen) {
            throw new RpcClientException(
                "RPC response length mismatch on rpc error");
          }

          final String exceptionClassName = header.hasExceptionClassName() ?
                header.getExceptionClassName() : 
                  "ServerDidNotSetExceptionClassName";
          final String errorMsg = header.hasErrorMsg() ? 
                header.getErrorMsg() : "ServerDidNotSetErrorMsg" ;
          final RpcErrorCodeProto erCode = 
                    (header.hasErrorDetail() ? header.getErrorDetail() : null);
          if (erCode == null) {
             LOG.warn("Detailed error code not set by server on rpc error");
          }
          RemoteException re = 
              ( (erCode == null) ? 
                  new RemoteException(exceptionClassName, errorMsg) :
              new RemoteException(exceptionClassName, errorMsg, erCode));
          if (status == RpcStatusProto.ERROR) {
            calls.remove(callId);
            call.setException(re);
          } else if (status == RpcStatusProto.FATAL) {
            // Close the connection
            markClosed(re);
          }
        }
      } catch (IOException e) {
        markClosed(e);
      }
    }
    ----------------------------------------------------------------------------------------------------------
    private synchronized void markClosed(IOException e) {
      if (shouldCloseConnection.compareAndSet(false, true)) {
        closeException = e;
        notifyAll();
      }
    }
    ----------------------------------------------------------------------------------------------------------
     /* wait till someone signals us to start reading RPC response or
     * it is idle too long, it is marked as to be closed, 
     * or the client is marked as not running.
     * 
     * Return true if it is time to read a response; false otherwise.
     */
    private synchronized boolean waitForWork() {
      if (calls.isEmpty() && !shouldCloseConnection.get()  && running.get())  {
        long timeout = maxIdleTime-
              (Time.now()-lastActivity.get());
        if (timeout>0) {
          try {
            wait(timeout);
          } catch (InterruptedException e) {}
        }
      }

      if (!calls.isEmpty() && !shouldCloseConnection.get() && running.get()) {
        return true;
      } else if (shouldCloseConnection.get()) {
        return false;
      } else if (calls.isEmpty()) { // idle connection closed or stopped
        markClosed(null);
        return false;
      } else { // get stopped but there are still pending requests 
        markClosed((IOException)new IOException().initCause(
            new InterruptedException()));
        return false;
      }
    }
```

上面四个方法就是这个问题的问答，当请求失败或者请求完毕连接被挂了，就调用notifyAll方法来唤醒当前线程。

另外，当获取响应结果后，从响应头中获取callId来与calls中的callId来对应得到相应call的响应结果。

