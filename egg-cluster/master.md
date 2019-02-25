先声明 egg-cluster 是个大坑，由于涉及到 IPC，master-agent-worker 进程间的协作与进程守护等问题，所以其实现较为复杂且代码量较大（其中涉及到 options.framework => egg => egg-core 中大量的调用，其中启动 agent 与 worker 服务时多会注入 egg-core 配置），看不下去的话看看文字说明的粗略分析都可以，看的下去的话就跟着我看源码吧，如果有错的地方就帮我指出来一下吧，我会仔细看看改的～～～

由于 egg-cluster 中是调用的 Master 类的 ready 方法，所以我们在其内部先大概预览一下结构，搜索 ready 发现其为 [get-ready](https://www.npmjs.com/package/get-ready) 包，用处为帮助某个对象注册一个 ready 事件，ok 两个 collaborators 都是蚂蚁金服的，也就是说都是自己人写的，花 2 分钟看了下源码。。。其实就是一个简单监听发布模式：
1. 当你 ready 的参数为 undefined 时返回一个 promise.resolve 并且 push 入调用栈；
2. 当你 ready 的参数为 Function 时直接 push 入调用栈；
3. 当不为以上两种类型时触发内部 emit 方法(比如 master.js 中使用 `this.ready(true)`)，为 true 时会改变内部常量 IS_READY 为 true，其实随便传个啥，反正能通过三目判断为 true 即可；
4. 触发的 emit 为使用 Array.prototype.splice 截取整个调用栈（等同于清理调用栈）通过 forEach 遍历调用栈，并且通过 process.nextTick 顺序执行栈内的函数队列；

ok，我们开始分析 master.js 的构造函数，其实源码已经有很多备注了，我也只是为大家粗略分析

1. 一旦开启 master 进程后 agent 代理层的启动退出与重启监听，监听 agent-start 改变 agent 各个状态以及 once 一次监听 agent-start 用于利用 cluster 启动多个 worker 从进程；
2. 通过继承 egg-core 中的 lifecycle agent-start 事件的函数置入其 get-ready 中，在 agent 服务启动后，实例化 agent 服务时将触发 `[INIT_READY]()` 使 licecycle get-ready 存储的调用栈全部调用，向 master 进程发送消息触发 agent-start 事件，开始进行 forkAppWorkers（fork 多个应用从进程）；
3. cluster 启动多个 worker 从进程并监听 端口成功启动 后发送子进程 pid 给 master 触发 app-start 事件进行 app 层进程间通信处理，并在最后触发 ready 调用栈中所有函数；
4. 在这个过程中涉及到进程守护的处理，如监听各进程的 退出、断开连接、监听成功 等...

整个周期正如文档中的此图
```
(new Master)
+---------+           +---------+          +---------+
|  Master |           |  Agent  |          |  Worker |
+---------+           +----+----+          +----+----+
     |      fork agent     |                    |
     |(使用 detect-port 触发)|
     +-------------------->|                    |
     |      agent ready    |                    |
     |(继承于 egg-core 的 lifecycle get-ready 触发 agent-start 事件)
     |<--------------------+                    |
     |                     |    fork worker     |
     | (触发 agent-start 事件执行 forkAppWorkers) |
     +----------------------------------------->|
     |     worker ready    |                    |
     |(由 forkAppWorkers 中 cluster 监听 listen 事件，当每个工作进程启动服务并 listen 成功时触发 app-start)
     |<-----------------------------------------+
     |      Egg ready      |                    |
     |(每个工作进程触发 app-start 的 onAppStart 函数，判断如果所有工作进程全部启动，则触发 egg-ready 周期发送给应用层)
     +-------------------->|                    |
     |      Egg ready      |                    |
     |(最后 onAppStart 中触发 master 进程中的 get-ready 调用栈，输出 master 启动完毕的日志及时间，并将 egg-ready 动作通知所有层: parent app agent)
     +----------------------------------------->|
```

最后分析完毕后，我们可以直接跳到去看 `this.options.framework` => `require('egg')`，去看看 egg 模块在干什么

```js
class Master extends EventEmitter {
//  先明确传入的参数对象为 process.argv[2]，即为前面提到过的 eggArgs，看源码的注释也清楚写明了各个参数的备注
/**
 * @constructor
 * @param {Object} options
 *  - {String} [framework] - specify framework that can be absolute path or npm package
 *  - {String} [baseDir] directory of application, default to `process.cwd()`
 *  - {Object} [plugins] - customized plugins, for unittest
 *  - {Number} [workers] numbers of app workers, default to `os.cpus().length`
 *  - {Number} [port] listening port, default to 7001(http) or 8443(https)
 *  - {Object} [https] https options, { key, cert }, full path
 *  - {Array|String} [require] will inject into worker/agent process
 */
  constructor(options) {
    // 使用 super 关键字 call this（EventEmitter.prototype.constructor.call(this)）
    super();
    // 默认配置合并与断言处理
    this.options = parseOptions(options);
    // 抽象了一些用于操纵 agent 与 worker 的模块
    this.workerManager = new Manager();
    // 实例化 Messenger 类，使用 IPC 通道向各进程发送消息
    this.messenger = new Messenger(this);

    // 实例化 get-ready
    ready.mixin(this);

    // 判断环境常量是否为生产环境或不为开发环境与单元测试环境
    this.isProduction = isProduction();
    // 初始化各个状态以及默认端口
    this.agentWorkerIndex = 0;
    this.closed = false;
    this[REALPORT] = this.options.port;

    // app started or not
    this.isStarted = false;
    this.logger = new ConsoleLogger({ level: process.env.EGG_MASTER_LOGGER_LEVEL || 'INFO' });
    this.logMethod = 'info';
    if (process.env.EGG_SERVER_ENV === 'local' || process.env.NODE_ENV === 'development') {
      this.logMethod = 'debug';
    }

    // get the real framework info
    const frameworkPath = this.options.framework;
    const frameworkPkg = utility.readJSONSync(path.join(frameworkPath, 'package.json'));

    // 一下都为日志判断打印
    this.log(`[master] =================== ${frameworkPkg.name} start =====================`);
    this.logger.info(`[master] node version ${process.version}`);
    if (process.alinode) this.logger.info(`[master] alinode version ${process.alinode}`);
    this.logger.info(`[master] ${frameworkPkg.name} version ${frameworkPkg.version}`);

    if (this.isProduction) {
      this.logger.info('[master] start with options:%s%s',
        os.EOL, JSON.stringify(this.options, null, 2));
    } else {
      this.log('[master] start with options: %j', this.options);
    }
    this.log('[master] start with env: isProduction: %s, EGG_SERVER_ENV: %s, NODE_ENV: %s',
      this.isProduction, process.env.EGG_SERVER_ENV, process.env.NODE_ENV);

    const startTime = Date.now();

    this.ready(() => {
      // 当触发 ready 调用栈时执行以下
      this.isStarted = true;
      const stickyMsg = this.options.sticky ? ' with STICKY MODE!' : '';
      this.logger.info('[master] %s started on %s (%sms)%s',
        frameworkPkg.name, this[APP_ADDRESS], Date.now() - startTime, stickyMsg);

      // 与各进程通信，触发 egg-ready 周期
      const action = 'egg-ready';
      this.messenger.send({ action, to: 'parent', data: { port: this[REALPORT], address: this[APP_ADDRESS] } });
      this.messenger.send({ action, to: 'app', data: this.options });
      this.messenger.send({ action, to: 'agent', data: this.options });

      // start check agent and worker status
      // 每 10s 检测一次 agent 与 worker，如果 代理层 与 worker 都存在，则异常数归 0 ，否则如果异常数量大于等于 3，输出异常
      if (this.isProduction) {
        this.workerManager.startCheck();
      }
    });

    // 监听 agent-exit 事件，退出 agent 进程时会触发，阅读源码调试时可通过 ctrl + c 退出 egg-bin dev 即可触发
    this.on('agent-exit', this.onAgentExit.bind(this));
    // 监听 agent-start 事件，触发时会判断从进程条件分发消息给指定层通知 agent 层服务启动成功
    this.on('agent-start', this.onAgentStart.bind(this));
    this.on('app-exit', this.onAppExit.bind(this));
    // 利用 cluster 启动应用服务，监听 listening 事件，在调用 listen() 成功后，会触发 app-start 事件
    this.on('app-start', this.onAppStart.bind(this));
    // 重启工作进程
    this.on('reload-worker', this.onReload.bind(this));

    // fork app workers after agent started
    this.once('agent-start', this.forkAppWorkers.bind(this));
    // get the real port from options and app.config
    // app worker will send after loading
    this.on('realport', port => {
      if (port) this[REALPORT] = port;
    });

    // 监听退出进程的不同情况，这里都是针对当前 master 进程的
    // https://nodejs.org/api/process.html#process_signal_events
    // https://en.wikipedia.org/wiki/Unix_signal
    // kill(2) Ctrl-C
    process.once('SIGINT', this.onSignal.bind(this, 'SIGINT'));
    // kill(3) Ctrl-\
    process.once('SIGQUIT', this.onSignal.bind(this, 'SIGQUIT'));
    // kill(15) default
    process.once('SIGTERM', this.onSignal.bind(this, 'SIGTERM'));

    process.once('exit', this.onExit.bind(this));

    // 监听指定端口的启动，看了下 detect-port 的源码，当直接没有 port 参数时，端口默认为 0，即会随机分配一个可用端口，即会使用 net 模块启动 TCP 服务，默认监听 0.0.0.0:[port]
    // 这里我们可以看出 master 是不左右任何业务的，它可以说只做进程分发，因为启动时 port 并不是我们应用监听的 port 而是随机一个，官方文档也是这么描述 master 进程的，ok，没毛病
    detectPort((err, port) => {

      /* istanbul ignore if */
      if (err) {
        err.name = 'ClusterPortConflictError';
        err.message = '[master] try get free port error, ' + err.message;
        this.logger.error(err);
        process.exit(1);
      }
      this.options.clusterPort = port;
      // 当前进程即为 master 进程（即使用 startCluster api 执行 master.js 的进程，由 detectPort 启动的 TCP 服务），此时当前的 master 进程触发启动 agent 进程，会去 fork 一个子进程 agentWorker
      // 在启动完成后触发 agent-start 事件，所以我们再次回到上面，如何触发的解析在下面具体函数内
      this.forkAgentWorker();
    });

    // exit when agent or worker exception
    this.workerManager.on('exception', ({ agent, worker }) => {
      const err = new Error(`[master] ${agent} agent and ${worker} worker(s) alive, exit to avoid unknown state`);
      err.name = 'ClusterWorkerExceptionError';
      err.count = { agent, worker };
      this.logger.error(err);
      process.exit(1);
    });
  }
  /**
   * Agent Worker exit handler
   * Will exit during startup, and refork during running.
   * @param {Object} data
   *  - {Number} code - exit code
   *  - {String} signal - received signal
   */
  onAgentExit(data) {
    if (this.closed) return;

    this.messenger.send({ action: 'egg-pids', to: 'app', data: [] });
    const agentWorker = this.agentWorker;
    this.workerManager.deleteAgent(this.agentWorker);

    const err = new Error(util.format('[master] agent_worker#%s:%s died (code: %s, signal: %s)',
      agentWorker.id, agentWorker.pid, data.code, data.signal));
    err.name = 'AgentWorkerDiedError';
    this.logger.error(err);

    // remove all listeners to avoid memory leak
    // 退出 agent 进程，移除所有 agent 上的监听事件，如若 this.isStarted 为 true，则重启 agent 进程
    agentWorker.removeAllListeners();

    if (this.isStarted) {
      this.log('[master] try to start a new agent_worker after 1s ...');
      setTimeout(() => {
        this.logger.info('[master] new agent_worker starting...');
        // 这个太重点了，涉及到 agent 的重启与退出监听，我必须附上
        this.forkAgentWorker();
      }, 1000);
      this.messenger.send({
        action: 'agent-worker-died',
        to: 'parent',
      });
    } else {
      this.logger.error('[master] agent_worker#%s:%s start fail, exiting with code:1',
        agentWorker.id, agentWorker.pid);
      process.exit(1);
    }
  }

  onAgentStart() {
    this.agentWorker.status = 'started';

    // Send egg-ready when agent is started after launched
    if (this.isAllAppWorkerStarted) {
      this.messenger.send({ action: 'egg-ready', to: 'agent', data: this.options });
    }

    this.messenger.send({ action: 'egg-pids', to: 'app', data: [ this.agentWorker.pid ] });
    // should send current worker pids when agent restart
    if (this.isStarted) {
      this.messenger.send({ action: 'egg-pids', to: 'agent', data: this.workerManager.getListeningWorkerIds() });
    }

    this.messenger.send({ action: 'agent-start', to: 'app' });
    this.logger.info('[master] agent_worker#%s:%s started (%sms)',
      this.agentWorker.id, this.agentWorker.pid, Date.now() - this.agentStartTime);
  }

  /**
   * App Worker exit handler
   * @param {Object} data
   *  - {String} workerPid - worker id
   *  - {Number} code - exit code
   *  - {String} signal - received signal
   */
  onAppExit(data) {
    if (this.closed) return;

    const worker = this.workerManager.getWorker(data.workerPid);

    if (!worker.isDevReload) {
      const signal = data.signal;
      const message = util.format(
        '[master] app_worker#%s:%s died (code: %s, signal: %s, suicide: %s, state: %s), current workers: %j',
        worker.id, worker.process.pid, worker.process.exitCode, signal,
        worker.exitedAfterDisconnect, worker.state,
        Object.keys(cluster.workers)
      );
      if (this.options.isDebug && signal === 'SIGKILL') {
        // exit if died during debug
        this.logger.error(message);
        this.logger.error('[master] worker kill by debugger, exiting...');
        setTimeout(() => this.close(), 10);
      } else {
        const err = new Error(message);
        err.name = 'AppWorkerDiedError';
        this.logger.error(err);
      }
    }

    // remove all listeners to avoid memory leak
    worker.removeAllListeners();
    this.workerManager.deleteWorker(data.workerPid);
    // send message to agent with alive workers
    this.messenger.send({ action: 'egg-pids', to: 'agent', data: this.workerManager.getListeningWorkerIds() });

    if (this.isAllAppWorkerStarted) {
      // cfork will only refork at production mode
      this.messenger.send({
        action: 'app-worker-died',
        to: 'parent',
      });

    } else {
      // exit if died during startup
      this.logger.error('[master] app_worker#%s:%s start fail, exiting with code:1',
        worker.id, worker.process.pid);
      process.exit(1);
    }
  }

  /**
   * after app worker
   * @param {Object} data
   *  - {String} workerPid - worker id
   *  - {Object} address - server address
   */
  onAppStart(data) {
    const worker = this.workerManager.getWorker(data.workerPid);
    const address = data.address;

    // ignore unspecified port
    // and it is ramdom port when use sticky
    if (!this.options.sticky
      && !isUnixSock(address)
      && (String(address.port) !== String(this[REALPORT]))) {
      return;
    }

    // send message to agent with alive workers
    this.messenger.send({
      action: 'egg-pids',
      to: 'agent',
      data: this.workerManager.getListeningWorkerIds(),
    });

    this.startSuccessCount++;

    const remain = this.isAllAppWorkerStarted ? 0 : this.options.workers - this.startSuccessCount;
    this.log('[master] app_worker#%s:%s started at %s, remain %s (%sms)',
      worker.id, data.workerPid, address.port, remain, Date.now() - this.appStartTime);

    // Send egg-ready when app is started after launched
    if (this.isAllAppWorkerStarted) {
      this.messenger.send({ action: 'egg-ready', to: 'app', data: this.options });
    }

    // if app is started, it should enable this worker
    if (this.isAllAppWorkerStarted) {
      worker.disableRefork = false;
    }

    if (this.isAllAppWorkerStarted || this.startSuccessCount < this.options.workers) {
      return;
    }

    this.isAllAppWorkerStarted = true;

    // enable all workers when app started
    for (const id in cluster.workers) {
      const worker = cluster.workers[id];
      worker.disableRefork = false;
    }

    address.protocal = this.options.https ? 'https' : 'http';
    address.port = this.options.sticky ? this[REALPORT] : address.port;
    this[APP_ADDRESS] = getAddress(address);

    if (this.options.sticky) {
      this.startMasterSocketServer(err => {
        if (err) return this.ready(err);
        this.ready(true);
      });
    } else {
      this.ready(true);
    }
  }

  forkAgentWorker() {
    // 记录 agent 进程开始的时间戳，方便日志的记录
    this.agentStartTime = Date.now();

    const args = [ JSON.stringify(this.options) ];
    const opt = {};

    // add debug execArgv
    const debugPort = process.env.EGG_AGENT_DEBUG_PORT || 5800;
    if (this.options.isDebug) opt.execArgv = process.execArgv.concat([ `--${semver.gte(process.version, '8.0.0') ? 'inspect' : 'debug'}-port=${debugPort}` ]);

    // 使用 child_process 模块 fork 一个子进程 agent，内部已经 debug 了 options 并将
    // 在开发模式下改变 option 会进行 debug，其中的 graceful-process 用于监听该进程的退出，及优雅退出进程，其中做了各种情况退出、断线监听，并且由判断是否为 cluster 启动的 worker 进程
    // 其中的 new Agent 来自于 egg/lib/agent.js 继承于 egg-core => this.close => egg-core/lib/lifecycle.js close => 删除所有监听事件，所以此时 agent 层已经进行了配置注入，如果配置连接出错了，那 agent 层都启动不起来的哦～
    // 还需注意其中的一个 agent.ready 的回调，这个 ready 是来自 egg-core/lib/lifecycle.js 的，其中的注册事件向 master 发送了消息触发 agent-start 事件，从而触发 master 中的 onAgentStart 与 forkAppWorkers
    // 我们之前提到过，这里的 get-ready 传入函数只是传入一个调用栈中存储，那么存入的触发 agent-start 到底在哪调用了，这就要看到上段提到的 egg-core/lib/lifecycle.js 了
    // 1. 其构造函数中调用了 this[INIT_READY]，它是使用的 ready-callback 包，服务启动成功后将会触发其中的 ready 栈
    // 2. 也就是说会触发最后的一个 this.ready(err || true)，即服务不出错则会触发 get-ready 的调用栈从而触发之前 agent_work.js 中 agent.ready 内的进程通信，触发 agent-start 事件
    const agentWorker = childprocess.fork(this.getAgentWorkerFile(), args, opt);
    agentWorker.status = 'starting';
    agentWorker.id = ++this.agentWorkerIndex;
    this.workerManager.setAgent(agentWorker);
    this.log('[master] agent_worker#%s:%s start with clusterPort:%s',
      agentWorker.id, agentWorker.pid, this.options.clusterPort);

    // send debug message（如果是 egg-bin dev 的 debug 模式时）
    if (this.options.isDebug) {
      this.messenger.send({ to: 'parent', from: 'agent', action: 'debug', data: { debugPort, pid: agentWorker.pid } });
    }
    // 传递 agent 进程中的消息
    // forwarding agent' message to messenger
    agentWorker.on('message', msg => {
      if (typeof msg === 'string') msg = { action: msg, data: msg };
      msg.from = 'agent';
      this.messenger.send(msg);
    });
    // 监听 agent 层的 error 事件
    agentWorker.on('error', err => {
      err.name = 'AgentWorkerError';
      err.id = agentWorker.id;
      err.pid = agentWorker.pid;
      this.logger.error(err);
    });
    // agent exit message
    // 监听 agent 层的退出事件，当退出时利用 IPC 发送消息给 master 层，并触发 agent-exit 事件
    agentWorker.once('exit', (code, signal) => {
      this.messenger.send({
        action: 'agent-exit',
        data: { code, signal },
        to: 'master',
        from: 'agent',
      });
    });
  }

  forkAppWorkers() {
    // 记录启动的时间戳，方便后续日志输出
    this.appStartTime = Date.now();
    this.isAllAppWorkerStarted = false;
    this.startSuccessCount = 0;

    const args = [ JSON.stringify(this.options) ];
    this.log('[master] start appWorker with args %j', args);

    // 使用 cfork 包启动 egg-cluster/lib/app_worker.js
    cfork({
      // 这个脚本就简单很多了，根据 options 是 http 还是 https 来启动对应模块服务，最后 listen 参数指定的 port、hostname 即可，默认为 127.0.0.1:7001
      exec: this.getAppWorkerFile(),
      args,
      silent: false,
      // 来自 egg-scripts 的 worker 进程数，默认是 os.cpus().length
      count: this.options.workers,
      // don't refork in local env
      refork: this.isProduction,
    });

    let debugPort = process.debugPort;
    // 监听 worker 进程的 fork
    cluster.on('fork', worker => {
      worker.disableRefork = true;
      this.workerManager.setWorker(worker);
      worker.on('message', msg => {
        if (typeof msg === 'string') msg = { action: msg, data: msg };
        msg.from = 'app';
        this.messenger.send(msg);
      });
      this.log('[master] app_worker#%s:%s start, state: %s, current workers: %j',
        worker.id, worker.process.pid, worker.state, Object.keys(cluster.workers));

      // send debug message, due to `brk` scence, send here instead of app_worker.js
      if (this.options.isDebug) {
        debugPort++;
        this.messenger.send({ to: 'parent', from: 'app', action: 'debug', data: { debugPort, pid: worker.process.pid } });
      }
    });
    // 监听断线
    cluster.on('disconnect', worker => {
      this.logger.info('[master] app_worker#%s:%s disconnect, suicide: %s, state: %s, current workers: %j',
        worker.id, worker.process.pid, worker.exitedAfterDisconnect, worker.state, Object.keys(cluster.workers));
    });
    // 监听工作进程的退出，退出时触发 app-exit
    cluster.on('exit', (worker, code, signal) => {
      this.messenger.send({
        action: 'app-exit',
        data: { workerPid: worker.process.pid, code, signal },
        to: 'master',
        from: 'app',
      });
    });
    // 当工作进程调用 listen 的时候被触发，然后触发 app-start 事件，而 listen 就在 egg-cluster/lib/app_worker.js 中 server.listen 调用，这个触发实在比 agent 进程容易多了（在打字的我表示舒了一口气）
    cluster.on('listening', (worker, address) => {
      this.messenger.send({
        action: 'app-start',
        data: { workerPid: worker.process.pid, address },
        to: 'master',
        from: 'app',
      });
    });
  }
}
```
