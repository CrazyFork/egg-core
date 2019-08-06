
# 目标
* 公司用, 所以就读下, eggjs文档毕竟还有很多事情没有交代清楚
* how unit test are structured

# Links

* v8 - https://v8.dev/docs

##

-> lifecycle 的设计:
里边用了很多异步事件驱动的; 代码逻辑结构跳来跳去的. 非常不能理解为啥这么设计(到现在也没看懂 :(, 
我理解启动阶段有问题就退出, 应该是同步的代码结构. 真的是...

设计比较令人难理解的是, 如果我没有猜错 lifecycle.js 中的代码, (看下面) Boots 数组里边应该只允许有一个
Boot 实例才对; 要不然这个 get-ready 的库真的要区分多个boot全部ok, 才call `this.loadReady` .  但这个感觉就不可能
算了, 不纠结了.

```js
triggerDidLoad() {
  debug('register didLoad');
  for (const boot of this[BOOTS]) {
    const didLoad = boot.didLoad && boot.didLoad.bind(boot);
    if (didLoad) {
      // 到下一阶段
      this[REGISTER_READY_CALLBACK](didLoad, this.loadReady, 'Did Load');
    }
  }
}
```

-> make middleware private


```js

    const middlewarePaths = opt.directory;
    this.loadToApp(middlewarePaths, 'middlewares', opt);

    for (const name in app.middlewares) {
      // using defineProperty to make app.middleware unsettable.
      Object.defineProperty(app.middleware, name, {
        get() {
          return app.middlewares[name];
        },
        enumerable: false, // neccessary, but why ?
        configurable: false,
      });
    }
```

-> debugging, using debug lib
app.logger are fit for application developer usage, while if you are a lib developer, it's much more appropriate to use
this debug lib.



```raw
lib
├── egg.js                     // Koa Application 的子类; 可以理解为一个Application; fields 包括 context, controller, service, 还有路由的一些方法
├── lifecycle.js               // lifecycle 的methods, 统管lifecycle, 包括对应的 Hook 实例的注入和trigger in different lifecycle stage
├── loader
│   ├── context_loader.js
│   ├── egg_loader.js          // egg 的核心loader, 
│   ├── file_loader.js         // loader 的核心; 底层嵌入了nodejs loader; 会对对应目录下文件(glob)形式解析; 然后将文件目录转换成props注入到target的属性上面, 对应inject的对象也会被注入到文件里边, 一参数的形式
│   └── mixin
│       ├── config.js         // 加载config, 
│       ├── controller.js     // 用来加载controller
│       ├── custom.js
│       ├── custom_loader.js
│       ├── extend.js
│       ├── middleware.js
│       ├── plugin.js
│       ├── router.js
│       └── service.js
└── utils
    ├── base_context_class.js   // 鬼知道是干啥的? 通过名字猜
    ├── index.js                // 定义了一些loadFile, error stack, etc...方法; loadFile 只会两种状态buffer文件内容; js解析后的exports
    ├── sequencify.js           // 用于收集任务的执行顺序的, 带有依赖任务的执行顺序的那种
    └── timing.js               // 记录每个pid timing的util
```





esmodule
```js
// 原来esmodule是搞这个事情
if (obj.__esModule) return 'default' in obj ? obj.default : obj;


// bm
fullPath = require.resolve(filepath);
```

### 启动顺序(具体的顺序要看eggjs外层的包装库是怎么实现的)

```js
        app.Helper = class Helper {};
        app.loader.loadPlugin();
        app.loader.loadConfig();
        app.loader.loadApplicationExtend();
        app.loader.loadAgentExtend();
        app.loader.loadRequestExtend();
        app.loader.loadResponseExtend();
        app.loader.loadContextExtend();
        app.loader.loadHelperExtend();
        app.loader.loadCustomApp();
        app.loader.loadService();
        app.loader.loadController();
        app.loader.loadRouter();
        app.loader.loadPlugin();
        app.loader.loadMiddleware();
```



### sequencify.js 执行顺序, 
主要看代码
* 从顶部开始以names遍历, 沿着dependencies往下走,
* 任何cyclular dep被记录, 访问的parent节点被记录
* reach to leaf node, set result[name] = true, 记录这个任务必须执行, push 到任务队列里边
* 然后将parent再push到任务队列里边. 总体就是一个将树的深度遍历, 至不过执行顺序会将parent节点下的children节点先行遍历一遍.


### 配置的注入顺序

defined in config.js, loadConfig()
代码我总看的虚, 关于middleware的注入配置; 详细看源码; 主要就是说array的配置, 同样的index会被override掉

注意覆盖顺序是从下往上. 下面优先级最高
```js

    //   plugin config.default
    //     framework config.default
    //       app config.default
    //         plugin config.{env}
    //           framework config.{env}
    //             app config.{env}
```


### plugin 的执行顺序
执行顺序就是按照sequencify 的执行顺序走的; 无非就是会设置对应的plugin的禁用状态;
### Hook, BootHook
```
class Hook {
  constructor(app) {
    this.app = app;
  }
  configDidLoad() {
    hook(this.app);
  }
}
```

* beforeClose, 可以在app进行关闭前的hook, 会被执行


### lifecycle 


```
// 生命周期和boot明确相关; 每个boot的生命周期函数会被扔在this[REGISTER_READY_CALLBACK]函数处理
  triggerWillReady() {
    debug('register willReady');
    this.bootReady.start();
    for (const boot of this[BOOTS]) {
      const willReady = boot.willReady && boot.willReady.bind(boot);
      if (willReady) {
        this[REGISTER_READY_CALLBACK](willReady, this.bootReady, 'Will Ready');
      }
    }
  }
```



define in plugins.js
` getOrderPlugins(allPlugins, enabledPluginNames, appPlugins) { `

## third party libs

```js

this._extendPlugins(this.allPlugins, eggPlugins);
this._extendPlugins(this.allPlugins, appPlugins);
this._extendPlugins(this.allPlugins, customPlugins);
// 虽然上边那么走; 但是for in 遍历的顺序是没法保证的
for (const name in this.allPlugins) {
  // 解析插件的完整路径

  ...
}
// 插件的顺序, 按照for in的顺序, 解析第一个插件的dep叶节点的插件执行
// 解析依赖; implicit 依赖; missing 依赖, 统一格式化依赖路径...
sequencify(plugins...)


```


### egg loader

```js
// 文件的组织形式, prototype的继承
const loaders = [
  require('./mixin/plugin'),
  require('./mixin/config'),
  require('./mixin/extend'),
  require('./mixin/custom'),
  require('./mixin/service'),
  require('./mixin/middleware'),
  require('./mixin/controller'),
  require('./mixin/router'),
  require('./mixin/custom_loader'),
];

for (const loader of loaders) {
  Object.assign(EggLoader.prototype, loader);
}
```

### middleware 的注入顺序

middleware 注入顺序是 coreMiddlewares -> app middlewares, 

-> middleware 的限制, 
* 只有framework 有能力注入coreMiddleware.
* plugin 只允许定义middleware, app 决定用哪些

```js
    if (isPlugin || isApp) {
      assert(!config.coreMiddleware, 'Can not define coreMiddleware in app or plugin');
    }
    if (!isApp) {
      assert(!config.middleware, 'Can not define middleware in ' + filepath);
    }
```

### router 的生效时机
?也就是说要在beforeStart前执行.
A: 不是的, middleware 注入了, 后面随便调用 `app.use`

真正的注入时机实在beforeStart之后,  https://eggjs.org/en/advanced/loader.html, 看app的时机图.


```js
get router() {
  if (this[ROUTER]) {
    return this[ROUTER];
  }
  const router = this[ROUTER] = new Router({ sensitive: true }, this);
  // register router middleware
  this.beforeStart(() => {
    this.use(router.middleware());
  });
  return router;
}



registerBeforeStart(scope) {
  this[REGISTER_READY_CALLBACK](scope, this.loadReady, 'Before Start');
}
```


# egg-core

[![NPM version][npm-image]][npm-url]
[![build status][travis-image]][travis-url]
[![Test coverage][codecov-image]][codecov-url]
[![David deps][david-image]][david-url]
[![Known Vulnerabilities][snyk-image]][snyk-url]
[![npm download][download-image]][download-url]

[npm-image]: https://img.shields.io/npm/v/egg-core.svg?style=flat-square
[npm-url]: https://npmjs.org/package/egg-core
[travis-image]: https://img.shields.io/travis/eggjs/egg-core.svg?style=flat-square
[travis-url]: https://travis-ci.org/eggjs/egg-core
[codecov-image]: https://codecov.io/github/eggjs/egg-core/coverage.svg?branch=master
[codecov-url]: https://codecov.io/github/eggjs/egg-core?branch=master
[david-image]: https://img.shields.io/david/eggjs/egg-core.svg?style=flat-square
[david-url]: https://david-dm.org/eggjs/egg-core
[snyk-image]: https://snyk.io/test/npm/egg-core/badge.svg?style=flat-square
[snyk-url]: https://snyk.io/test/npm/egg-core
[download-image]: https://img.shields.io/npm/dm/egg-core.svg?style=flat-square
[download-url]: https://npmjs.org/package/egg-core

A core Pluggable framework based on [koa](https://github.com/koajs/koa).

**Don't use it directly, see [egg].**

## Usage

Directory structure

```
├── package.json
├── app.js (optional)
├── agent.js (optional)
├── app
|   ├── router.js
│   ├── controller
│   │   └── home.js
|   ├── extend (optional)
│   |   ├── helper.js (optional)
│   |   ├── filter.js (optional)
│   |   ├── request.js (optional)
│   |   ├── response.js (optional)
│   |   ├── context.js (optional)
│   |   ├── application.js (optional)
│   |   └── agent.js (optional)
│   ├── service (optional)
│   ├── middleware (optional)
│   │   └── response_time.js
│   └── view (optional)
|       ├── layout.html
│       └── home.html
├── config
|   ├── config.default.js
│   ├── config.prod.js
|   ├── config.test.js (optional)
|   ├── config.local.js (optional)
|   ├── config.unittest.js (optional)
│   └── plugin.js
```

Then you can start with code below

```js
const Application = require('egg-core').EggCore;
const app = new Application({
  baseDir: '/path/to/app'
});
app.ready(() => app.listen(3000));
```

## EggLoader

EggLoader can easily load files or directories in your [egg] project. In addition, you can customize the loader with low level APIs.

### constructor

- {String} baseDir - current directory of application
- {Object} app - instance of egg application
- {Object} plugins - merge plugins for test
- {Logger} logger - logger instance，default is console

### High Level APIs

#### loadPlugin

Load config/plugin.js

#### loadConfig

Load config/config.js and config/{serverEnv}.js

#### loadController

Load app/controller

#### loadMiddleware

Load app/middleware

#### loadApplicationExtend

Load app/extend/application.js

#### loadContextExtend

Load app/extend/context.js

#### loadRequestExtend

Load app/extend/request.js

#### loadResponseExtend

Load app/extend/response.js

#### loadHelperExtend

Load app/extend/helper.js

#### loadCustomApp

Load app.js, if app.js export boot class, then trigger configDidLoad

#### loadCustomAgent

Load agent.js, if agent.js export boot class, then trigger configDidLoad

#### loadService

Load app/service

### Low Level APIs

#### getServerEnv()

Retrieve application environment variable values via `serverEnv`. You can access directly by calling `this.serverEnv` after instantiation.

serverEnv | description
---       | ---
default   | default environment
test      | system integration testing environment
prod      | production environment
local     | local environment on your own computer
unittest  | unit test environment

#### getEggPaths()

To get directories of the frameworks. A new framework is created by extending egg, then you can use this function to get all frameworks.

#### getLoadUnits()

A loadUnit is a directory that can be loaded by EggLoader, cause it has the same structure.

This function will get add loadUnits follow the order:

1. plugin
2. framework
3. app

loadUnit has a path and a type. Type must be one of those values: *app*, *framework*, *plugin*.

```js
{
  path: 'path/to/application',
  type: 'app'
}
```

#### getAppname()

To get application name from *package.json*

#### appInfo

Get the infomation of the application

- pkg: `package.json`
- name: the application name from `package.json`
- baseDir: current directory of application
- env: equals to serverEnv
- HOME: home directory of the OS
- root: baseDir when local and unittest, HOME when other environment

#### loadFile(filepath)

To load a single file. **Note:** The file must export as a function.

#### loadToApp(directory, property, LoaderOptions)

To load files from directory in the application.

Invoke `this.loadToApp('$baseDir/app/controller', 'controller')`, then you can use it by `app.controller`.

#### loadToContext(directory, property, LoaderOptions)

To load files from directory, and it will be bound the context.

```js
// define service in app/service/query.js
module.exports = class Query {
  constructor(ctx) {
    // get the ctx
  }

  get() {}
};

// use the service in app/controller/home.js
module.exports = function*() {
  this.body = this.service.query.get();
};
```

#### loadExtend(name, target)

Loader app/extend/xx.js to target, For example,

```js
this.loadExtend('application', app);
```

### LoaderOptions

Param          | Type           | Description
-------------- | -------------- | ------------------------
directory      | `String/Array` | directories to be loaded
target         | `Object`       | attach the target object from loaded files
match          | `String/Array` | match the files when load, default to `**/*.js`(if process.env.EGG_TYPESCRIPT was true, default to `[ '**/*.(js|ts)', '!**/*.d.ts' ]`)
ignore         | `String/Array` | ignore the files when load
initializer    | `Function`     | custom file exports, receive two parameters, first is the inject object(if not js file, will be content buffer), second is an `options` object that contain `path`
caseStyle      | `String/Function` | set property's case when converting a filepath to property list.
override       | `Boolean`      | determine whether override the property when get the same name
call           | `Boolean`      | determine whether invoke when exports is function
inject         | `Object`       | an object that be the argument when invoke the function
filter         | `Function`     | a function that filter the exports which can be loaded

## Questions & Suggestions

Please open an issue [here](https://github.com/eggjs/egg/issues).

## License

[MIT](LICENSE)

[egg]: https://github.com/eggjs/egg
