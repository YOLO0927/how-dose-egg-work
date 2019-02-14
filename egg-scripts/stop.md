该说的前置知识，我们前面都说过了，这里直接上代码开干！

stop 非常简单，无非就是过滤出 node 进程，再全部杀掉！

```js
class StopCommand extends Command {

  constructor(rawArgv) {
    super(rawArgv);
    // start.js 中说过了，声明命令行使用方法
    this.usage = 'Usage: egg-scripts stop [--title=example]';
    this.serverBin = path.join(__dirname, '../start-cluster');
    // stop 指定 title 的应用
    this.options = {
      title: {
        description: 'process title description, use for kill grep',
        type: 'string',
      },
    };
  }

  // 运行时 ci 界面打印的日志内容
  get description() {
    return 'Stop server';
  }

  * run(context) {
    /* istanbul ignore next */
    // windows 32 位凉凉，自己去 kill 掉 master 解决吧
    if (process.platform === 'win32') {
      this.logger.warn('Windows is not supported, try to kill master process which command contains `start-cluster` or `--type=egg-server` yourself, good luck.');
      process.exit(0);
    }

    const { argv } = context;

    this.logger.info(`stopping egg application ${argv.title ? `with --title=${argv.title}` : ''}`);

    // node /Users/tz/Workspaces/eggjs/egg-scripts/lib/start-cluster {"title":"egg-server","workers":4,"port":7001,"baseDir":"/Users/tz/Workspaces/eggjs/test/showcase","framework":"/Users/tz/Workspaces/eggjs/test/showcase/node_modules/egg"}
    // 过滤出所有 node 进程，当存在指定 title 时过滤出指定 title 的进程，若不存在则过滤出是 start-cluster 运行的进程
    // 需注意 egg-bin 也是 start-cluster 启动的，只是子线程只有 1 个
    let processList = yield this.helper.findNodeProcess(item => {
      const cmd = item.cmd;
      return argv.title ?
        cmd.includes('start-cluster') && cmd.includes(`"title":"${argv.title}"`) :
        cmd.includes('start-cluster');
    });
    // 过滤出对应进程列表的 pid
    let pids = processList.map(x => x.pid);

    // 若找到符合条件的对应进程，则打印出日志，并全部 kill 掉，这里的 kill 方法是使用 forEach 循环 kill 的，大家自己去看 /lib/helper.js 就好
    // 若没有符合条件的，则打印 "未侦查到正在运行的 egg 进程"
    if (pids.length) {
      this.logger.info('got master pid %j', pids);
      this.helper.kill(pids);
    } else {
      this.logger.warn('can\'t detect any running egg process');
    }

    // wait for 5s to confirm whether any worker process did not kill by master
    // 等待 5s 后再次进行过滤进程判断查看是否进程还存在，若存在则再次 kill 掉（理论上等待 5s 后不用重复这一步，除非你电脑卡死了。。。）
    yield sleep('5s');

    // node --debug-port=5856 /Users/tz/Workspaces/eggjs/test/showcase/node_modules/_egg-cluster@1.8.0@egg-cluster/lib/agent_worker.js {"framework":"/Users/tz/Workspaces/eggjs/test/showcase/node_modules/egg","baseDir":"/Users/tz/Workspaces/eggjs/test/showcase","port":7001,"workers":2,"plugins":null,"https":false,"key":"","cert":"","title":"egg-server","clusterPort":52406}
    // node /Users/tz/Workspaces/eggjs/test/showcase/node_modules/_egg-cluster@1.8.0@egg-cluster/lib/app_worker.js {"framework":"/Users/tz/Workspaces/eggjs/test/showcase/node_modules/egg","baseDir":"/Users/tz/Workspaces/eggjs/test/showcase","port":7001,"workers":2,"plugins":null,"https":false,"key":"","cert":"","title":"egg-server","clusterPort":52406}
    processList = yield this.helper.findNodeProcess(item => {
      const cmd = item.cmd;
      return argv.title ?
        (cmd.includes('egg-cluster/lib/app_worker.js') || cmd.includes('egg-cluster/lib/agent_worker.js')) && cmd.includes(`"title":"${argv.title}"`) :
        (cmd.includes('egg-cluster/lib/app_worker.js') || cmd.includes('egg-cluster/lib/agent_worker.js'));
    });
    pids = processList.map(x => x.pid);

    if (pids.length) {
      this.logger.info('got worker/agent pids %j that is not killed by master', pids);
      this.helper.kill(pids, 'SIGKILL');
    }

    this.logger.info('stopped');
  }
}
```
