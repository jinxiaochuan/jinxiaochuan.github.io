---
title: webpack 之 LoaderRunner 源码解读
date: 2022-02-22 17:27:27
tags: Webpack
---

#### 回顾 webpack 构建编译

我们知道，`webpack` 整个的编译过程

> `compiler.run -> [beforeRun hook -> run hook] -> compiler.compile -> [beforeCompile hook -> compile hook] -> compiler.newCompilation(params) -> [make hook] -> compilation.finish -> compilation.seal -> [afterCompile hook]`
>
> 其中最核心的编译阶段是 `make hook` 的触发调用。而订阅 `make hook` 的 `plugin` 为 `SingleEntryPlugin`、`MultiEntryPlugin`、`DynamicEntryPlugin`。
> 通过 `SingleEntryPlugin` 等 `plugin` 作为构建的入口起始点，即从入口文件开始构建编译。
>
> `make hook` 触发后将执行 `compilation.addEntry -> compilation._addModuleChain -> buildModule -> module.build -> module.doBuild -> runLoaders`
>
> 即从入口文件开始进行构建打包，实际的构建处理是通过调用 `runLoaders` 实现。

#### loader-runner 功能

##### 执行流程 normal 和 pitch

一个 `loader` 可以定义两类函数，一个默认导出的函数 `normalLoader`，一个用于阻断常规流程的函数 `pitchLoader`

```bash
// a-loader.js
// normal loader:
function aLoader(resource) {
    // ...
    return resource
}
// pitch loader:
aLoader.pitch = function() {}
module.exports = aLoader
```

如果我们配置 `use: [ 'a-loader', 'b-loader', 'c-loader' ]`，且三个 `loader` 都没有 `pitchLoader` 或 `pitchLoader` 无返回值，`loader` 将会以以下流程执行：

```bash
// |- a-loader `pitch` 没有或无返回值
//   |- b-loader `pitch` 没有或无返回值
//     |- c-loader `pitch` 没有或无返回值
//       |- load resource
//     |- c-loader normal execution
//   |- b-loader normal execution
// |- a-loader normal execution
```

如果在 `b-loader` 的 `pitch` 函数返回了某个值，流程将会变成下面这样：

```bash
// |- a-loader `pitch`
//   |- b-loader `pitch` 有返回值
// |- a-loader normal execution
```

##### 支持同步/异步

`loader` 可以支持以同步或异步(`callback, Promise`)方式运行，调用 `this.async()` 获取回调，并在执行完毕后调用。

```bash
module.exports = function(resource) {
    const callback = this.async()
    asyncFunc((err, res) => {
        callback(err, res)
    })
}
```

##### loader.raw

通过这个参数指定 `loader` 接收一个 `buffer` 类型的资源或 `string` 类型的资源

#### loader-runner 核心源码解析

##### runLoaders 入口函数

```bash
function runLoaders(options, callback) {
  // 定义上下文
  var loaderContext = options.context || {};
  // 待解析的文件
  loaderContext.resourcePath = options.resource;
  // 待解析文件的目录
  loaderContext.context = dirname(options.resource);
  // 当前执行到第几个loader
  loaderContext.loaderIndex = 0;
  // 创建loader对象
  loaderContext.loaders = options.loaders

  // 执行Pitch阶段
	var processOptions = {
		resourceBuffer: null,
		readResource: fs.readFile.bind(fs),
	};
  iteratePitchingLoaders(processOptions, loaderContext, (err, res) => {
		callback(null, {
      // 最后经过loader输出的值，可能为buffer或string
      result: result,
      // 最原始的资源buffer
      resourceBuffer: processOptions.resourceBuffer,
      // 是否需要缓存结果
      cacheable: requestCacheable,
      // loader需要监听的文件
      fileDependencies: fileDependencies,
      // loader需要监听的文件夹
      contextDependencies: contextDependencies
		});
  })
}
```

##### iteratePitchingLoaders

这里采用了递归的方法来处理 `loader` 链式操作，当 `pitch` 都执行完开始加载资源，当 `pitch` 有返回值直接跳过加载资源，往回执行 `normalLoader`

```bash
function iteratePitchingLoaders(options, loaderContext, callback) {
  // 如果所有loader的pitch都执行完，就开始执行 processResource 并进行处理源文件
	if(loaderContext.loaderIndex >= loaderContext.loaders.length) {
    return processResource(options, loaderContext, callback);
  }

	var loader = loaderContext.loaders[loaderContext.loaderIndex];

	// 奇葩的递归执行操作，循环递增条件放在这里
	if(loader.pitchExecuted) {
		loaderContext.loaderIndex++;
		return iteratePitchingLoaders(options, loaderContext, callback);
  }

  // 加载执行loader.pitch
	loadLoader(loader, function(err) {
		var fn = loader.pitch;
		loader.pitchExecuted = true;
		if(!fn) return iteratePitchingLoaders(options, loaderContext, callback);

    runSyncOrAsync(fn, loaderContext, [loaderContext.remainingRequest,  loaderContext.previousRequest, loader.data = {}], function(err) {
      if(err) return callback(err);
      var args = Array.prototype.slice.call(arguments, 1);
      // pitch有返回值，直接跳过后面的loader，并把返回值给其他loader
      if(args.length > 0) {
        loaderContext.loaderIndex--;
        iterateNormalLoaders(options, loaderContext, args, callback);
      } else {
        iteratePitchingLoaders(options, loaderContext, callback);
      }}
    )}
  );
}
```

##### processResource

这里会加载待处理的资源文件，并将其加入到文件监听中，然后开始执行 `normalLoadr`：

```bash
function processResource(options, loaderContext, callback) {
	loaderContext.loaderIndex = loaderContext.loaders.length - 1;

	var resourcePath = loaderContext.resourcePath;
	if(resourcePath) {
		loaderContext.addDependency(resourcePath);
		options.readResource(resourcePath, function(err, buffer) {
			if(err) return callback(err);
			options.resourceBuffer = buffer;
			iterateNormalLoaders(options, loaderContext, [buffer], callback);
		});
	} else {
		iterateNormalLoaders(options, loaderContext, [null], callback);
	}
}
```

##### iterateNormalLoaders

这里递归执行 `normalLoader`，在执行前会进行资源类型的转换

```bash
function iterateNormalLoaders(options, loaderContext, args, callback) {
  // 所有loader执行完毕
	if(loaderContext.loaderIndex < 0)
		return callback(null, args);

	var currentLoaderObject = loaderContext.loaders[loaderContext.loaderIndex];

	// iterate
	if(currentLoaderObject.normalExecuted) {
		loaderContext.loaderIndex--;
		return iterateNormalLoaders(options, loaderContext, args, callback);
	}

	var fn = currentLoaderObject.normal;
  // 标记当前loader已执行过
	currentLoaderObject.normalExecuted = true;
	if(!fn) {
		return iterateNormalLoaders(options, loaderContext, args, callback);
	}

	convertArgs(args, currentLoaderObject.raw);

	runSyncOrAsync(fn, loaderContext, args, function(err) {
		if(err) return callback(err);

		var args = Array.prototype.slice.call(arguments, 1);
		iterateNormalLoaders(options, loaderContext, args, callback);
	});
}
```

执行 `runSyncOrAsync` 之前，通过调用 `convertArgs` 以及当前 `loader` 配置的 `raw` 值来决定是否将上一个 `loader` 传入的 `result` 转化为 `buffer`

```bash
function convertArgs(args, raw) {
	if(!raw && Buffer.isBuffer(args[0]))
		args[0] = utf8BufferToString(args[0]);
	else if(raw && typeof args[0] === "string")
		args[0] = new Buffer(args[0], "utf-8"); // eslint-disable-line
}
```

##### runSyncOrAsync

函数内部的 `isSync` 和 `isDone` 很重要，`isSync` 是来控制同步还是异步 `loader` 的，`isDone` 是防止 `callback` 被触发多次。
`context.async` 是一个闭包函数，它返回的是 `innerCallback`，而 `innerCallback` 内部才是真正执行 `runSyncOrAsync` 的 `callback` 函数，这个 `callback` 会进入下一次的 `iterateNormalLoaders` 逻辑。
同时 `innerCallback` 也是 `context.callback` 的一个引用。真正执行 `loader` 的 `normal` 的函数语句在下面的这个立即执行函数里面。

```bash
function runSyncOrAsync(fn, context, args, callback) {
	var isSync = true;
	var isDone = false;
	var isError = false; // internal error
	var reportedError = false;
	context.async = function async() {
		if(isDone) {
			if(reportedError) return;
			throw new Error('async(): The callback was already called.');
		}
		isSync = false;
		return innerCallback;
	};
	var innerCallback = context.callback = function() {
		if(isDone) {
			if(reportedError) return;
			throw new Error('callback(): The callback was already called.');
		}
		isDone = true;
		isSync = false;
		try {
			callback.apply(null, arguments);
		} catch(e) {
			isError = true;
			throw e;
		}
	};
	try {
		var result = (function LOADER_EXECUTION() {
			return fn.apply(context, args);
		}());
		if(isSync) {
			isDone = true;
			if(result === undefined)
				return callback();
			if(result && typeof result === "object" && typeof result.then === "function") {
				return result.then(function(r) {
					callback(null, r);
				}, callback);
			}
			return callback(null, result);
		}
	} catch(e) {
		if(isError) throw e;
		if(isDone) {
			// loader is already "done", so we cannot use the callback function
			// for better debugging we print the error on the console
			if(typeof e === "object" && e.stack) console.error(e.stack);
			else console.error(e);
			return;
		}
		isDone = true;
		reportedError = true;
		callback(e);
	}
}
```

#### 参考

[webpack 之 LoaderRunner 全方位揭秘](https://juejin.cn/post/6844903858552979470)
[Webpack 源码分析 - loader-runner](https://juejin.cn/post/6844904058591920141)
[github loader-runner](https://github.com/webpack/loader-runner/blob/master/lib/LoaderRunner.js)
