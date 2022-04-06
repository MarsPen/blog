<h1 align="center">📖 setState 执行机制</h1>

提到 react 的 setState 大家应该都很熟悉，那么下面我们以实际开发中遇到的几个问题，来引入 setState 的机制

### 问题集合

> setState 是同步的还是异步的？

**1. 生命周期和合成事件中**

```js
componentDidMount() {
  console.log("开始调用", this.state.count); // 开始调用 0
  this.setState({
    count: this.state.count + 1
  });
  console.log("count1", this.state.count); // count1 0
  this.setState({
    count: this.state.count + 1
  });
  console.log("count2", this.state.count); // count2 0
  console.log("结束调用", this.state.count); // 结束调用 0
}
```

**2. 异步代码和原生事件中**

```js

componentDidMount() {
  setTimeout(() => {
    console.log("开始调用", this.state.count); // 开始调用 0
    this.setState({
      count: this.state.count + 1
    });
    console.log("count1", this.state.count); // count1 1
    this.setState({
      count: this.state.count + 1
    });
    console.log("count2", this.state.count); // count2 2
    console.log("结束调用", this.state.count); // 结束调用 2
  }, 0);
}

```

**3. 连续多次 setState**

如果想要在 componentDidMount 阶段立即拿到数据可以采用如下模式

```js
componentDidMount() {
  console.log("开始调用", this.state.count);
  this.setState({ count: this.state.count + 1},() => {
    console.log("count1", this.state.count);
  });

  this.setState({count: this.state.count + 1},() => {
    console.log("count2", this.state.count);
  });
  console.log("结束调用", this.state.count);
}
```

执行结果

```md
开始调用 0
结束调用 0
count1 1
count2 1
```

上面的示例代码，我们已经看出了问题

- 为什么连续 setState 之后值结果相同。
- 为什么在生命周期和合成事件中不能及时的拿到结果。
- 为什么异步代码和原生事件中就能拿到结果了，到底是同步的还是异步的。


通过官网对<a href="https://reactjs.org/docs/react-component.html#setstate">setState()</a>介绍，我们大概总结如下这几方面

- 通过队列的形式保存组件状态并通知 React 这个组件和它的子组件需要重新渲
- 并不能总是立即更新组件，也不能保证更新后数据立即生效
- 有时候会批量推迟更新，如果需要在调用后直接拿到数据可利用回调或者 componentDidUpdate（官网建议）
- 会执行浅合并来生成一个新的 state 
- 在同一周期靠后的 setState()将会覆盖前一个 setSate() 的值(相同属性名)

通过上面的总结，我们可能已经解决了提出的问题。但实际上远远没那么简单，下面我们就来看看源码中 setState。此文章分析为 16.9 版本（ 16.9关于调度和 setState 改动还是蛮大的网上通过批处理控制标志已经在源码中不见了）

由于本人能力有限所以不会将整个流程进行分析完毕因为在setState 包括生命周期，组件渲染，事件合成，批处理等多处都会涉及。

我们通过源码来看一下 setState 的执行过程。由于源码很多很长，如果不想了解源码可以查看文章后面的总结

### setState入口<hr>

定义在文件 `packages/react/src/ReactBaseClasses.js`

```js
/**
* 总是使用 setState 来改变状态，而不是使用 this.state
* 无法保证 setState 会立即更新，所以调用 this.state 可能还是旧值 
* 无法保证对 setState 的调用将同步运行，他们可能会被批量处理
* 可以提供一个可选的回调，它将在对 setState 的调用实际完成时执行
* setState 第一个参数为 function 时，会在将来的某个时候和组件参数(state,props, context)一起被调用(不是同步调用)。这些值可以与this.state不同
* 因为 function 可能在 receiveProps 之后 shouldComponentUpdate之前调用，new state、props、content还没更新到当前this指向的这些值。
*/
Component.prototype.setState = function(partialState, callback) {
  invariant(
    typeof partialState === "object" ||
      typeof partialState === "function" ||
      partialState == null,
    "setState(...): takes an object of state variables to update or a " +
      "function which returns an object of state variables."
  );
  this.updater.enqueueSetState(this, partialState, callback, "setState");
};
```

我们看到 setState 传入了两个参数

- partialState 通过 invariant 函数验证我们知道需要对象、函数、null
- callback 回调函数

partialState 这个参数传入的是 state，也许是整个state，也许是部分，但是最后都会被执行浅合并。

```js
let dontMutate = true;
if (dontMutate) {
  dontMutate = false;
  nextState = Object.assign({}, nextState, partialState);
} else {
  Object.assign(nextState, partialState);
}
```

接下来执行了 

```js
this.updater.enqueueSetState(this, partialState, callback, "setState");
```

在这里调用了 this.updater 中的 enqueueSetState, 这是一个 setState 的队列（准确的说它是一个链表）

```js
function Component(props, context, updater) {
  this.props = props;
  this.context = context;
  // 如果一个组件有字符串引用，将在以后分配一个不同的对象
  this.refs = emptyObject;
  // 初始化默认的 updater ，但是在render 的时候被注入真正的值
  this.updater = updater || ReactNoopUpdateQueue;
}
```
通过上面的注释我们看到 updater 有一个默认值，但是真正的值是在 render 的时候被注入的

### updater<hr>

那么接下来我们看看这个 updater 的数据结构，定义在 `react\packages\react-reconciler\src\ReactFiberClassComponent`

```js
//classComponent 初始化的时候拿到的 update 结构
const classComponentUpdater = {
  isMounted,
  enqueueSetState(inst, payload, callback) {
    // inst 就是传入的 this，通过 this 获取 fiber 对象 
    const fiber = getInstance(inst);
    // 计算当前时间
    const currentTime = requestCurrentTime();
    // 异步加载的设置
    const suspenseConfig = requestCurrentSuspenseConfig();
    // 根据超时时间设置任务的优先级
    const expirationTime = computeExpirationForFiber(
      currentTime,
      fiber,
      suspenseConfig,
    );
    // 创建 update 结构
    const update = createUpdate(expirationTime, suspenseConfig);
    // 传进来的 state
    update.payload = payload;
    // callback 就是 setState 的回调函数 setstate({},()=>{})
    if (callback !== undefined && callback !== null) {
      // ...
      update.callback = callback;
    }

    if (revertPassiveEffectsChange) {
      flushPassiveEffects();
    }
    // 将 update 结构放入队列
    enqueueUpdate(fiber, update);
    // 任务调度流程
    scheduleWork(fiber, expirationTime);
  },
  // 替换 state
  enqueueReplaceState(inst, payload, callback) {
    const fiber = getInstance(inst);
    const currentTime = requestCurrentTime();
    const suspenseConfig = requestCurrentSuspenseConfig();
    const expirationTime = computeExpirationForFiber(
      currentTime,
      fiber,
      suspenseConfig,
    );

    const update = createUpdate(expirationTime, suspenseConfig);
    update.tag = ReplaceState;
    update.payload = payload;

    if (callback !== undefined && callback !== null) {
      // ...
      update.callback = callback;
    }

    if (revertPassiveEffectsChange) {
      flushPassiveEffects();
    }
    enqueueUpdate(fiber, update);
    scheduleWork(fiber, expirationTime);
  },
  // 强制更新 state
  enqueueForceUpdate(inst, callback) {
    const fiber = getInstance(inst);
    const currentTime = requestCurrentTime();
    const suspenseConfig = requestCurrentSuspenseConfig();
    const expirationTime = computeExpirationForFiber(
      currentTime,
      fiber,
      suspenseConfig,
    );

    const update = createUpdate(expirationTime, suspenseConfig);
    update.tag = ForceUpdate;

    if (callback !== undefined && callback !== null) {
      // ...
      update.callback = callback;
    }

    if (revertPassiveEffectsChange) {
      flushPassiveEffects();
    }
    enqueueUpdate(fiber, update);
    scheduleWork(fiber, expirationTime);
  }
}
```

接下来我们主要看一下 enqueueSetState，通过上面的注释我们很清楚的知道每个方法的作用。那么我们来总结一下 enqueueSetState 的流程

- 获取 this 上的 fiber 对象
- 计算 currentTime 时间
- 根据 fiber 对象和 currentTime 时间，计算超时时间 
- 根据 expirationTime 创建 update 对象
- 将传入的 payload 也就是 state 赋值到 update.payload
- 如果 callback 存在将 callback 赋值到 update.callback
- 将 update 推入 updateQueue 队列
- 进行任务调度

关于 requestCurrentTime、 requestCurrentSuspenseConfig、computeExpirationForFiber 在这里就不多做解释。请查看<a  href="02-fiber 构建.md"> FiberRoot 构建过程</a>和<a href="03-fiber 调度.md">FiberRoot 调度过程</a>

### createUpdate <hr>
上面代码 执行一共有三个方法分别是 enqueueSetState、 enqueueReplaceState、enqueueForceUpdate，他们主要是的区别就是在于 tag 上。也就是 React 的状态更新分为四种情况

```js
export const UpdateState = 0;
export const ReplaceState = 1;
export const ForceUpdate = 2;
export const CaptureUpdate = 3;

export function createUpdate(
  expirationTime: ExpirationTime,
  suspenseConfig: null | SuspenseConfig,
): Update<*> {
  return {
    //超时时间
    expirationTime,
    suspenseConfig,
    tag: UpdateState, //0更新 1替换 2强制更新 3捕获更新
    //更新的 state
    payload: null,
    //对应的回调，比如setState({}, callback )
    callback: null,
    //指向下一个更新
    next: null,
    //指向下一个side effect
    nextEffect: null,
  };
}
``` 

接下来我们主要来看看 enqueueUpdate 这个函数,也就是将 update 推入 updateQueue 队列

```js
// 每次 setState 都会 update，每次 update，都会入加入 updateQueue
export function enqueueUpdate<State>(fiber: Fiber, update: Update<State>) {
  // 更新队列是延迟创建的.
  // fiber => current
  // alternate => workInProgress
  const alternate = fiber.alternate;
  // current 队列
  let queue1;
  // alternate 队列
  let queue2;

  // current到alternate即workInProgress有一个映射关系
  // 要保证current和workInProgress的updateQueue是一致的

  // 如果 alternate 队列是空的 
  if (alternate === null) {
    // 只有一个 fiber.
    queue1 = fiber.updateQueue;
    queue2 = null;
    //如果 current 仍为空，则初始化更新队列
    if (queue1 === null) {
      queue1 = fiber.updateQueue = createUpdateQueue(fiber.memoizedState);
    }
  } else {
    // 如果 alternate 不为空 则分别更新各自的队列
    queue1 = fiber.updateQueue;
    queue2 = alternate.updateQueue;

    if (queue1 === null) {
      if (queue2 === null) {
        // queue1 和 queue2 的 fiber 都没有更新队列。则创建一个新队列.
        queue1 = fiber.updateQueue = createUpdateQueue(fiber.memoizedState);
        queue2 = alternate.updateQueue = createUpdateQueue(
          alternate.memoizedState,
        );
      } else {
        // 如果 queue2存在，则利用 cloneUpdateQueue 克隆一个队列到 queue1
        queue1 = fiber.updateQueue = cloneUpdateQueue(queue2);
      }
    } else {
      if (queue2 === null) {      
        queue2 = alternate.updateQueue = cloneUpdateQueue(queue1);
      } else {
        // Both owners have an update queue.
      }
    }
  }
  if (queue2 === null || queue1 === queue2) {
    // 只有一个队列.
    appendUpdateToQueue(queue1, update);
  } else {
     // 将更新附加到两个队列，
     // 同时考虑到列表的持久结构不希望将相同的更新添加多次
    if (queue1.lastUpdate === null || queue2.lastUpdate === null) {
      // 其中一个队列不是空的。将更新添加到两个队列.
      appendUpdateToQueue(queue1, update);
      appendUpdateToQueue(queue2, update);
    } else {
      // 两个队列都不是空的。由于结构共享，这两个列表中的最新更新是相同的。
      // 因此，只向其中一个列表追加。
      appendUpdateToQueue(queue1, update);
      // 要更新 queue2 的 `lastUpdate` 指针.
      queue2.lastUpdate = update;
    }
  }
  // ...
}
```
总结一下上面的注释

- queue1 是 fiber.updateQueue;
- queue2 是 alternate.updateQueue
- 如果都为 null，则调用 createUpdateQueue() 获取初始队列
- 如果有一个为 null，则调用 cloneUpdateQueue() 从对方中获取队列
- 如果都不为 null，则调用 appendUpdateToQueue 将 update 作为 lastUpdate

### createUpdateQueue <hr>

我们先来看看如下代码 createUpdateQueue 创建更新队列

```js
export function createUpdateQueue<State>(baseState: State): UpdateQueue<State> {
  const queue: UpdateQueue<State> = {
    // 更新后的 state
    baseState,
    // 队列中的第一个 update
    firstUpdate: null,
    // 队列中的最后一个 update
    lastUpdate: null,
    // 队列中第一个捕获更新的 update
    firstCapturedUpdate: null,
    // 队列中最后一个捕获更新的 update
    lastCapturedUpdate: null,
    // 第一个side effect
    firstEffect: null,
    // 最后一个side effect
    lastEffect: null,
    // 第一个side 捕获更新 effect
    firstCapturedEffect: null,
    // 最后一个side 捕获更新 effect
    lastCapturedEffect: null,
  };
  return queue;
}
```
通过上面的注释我们来整理一下

- baseState 在组件 setState 后，渲染并更新state，在下次更新时，拿的就是这次更新过的state

- firstUpdate 和 lastUpdate 之间的 update 通过上个 update 的 next 串联

### cloneUpdateQueue

```js
// 克隆更新队列
function cloneUpdateQueue<State>(
  currentQueue: UpdateQueue<State>,
): UpdateQueue<State> {
  const queue: UpdateQueue<State> = {
    baseState: currentQueue.baseState,
    firstUpdate: currentQueue.firstUpdate,
    lastUpdate: currentQueue.lastUpdate,
    firstCapturedUpdate: null,
    lastCapturedUpdate: null,

    firstEffect: null,
    lastEffect: null,

    firstCapturedEffect: null,
    lastCapturedEffect: null,
  };
  return queue;
}
```


通过以上代码我们知道 UpdateQueue 实际上是一个<a href="https://github.com/trekhleb/javascript-algorithms/blob/master/README.zh-CN.md">链表</a>

<img src="/assets/react-updateQueue.jpg" />

通过上面的图片展示,我们可以很清楚的看见这是一个链表结构。当然具体的操作链表的方法我就不在这里展开了。

具体的更新队列步骤如下

- 创建更新队列 createUpdateQueue()
- 更新 state getStateFromUpdate()
- 处理更新 processUpdateQueue()
- 提交更新 commitUpdateQueue()


### 总结

react 事件系统中和通过生命周期阶段调用的 setState 在推入队列前面的阶段实际上都可以属于同步阶段，但在执行批处理到渲染阶段都会触发批处理由于自定义的一套事件系统所以都是异步。

使用原生事件监听和 settimeout 这种当时不会触发此事件系统 ，也不会触发批处理，所以可以更新 this.setState 所以是同步的。

### 推荐阅读

 - <a href="https://stackoverflow.com/questions/48563650/does-react-keep-the-order-for-state-updates/48610973#48610973">为什么会批量执行</a>
 - <a href="https://github.com/facebook/react/issues/9439">某些情况下 setState 的工作方式</a>
 - <a href="https://github.com/facebook/react/issues/11527#issuecomment-360199710">为什么不直接更新 this.state</a>

