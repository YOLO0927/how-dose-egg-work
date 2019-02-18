egg-cluster 模块将会揭晓 egg 是如何将 egg-scripts 分离出的自定义命令通过 cluster 模块集群部署以及 Master-agent-work 机制是如何实现的

预备知识:
  1. Node 核心模块 [cluster](http://nodejs.cn/api/cluster.html) 模块；
  2. 如果使用 IPC 通道发送句柄(Handle)或消息，最好了解 进程间通信（IPC）除了 socket 还有什么 Handle 可以通过 IPC 传递；

不了解的同学可以简单看下我这篇文章对 [《Node 深入浅出》第九章的总结](https://blog.csdn.net/yolo0927/article/details/81224942) ，内部即为 Master-Work（主从模式）如何通过多个 http 服务监听同一端口简单的原理实现，及如何使用 cluster 模块快速实现；

至于 cluster 模块是如何实现的，官方已经推荐了文章 [Cluster 实现原理](https://cnodejs.org/topic/56e84480833b7c8a0492e20c)

阅读 egg-cluster 时大家应该结合文档对于 [多进程模型和进程](https://eggjs.org/zh-cn/core/cluster-and-ipc.html#agent-%E6%9C%BA%E5%88%B6) 来看，更容易理解，首先我肯定是对着看了然后记录的；

```js
exports.startCluster = function(options, callback) {
  new Master(options).ready(callback);
};
```

这就是 egg-cluster 的入口了，我们通过线索直接去解析 /lib/master 的 ready 是如何实现的即可 [master.md](./master.md)
