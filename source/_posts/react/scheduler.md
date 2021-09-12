---
title: 一篇关于scheduler的长长文
date: 2021-09-11 22:47:26
categories: react源码解析
tags:
  - react
  - scheduler
---

序：
  本来打算想新建个react的blog页面 把这些文章迁移到自己的个人网站上面，忽然一想 我这么没有实力的开发 搞这些乱七八糟的干什么 本来文章都没有几篇，迁移还浪费时间 不如直接在这里继续吧， 加油 家教人！！

<!--more-->

# Scheduler源码解析

前段时间我在公司的分享就是关于 react scheduler 调度相关的主题 可能因为时间不够或者我自己的语言整理不是很好 个人感觉 效果不是很好 所以 我把之前整理的和现在理解的全部整理在这里， 就当时一个新的分享文章去做好了。


## Scheduler 是什么？
最近这段时间 react18好像也出来了， 我们在使用react17版本的时候 最多使用的就是关于 `hooks` 相关的语法， react官方也说过 17 算是一个过渡版本，对于 `concurrent`模式的使用我们基本上很少涉及到 现在大部分的react项目还是使用老的 `ReactDOM.render` 来进行构建。


当我们使用 `ReactDOM.render` 来构建项目时， 我们做初始化`performance`的时候 会看到如下的图片
<img src="https://liuxing-oss-1256582279.cos.ap-nanjing.myqcloud.com/share%2F%E5%90%8C%E6%AD%A5performance%E5%88%86%E6%9E%90.png" />

我们可以看到 整个 `reconcile`和  `render`的过程在 一个 `Task`中 并且因为渲染了 3000个节点的原因 这个`Task`是标红的 这种将 协调 和 渲染 塞到同一个`Task` 的行为就成为 同步渲染模式。

在解释`concurrent`模式之前 我们先了解 `concurrent` 的中文翻译是什么
<img src="https://liuxing-oss-1256582279.cos.ap-nanjing.myqcloud.com/share%2Fconcurrent%E7%BF%BB%E8%AF%91.png" />

这其实是一个并发的任务 通过任务切片 限制一个任务的执行时间, 实现异步可中断更新。 
而这些功能的实现 就需要 `scheduler` 调度器 来完成。

我们看下 当我们启用 `concurrent` 模式的时候 首次 `render` 的 performance 图 是什么样子的
<img src="https://liuxing-oss-1256582279.cos.ap-nanjing.myqcloud.com/share%2Fconcurrent%E7%9A%84performance.%E5%88%86%E6%9E%90png.png">


可以看到 首次 render 分成了多个任务 这就是我们上面说的时间切片 将每一个任务 大概切成了 5ms 减少阻塞问题 并且 5ms执行之后 再查询是否有更高优先级的任务塞进来 如何有的话 打算当前的 reconciler 过程 执行高优先级的 reconciler 这就是 优先级调度执行

## Scheduler 的功能
其实 在我们上面的内容能够看出来 `Scheduler` 主要包含下面两个主要功能
  - 时间切片
  - 优先级调度

下面一个个来介绍这些调度的功能实现

### 时间切片功能

在 源码中 `React`选择实现时间切片的功能实现通过以下两个api 一个是 `setTimeout`, 另一个是 `MessageChannel` 他们俩个都是属于 宏任务的执行 只不过 `MessageChannel` 的执行时间会比 `setTimeout` 早一些。

#### 问题一 那么在什么时间进行时间切片呢？？

在上面开始concurrent的performance 中， 我们继续放大其实可以看出来，所谓的异步可中断更新 其实都是 `reconciler` 的 中断（react 三大阶段 Scheduler Reconciler Commit），由此 不难猜出 其实所有时间切片的判断都是在 `reconciler`阶段 而 `commit`阶段是 **同步的** 也就不存在时间切片。

我们从 `reconciler` 入口函数 `performConcurrentWorkOnRoot` 开始找 执行顺序如下
```
performConcurrentWorkOnRoot -> renderRootConcurrent -> workLoopConcurrent
```
在 `workLoopConcurrent` 函数下 首次执行的就是 关于 时间切片的判断 以及 是否跳出的逻辑.

代码如下所示
```javascript
function workLoopConcurrent() {
  // Perform work until Scheduler asks us to yield
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

function workLoopSync() {
  // Already timed out, so perform work without checking if we need to yield.
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}
```

其中 `shouldYield` 就是 `concurrent`模式下才会进行的判断。

接下来 在对 `shouldYield` 进行溯源时 我们发现 这个函数是由 `SchedulerWithReactIntegration` 提供 这个文件就是类似于 是 `React` 与 `Scheduler` 的桥梁 负责翻译工作。

最终 最终溯源到 `shouldYield` 在 Scheduler中定义的方法名为 `shouldYieldToHost` 而 `shouldYieldToHost`的具体实现是根据浏览器是否支持MessageChannel 以及 是否是DOM环境判断的

判断条件如下
```js
if (
  // If Scheduler runs in a non-DOM environment, it falls back to a naive
  // implementation using setTimeout.
  typeof window === 'undefined' ||
  // Check if MessageChannel is supported, too.
  typeof MessageChannel !== 'function'
) 
```

无Dom环境 或者不支持 MessageChannel
```js
// 低版本浏览器的 不支持 shouldYield 判断呢 直接返回false
  shouldYieldToHost = function() {
    return false;
  };
```

支持 MessageChannel
```js
import {enableIsInputPending} from '../SchedulerFeatureFlags';
if (
  enableIsInputPending &&
  navigator !== undefined &&
  navigator.scheduling !== undefined &&
  navigator.scheduling.isInputPending !== undefined
) {
  const scheduling = navigator.scheduling;
  // 真的 concurrent模式下的 shouldYield
  // ReactFiberWorkLoop 文件下 workLoopConcurrent(render阶段开始) 进行调用
  shouldYieldToHost = function() {
    // 获取当前的执行时间
    const currentTime = getCurrentTime();
    // 这个deadline 在 performWorkUntilDeadline 中定义
    if (currentTime >= deadline) {
      // There's no time left. We may want to yield control of the main
      // thread, so the browser can perform high priority tasks. The main ones
      // are painting and user input. If there's a pending paint or a pending
      // input, then we should yield. But if there's neither, then we can
      // yield less often while remaining responsive. We'll eventually yield
      // regardless, since there could be a pending paint that wasn't
      // accompanied by a call to `requestPaint`, or other main thread tasks
      // like network events.
      // 如果 needsPaint的标志位为 true 或者 调度器判断用户处于输入状态
      // 为了不导致卡顿 返回true 中断 协调阶段
      if (needsPaint || scheduling.isInputPending()) {
        // There is either a pending paint or a pending input.
        return true;
      }
      // There's no pending input. Only yield if we've reached the max
      // yield interval.
      // 判断当前时间 是否 大于 最大等待间断时间
      return currentTime >= maxYieldInterval;
    } else {
      // There's still time left in the frame.
      return false;
    }
  };

  requestPaint = function() {
    needsPaint = true;
  };
} else {
  // `isInputPending` is not available. Since we have no way of knowing if
  // there's pending input, always yield at the end of the frame.
  shouldYieldToHost = function() {
    return getCurrentTime() >= deadline;
  };
```

其中 关于 navigator 的可以参考 <a href="https://developer.mozilla.org/zh-CN/docs/Web/API/Navigator" target="_blank">mdn官方文档</a>

以上就是 在reconciler 中关于 `shouldYield`的判断了。

#### 问题二 为什么上面的切片时间大部分都是5ms多呢？

在上面的代码中 我们可以看到这样一个判断
```js
if (currentTime >= deadline) {
```
这里的deadline 就是 在 `performWorkUntilDeadline`中定义的

```js
let yieldInterval = 5;
const performWorkUntilDeadline = () => {
    if (scheduledHostCallback !== null) {
      const currentTime = getCurrentTime();
      // Yield after `yieldInterval` ms, regardless of where we are in the vsync
      // cycle. This means there's always time remaining at the beginning of
      // the message event.
      // 计算deadline，deadline会参与到
      // shouldYieldToHost（根据时间片去限制任务执行）的计算中
      deadline = currentTime + yieldInterval;
```
这样代码一追踪  `yieldInterval` 定义的值就是5 所以每次执行的task大概都是5ms

以上就是所有的关于时间切片部分我个人的理解了

### 优先级调度

 <!-- TODO: 理解有问题？ -->
#### unstable_runWithPriority

<!-- 那么在Scheduler中 的触发器是什么呢 其实就是 `unstable_runWithPriority` 这个会在 commit阶段的入口来执行 -->

调用代码如下
```js
function commitRoot(root) {
  const renderPriorityLevel = getCurrentPriorityLevel();
  runWithPriority(
    ImmediateSchedulerPriority,
    commitRootImpl.bind(null, root, renderPriorityLevel),
  );
  return null;
}
```

而 `unstable_runWithPriority`代码如下
```js
// 调度器 的调度方法
// priorityLevel react的优先级
// eventHandler 回调函数
function unstable_runWithPriority(priorityLevel, eventHandler) {
  // 第一步 将react优先级换算成 调度优先级
  switch (priorityLevel) {
    case ImmediatePriority:
    case UserBlockingPriority:
    case NormalPriority:
    case LowPriority:
    case IdlePriority:
      break;
    default:
      priorityLevel = NormalPriority;
  }

  // 记录一个 previous 字段 存储当前 调度优先级
  var previousPriorityLevel = currentPriorityLevel;
  // 修改当前优先级为 react优先级
  currentPriorityLevel = priorityLevel;

  try {
    // 执行 eventHandler 的过程中可能会调到 currentPriorityLevel
    return eventHandler();
  } finally {
    // 恢复调度优先级
    currentPriorityLevel = previousPriorityLevel;
  }
}
```

#### scheduleCallback

我们知道 在`concurrent`模式中  `renconciler` 的入口是 `performConcurrentWorkOnRoot` 那调度 `performConcurrentWorkOnRoot` 的`ensureRootIsScheduled`函数中有这样一样代码

```js
// Schedule a new callback.
// 调度一个新任务
let newCallbackNode;
// 同步优先级：调用scheduleSyncCallback去同步执行任务。
if (newCallbackPriority === SyncLanePriority) {
  // Special case: Sync React callbacks are scheduled on a special
  // internal queue
  newCallbackNode = scheduleSyncCallback(
    performSyncWorkOnRoot.bind(null, root),
  );
} else if (newCallbackPriority === SyncBatchedLanePriority) {
  // 同步批量执行：调用scheduleCallback将任务以立即执行的优先级去加入调度。
  newCallbackNode = scheduleCallback(
    ImmediateSchedulerPriority,
    performSyncWorkOnRoot.bind(null, root),
  );
} else {
  // 属于concurrent模式的优先级：调用scheduleCallback将任务以上面获取到的新任务优先级去加入调度。
  // 根据任务优先级获取Scheduler的调度优先级
  const schedulerPriorityLevel = lanePriorityToSchedulerPriority(
    newCallbackPriority,
  );
  // React中通过下面的代码，让fiber树的构建任务进入调度流程：
  newCallbackNode = scheduleCallback(
    schedulerPriorityLevel,
    performConcurrentWorkOnRoot.bind(null, root),
  );
}
```

我们可以看到在调用 `scheduleCallback` 第一个入参就是调度优先级 第二个就是注册的函数

#### schedulerPriorityLevel 调度优先级
下面我们来解析第一个调度优先级的意义， 首先需要知道的是都有那些调度的优先级定义

代码在 `SchedulerPriorities.js`中

```js
export const NoPriority = 0; // 没有任何优先级
export const ImmediatePriority = 1; // 立即执行的优先级，级别最高
export const UserBlockingPriority = 2; // 用户阻塞级别的优先级
export const NormalPriority = 3; // 正常的优先级
export const LowPriority = 4; // 较低的优先级
export const IdlePriority = 5; // 优先级最低，表示任务可以闲置
```

根据数字不同 分成了 6个等级的优先级， 那在 `scheduleCallback` 是如何使用这些优先级的呢？

继续看代码
```js
// Times out immediately
var IMMEDIATE_PRIORITY_TIMEOUT = -1;
// Eventually times out
var USER_BLOCKING_PRIORITY_TIMEOUT = 250;
var NORMAL_PRIORITY_TIMEOUT = 5000;
var LOW_PRIORITY_TIMEOUT = 10000;
// Never times out
var IDLE_PRIORITY_TIMEOUT = maxSigned31BitInt;
// 计算timeout
function unstable_scheduleCallback(priorityLevel, callback, options) {
  //... 省略其他代码
  var timeout;
  // 将 react优先级 换成 调度器 优先级
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
  // 设置过期时间
  // 计算任务的过期时间，任务开始时间 + timeout
  // 若是立即执行的优先级（ImmediatePriority），
  // 它的过期时间是startTime - 1，意味着立刻就过期
  var expirationTime = startTime + timeout;
  // ... 省略其他代码
}
```
根据上面的代码可以看出 调度优先级 主要用来计算 `timeout`的值


那`timeout`是用来干嘛的呢 `timeout`是作为过期任务的排序标准，下面介绍任务队列的关系

#### 任务队列管理

在 Scheduler中 把 任务分为两种 一种是 未过期任务 用  `timeQueue`存储， 另一种是 已过期任务 用 `taskQueue`存储。

当一个任务(`unstable_scheduleCallback`创建任务)被创建的时候 如何区分该任务属于那种任务类型呢？

1. 判断开始时间 即 `startTime`
代码如下
```js
// 获取当前时间 它是计算任务开始时间、过期时间和判断任务是否过期的依据
var currentTime = getCurrentTime();

// 确定任务开始时间
var startTime;
// 从options中尝试获取delay，也就是推迟时间
if (typeof options === 'object' && options !== null) {
  var delay = options.delay;
  if (typeof delay === 'number' && delay > 0) {
    // 如果有delay，那么任务开始时间就是当前时间加上delay
    startTime = currentTime + delay;
  } else {
    // 没有delay，任务开始时间就是当前时间，也就是任务需要立刻开始
    startTime = currentTime;
  }
} else {
  startTime = currentTime;
}
```

2. 判断过期时间 就是上面计算 timeout的过程

3. 创建一个新的任务
```js

  // 创建一个新的 task
  // 创建调度任务
  var newTask = {
    id: taskIdCounter++,
    // 真正的任务函数，重点
    callback,
    // 任务优先级
    priorityLevel,
    // 任务开始的时间，表示任务何时才能执行
    startTime,
    // 任务的过期时间
    expirationTime,
    // 在小顶堆队列中排序的依据
    sortIndex: -1,
  };
```

4. 任务类型的判断
```js
// 下面的if...else判断各自分支的含义是：

  // 如果任务未过期，则将 newTask 放入timerQueue， 调用requestHostTimeout，
  // 目的是在timerQueue中排在最前面的任务的开始时间的时间点检查任务是否过期，
  // 过期则立刻将任务加入taskQueue，开始调度

  // 如果任务已过期，则将 newTask 放入taskQueue，调用requestHostCallback，
  // 开始调度执行taskQueue中的任务

  if (startTime > currentTime) {
    // This is a delayed task.
    newTask.sortIndex = startTime;
    // startTime 作为 优先级的 排序字段
    // 将当前任务 根据 sortIndex 插入 timerQueue 小顶堆 中
    push(timerQueue, newTask);
    // 如果 taskQueue 中无过期任务 && 当前任务是 最先要执行的任务
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      // All tasks are delayed, and this is the task with the earliest delay.
      // 如果现在taskQueue中没有任务，并且当前的任务是timerQueue中排名最靠前的那一个
      // 那么需要检查timerQueue中有没有需要放到taskQueue中的任务，这一步通过调用
      // requestHostTimeout实现
      if (isHostTimeoutScheduled) {
        // Cancel an existing timeout.
         // 因为即将调度一个requestHostTimeout，所以如果之前已经调度了，那么取消掉
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      // Schedule a timeout.
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    newTask.sortIndex = expirationTime;
    push(taskQueue, newTask);
    // Schedule a host callback, if needed. If we're already performing work,
    // wait until the next time we yield.
    // 开始执行任务，使用flushWork去执行taskQueue
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    }
  }
```

上面的代码简单梳理 可以理解为
 - taskQueue:
   - 根据任务的过期时间`expirationTime`来排序， 过期时间越早说明越紧急。
 - timeQueue:
   - 根据任务的开始时间`startTime`排序 开始时间越早 说明会越早开始。

这里对于任务队列的管理 为了能在 `O(1)`的复杂度中找到需要最先执行的任务 采用了<a href="https://www.cnblogs.com/lanhaicode/p/10546257.html" target="_blank">小顶锥</a>来实现

<b>还有一个需要强调的点</b>

在上面的代码中 我们看到  `scheduleCallback` 的第二个参数就是 `performConcurrentWorkOnRoot.bind(null, root)` 函数 这个函数就放到 task 的 `callback`中



#### requestHostCallback 调度任务执行
在上面的代码 中 在最后 有这样一段代码
```js
// Schedule a host callback, if needed. If we're already performing work,
  // wait until the next time we yield.
  // 开始执行任务，使用flushWork去执行taskQueue
  if (!isHostCallbackScheduled && !isPerformingWork) {
    isHostCallbackScheduled = true;
    requestHostCallback(flushWork);
  }
```
这里就是去执行调度任务的地方 可以看到是执行了两个方法 一个是 `requestHostCallback` 另一个是 `flushWork`



##### requestHostCallback 实现
`requestHostCallback` 是在 Schelduler中定义的 根据最开始讲的 `setTimeout` 和 `MessageChannel` 两个实现

setTimeout实现
```js
requestHostCallback = function(cb) {
  if (_callback !== null) {
    // Protect against re-entrancy.
    setTimeout(requestHostCallback, 0, cb);
  } else {
    _callback = cb;
    setTimeout(_flushCallback, 0);
  }
};
```

MessageChannel实现
```js
const channel = new MessageChannel();
const port = channel.port2;
channel.port1.onmessage = performWorkUntilDeadline;
requestHostCallback = function(callback) {
  // 在这里 赋予 scheduledHostCallback 回调函数 到 performWorkUntilDeadline 去执行
  scheduledHostCallback = callback;
  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true;
    port.postMessage(null);
  }
};
```

这两种实现 最终都是都是执行 `flushWork`

只不过 setTimeout 是直接去调用执行 `flushWork`

而 `MessageChannel`多了一个中间层 也就是 `performWorkUntilDeadline`

而我们主要研究的就是 `performWorkUntilDeadline`的实现

##### performWorkUntilDeadline 实现
```js
// performWorkUntilDeadline作为执行者，它的作用是按照时间片的限制去中断任务，
// 并通知调度者再次调度一个新的执行者去继续任务。按照这种认知去看它的实现，会很清晰。
// performWorkUntilDeadline内部调用的是scheduledHostCallback，它早在开始调度的时候就被requestHostCallback赋值为了flushWork，
const performWorkUntilDeadline = () => {
  if (scheduledHostCallback !== null) {
    const currentTime = getCurrentTime();
    // Yield after `yieldInterval` ms, regardless of where we are in the vsync
    // cycle. This means there's always time remaining at the beginning of
    // the message event.
    // 计算deadline，deadline会参与到
    // shouldYieldToHost（根据时间片去限制任务执行）的计算中
    deadline = currentTime + yieldInterval;
    // hasTimeRemaining表示任务是否还有剩余时间，
    // 它和时间片一起限制任务的执行。如果没有时间，
    // 或者任务的执行时间超出时间片限制了，那么中断任务。

    // 它的默认为true，表示一直有剩余时间
    // 因为MessageChannel的port在postMessage，
    // 是比setTimeout还靠前执行的宏任务，这意味着
    // 在这一帧开始时，总是会有剩余时间
    // 所以现在中断任务只看时间片的了
    const hasTimeRemaining = true;
    try {
      // scheduledHostCallback去执行任务的函数，
      // 当任务因为时间片被打断时，它会返回true，表示
      // 还有任务，所以会再让调度者调度一个执行者
      // 继续执行任务
      const hasMoreWork = scheduledHostCallback(
        hasTimeRemaining,
        currentTime,
      );
      if (!hasMoreWork) {
        // 如果没有任务了，停止调度
        isMessageLoopRunning = false;
        scheduledHostCallback = null;
      } else {
        // If there's more work, schedule the next message event at the end
        // of the preceding one.
        // 如果还有任务，继续让调度者调度执行者，便于继续
        // 完成任务
        port.postMessage(null);
      }
    } catch (error) {
      // If a scheduler task throws, exit the current browser task so the
      // error can be observed.
      port.postMessage(null);
      throw error;
    }
  } else {
    isMessageLoopRunning = false;
  }
  // Yielding to the browser will give it a chance to paint, so we can
  // reset this.
  needsPaint = false;
};
```

我们可以把 `performWorkUntilDeadline` 当作一个执行者 它的作用就是按照时间片的限制去中断任务 并通知调度者再次调度一个新的任务去执行。

其中最重要的代码就是下面
```js
const hasMoreWork = scheduledHostCallback(
  hasTimeRemaining,
  currentTime,
);
```

这里的 `scheduledHostCallback` 就是 我们上面所说的 `flushWork`

继续向下溯源呢

##### flushWork 实现
```js
function flushWork(hasTimeRemaining, initialTime) {
  // ...省略
  try {
    if (enableProfiling) {
      try {
        return workLoop(hasTimeRemaining, initialTime);
      } catch (error) {
        if (currentTask !== null) {
          const currentTime = getCurrentTime();
          markTaskErrored(currentTask, currentTime);
          currentTask.isQueued = false;
        }
        throw error;
      }
    } else {
      // No catch in prod code path.
      return workLoop(hasTimeRemaining, initialTime);
    }
  }
  // ...省略
}
```

这里我们看到 `flushWork` 返回的结果就是 `workLoop` 的执行结果

继续向下溯源

##### workLoop 实现

```js
// workLoop 最后返回的值 给 performWorkUntilDeadline 使用
// 也就是下面的这个 const hasMoreWork = scheduledHostCallback(
//   hasTimeRemaining,
//   currentTime,
// );
// 要理解workLoop，需要回顾Scheduler的功能之一：通过时间片限制任务的执行时间。那么既然任务的执行被限制了，
// 它肯定有可能是尚未完成的，如果未完成被中断，那么需要将它恢复。
// 所以时间片下的任务执行具备下面的重要特点：会被中断，也会被恢复。
// workLoop中可以分为两大部分：循环taskQueue执行任务 和 任务状态的判断
function workLoop(hasTimeRemaining, initialTime) {
  let currentTime = initialTime;
  // 这里你可以先这么理解就是 每次执行workLoop 都要先整合 timeQueue中的任务 看它是不是过期 过期的话塞到 taskQueue 中
  advanceTimers(currentTime);
  // 从taskQueue 取出优先级最高的任务
  currentTask = peek(taskQueue);
  while (
    currentTask !== null &&
    !(enableSchedulerDebugging && isSchedulerPaused)
  ) {
    // currentTask就是当前正在执行的任务，
    // 它中止的判断条件是：任务并未过期，但已经没有剩余时间了
    // （由于hasTimeRemaining一直为true，这与MessageChannel作为宏任务的执行时机有关，我们忽略这个判断条件，只看时间片），
    // 或者应该让出执行权给主线程（时间片的限制），也就是说currentTask执行得好好的，可是时间不允许，那只能先break掉本次while循环，
    // 使得本次循环下面currentTask执行的逻辑都不能被执行到（此处是中断任务的关键）。
    // 但是被break的只是while循环，while下部还是会判断currentTask的状态。
    if (
      currentTask.expirationTime > currentTime &&
      (!hasTimeRemaining || shouldYieldToHost())
    ) {
      // This currentTask hasn't expired, and we've reached the deadline.
      break;
    }
    const callback = currentTask.callback;
    if (typeof callback === 'function') {
      currentTask.callback = null;
      currentPriorityLevel = currentTask.priorityLevel;
      const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;
      markTaskRun(currentTask, currentTime);
      // 调度的时候
      // scheduleCallback(
      //   schedulerPriorityLevel,
      //   performConcurrentWorkOnRoot.bind(null, root),
      // );

      // 其内部return自身的时候
      // function performConcurrentWorkOnRoot(root) {
      //   ...

      // 任务恢复
      //   if (root.callbackNode === originalCallbackNode) {
      //     return performConcurrentWorkOnRoot.bind(null, root);
      //   }
      // 这里还是会返回一个函数 然后 continuationCallback 是一个函数的话
      // 又会执行当前任务
      //   return null;
      // }
      // callback === performConcurrentWorkOnRoot.bind(null, root)
      const continuationCallback = callback(didUserCallbackTimeout);
      currentTime = getCurrentTime();
      // 若任务函数返回值为函数，那么就说明当前任务尚未完成，需要继续调用任务函数，
      // 否则任务完成。workLoop就是通过这样的办法判断单个任务的完成状态。
      if (typeof continuationCallback === 'function') {
        currentTask.callback = continuationCallback;
        markTaskYield(currentTask, currentTime);
      } else {
        if (currentTask === peek(taskQueue)) {
          pop(taskQueue);
        }
      }
      advanceTimers(currentTime);
    } else {
      pop(taskQueue);
    }
    currentTask = peek(taskQueue);
  }
  // 由于它只是被中止了，所以currentTask不可能是null，那么会return一个true告诉外部还没完事呢（此处是恢复任务的关键），
  // 否则说明全部的任务都已经执行完了，taskQueue已经被清空了，return一个false好让外部终止本次调度。
  // 而workLoop的执行结果会被flushWork return出去，flushWork实际上是scheduledHostCallback，
  // 当 performWorkUntilDeadline 检测到scheduledHostCallback的返回值（hasMoreWork）为false时，就会停止调度。
  // Return whether there's additional work
  if (currentTask !== null) {
    // 如果currentTask不为空，说明是时间片的限制导致了任务中断
    // return 一个 true告诉外部，此时任务还未执行完，还有任务，
    // 翻译成英文就是hasMoreWork
    return true;
  } else {
    // 如果currentTask为空，说明taskQueue队列中的任务已经都
    // 执行完了，然后从timerQueue中找任务，调用requestHostTimeout
    // 去把task放到taskQueue中，到时会再次发起调度，但是这次，
    // 会先return false，告诉外部当前的taskQueue已经清空，
    // 先停止执行任务，也就是终止任务调度
    const firstTimer = peek(timerQueue);
    if (firstTimer !== null) {
      // 延时执行
      requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
    }
    return false;
  }
}
```

workLoop的判断任务是否完成依据： <b>若任务函数返回值为函数 在代表当前任务没有完成 需要继续调度任务函数 否则 认为任务已经完成</b>


// TODO: 未完待续


