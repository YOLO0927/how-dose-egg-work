开发环境下的本地调试，在 egg 的一套架构中，我们通常是直接运行 `egg-bin dev`

粗览整个 egg-bin 模块的目录分布，与 egg-scripts 是一样的，都是基于使用 common-bin 而设置的，这里我们就不重复解释了，需要了解的同学大概看一下 egg-scripts 的解析就能明白了；

在看完 common-bin 的文档与 egg-scripts 的解析后，我们知道，同样对于 egg-bin 我们直接分析 /lib/cmd/dev.js 即可，我们这里暂时跳过用于测试的其他脚本；

直接看看 /lib/cmd/dev.js 吧

```js
class DevCommand extends Command {
  constructor(rawArgv) {
    super(rawArgv);
    this.usage = 'Usage: egg-bin dev [dir] [options]';

    // 从此处看出如果不指定，默认端口就是 7001，仍然是以 start-cluster 来使用 egg-cluster 启动，唯一的区别就是 dev 下是使用 debug 包来启动的，方便我们开发时实时打印日志及热重载
    this.defaultPort = 7001;
    this.serverBin = path.join(__dirname, '../start-cluster');

    this.options = {
      baseDir: {
        description: 'directory of application, default to `process.cwd()`',
        type: 'string',
      },
      cluster: {
        description: 'numbers of app workers, if not provide then only 1 worker, provide without value then `os.cpus().length`',
        type: 'number',
        alias: 'c',
      },
      port: {
        description: 'listening port, default to 7001',
        type: 'number',
        alias: 'p',
      },
      framework: {
        description: 'specify framework that can be absolute path or npm package',
        type: 'string',
      },
      require: {
        description: 'will add to execArgv --require',
        type: 'array',
        alias: 'r',
      },
    };
  }

  // 启动时的日志描述
  get description() {
    return 'Start server at local dev mode';
  }

  get context() {
    const context = super.context;
    const { argv, execArgvObj } = context;
    execArgvObj.require = execArgvObj.require || [];
    // add require to execArgv
    if (argv.require) {
      execArgvObj.require.push(...argv.require);
      argv.require = undefined;
    }
    return context;
  }

  * run(context) {
    // 序列化上下文参数，比较简单，不做多余解释了，需要先说明的是 index.js 中 get context 已经预先处理过兼容 ts 的情况了
    const devArgs = yield this.formatArgs(context);
    const env = {
      NODE_ENV: 'development',
      EGG_MASTER_CLOSE_TIMEOUT: 1000,
    };
    const options = {
      execArgv: context.execArgv,
      env: Object.assign(env, context.env),
    };
    // 建议大家先去简单看下 debug 这个包的用法，用法特别简单的一个 debug 工具
    // 这里是开始运行 egg-bin dev 时的 debug 日志输出
    debug('%s %j %j, %j', this.serverBin, devArgs, options.execArgv, options.env.NODE_ENV);
    // forkNode helper 这是 common-bin 自带的 helper，用于创建一个 node 子进程来运行这个指令
    // 我简单看了下 common-bin 中 forkNode 的源码，非常简单，就是使用 child_process.fork 来跑的，我们不需要像 ege-scripts 中一样使用 spawn 还需要写明为 node ...，因为 spawn 是使用 child_process.spawn 需要指定 command 为 node，所以参数也和 fork 这个 api 的参数一致 forkNode(modulePath[, args][, options])
    // this.serverBin 指定的运行模块路径仍为 start-cluster
    // 需要注意使用 debug 函数的地方即为每次 debug 监听输出日志的地方
    yield this.helper.forkNode(this.serverBin, devArgs, options);
  }

  /**
   * format egg startCluster args then change it to json string style
   * @method helper#formatArgs
   * @param {Object} context - { cwd, argv }
   * @return {Array} pass to start-cluster, [ '{"port":7001,"framework":"egg"}' ]
   */
  * formatArgs(context) {
    const { cwd, argv } = context;
    /* istanbul ignore next */
    argv.baseDir = argv._[0] || argv.baseDir || cwd;
    /* istanbul ignore next */
    if (!path.isAbsolute(argv.baseDir)) argv.baseDir = path.join(cwd, argv.baseDir);

    // 如未指定，则默认子线程数量为 1，即 master => agent => work(1)
    argv.workers = argv.cluster || 1;
    argv.port = argv.port || argv.p;
    argv.framework = utils.getFrameworkPath({
      framework: argv.framework,
      baseDir: argv.baseDir,
    });

    // remove unused properties
    argv.cluster = undefined;
    argv.c = undefined;
    argv.p = undefined;
    argv._ = undefined;
    argv.$0 = undefined;

    // auto detect available port
    if (!argv.port) {
      debug('detect available port');
      const port = yield detect(this.defaultPort);
      if (port !== this.defaultPort) {
        argv.port = port;
        console.warn(`[egg-bin] server port ${this.defaultPort} is in use, now using port ${port}\n`);
      }
      debug(`use available port ${port}`);
    }
    return [ JSON.stringify(argv) ];
  }
}
```

整体来说其实 egg-bin 是与 egg-scripts 的玩法一样的，唯一的不同就是添加了 debug 工具用于方便我们 debug；

需要注意的是通过 egg-scripts stop 也是可以停止 egg-bin 进程的，因为都是属于 node 进程，除非指定 title
