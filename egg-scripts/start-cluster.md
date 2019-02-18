首先大家都知道这个脚本是由 start.js 运行的，进入后看到这个脚本是没有文件名后缀的，但是在内部声明了使用 node 解释，下面我们分析一下

```js
// 指定脚本使用 node 环境来解释
#!/usr/bin/env node

'use strict';

// 将我们在 start.js 中 spawn('node', eggArgs, options) eggArgs 这个数组中的 stringify(argv, ignoreKeys) 解析出来
const options = JSON.parse(process.argv[2]);
// 此时的 options.framework 应为 /node_modules/egg
// 此时就是直接加载调用 egg 的 startCluster 方法启动，我们后续直接简单分析一下就知道了
require(options.framework).startCluster(options);
```

此后我们切到 egg 下查看 index.js 发现这样一行代码

```js
  exports.startCluster = require('egg-cluster').startCluster;
```

ok，得了您咧，线索跳至去简单分析 egg-cluster
