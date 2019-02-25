在前面我们查看 egg-scripts 时就发现，启动的 node 命令 `serverBin` 就是 startCluster 这个 api 时，我们说过它就是从默认选项 `this.options.framework` 中取的，而这个默认的框架就是 egg，而 egg 模块的启动入口 index.js 中我们看到了由许多 CommonJs 释出的方法，且注释齐全

在回到之前 egg-cluster 的 master.js 中，我们前面分析过启动工作进程是由 forkAppWorkers 这个方法启动的，在我的 egg-cluster 分析中已经写的很清楚干了什么，而在其中的 app_worker.js 中就写了每个工作进程是如何启动应用的，其中用到的就是这里 egg 释出的 Application api 去创建应用并最后监听指定端口，其中的 cfork 包是使用 cluster 模块启动的，所以我们没必要担心端口冲突的问题

而上面的简短说明就是告诉大家，egg 模块中释出的 api 都在前面启动进程时用到过，如果你看完了 egg-cluster 整个启动过程就知道
1. startCluster：egg-scripts 这个命令行工具的最终命令都是跑的 startCluster；
2. Application：在 egg-cluster 中启动从进程 worker 时用到，即触发 agent-start 事件的 forkAppWorkers 方法中；
3. Agent：在 egg-cluster 的 master.js 中使用 detectPort 检测完端口后调用 forkAgentWorker 时运行了 agent_worker.js 时调用；（其中更涉及到 egg-core 的 lifecycle.js，这里我不得不再强调一次这个 agent-start 事件是如何触发后，带动整个应用层的 worker 进程启动并触发 egg-ready 周期的）；
    * agent_worker.js 中 `new Agent` 后使用了 `agent.ready` 加入调用栈，而我们看到 egg/lib/agent.js 并没有开放 ready 接口，所以我们向上溯源找到父类 ./lib/egg.js 在这里发现它也在直接调用 `this.ready` 所以我们又溯源找到 egg-core/index.js => egg-core/lib/egg.js => ready => `return this.lifecycle.ready(...)` => egg-core/lib/lifecycle.js 在其构造函数中我们终于发现了创建的调用栈所在 `getReady.mixin(this)` 并在最后执行了这个函数 `this[INIT_READY]()`
    * 在其中我们发现了，而这里的 bootReady 就是使用了 ready-callback 包，它的功能就是启动服务后执行所有异步队列任务，而这里执行的就是 `this.ready(err || true)` 即执行 get-ready 的所有调用栈，而我们需要执行的 `process.send({ action: 'agent-start', to: 'master' })` 就在它的栈中，over
    ```js
    this.bootReady.ready(err => {
      this.ready(err || true);
    });
    ```
    * 触发过程流程：startCluster => egg-cluster/master.js => detectPort => forkAgentWorker => agent_worker.js => egg/lib/agent.js => egg/lib/egg.js => egg-core/index.js => egg-core/lib/egg.js => ready => `return this.lifecycle.ready(...)` => egg-core/lib/lifecycle.js => `this[INIT_READY]()`
4. AppWorkerLoader：egg/lib/application.js 中用到的加载器，在从进程 worker 中启动应用都会用到；
5. AgentWorkerLoader：agent 层启动类中用到 => egg/lib/agent.js 代理层配置加载器；
6. Controller、Service、BaseContextClass(ctx)、Subscription => egg-core/lib/utils/base_context_class.js 这里已经属于应用的挂载对象了，使用过 egg 的同学应该都知道；

```js
'use strict';

/**
* @namespace Egg
*/

/**
* Start egg application with cluster mode
* @since 1.0.0
*/
exports.startCluster = require('egg-cluster').startCluster;

/**
* @member {Application} Egg#Application
* @since 1.0.0
*/
exports.Application = require('./lib/application');

/**
* @member {Agent} Egg#Agent
* @since 1.0.0
*/
exports.Agent = require('./lib/agent');

/**
* @member {AppWorkerLoader} Egg#AppWorkerLoader
* @since 1.0.0
*/
exports.AppWorkerLoader = require('./lib/loader').AppWorkerLoader;

/**
* @member {AgentWorkerLoader} Egg#AgentWorkerLoader
* @since 1.0.0
*/
exports.AgentWorkerLoader = require('./lib/loader').AgentWorkerLoader;

/**
* @member {Controller} Egg#Controller
* @since 1.1.0
*/
exports.Controller = require('./lib/core/base_context_class');

/**
* @member {Service} Egg#Service
* @since 1.1.0
*/
exports.Service = require('./lib/core/base_context_class');

/**
* @member {Subscription} Egg#Subscription
* @since 1.10.0
*/
exports.Subscription = require('./lib/core/base_context_class');

/**
* @member {BaseContextClass} Egg#BaseContextClass
* @since 1.2.0
*/
exports.BaseContextClass = require('./lib/core/base_context_class');

```
