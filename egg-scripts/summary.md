首先我们大家使用时都很清楚，我们执行的 `npm run start` or `npm run stop` 是在使用 egg-scripts 模块对应执行的 `egg-scripts start` 与 `egg-scripts stop`，所以我们首先根据这个启动的入口去看看 egg-scripts 是如何 start 与 stop 的

下面是我们追本溯源的过程，每一步我都会大概解释为什么，看完大家就会明白 egg-scripts 是怎么运行的了，并且了解我学习的方式

1. 首先我们根据 CommonJs 规范去查看 package.json 的 main 指向以及 bin 指向
   * 了解 CommonJs 规范的同学应该知道，一个符合 CommonJs npm 包的模块是要存在 main 属性指向该模块的初始入口的，如果未指定则会默认未根目录下的 index.js，如若不存在则会递归查找各级目录下的 index.js，而 bin 目录下则是存放可执行文件的目录；
   * 在这里什么叫可执行文件了，简单来说就是你系统当前环境的解释器可以解释的文件，比如你系统是有 node 环境的，那么当然能解析 js 文件，如果你处于 linux 环境下，你直接写一个 shell 脚本都ok；
   * 如果 CommonJs 规范不了解，或者上面我所说的都没印象的话，建议先补一下相关的知识，搞清楚规范，不然看一个新包你还是不懂怎么看，如果找不到对应资料，可以查看我这篇文章大概了解一下 [模块机制与 CommonJs 规范](https://blog.csdn.net/yolo0927/article/details/79405181)

2. 分析 /index.js
  ```js
  'use strict';

  const path = require('path');
  const Command = require('./lib/command');

  class EggScripts extends Command {
    constructor(rawArgv) {
      super(rawArgv);
      this.usage = 'Usage: egg-scripts [command] [options]';

      // load directory
      this.load(path.join(__dirname, 'lib/cmd'));
    }
  }

  module.exports = exports = EggScripts;
  exports.Command = Command;
  exports.StartCommand = require('./lib/cmd/start');
  exports.StopCommand = require('./lib/cmd/stop');
  ```

  * 上面是 egg-scripts 入口的源码，初看之下我们只发现模块输出了 EggScripts 这个类，并且很严谨的使用了 `module.exports = exports = EggScripts` 来保证默认指向，在其中我们很容易通过命名就发现模块中存在 2 个方法，一个是 `StartCommand`，一个是 `StopCommand`，并且发现是通过 require 引入的 start 模块与 stop 模块，看到这里我们已经可以猜到了，这两个被引入的模块应该可能就是我们 egg-scripts 对应的 start 与 stop 脚本了，但这还只是猜测，所以接下来我们就沿着这个线索去解决它
  * 有了猜测之后，我们直接看到 EggScripts 类是继承于 Command 的，并且在构造函数 constructor 中使用 super 引入超集，这时我们就可以使用 this 去调用父类方法了，所以此时我们可以看到这个 `this.load` 一定是这个 Command 父类提供的，所以我们下一步就是去查看 Command 模块；

 3. 分析 /lib/command.js
    ```js
    'use strict';

    const BaseCommand = require('common-bin');
    const Logger = require('zlogger');
    const helper = require('./helper');

    class Command extends BaseCommand {
      constructor(rawArgv) {
        super(rawArgv);

        // 合并 egg-scripts 中 /lib/helper.js 中的 helper 入内
        Object.assign(this.helper, helper);

        this.parserOptions = {
          removeAlias: true,
          removeCamelCase: true,
          execArgv: true,
        };

        this.logger = new Logger({
          prefix: '[egg-scripts] ',
          time: false,
        });
      }
    }

    module.exports = Command;
    ```

    整体一看，非常简单，还是沿用我们分析 index.js 的思路，一看就发现，这里起作用的就是 BaseCommand 类，即 `common-bin` 模块，接下来我们将线索放到这个模块就可以了

    https://www.npmjs.com/package/common-bin

    通过 npm 的搜索，我们发现它原来就是 天猪和死马 他们写的，也就是说是 egg 的开源团队写的，然后看一下模块的简介，我直接粗略翻译给大家就是

    “根据 yargs 包抽象化的 bin 工具，提供了更加便利的使用体验以及支持 async 与 generator”

    看完简介其实我们就已经可以知道 egg-scripts 是个什么东西了
      1. yargs 包与 commander 包的功能是差不多的，我前面的博客提到过 vue-cli 是在干嘛，并且在上面 模块机制 这篇文章的最后教过大家如何学习 vue-init 去自定义命令，里面使用的就是 commander，所以这里我们猜测 common-bin 这个包其实就是封装了 yargs 去做的自定义命令工具；
      2. 接下来我们看一下 common-bin 文档提供的 example（demo 例子）发现了，egg-scripts 的目录结构以及内部实现基本与此 demo 一致，over，那就简单了，我们看看 common-bin 的使用方法就知道 egg-scripts 在干嘛了；
      3. 看文档 demo 的过程大家自己看，我直接说结果
          * 首先我们发现 egg-scripts 下 /index.js 中的 `this.load` 原来就是引入自定义命令的目录，而 /bin 目录下的 egg-scripts.js 中的 `new Command().start()` 就是执行 index.js，即我们的命令就为 bin 下这个模块名 => egg-scripts，而命令的选项就是 `this.load(path.join(__dirname, 'lib/cmd'))` /lib/cmd 下的 2 个脚本的名字，start 与 stop，我们最初的猜测完全正确，接下来我们只需要分析 /lib/cmd/start.js 与 /lib/cmd/start.js 在干什么，就能清楚 `egg-script start` 与 `egg-scripts stop`在干什么了
          * 接下来我们看到 start.js 与 stop.js 脚本中，大体粗略看结构我们发现有共性，都使用了 getter 访问器去定义 `description` 属性，以及都使用了生成器函数 `* run`，传入的参数为 context 上下文，我们大家都知道生成器是用来解决异步队列的问题，那么这里我们则猜测 start 与 stop 跑的函数就是这个函数，这里使用生成器解决异步队列的原因就是为了多条命令同时运行使能够按照队列顺序执行；
          * 最后我们看到 common-bin 模块文档的 Concept 处发现了这些函数的说明，ok，load、options、usage、run 都在里面，证实了我们的推测，我们翻译几条我们核心的东西

          ** Method: **

          load(fullPath)：注册命令的入口目录（ok，意思就是说，我们传入的参数就是目录地址，这个地址下的脚本名都会被注册为命令选项）;

          run(context)：
              1. 提供组装命令的处理者，并且提供没有子命令时的校验；（这里的子命令即为命令的参数等情况）；
              2. 支持 generator、async function、普通 function，它们最终将返回一个 promise；

          start()：开始你的程序，仅仅只在你的 bin 文件中使用一次（这就解释了 egg-scripts bin 目录下为什么要 start 了，因为不 start 就不会注册运行）

          ** properties: **

          description{String}：一个 getter 访问器，只用于展示执行对应命令时 ci 界面中出现的帮助日志说明；

          options{Object}：一个 setter 访问器，设置 yargs 的选项；

          parserOptions{Object}：控制 context 的解析规则；

    通过以上的结论分析，我们完全搞明白的 egg-scripts 的运行机制，接下来我们针对分析 start 与 stop 脚本干了什么，来解决我们最初的疑问，egg-scripts start 与 egg-stop 做了什么，上面提到的 common-bin 的 properties 将会在其中出现，我不会再次做解释了，大家可以自行查看文档描述

    ** 下面分别查看 ./start.md 与 ./stop.md **

    至于对 common-bin 源码感兴趣的同学就自己去翻阅啦，我也不可能一直解析到底层
