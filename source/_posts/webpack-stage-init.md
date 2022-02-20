---
title: Webpack 初始化阶段
date: 2022-02-19 22:52:44
tags: Webpack
---

#### Webpack 核心之初始化

> `Webpack` 核心功能官方解释
> `At its core, webpack is a static module bundler for modern JavaScript applications.`
> 将各种类型的资源，包括`图片`、`css`、`js` 等，转译、组合、拼接、生成 `JS` 格式的 `bundler` 文件

![webpack 初始化](https://pic1.zhimg.com/v2-c2fef8d21ef8785dda99c3360052e200_r.jpg)

##### Webpack 初始化核心代码

> Webpack 版本为 `v4.46.0`

```bash
const webpack = (options, callback) => {
	const webpackOptionsValidationErrors = validateSchema(
		webpackOptionsSchema,
		options
	);
	if (webpackOptionsValidationErrors.length) {
		throw new WebpackOptionsValidationError(webpackOptionsValidationErrors);
	}
	let compiler;
	if (Array.isArray(options)) {
		compiler = new MultiCompiler(
			Array.from(options).map(options => webpack(options))
		);
	} else if (typeof options === "object") {
		options = new WebpackOptionsDefaulter().process(options);

		compiler = new Compiler(options.context);
		compiler.options = options;
		new NodeEnvironmentPlugin({
			infrastructureLogging: options.infrastructureLogging
		}).apply(compiler);
		if (options.plugins && Array.isArray(options.plugins)) {
			for (const plugin of options.plugins) {
				if (typeof plugin === "function") {
					plugin.call(compiler, compiler);
				} else {
					plugin.apply(compiler);
				}
			}
		}
		compiler.hooks.environment.call();
		compiler.hooks.afterEnvironment.call();
		compiler.options = new WebpackOptionsApply().process(options, compiler);
	} else {
		throw new Error("Invalid argument: options");
	}
	if (callback) {
		if (typeof callback !== "function") {
			throw new Error("Invalid argument: callback");
		}
		if (
			options.watch === true ||
			(Array.isArray(options) && options.some(o => o.watch))
		) {
			const watchOptions = Array.isArray(options)
				? options.map(o => o.watchOptions || {})
				: options.watchOptions || {};
			return compiler.watch(watchOptions, callback);
		}
		compiler.run(callback);
	}
	return compiler;
};
```

##### 初始化参数

> 1. 从 `配置文件`、 `配置对象`、`Shell参数` 中读取，与 `默认配置` 结合得出最终的参数
> 2. 对应的核心代码为 `options = new WebpackOptionsDefaulter().process(options)` 且 `class WebpackOptionsDefaulter extends OptionsDefaulter {}`
> 3. 根据不同的合并规则进行 `options` 和 `默认配置` 的合并处理

```bash
class OptionsDefaulter {
	process(options) {
		options = Object.assign({}, options);
		for (let name in this.defaults) {
			switch (this.config[name]) {
				/**
				 * If {@link ConfigType} doesn't specified and current value is `undefined`, then default value will be assigned
				 */
				case undefined:
					if (getProperty(options, name) === undefined) {
						setProperty(options, name, this.defaults[name]);
					}
					break;
				/**
				 * Assign result of {@link CallConfigHandler}
				 */
				case "call":
					setProperty(
						options,
						name,
						this.defaults[name].call(this, getProperty(options, name), options)
					);
					break;
				/**
				 * Assign result of {@link MakeConfigHandler}, if current value is `undefined`
				 */
				case "make":
					if (getProperty(options, name) === undefined) {
						setProperty(options, name, this.defaults[name].call(this, options));
					}
					break;
				/**
				 * Adding {@link AppendConfigValues} at the end of the current array
				 */
				case "append": {
					let oldValue = getProperty(options, name);
					if (!Array.isArray(oldValue)) {
						oldValue = [];
					}
					oldValue.push(...this.defaults[name]);
					setProperty(options, name, oldValue);
					break;
				}
				default:
					throw new Error(
						"OptionsDefaulter cannot process " + this.config[name]
					);
			}
		}
		return options;
	}
}
```

##### 创建编译器对象

> 1. 通过上一步得到的 `options.context` 参数创建 `Compiler` 对象
> 2. 对应的核心代码为 `compiler = new Compiler(options.context);`

##### 初始化编译环境

> 1. 包括注入内置插件、注册各种模块工厂、初始化 RuleSet 集合、加载配置的插件等
> 2. 对应的核心代码为 `plugin.apply(compiler)` 和 `compiler.options = new WebpackOptionsApply().process(options, compiler);`

##### 开始编译

> 1. 执行 compiler 对象的 run 方法
> 2. 对应的核心代码为 `compiler.run(callback);`

```bash
run(callback) {
  if (this.running) return callback(new ConcurrentCompilationError());

  const finalCallback = (err, stats) => {
    this.running = false;

    if (err) {
      this.hooks.failed.call(err);
    }

    if (callback !== undefined) return callback(err, stats);
  };

  const startTime = Date.now();

  this.running = true;

  const onCompiled = (err, compilation) => {
    if (err) return finalCallback(err);

    if (this.hooks.shouldEmit.call(compilation) === false) {
      const stats = new Stats(compilation);
      stats.startTime = startTime;
      stats.endTime = Date.now();
      this.hooks.done.callAsync(stats, err => {
        if (err) return finalCallback(err);
        return finalCallback(null, stats);
      });
      return;
    }

    this.emitAssets(compilation, err => {
      if (err) return finalCallback(err);

      if (compilation.hooks.needAdditionalPass.call()) {
        compilation.needAdditionalPass = true;

        const stats = new Stats(compilation);
        stats.startTime = startTime;
        stats.endTime = Date.now();
        this.hooks.done.callAsync(stats, err => {
          if (err) return finalCallback(err);

          this.hooks.additionalPass.callAsync(err => {
            if (err) return finalCallback(err);
            this.compile(onCompiled);
          });
        });
        return;
      }

      this.emitRecords(err => {
        if (err) return finalCallback(err);

        const stats = new Stats(compilation);
        stats.startTime = startTime;
        stats.endTime = Date.now();
        this.hooks.done.callAsync(stats, err => {
          if (err) return finalCallback(err);
          return finalCallback(null, stats);
        });
      });
    });
  };

  this.hooks.beforeRun.callAsync(this, err => {
    if (err) return finalCallback(err);

    this.hooks.run.callAsync(this, err => {
      if (err) return finalCallback(err);

      this.readRecords(err => {
        if (err) return finalCallback(err);

        this.compile(onCompiled);
      });
    });
  });
}
```

> `this.compile(onCompiled)` 是整个编译过程启动的入口
> 一个 `compilation` 对象表现了当前的模块资源、编译生成资源、变化的文件、以及被跟踪依赖的状态信息，代表了一次资源的构建。

```bash
compile(callback) {
  const params = this.newCompilationParams();
  this.hooks.beforeCompile.callAsync(params, err => {
    if (err) return callback(err);

    this.hooks.compile.call(params);

    const compilation = this.newCompilation(params);

    this.hooks.make.callAsync(compilation, err => {
      if (err) return callback(err);

      compilation.finish(err => {
        if (err) return callback(err);

        compilation.seal(err => {
          if (err) return callback(err);

          this.hooks.afterCompile.callAsync(compilation, err => {
            if (err) return callback(err);

            return callback(null, compilation);
          });
        });
      });
    });
  });
}

newCompilationParams() {
  const params = {
    normalModuleFactory: this.createNormalModuleFactory(),
    contextModuleFactory: this.createContextModuleFactory(),
    compilationDependencies: new Set()
  };
  return params;
}

newCompilation(params) {
  const compilation = this.createCompilation();
  compilation.fileTimestamps = this.fileTimestamps;
  compilation.contextTimestamps = this.contextTimestamps;
  compilation.name = this.name;
  compilation.records = this.records;
  compilation.compilationDependencies = params.compilationDependencies;
  this.hooks.thisCompilation.call(compilation, params);
  this.hooks.compilation.call(compilation, params);
  return compilation;
}
```

##### 总结

> 1. 将 `process.args + webpack.config.js` 合并成用户配置
> 2. 调用 `validateSchema` 校验配置
> 3. 调用 `new WebpackOptionsDefaulter().process(options)` 合并出最终配置
> 4. 创建 `compiler` 对象
> 5. 遍历用户定义的 `plugins` 集合，执行插件的 `apply` 方法
> 6. 调用 `new WebpackOptionsApply().process(options, compiler)` 方法，加载各种内置插件
>    > 主要逻辑集中在 `WebpackOptionsApply` 类，`webpack` 内置了数百个插件，这些插件并不需要我们手动配置，`WebpackOptionsApply` 会在初始化阶段根据配置内容动态注入对应的插件，比如包括：
>    >
>    > - 注入 `EntryOptionPlugin` 插件，处理 `entry` 配置
>    > - 根据 `devtool` 值判断后续用那个插件处理 `sourcemap`，可选值：`EvalSourceMapDevToolPlugin`、`SourceMapDevToolPlugin`、`EvalDevToolModulePlugin`
>    > - 根据 `optimization.${key}` 配置（`${key}` - `removeAvailableModules`、`removeEmptyChunks`、`mergeDuplicateChunks`、`flagIncludedChunks`、`sideEffects`、`providedExports`、`usedExports`、`concatenateModules`、`splitChunks`、`runtimeChunk`、`noEmitOnErrors`、`checkWasmTypes`）自动加载对应的插件
>    > - 注入 `RuntimePlugin` ，用于根据代码内容动态注入 `webpack` 运行时
> 7. `compiler.run` 实际调用 `compiler.compile` 函数，实际调用 `this.hooks.make.callAsync`，从而触发 `EntryPlugin` 的 `compiler.hooks.make.tapAsync` 回调，在回调中执行 `compilation.addEntry` 函数；`compilation.addEntry` 函数内部经过一坨与主流程无关的 `hook` 之后，再调用 `handleModuleCreate` 函数，正式开始构建内容

#### 参考

[[万字总结] 一文吃透 Webpack 核心原理](https://zhuanlan.zhihu.com/p/363928061)
[[源码解读] Webpack 插件架构深度讲解](https://zhuanlan.zhihu.com/p/367931462)
