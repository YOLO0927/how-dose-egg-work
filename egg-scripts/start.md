start 脚本命令中，我们重点去分析 构造函数 与 上述提到过的 run 函数即可，因为构造函数中继承超集后，会将命令所需的属性与配置一一添加，而 run 函数即为 start 命令执行的函数，到最后则都是跑的 /lib/start-cluster 后续会简单解析

先给大家科普一个 node 进程的参数都是什么，知道的同学就不用看了，了解过 node 进程的同学都应该知道，使用 process.argv 获取继承参数列表，参数分布应为
example command => node test.js --name=YOLO
```js
  [
    node解释器的地址{String}(即你的 node 安装目录下的 bin 下的 node 解释器),
    test.js 这个脚本的地址{String}，
    '--name=YOLO'{Any}(你写的序列化参数)，
    ...
  ]
```

在不懂的话自己随便跑一个测试脚本打印看看吧

下面为需要解析的源码，我们已经将几个辅助函数剔除了

```js
class StartCommand extends Command {
  constructor(rawArgv) {
    super(rawArgv);
    // 设置命令的使用方法，这也是为什么 egg-scripts start 能运行的原因
    this.usage = 'Usage: egg-scripts start [options] [baseDir]';
    this.serverBin = path.join(__dirname, '../start-cluster');

    // yargs usage 中的 options，例如 egg 官方 demo 中的 egg-script start title=egg-application，在停止多线程服务时，我们也可指定 title，更多选项可以查看 egg 文档内的 应用部署
    this.options = {
      title: {
        description: 'process title description, use for kill grep, default to `egg-server-${APP_NAME}`',
        type: 'string',
      },
      // 使用 node cluster 模块集群部署时的子进程数，默认为 cpu 个数
      workers: {
        description: 'numbers of app workers, default to `os.cpus().length`',
        type: 'number',
        alias: [ 'c', 'cluster' ],
        default: process.env.EGG_WORKERS,
      },
      // 应用监听的端口
      port: {
        description: 'listening port, default to `process.env.PORT`',
        type: 'number',
        alias: 'p',
        default: process.env.PORT,
      },
      // 应用启动的环境常量，默认为 EGG_SERVER_ENV
      env: {
        description: 'server env, default to `process.env.EGG_SERVER_ENV`',
        default: process.env.EGG_SERVER_ENV,
      },
      // 默认是 egg
      framework: {
        description: 'specify framework that can be absolute path or npm package',
        type: 'string',
      },
      // 是否后台启动进程守护，即进程的自动重启、日志打印等
      daemon: {
        description: 'whether run at background daemon mode',
        type: 'boolean',
      },
      stdout: {
        description: 'customize stdout file',
        type: 'string',
      },
      stderr: {
        description: 'customize stderr file',
        type: 'string',
      },
      // 应用最长启动超时时间，默认为 300s
      timeout: {
        description: 'the maximum timeout when app starts',
        type: 'number',
        default: 300 * 1000,
      },
      'ignore-stderr': {
        description: 'whether ignore stderr when app starts',
        type: 'boolean',
      },
    };
  }

  // 命令执行时 ci 界面的日志说明
  get description() {
    return 'Start server at prod mode';
  }

  * run(context) {
    const { argv, env, cwd, execArgv } = context;

    // HOME 为当前系统用户 node 环境下的启动目录，例如我这里是 /Users/yanglu
    const HOME = homedir();
    // 日志存放目录，argv 为命令后所有的选项参数，下面就是一系列的抽出参数并处理了
    const logDir = path.join(HOME, 'logs');

    // egg-script start
    // egg-script start ./server
    // egg-script start /opt/app
    let baseDir = argv._[0] || cwd;
    // baseDir 默认为应用项目存放目录
    if (!path.isAbsolute(baseDir)) baseDir = path.join(cwd, baseDir);
    argv.baseDir = baseDir;

    const isDaemon = argv.daemon;

    argv.framework = yield this.getFrameworkPath({
      framework: argv.framework,
      baseDir,
    });

    // 处理默认框架名，若 package.json 没写 name 则默认为 egg
    this.frameworkName = yield this.getFrameworkName(argv.framework);

    const pkgInfo = require(path.join(baseDir, 'package.json'));
    // 处理应用 title
    argv.title = argv.title || `egg-server-${pkgInfo.name}`;

    // 赋值输出与输出日志地址
    argv.stdout = argv.stdout || path.join(logDir, 'master-stdout.log');
    argv.stderr = argv.stderr || path.join(logDir, 'master-stderr.log');

    // normalize env
    env.HOME = HOME;
    env.NODE_ENV = 'production';

    env.PATH = [
      // for nodeinstall
      path.join(baseDir, 'node_modules/.bin'),
      // support `.node/bin`, due to npm5 will remove `node_modules/.bin`
      path.join(baseDir, '.node/bin'),
      // adjust env for win
      env.PATH || env.Path,
    ].filter(x => !!x).join(path.delimiter);

    // for alinode
    env.ENABLE_NODE_LOG = 'YES';
    env.NODE_LOG_DIR = env.NODE_LOG_DIR || path.join(logDir, 'alinode');
    // 在上面所说的日志目录中创建 alinode 目录
    yield mkdirp(env.NODE_LOG_DIR);

    // 处理环境常量，若无参数声明则默认 prod
    // cli argv -> process.env.EGG_SERVER_ENV -> `undefined` then egg will use `prod`
    if (argv.env) {
      // if undefined, should not pass key due to `spwan`, https://github.com/nodejs/node/blob/master/lib/child_process.js#L470
      env.EGG_SERVER_ENV = argv.env;
    }

    const options = {
      execArgv,
      env,
      stdio: 'inherit',
      detached: false,
    };

    // 日志输出，遵循了 C 语言的 printf 输出方式，学过的同学应该知道 %s 是 C 语言字符串的通配符
    // 这里的意思是 ci 界面在启动时将会打印日志 Starting [this.frameworkName] application at [baseDir]
    this.logger.info('Starting %s application at %s', this.frameworkName, baseDir);

    // remove unused properties from stringify, alias had been remove by `removeAlias`
    const ignoreKeys = [ '_', '$0', 'env', 'daemon', 'stdout', 'stderr', 'timeout', 'ignore-stderr' ];
    const eggArgs = [ this.serverBin, stringify(argv, ignoreKeys), `--title=${argv.title}` ];
    // 上面解释过打印方式了
    this.logger.info('Run node %s', eggArgs.join(' '));

    // whether run in the background.
    // 是否在后台运行
    if (isDaemon) {
      this.logger.info(`Save log file to ${logDir}`);
      const [ stdout, stderr ] = yield [ getRotatelog(argv.stdout), getRotatelog(argv.stderr) ];
      options.stdio = [ 'ignore', stdout, stderr, 'ipc' ];
      options.detached = true;

      // 同理这里的 spawn 使用的是 child_process.spawn，在当前进程衍生一个新的子进程去跑服务，需要运行的命令为 node，字符串参数列表为 eggArgs，即在 start-cluster 中进程 process.argv 数组的一个值即为 node 地址，第二个为 this.serverBin（即需要用 node 解释的脚本路径），第三个为 stringify(argv, ignoreKeys)，注意重点这里的 argv.framework 为 egg
      const child = this.child = spawn('node', eggArgs, options);
      // 是否就绪：false
      this.isReady = false;
      // 监听子进程 message 事件，当子进程使用 process.send 发送信息时接收并进行处理
      child.on('message', msg => {
        /* istanbul ignore else */
        // 当收到子进程消息，且 msg.action 为 egg-ready，则代表已经使用 cluster 启动成功
        if (msg && msg.action === 'egg-ready') {
          this.isReady = true;
          // 打印启动成功日志
          this.logger.info('%s started on %s', this.frameworkName, msg.data.address);
          // 解绑子进程
          child.unref();
          // 断开与子进程的 IPC 通道，进程间不再能够传递信息
          child.disconnect();
          // 退出当前进程，且退出码为 0
          process.exit(0);
        }
      });

      // check start status
      // 队列打印日志 Wait Start: %d... 一秒一次，若出错则会同时退出子进程并在 sleep 1s 后退出当前进程
      yield this.checkStatus(argv);
    } else {
      // signal event had been handler at common-bin helper
      // 若不在后台运行，执行 common-bin helper spawn => spawn(cmd, args, opt)，此方法将会开启一个新进程来执行，这里我们看到若不在后台运行则为最简单的 node /node_modules/egg-scripts/lib/start-cluster，也就是说去运行 start-cluster 脚本，使用 cluster 模块多线程启动服务，参数则为上述我们抽象好的 options 对象
      this.helper.spawn('node', eggArgs, options);
    }
  }
}
```
