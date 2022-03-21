---
title: React Scheduler 实现原理
date: 2022-03-21 23:17:17
tags: React
---

#### Scheduler 时间分片

如果「组件 Render 过程耗时」或「参与调和阶段的虚拟 DOM 节点很多」时，那么一次性完成所有组件的调和阶段就会花费较长时间。
为了避免长时间执行调和阶段而引起页面卡顿，React 团队提出了 Fiber 架构和 Scheduler 任务调度。
Fiber 架构的目的是「能独立执行每个虚拟 DOM 的调和阶段」，而不是每次执行整个虚拟 DOM 树的调和阶段。
Scheduler 的主要功能是时间分片，每隔一段时间就把主线程还给浏览器，避免长时间占用主线程。

#### React 与 Scheduler 交互

如果只考虑 React 和 Scheduler 的交互，则组件更新的流程如下：

1. React 组件状态更新，向 Scheduler 中存入一个任务，该任务为 React 更新算法。
2. Scheduler 调度该任务，执行 React 更新算法。
3. React 在调和阶段更新一个 Fiber 之后，会询问 Scheduler 是否需要暂停。如果不需要暂停，则重复步骤 3，继续更新下一个 Fiber。
4. 如果 Scheduler 表示需要暂停，则 React 将返回一个函数，该函数用于告诉 Scheduler 任务还没有完成。Scheduler 将在未来某时刻调度该任务。

#### 为何使用 MessageChannel 实现

```bash
# DOM and Worker environments.
# We prefer MessageChannel because of the 4ms setTimeout clamping.
let schedulePerformWorkUntilDeadline;
const channel = new MessageChannel();
const port = channel.port2;
# 每当调用 schedulePerformWorkUntilDeadline 时，则会执行 performWorkUntilDeadline
channel.port1.onmessage = performWorkUntilDeadline;
schedulePerformWorkUntilDeadline = () => {
  port.postMessage(null);
};
```

> ##### _❌setTimeout(fn, 0) 是我们最常用的创建宏任务的手段，为什么 React 没选择用它实现 Scheduler 呢？_
>
> 原因是递归执行 `setTimeout(fn, 0)` 时，最后间隔时间会变成 4 毫秒，而不是最初的 1 毫秒。
> 其实在源码中已经备注了选择 `MessageChannel` 的原因是：`We prefer MessageChannel because of the 4ms setTimeout clamping.`
> 可在浏览器中执行以下代码：
>
> ```bash
> var count = 0
> var startVal = +new Date()
> console.log("start time", 0, 0)
> function func() {
>  setTimeout(() => {
>    console.log("exec time", ++count, +new Date() - startVal)
>    if (count === 50) {
>      return
>    }
>    func()
>  }, 0)
> }
> func()
> ```
>
> ![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c550a01a07374e4082c35884412be5b9~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)
>
> 如果使用 setTimeout(fn, 0) 实现 Scheduler，就会浪费 4 毫秒。因为 60 FPS 要求每帧间隔不超过 16.66 ms，所以 4ms 是不容忽视的浪费。
>
> ##### _❌ 为什么不选择 requestAnimationFrame(fn)？_
>
> `window.requestAnimationFrame() 告诉浏览器——你希望执行一个动画，并且要求浏览器在下次重绘之前调用指定的回调函数更新动画。该方法需要传入一个回调函数作为参数，该回调函数会在浏览器下一次重绘之前执行`
> 我们知道 rAF() 是在页面更新之前被调用。
>
> 如果第一次任务调度不是由 rAF() 触发的，例如直接执行 scheduler.scheduleTask()，那么在本次页面更新前会执行一次 rAF() 回调，该回调就是第二次任务调度。所以使用 rAF() 实现会导致在本次页面更新前执行了两次任务。
> 为什么是两次，而不是三次、四次？因为在 rAF() 的回调中再次调用 rAF()，会将第二次 rAF() 的回调放到下一帧前执行，而不是在当前帧前执行。
> 另一个原因是 rAF() 的触发间隔时间不确定，如果浏览器间隔了 10ms 才更新页面，那么这 10ms 就浪费了。

#### Scheduler 任务优先级

根据任务的优先级 `priorityLevel` 设置不同的 `timeout` ，并以 `expirationTime = startTime + timeout` 作为 `task.sortIndex`。通过 `push(taskQueue, newTask)` 执行 `最小堆排序`，按照 `sortIndex` 进行排序优先(`sortIndex 越小，优先级越高`)，堆顶即为优先级最高的 `task`;

```bash
# Max 31 bit integer. The max integer size in V8 for 32-bit systems.
# Math.pow(2, 30) - 1
# 0b111111111111111111111111111111
var maxSigned31BitInt = 1073741823;
# Times out immediately
var IMMEDIATE_PRIORITY_TIMEOUT = -1;
# Eventually times out
var USER_BLOCKING_PRIORITY_TIMEOUT = 250;
var NORMAL_PRIORITY_TIMEOUT = 5000;
var LOW_PRIORITY_TIMEOUT = 10000;
# Never times out
var IDLE_PRIORITY_TIMEOUT = maxSigned31BitInt;

function unstable_scheduleCallback(priorityLevel, callback, options) {
  var currentTime = getCurrentTime();

  var startTime;
  if (typeof options === 'object' && options !== null) {
    var delay = options.delay;
    if (typeof delay === 'number' && delay > 0) {
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
  } else {
    startTime = currentTime;
  }

  var timeout;
  switch (priorityLevel) {
    case ImmediatePriority:
      timeout = IMMEDIATE_PRIORITY_TIMEOUT;
      break;
    case UserBlockingPriority:
      timeout = USER_BLOCKING_PRIORITY_TIMEOUT;
      break;
    case IdlePriority:
      timeout = IDLE_PRIORITY_TIMEOUT;
      break;
    case LowPriority:
      timeout = LOW_PRIORITY_TIMEOUT;
      break;
    case NormalPriority:
    default:
      timeout = NORMAL_PRIORITY_TIMEOUT;
      break;
  }

  var expirationTime = startTime + timeout;

  var newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    sortIndex: -1,
  };
  if (enableProfiling) {
    newTask.isQueued = false;
  }

  if (startTime > currentTime) {
    # This is a delayed task.
    newTask.sortIndex = startTime;
    push(timerQueue, newTask);
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      # All tasks are delayed, and this is the task with the earliest delay.
      if (isHostTimeoutScheduled) {
        # Cancel an existing timeout.
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      # Schedule a timeout.
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    newTask.sortIndex = expirationTime;
    push(taskQueue, newTask);
    if (enableProfiling) {
      markTaskStart(newTask, currentTime);
      newTask.isQueued = true;
    }
    # Schedule a host callback, if needed. If we're already performing work,
    # wait until the next time we yield.
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    }
  }

  return newTask;
}
```

`requestHostCallback(flushWork)` 执行即触发调度任务，实际通过执行 `schedulePerformWorkUntilDeadline()` 触发 `port.postMessage(null)`，从而触发 `performWorkUntilDeadline` 执行。
此时通过 `scheduledHostCallback(hasTimeRemaining, currentTime)` 实际是触发 `flushWork(hasTimeRemaining, currentTime) -> workLoop(hasTimeRemaining, initialTime)`

```bash
function requestHostCallback(callback) {
  scheduledHostCallback = callback;
  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true;
    schedulePerformWorkUntilDeadline();
  }
}
```

```bash
const performWorkUntilDeadline = () => {
  if (scheduledHostCallback !== null) {
    const currentTime = getCurrentTime();
    # Keep track of the start time so we can measure how long the main thread
    # has been blocked.
    startTime = currentTime;
    const hasTimeRemaining = true;

    # If a scheduler task throws, exit the current browser task so the
    # error can be observed.
    #
    # Intentionally not using a try-catch, since that makes some debugging
    # techniques harder. Instead, if `scheduledHostCallback` errors, then
    # `hasMoreWork` will remain true, and we'll continue the work loop.
    let hasMoreWork = true;
    try {
      hasMoreWork = scheduledHostCallback(hasTimeRemaining, currentTime);
    } finally {
      if (hasMoreWork) {
        # If there's more work, schedule the next message event at the end
        # of the preceding one.
        schedulePerformWorkUntilDeadline();
      } else {
        isMessageLoopRunning = false;
        scheduledHostCallback = null;
      }
    }
  } else {
    isMessageLoopRunning = false;
  }
  # Yielding to the browser will give it a chance to paint, so we can
  # reset this.
  needsPaint = false;
};
```

#### SchedulerMinHeap 源码实现

```bash
type Heap = Array<Node>;
type Node = {|
  id: number,
  sortIndex: number,
|};

export function push(heap: Heap, node: Node): void {
  const index = heap.length;
  heap.push(node);
  siftUp(heap, node, index);
}

export function peek(heap: Heap): Node | null {
  return heap.length === 0 ? null : heap[0];
}

export function pop(heap: Heap): Node | null {
  if (heap.length === 0) {
    return null;
  }
  const first = heap[0];
  const last = heap.pop();
  if (last !== first) {
    heap[0] = last;
    siftDown(heap, last, 0);
  }
  return first;
}

function siftUp(heap, node, i) {
  let index = i;
  while (index > 0) {
    const parentIndex = (index - 1) >>> 1;
    const parent = heap[parentIndex];
    if (compare(parent, node) > 0) {
      # The parent is larger. Swap positions.
      heap[parentIndex] = node;
      heap[index] = parent;
      index = parentIndex;
    } else {
      # The parent is smaller. Exit.
      return;
    }
  }
}

function siftDown(heap, node, i) {
  let index = i;
  const length = heap.length;
  const halfLength = length >>> 1;
  while (index < halfLength) {
    const leftIndex = (index + 1) * 2 - 1;
    const left = heap[leftIndex];
    const rightIndex = leftIndex + 1;
    const right = heap[rightIndex];

    # If the left or right node is smaller, swap with the smaller of those.
    if (compare(left, node) < 0) {
      if (rightIndex < length && compare(right, left) < 0) {
        heap[index] = right;
        heap[rightIndex] = node;
        index = rightIndex;
      } else {
        heap[index] = left;
        heap[leftIndex] = node;
        index = leftIndex;
      }
    } else if (rightIndex < length && compare(right, node) < 0) {
      heap[index] = right;
      heap[rightIndex] = node;
      index = rightIndex;
    } else {
      # Neither child is smaller. Exit.
      return;
    }
  }
}

function compare(a, b) {
  # Compare sort index first, then task id.
  const diff = a.sortIndex - b.sortIndex;
  return diff !== 0 ? diff : a.id - b.id;
}
```

#### 参考

[React Scheduler 为什么使用 MessageChannel 实现](https://juejin.cn/post/6953804914715803678)
[手写简易版 React 来彻底搞懂 fiber 架构](https://juejin.cn/post/7063321486135656479)
[React 源码笔记 -requestIdleCallback 是什么](https://juejin.cn/post/7041457381938561055)
[图解 React 源码 - React 中的优先级管理](https://juejin.cn/post/6993139933573546021)
[2022 年了,真的懂 requestAnimationFrame 么？](https://juejin.cn/post/7062178363800027173)
