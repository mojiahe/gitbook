# 2.3 RPC源码解析（一）

---

在上一节的记录中，明确了RPC的主要流程分为三步。这一节来探究一下客户端是如何生成proxy动态对象。

注意：hadoop的RPC封装全部位于org.apache.hadoop.ipc这个package下。

> 客户端如何生成proxy动态对象？

现在模拟一下RPC的过程，简单的代码跟踪如下：

