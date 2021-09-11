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

`未完待续`