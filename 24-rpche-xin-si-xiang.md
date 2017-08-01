# 2.2 RPC核心思想

---

> ##### 什么是RPC？

WIKI的解析：远程过程调用（英语：Remote Procedure Call，缩写为 RPC）是一个计算机通信协议。该协议允许运行于一台计算机的程序调用另一台计算机的子程序，而程序员无需额外地为这个交互作用编程。如果涉及的软件采用面向对象编程，那么远程过程调用亦可称作远程调用或远程方法调用，例：Java RMI。

简单地说RPC就是调用远程的方法，但是其中用到了较高级的处理方式，例如动态代理、反射、java nio等等，这些都是实实在在的可学之处。

> ##### 图解Hadoop RPC核心流程

![](/assets/RPC核心思想.jpg)主要流程：

1. 服务器端建立socket通信，等待客户端请求。
2. 客户端生成socket代理对象。
3. 客户端发送数据，请求服务器。


