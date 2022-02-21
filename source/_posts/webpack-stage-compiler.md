---
title: Webpack 构建编译阶段
date: 2022-02-21 13:12:08
tags: Webpack
---

#### Webpack 核心之构建编译

调用 `compiler.run` 方法来启动构建

```bash
run(callback) {
    // 编译结束回调函数
    const onCompiled = (err, compilation) => {
    	this.hooks.done.callAsync(stats, err => {
    		return finalCallback(null, stats);
    	});
    };

    // 执行订阅了 compiler.beforeRun 钩子插件的回调
    this.hooks.beforeRun.callAsync(this, err => {
        // 执行订阅了 compiler.run 钩子插件的回调
    	this.hooks.run.callAsync(this, err => {
    		this.compile(onCompiled);
    	});
    });
}
```

`compiler.compile` 开始真正执行我们的构建流程，在 `compile` 阶段，`Compiler` 对象会开始实例化两个核心的工厂对象，分别是 `NormalModuleFactory` 和 `ContextModuleFactory`。工厂对象顾名思义就是用来创建实例的，它们后续用来创建 `module` 实例的，包括 `NormalModule` 以及 `ContextModule` 实例。

```bash
compile(callback) {
    // 实例化核心工厂对象
    const params = this.newCompilationParams();
    // 执行订阅了 compiler.beforeCompile 钩子插件的回调
    this.hooks.beforeCompile.callAsync(params, err => {
        // 执行订阅了 compiler.compile 钩子插件的回调
        this.hooks.compile.call(params);
        // 创建此次编译的 `Compilation` 对象
        const compilation = this.newCompilation(params);

        // 执行订阅了 compiler.make 钩子插件的回调
        this.hooks.make.callAsync(compilation, err => {

            compilation.finish(err => {
                compilation.seal(err => {
                    this.hooks.afterCompile.callAsync(compilation, err => {
                		return callback(null, compilation);
                	});
                })
            })
        })
    })
}
```

在讲 `this.hooks.make.callAsync` 之前不得不提，`make hook` 实际是在 `SingleEntryPlugin` 进行注册的。

#### 编译构建阶段

##### SingleEntryPlugin -> compilation.addEntry

在 `lib/WebpackOptionsApply.js` 中注册 `EntryOptionPlugin`，并立即执行 `compiler.hooks.entryOption.call()`

```bash
new EntryOptionPlugin().apply(compiler);
compiler.hooks.entryOption.call(options.context, options.entry);
```

在 `lib/EntryOptionPlugin.js` 中注册了 `entryOption hook`，无论是 `SingleEntryPlugin`、`MultiEntryPlugin` 还是 `DynamicEntryPlugin` 均是注册了 `make hook`。

```bash
const itemToPlugin = (context, item, name) => {
	if (Array.isArray(item)) {
		return new MultiEntryPlugin(context, item, name);
	}
	return new SingleEntryPlugin(context, item, name);
};

module.exports = class EntryOptionPlugin {
	apply(compiler) {
		compiler.hooks.entryOption.tap("EntryOptionPlugin", (context, entry) => {
			if (typeof entry === "string" || Array.isArray(entry)) {
				itemToPlugin(context, entry, "main").apply(compiler);
			} else if (typeof entry === "object") {
				for (const name of Object.keys(entry)) {
					itemToPlugin(context, entry[name], name).apply(compiler);
				}
			} else if (typeof entry === "function") {
				new DynamicEntryPlugin(context, entry).apply(compiler);
			}
			return true;
		});
	}
};
```

以 `SingleEntryPlugin` 为例，`make hook` 注册于此，即当 `compiler.compile` 中执行 `this.hooks.make.callAsync`，实际会触发执行 `compilation.addEntry`，即真正的编译于此处开启。

```bash
class SingleEntryPlugin {
	/**
	 * An entry plugin which will handle
	 * creation of the SingleEntryDependency
	 *
	 * @param {string} context context path
	 * @param {string} entry entry path
	 * @param {string} name entry key name
	 */
	constructor(context, entry, name) {
		this.context = context;
		this.entry = entry;
		this.name = name;
	}

	/**
	 * @param {Compiler} compiler the compiler instance
	 * @returns {void}
	 */
	apply(compiler) {
		compiler.hooks.compilation.tap(
			"SingleEntryPlugin",
			(compilation, { normalModuleFactory }) => {
				compilation.dependencyFactories.set(
					SingleEntryDependency,
					normalModuleFactory
				);
			}
		);

		compiler.hooks.make.tapAsync(
			"SingleEntryPlugin",
			(compilation, callback) => {
				const { entry, name, context } = this;

				const dep = SingleEntryPlugin.createDependency(entry, name);
				compilation.addEntry(context, dep, name, callback);
			}
		);
	}

	/**
	 * @param {string} entry entry request
	 * @param {string} name entry name
	 * @returns {SingleEntryDependency} the dependency
	 */
	static createDependency(entry, name) {
		const dep = new SingleEntryDependency(entry);
		dep.loc = { name };
		return dep;
	}
}
```

##### compilation.addEntry -> \_addModuleChain

`_addModuleChain` 中接收参数 `dependency` 传入的入口依赖，使用对应的工厂函数 `NormalModuleFactory.create` 方法生成一个空的 `module` 对象，回调中会把此 `module` 存入 `compilation.modules` 对象和 `dependencies.module` 对象中，由于是入口文件，也会存入 `compilation.entries` 中。随后执行 `buildModule` 进入真正的构建 `module` 内容的过程。

```bash
_addModuleChain(context, dependency, onModule, callback) {
    // ...

    // 根据依赖查找对应的工厂函数
    const Dep = /** @type {DepConstructor} */ (dependency.constructor);
    const moduleFactory = this.dependencyFactories.get(Dep);

    this.semaphore.acquire(() => {
        moduleFactory.create(
            {
                dependencies: [dependency]
                ...
            },
            (err, module) => {
                // ...

                const afterBuild = () => {
                    if (addModuleResult.dependencies) {
                        this.processModuleDependencies(module, err => {
                            if (err) return callback(err);
                            callback(null, module);
                        });
                    } else {
                        return callback(null, module);
                    }
                };


                if (addModuleResult.build) {
                    this.buildModule(module, false, null, null, err => {
                        if (err) {
                            this.semaphore.release();
                            return errorAndCallback(err);
                        }

                        if (currentProfile) {
                            const afterBuilding = Date.now();
                            currentProfile.building = afterBuilding - afterFactory;
                        }

                        this.semaphore.release();
                        afterBuild();
                    });
                }
            }
        );
    });
}
```

##### \_addModuleChain -> runLoaders

`buildModule` 方法主要执行 `module.build()`，对应的是 `NormalModule.build()`，实际调用 `doBuild`

```bash
// NormalModule.js
build(options, compilation, resolver, fs, callback) {
    return this.doBuild(options, compilation, resolver, fs, err => {
        ...
    }
}
```

一句话说，`doBuild` 调用了相应的 `loaders` ，把我们的模块转成标准的 JS 模块。这里，使用 `babel-loader` 来编译 `index.js` ，`source` 就是 `babel-loader` 编译后的代码。`runLoaders` 实际为 `loader-runner` 模块导出函数。

```bash
doBuild(options, compilation, resolver, fs, callback) {
    const loaderContext = this.createLoaderContext(
        resolver,
        options,
        compilation,
        fs
    );

    runLoaders(
        {
            resource: this.resource,
            loaders: this.loaders,
            context: loaderContext,
            readResource: fs.readFile.bind(fs)
        },
        (err, result) => {
            if (result) {
                this.buildInfo.cacheable = result.cacheable;
                this.buildInfo.fileDependencies = new Set(result.fileDependencies);
                this.buildInfo.contextDependencies = new Set(
                    result.contextDependencies
                );
            }

            if (err) {
                if (!(err instanceof Error)) {
                    err = new NonErrorEmittedError(err);
                }
                const currentLoader = this.getCurrentLoader(loaderContext);
                const error = new ModuleBuildError(this, err, {
                    from:
                        currentLoader &&
                        compilation.runtimeTemplate.requestShortener.shorten(
                            currentLoader.loader
                        )
                });
                return callback(error);
            }

            const resourceBuffer = result.resourceBuffer;
            const source = result.result[0];
            const sourceMap = result.result.length >= 1 ? result.result[1] : null;
            const extraInfo = result.result.length >= 2 ? result.result[2] : null;

            if (!Buffer.isBuffer(source) && typeof source !== "string") {
                const currentLoader = this.getCurrentLoader(loaderContext, 0);
                const err = new Error(
                    `Final loader (${
                        currentLoader
                            ? compilation.runtimeTemplate.requestShortener.shorten(
                                    currentLoader.loader
                                )
                            : "unknown"
                    }) didn't return a Buffer or String`
                );
                const error = new ModuleBuildError(this, err);
                return callback(error);
            }

            this._source = this.createSource(
                this.binary ? asBuffer(source) : asString(source),
                resourceBuffer,
                sourceMap
            );
            this._sourceSize = null;
            this._ast =
                typeof extraInfo === "object" &&
                extraInfo !== null &&
                extraInfo.webpackAST !== undefined
                    ? extraInfo.webpackAST
                    : null;
            return callback();
        }
    );
}
```

经过 `doBuild` 之后，我们的任何模块都被转成了标准的 JS 模块

![webpack 构建](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/11/15/16e6e1282cb38143~tplv-t2oaga2asx-watermark.awebp)

#### 参考

[webpack 构建流程分析](https://juejin.cn/post/6844904000169607175)
