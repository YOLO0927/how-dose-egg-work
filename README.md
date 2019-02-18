#### 本文诞生的起因
1. 在使用 egg 之后，首先大家都知道它是基于 koa 封装的一个企业级应用 web 框架，那么到底它是如何封装、如何启动、如何集群部署、如何停止多线程服务的呢
2. 在蚂蚁金服的面试中也多次问到我是否看过 egg-core 的源码（我当时没看），这也侧面让我明白到其在内部的地位，因为我们都知道 egg 是在阿里内部经过大量实践后才经过处理后开源出来的，所以在这之后我花了一些时间去消化 egg-core 的源码，然后知道了 egg 跑起来是如何做环境区分的、如何根据环境变量去做配置、插件、全局中间件（在插件中判断）注入应用的，插件是如何作为一个单独应用被注入等
3. 随后我发现在自己粗略分析的过程中其实也是通过多个入口去看的（因为我的学习习惯就是追本溯源），通过不同的入口区分不同的情况，比如本地开发跑的是 egg-bin 脚本，线上测试与生产跑的是 egg-scripts 脚本，前置步骤其实是较为繁琐的，看完后过2天我已经将核心忘的差不多了

#### 针对上述问题的解决方案
遂有了这篇文章的诞生，旨在记录这个运行的大概过程，当然我也不保证自己写的会全是正确的，所以如果有错误希望大家指出，我会查看并仔细修改，共勉～

#### so，本文将从几个角度分析 egg 是如何运行起来的
1. [x] 开发环境入口 egg-bin
2. [x] 线上测试以及生产环境入口 egg-scripts
   * egg-scripts start
   * egg-scripts stop
   * 源码分析入口为 /egg-scirpts/summary.md（egg-scripts 解析大纲）
3. [x] egg-scripts 对于集群部署 cluster 模块的运用
   * egg-cluster（太特么多了）
4. [ ] egg-core
   * 如何进行环境区分
   * 配置注入（插件与 config 配置的注入）
   * 继承 KoaApplication 启动应用
