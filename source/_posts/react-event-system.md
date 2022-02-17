---
title: React 合成事件系统
date: 2022-02-16 22:12:01
tags: React
---

![React 合成事件系统](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/46124c3589a1468aac72590d16f4787a~tplv-k3u1fbpfcp-watermark.awebp)

#### JSX 事件绑定 -> Fiber

```bash
class Index extends React.Component{
    handerClick= (value) => console.log(value)
    render(){
        return <div>
            <button onClick={ this.handerClick } > 按钮点击 </button>
        </div>
    }
}
```

经过 `babel` 转换成 `React.createElement` 形式
![React createElement](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/02eb66989a5444839c4e758b795869e7~tplv-k3u1fbpfcp-watermark.awebp)
最终转成 `fiber` 对象形式如下
![React Fiber](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a2bd1a74076c40d1b5c5e7b53c341f7f~tplv-k3u1fbpfcp-watermark.awebp)
`fiber` 对象上的 `memoizedProps` 和 `pendingProps` 保存了我们的事件。

#### 什么是 React 合成事件？

```bash
class Index extends React.Component{
    componentDidMount(){
        console.log(this)
    }
    handerClick= (value) => console.log(value)
    handerChange=(value) => console.log(value)
    render(){
        return <div>
            <button onClick={this.handerClick} > 按钮点击 </button>
            <input placeholder="请输入内容" onChange={this.handerChange}  />
        </div>
    }
}
```

我们先看一下 `input dom` 元素上绑定的事件
![input dom event listeners](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b7781dbc5af7455492f97903bdb2f54b~tplv-k3u1fbpfcp-watermark.awebp)
然后我们看一下 `document` 上绑定的事件
![document event listeners](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2e956b39fff940bb919ac75aa4bd2cc3~tplv-k3u1fbpfcp-watermark.awebp)
我们发现，我们给 `<input>` 绑定的 `onChange`，并没有直接绑定在 `input` 上，而是统一绑定在了 `document` 上，然后我们 `onChange` 被处理成很多事件监听器，比如 `blur` , `change` , `input` , `keydown` , `keyup` 等。

> 综上我们可以得出结论：
> ① 我们在 `jsx` 中绑定的事件(`handerClick`，`handerChange`),根本就没有注册到真实的 `dom` 上。是绑定在 `document` 上统一管理的。
> ② 真实的 `dom` 上的 `click` 事件被单独处理,已经被 `react` 底层替换成空函数。
> ③ 我们在 `react` 绑定的事件,比如 `onChange`，在 `document` 上，可能有多个事件与之对应。
> ④ `react` 并不是一开始，把所有的事件都绑定在 `document` 上，而是采取了一种按需绑定，比如发现了 `onClick` 事件,再去绑定 `document click` 事件。

##### 事件合成的插件机制

###### 必要概念

①namesToPlugins - 事件名 -> 事件模块插件的映射

```bash
const namesToPlugins = {
  SimpleEventPlugin,
  EnterLeaveEventPlugin,
  ChangeEventPlugin,
  SelectEventPlugin,
  BeforeInputEventPlugin,
}
```

②plugins - 注册的所有插件列表,初始化为空

```bash
const plugins = [LegacySimpleEventPlugin, LegacyEnterLeaveEventPlugin, ...];
```

③registrationNameModules - 合成的事件 -> 事件插件的映射

```bash
{
    onBlur: SimpleEventPlugin,
    onClick: SimpleEventPlugin,
    onClickCapture: SimpleEventPlugin,
    onChange: ChangeEventPlugin,
    onChangeCapture: ChangeEventPlugin,
    onMouseEnter: EnterLeaveEventPlugin,
    onMouseLeave: EnterLeaveEventPlugin,
    ...
}
```

④ 事件插件 - 以 `SimpleEventPlugin` 为例

```bash
const SimpleEventPlugin = {
    eventTypes:{
        'click':{ /* 处理点击事件  */
            phasedRegistrationNames:{
                bubbled: 'onClick',       // 对应事件冒泡阶段 - onClick
                captured:'onClickCapture' // 对应事件捕获阶段 - onClickCapture
            },
            dependencies: ['click'], // 事件依赖
            ...
        },
        'blur':{ /* 处理失去焦点事件 */ },
        ...
    }
    extractEvents:function(topLevelType,targetInst,nativeEvent,nativeEventTarget){ /* eventTypes 里面的事件对应的统一事件处理函数，接下来会重点讲到 */ }
}
```

即：事件插件是一个对象，有两个属性，`extractEvents` 作为事件统一处理函数；`eventTypes` 对象保存了原生事件名和对应的配置项 dispatchConfig 的映射关系。

由于 `React v16` 的事件是统一绑定在 `document` 上的，`React` 用独特的事件名称比如 `onClick` 和 `onClickCapture`，来说明我们给绑定的函数到底是在冒泡事件阶段，还是捕获事件阶段执行。

⑤ registrationNameDependencies - 记录合成事件 -> 原生事件的映射关系

比如 `onClick` 和原生事件 `click` 对应关系
比如 `onChange` 对应 `change` , `input` , `keydown` , `keyup` 事件

```bash
{
    onBlur: ['blur'],
    onClick: ['click'],
    onClickCapture: ['click'],
    onChange: ['blur', 'change', 'click', 'focus', 'input', 'keydown', 'keyup', 'selectionchange'],
    onMouseEnter: ['mouseout', 'mouseover'],
    onMouseLeave: ['mouseout', 'mouseover'],
    ...
}
```

###### 事件初始化

对于事件合成，v16.13.1 版本 react 采用了初始化注册方式。

```bash
/* 第一步：注册事件：react-dom/src/client/ReactDOMClientInjection.js  */
injectEventPluginsByName({
    SimpleEventPlugin: SimpleEventPlugin,
    EnterLeaveEventPlugin: EnterLeaveEventPlugin,
    ChangeEventPlugin: ChangeEventPlugin,
    SelectEventPlugin: SelectEventPlugin,
    BeforeInputEventPlugin: BeforeInputEventPlugin,
});


/* 注册事件插件 */
export function injectEventPluginsByName(injectedNamesToPlugins){
     for (const pluginName in injectedNamesToPlugins) {
         namesToPlugins[pluginName] = injectedNamesToPlugins[pluginName]
     }
     recomputePluginOrdering()
}
```

`injectEventPluginsByName` 做的事情很简单，形成上述的 `namesToPlugins`，然后执行 `recomputePluginOrdering`

```bash
const eventPluginOrder = [ 'SimpleEventPlugin' , 'EnterLeaveEventPlugin','ChangeEventPlugin','SelectEventPlugin' , 'BeforeInputEventPlugin' ]

function recomputePluginOrdering(){
    for (const pluginName in namesToPlugins) {
        /* 找到对应的事件处理插件，比如 SimpleEventPlugin  */
        const pluginModule = namesToPlugins[pluginName];
        const pluginIndex = eventPluginOrder.indexOf(pluginName);
        /* 填充 plugins 数组  */
        plugins[pluginIndex] = pluginModule;
        const publishedEvents = pluginModule.eventTypes;
    for (const eventName in publishedEvents) {
       // publishedEvents[eventName] -> eventConfig , pluginModule -> 事件插件 ， eventName -> 事件名称
        publishEventForPlugin(publishedEvents[eventName],pluginModule,eventName,)
    }
    }
}
```

`recomputePluginOrdering` 作用很明确了，形成上面说的那个 `plugins` 数组。然后就是重点的函数 `publishEventForPlugin`

```bash
/*
  dispatchConfig -> 原生事件对应配置项 { phasedRegistrationNames :{  冒泡 捕获  } ，   }
  pluginModule -> 事件插件 比如SimpleEventPlugin
  eventName -> 原生事件名称。
*/
function publishEventForPlugin (dispatchConfig,pluginModule,eventName){
    eventNameDispatchConfigs[eventName] = dispatchConfig;
    /* 事件 */
    const phasedRegistrationNames = dispatchConfig.phasedRegistrationNames;
    if (phasedRegistrationNames) {
    for (const phaseName in phasedRegistrationNames) {
        if (phasedRegistrationNames.hasOwnProperty(phaseName)) {
            // phasedRegistrationName React事件名 比如 onClick / onClickCapture
            const phasedRegistrationName = phasedRegistrationNames[phaseName];
            // 填充形成 registrationNameModules React 合成事件 -> React 处理事件插件映射关系
            registrationNameModules[phasedRegistrationName] = pluginModule;
            // 填充形成 registrationNameDependencies React 合成事件 -> 原生事件 映射关系
            registrationNameDependencies[phasedRegistrationName] = pluginModule.eventTypes[eventName].dependencies;
        }
    }
    return true;
    }
}
```

###### 事件合成总结

初始化事件合成阶段主要做了：形成了上述的几个重要对象，构建初始化 React 合成事件和原生事件的对应关系，合成事件和对应的事件处理插件关系。

##### 事件绑定流程

###### diffProperties 处理 React 合成事件

![react diffProperties](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dec68c8a3d6d47d18aaecd565861cb97~tplv-k3u1fbpfcp-watermark.awebp)

React 在调合子节点后，进入 diff 阶段，如果判断是 HostComponent(dom 元素)类型的 fiber，会用 diff props 函数 diffProperties 单独处理

```bash
// react-dom/src/client/ReactDOMComponent.js
function diffProperties(){
    /* 判断当前的 propKey 是不是 React合成事件 */
    if(registrationNameModules.hasOwnProperty(propKey)){
         /* 这里多个函数简化了，如果是合成事件， 传入成事件名称 onClick ，向document注册事件  */
         legacyListenToEvent(registrationName, document）;
    }
}
```

diffProperties 函数在 diff props 如果发现是合成事件(onClick) 就会调用 legacyListenToEvent 函数。注册事件监听器

###### legacyListenToEvent 注册事件监听器

```bash
//  registrationName -> onClick 事件
//  mountAt -> document or container
function legacyListenToEvent(registrationName，mountAt){
   const dependencies = registrationNameDependencies[registrationName]; // 根据 onClick 获取  onClick 依赖的事件数组 [ 'click' ]。
    for (let i = 0; i < dependencies.length; i++) {
    const dependency = dependencies[i];
    //这个经过多个函数简化，如果是 click 基础事件，会走 legacyTrapBubbledEvent ,而且都是按照冒泡处理
     legacyTrapBubbledEvent(dependency, mountAt);
  }
}
```

> 注意 ⚠️：在 `legacyListenToEvent` 函数中，先找到 `React` 合成事件对应的原生事件集合，比如 `onClick -> ['click']` , `onChange -> [blur , change , input , keydown , keyup]`，然后遍历依赖项的数组，绑定事件，这就解释了，为什么我们在刚开始的 demo 中，只给元素绑定了一个 onChange 事件，结果在 document 上出现很多事件监听器的原因，就是在这个函数上处理的

`legacyTrapBubbledEvent` 就是执行将绑定真正的 dom 事件的函数 legacyTrapBubbledEvent(冒泡处理)。

```bash
function legacyTrapBubbledEvent(topLevelType,element){
   addTrappedEventListener(element,topLevelType,PLUGIN_EVENT_SYSTEM,false)
}
```

> 注意 ⚠️：React 是采用事件绑定，React 对于 click 等基础事件，会默认按照事件 `冒泡阶段` 的事件处理，不过这也不绝对的，比如一些事件的处理，有些特殊的事件，如 `scroll`、`focus`、`blur` 是按照 `事件捕获` 处理的。

```bash
case TOP_SCROLL: {                                // scroll 事件
    legacyTrapCapturedEvent(TOP_SCROLL, mountAt); // legacyTrapCapturedEvent 事件捕获处理。
    break;
}
case TOP_FOCUS: // focus 事件
case TOP_BLUR:  // blur 事件
legacyTrapCapturedEvent(TOP_FOCUS, mountAt);
legacyTrapCapturedEvent(TOP_BLUR, mountAt);
break;
```

###### 绑定 dispatchEvent，进行事件监听

如上述的 scroll 事件，focus 事件 ，blur 事件等，是默认按照事件捕获逻辑处理。接下来就是最重要关键的一步。React 是如何绑定事件到 document？ 事件处理函数函数又是什么？问题都指向了上述的 addTrappedEventListener，让我们来揭开它的面纱。

```bash
/*
  targetContainer -> document
  topLevelType ->  click
  capture = false
*/
function addTrappedEventListener(targetContainer,topLevelType,eventSystemFlags,capture){
   const listener = dispatchEvent.bind(null,topLevelType,eventSystemFlags,targetContainer)
   if(capture){
       // 事件捕获阶段处理函数。
   }else{
       /* TODO: 重要, 这里进行真正的事件绑定。*/
      targetContainer.addEventListener(topLevelType,listener,false) // document.addEventListener('click',listener,false)
   }
}
```

这个函数内容虽然不多，但是却非常重要,首先绑定我们的事件统一处理函数 `dispatchEvent`，绑定几个默认参数，事件类型 `topLevelType` demo 中的 `click` ，还有绑定的容器 `doucment`。然后真正的事件绑定,添加事件监听器 `addEventListener`， 事件绑定阶段完毕。

##### 事件触发流程

###### extractEvents 形成事件对象 event 和 事件处理函数队列

![DOM -> Fiber](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb9df1e3d518405aaac807e9ba2ade89~tplv-k3u1fbpfcp-watermark.awebp)

```bash
const  SimpleEventPlugin = {
    extractEvents:function(topLevelType,targetInst,nativeEvent,nativeEventTarget){
        const dispatchConfig = topLevelEventsToDispatchConfig.get(topLevelType);
        if (!dispatchConfig) {
            return null;
        }
        switch(topLevelType){
            default:
            EventConstructor = SyntheticEvent;
            break;
        }
        /* 产生事件源对象 */
        const event = EventConstructor.getPooled(dispatchConfig,targetInst,nativeEvent,nativeEventTarget)
        const phasedRegistrationNames = event.dispatchConfig.phasedRegistrationNames;
        const dispatchListeners = [];
        const {bubbled, captured} = phasedRegistrationNames; /* onClick / onClickCapture */
        const dispatchInstances = [];
        /* 从事件源开始逐渐向上，查找dom元素类型HostComponent对应的fiber ，收集上面的React合成事件，onClick / onClickCapture  */
         while (instance !== null) {
              const {stateNode, tag} = instance;
              if (tag === HostComponent && stateNode !== null) { /* DOM 元素 */
                   const currentTarget = stateNode;
                   if (captured !== null) { /* 事件捕获 */
                        /* 在事件捕获阶段,真正的事件处理函数 */
                        const captureListener = getListener(instance, captured);
                        if (captureListener != null) {
                        /* 对应发生在事件捕获阶段的处理函数，逻辑是将执行函数unshift添加到队列的最前面 */
                            dispatchListeners.unshift(captureListener);
                            dispatchInstances.unshift(instance);
                            dispatchCurrentTargets.unshift(currentTarget);
                        }
                    }
                    if (bubbled !== null) { /* 事件冒泡 */
                        /* 事件冒泡阶段，真正的事件处理函数，逻辑是将执行函数push到执行队列的最后面 */
                        const bubbleListener = getListener(instance, bubbled);
                        if (bubbleListener != null) {
                            dispatchListeners.push(bubbleListener);
                            dispatchInstances.push(instance);
                            dispatchCurrentTargets.push(currentTarget);
                        }
                    }
              }
              instance = instance.return;
         }
          if (dispatchListeners.length > 0) {
              /* 将函数执行队列，挂到事件对象event上 */
            event._dispatchListeners = dispatchListeners;
            event._dispatchInstances = dispatchInstances;
            event._dispatchCurrentTargets = dispatchCurrentTargets;
         }
        return event
    }
}
```

> 事件插件系统的核心 extractEvents 主要做的事是:
>
> ① 首先形成 React 事件独有的合成事件源对象，这个对象，保存了整个事件的信息。将作为参数传递给真正的事件处理函数(handerClick)。
> ② 然后声明事件执行队列 ，按照冒泡和捕获逻辑，从事件源开始逐渐向上，查找 dom 元素类型 HostComponent 对应的 fiber ，收集上面的 React 合成事件，例如 onClick / onClickCapture ，对于冒泡阶段的事件(onClick)，将 push 到执行队列后面 ， 对于捕获阶段的事件(onClickCapture)，将 unShift 到执行队列的前面。
> ③ 最后将事件执行队列，保存到 React 事件源对象上。等待执行。

> 举个 🌰
>
> ```bash
> render(){
>    return <div onClick={ this.handerClick2 } onClickCapture={ this.handerClick3 }>
>        <button onClick={ this.handerClick }  onClickCapture={ this.handerClick1 }>点击</button>
>    </div>
> }
> ```
>
> ![示例](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/514a83eb13df4dd58ec0ebc1dca1873d~tplv-k3u1fbpfcp-watermark.awebp)

###### 事件触发

React 的事件源对象 `SyntheticEvent`

```bash
function SyntheticEvent( dispatchConfig,targetInst,nativeEvent,nativeEventTarget){
  this.dispatchConfig = dispatchConfig;
  this._targetInst = targetInst;
  this.nativeEvent = nativeEvent;
  this._dispatchListeners = null;
  this._dispatchInstances = null;
  this._dispatchCurrentTargets = null;
  this.isPropagationStopped = () => false; /* 初始化，返回为false  */

}
SyntheticEvent.prototype={
    stopPropagation(){ this.isPropagationStopped = () => true;  }, /* React单独处理，阻止事件冒泡函数 */
    preventDefault(){ },  /* React单独处理，阻止事件捕获函数  */
    ...
}
```

`事件执行队列` 和 `事件源对象` 都形成了，接下来就是最后一步事件触发了。上面大家有没有注意到一个函数 `runEventsInBatch`，所有事件绑定函数，就是在这里触发的

```bash
function runEventsInBatch(){
    const dispatchListeners = event._dispatchListeners;
    const dispatchInstances = event._dispatchInstances;
    if (Array.isArray(dispatchListeners)) {
    for (let i = 0; i < dispatchListeners.length; i++) {
      if (event.isPropagationStopped()) { /* 判断是否已经阻止事件冒泡 */
        break;
      }

      dispatchListeners[i](event)
    }
  }
  /* 执行完函数，置空两字段 */
  event._dispatchListeners = null;
  event._dispatchInstances = null;
}
```

> 注意 ⚠️：`React` 对于 `阻止冒泡`，就是通过 `isPropagationStopped` 判断是否已经阻止事件冒泡。如果我们在事件函数执行队列中，某一会函数中，调用 `e.stopPropagation()` 就会赋值给 `isPropagationStopped = () => true`，当再执行 `e.isPropagationStopped()` 就会返回 `true` ,接下来事件处理函数，就不会执行了。
> 同理，对于 `阻止浏览器默认行为` 也必须使用 `e.preventDefault()` 即
>
> ```bash
> // 以下逻辑不能阻止浏览器默认行为。
> handerClick() {
>    return false
> }
> // 应该改为
> handerClick(e) {
>    e.preventDefault()
> }
> ```

###### 其他概念-事件池

```bash
 handerClick = (e) => {
    console.log(e.target) // button
    setTimeout(()=>{
        console.log(e.target) // null
    },0)
}
```

对于一次点击事件的处理函数，在正常的函数执行上下文中打印 e.target 就指向了 dom 元素，但是在 setTimeout 中打印却是 null，如果这不是 React 事件系统，两次打印的应该是一样的，但是为什么两次打印不一样呢？

> 因为在 React 采取了一个事件池的概念，每次我们用的事件源对象，在事件函数执行之后，可以通过 `releaseTopLevelCallbackBookKeeping` 等方法将事件源对象释放到事件池中，这样的好处每次我们不必再创建事件源对象，可以从事件池中取出一个事件源对象进行复用，在事件处理函数执行完毕后,会释放事件源到事件池中，清空属性，这就是 setTimeout 中打印为什么是 null 的原因了。

#### 关于 React v17 版本的事件系统

React v17 整体改动不是很大，但是事件系统的改动却不小，首先上述的很多执行函数，在 v17 版本不复存在了

##### 事件代理不再绑定至 document，而是 root

事件统一绑定 `container` 上，`ReactDOM.render(app， container)`;而不是 `document` 上，这样好处是有利于 `微前端` 的，`微前端` 一个前端系统中可能有多个应用，如果继续采取全部绑定在 `document` 上，那么可能多应用下会出现问题。

![react event v16 vs v17](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/83f4440adffa41b7a82cdb97e7951168~tplv-k3u1fbpfcp-watermark.awebp)

##### 对齐原生浏览器事件

React 17 中终于支持了 `原生捕获事件` 的支持， 对齐了浏览器原生标准。同时 `onScroll` 事件不再进行事件冒泡(解决 v16 问题：当滚动子元素时，父元素上的 onScroll 回调会触发)。`onFocus` 和 `onBlur` 使用原生 `focusin`， `focusout` 合成。

##### 取消事件池

React 17 取消事件池复用，也就解决了上述在 setTimeout 打印，找不到 e.target 的问题。

#### 参考

[「react 进阶」一文吃透 react 事件系统原理](https://juejin.cn/post/6955636911214067720)
[React v17.0 的 6 大变化](https://juejin.cn/post/6862660262995066894)
