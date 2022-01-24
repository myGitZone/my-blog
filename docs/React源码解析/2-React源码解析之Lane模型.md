---
group:
  order: 2
title: React源码解析之Lane模型
---

## Lane模型
React需要设计一套满足如下需要的优先级机制：

- 可以表示优先级的不同
- 可能同时存在几个同优先级的更新，所以还得能表示批的概念
- 方便进行优先级相关计算

为了满足如上需求，React设计了lane模型。接下来我们来看lane模型如何满足以上3个条件。

### 表示优先级的不同
<img style="border: 1px solid #eee;" src="../../images/lane.jpeg" alt="" />

不同的赛车疾驰在不同的赛道。内圈的赛道总长度更短，外圈更长。某几个临近的赛道的长度可以看作差不多长。

lane模型借鉴了同样的概念，使用31位的二进制表示31条赛道，位数越小的赛道优先级越高，某些相邻的赛道拥有相同优先级。

如下：
```javascript
export const NoLanes: Lanes = /*                        */ 0b0000000000000000000000000000000;
export const NoLane: Lane = /*                          */ 0b0000000000000000000000000000000;

export const SyncLane: Lane = /*                        */ 0b0000000000000000000000000000001;
export const SyncBatchedLane: Lane = /*                 */ 0b0000000000000000000000000000010;

export const InputDiscreteHydrationLane: Lane = /*      */ 0b0000000000000000000000000000100;
const InputDiscreteLanes: Lanes = /*                    */ 0b0000000000000000000000000011000;

const InputContinuousHydrationLane: Lane = /*           */ 0b0000000000000000000000000100000;
const InputContinuousLanes: Lanes = /*                  */ 0b0000000000000000000000011000000;

export const DefaultHydrationLane: Lane = /*            */ 0b0000000000000000000000100000000;
export const DefaultLanes: Lanes = /*                   */ 0b0000000000000000000111000000000;

const TransitionHydrationLane: Lane = /*                */ 0b0000000000000000001000000000000;
const TransitionLanes: Lanes = /*                       */ 0b0000000001111111110000000000000;

const RetryLanes: Lanes = /*                            */ 0b0000011110000000000000000000000;

export const SomeRetryLane: Lanes = /*                  */ 0b0000010000000000000000000000000;

export const SelectiveHydrationLane: Lane = /*          */ 0b0000100000000000000000000000000;

const NonIdleLanes = /*                                 */ 0b0000111111111111111111111111111;

export const IdleHydrationLane: Lane = /*               */ 0b0001000000000000000000000000000;
const IdleLanes: Lanes = /*                             */ 0b0110000000000000000000000000000;

export const OffscreenLane: Lane = /*                   */ 0b1000000000000000000000000000000;
```
其中，同步优先级占用的赛道为第一位：
```javascript
export const SyncLane: Lane = /*                        */ 0b0000000000000000000000000000001;
```
从SyncLane往下一直到SelectiveHydrationLane，赛道的优先级逐步降低。

### 表示“批”的概念
可以看到其中有几个变量占用了几条赛道，比如：
```javascript
const InputDiscreteLanes: Lanes = /*                    */ 0b0000000000000000000000000011000;
export const DefaultLanes: Lanes = /*                   */ 0b0000000000000000000111000000000;
const TransitionLanes: Lanes = /*                       */ 0b0000000001111111110000000000000;
```
这就是批的概念，被称作lanes（区别于优先级的lane）。
其中InputDiscreteLanes是“用户交互”触发更新会拥有的优先级范围。

DefaultLanes是“请求数据返回后触发更新”拥有的优先级范围。
TransitionLanes是Suspense、useTransition、useDeferredValue拥有的优先级范围。
这其中有个细节，越低优先级的lanes占用的位越多。比如InputDiscreteLanes占了2个位，TransitionLanes占了9个位。

原因在于：越低优先级的更新越容易被打断，导致积压下来，所以需要更多的位。相反，最高优的同步更新的SyncLane不需要多余的lanes。

### 方便进行优先级相关计算
既然lane对应了二进制的位，那么优先级相关计算其实就是位运算。

比如：

计算a、b两个lane是否存在交集，只需要判断a与b按位与的结果是否为0：
```javascript
export function includesSomeLane(a: Lanes | Lane, b: Lanes | Lane) {
  return (a & b) !== NoLanes;
}
```
计算b这个lanes是否是a对应的lanes的子集，只需要判断a与b按位与的结果是否为b：
```javascript
export function isSubsetOfLanes(set: Lanes, subset: Lanes | Lane) {
  return (set & subset) === subset;
}
```
将两个lane或lanes的位合并只需要执行按位或操作：
```javascript
export function mergeLanes(a: Lanes | Lane, b: Lanes | Lane): Lanes {
  return a | b;
}
```
从set对应lanes中移除subset对应lane（或lanes），只需要对subset的lane（或lanes）执行按位非，结果再对set执行按位与。
```javascript
export function removeLanes(set: Lanes, subset: Lanes | Lane): Lanes {
  return set & ~subset;
}
```

## React事件优先级
```javascript
// 离散事件优先级，例如：点击事件，input输入等触发的更新任务，优先级最高
export const DiscreteEventPriority: EventPriority = SyncLane;
// 连续事件优先级，例如：滚动事件，拖动事件等，连续触发的事件
export const ContinuousEventPriority: EventPriority = InputContinuousLane;
// 默认事件优先级，例如：setTimeout触发的更新任务
export const DefaultEventPriority: EventPriority = DefaultLane;
// 闲置事件优先级，优先级最低
export const IdleEventPriority: EventPriority = IdleLane;
```
可以看到React的事件优先级的值还是使用的Lane的值，那为什么不直接使用Lane呢？我觉得可能是为了不与Lane机制耦合，后面事件优先级有什么变动的话，可以直接修改而不会影响到Lane。

### Lane优先级转换为React事件优先级：
```javascript
export function lanesToEventPriority(lanes: Lanes): EventPriority {
  // 找到优先级最高的lane
  const lane = getHighestPriorityLane(lanes);
  if (!isHigherEventPriority(DiscreteEventPriority, lane)) {
    return DiscreteEventPriority;
  }
  if (!isHigherEventPriority(ContinuousEventPriority, lane)) {
    return ContinuousEventPriority;
  }
  if (includesNonIdleWork(lane)) {
    return DefaultEventPriority;
  }
  return IdleEventPriority;
}
```

## Scheduler优先级
```javascript
export const NoPriority = 0; //没有优先级
export const ImmediatePriority = 1; // 立即执行任务的优先级，级别最高
export const UserBlockingPriority = 2; // 用户阻塞的优先级
export const NormalPriority = 3; // 正常优先级
export const LowPriority = 4; // 较低的优先级
export const IdlePriority = 5; // 优先级最低，闲表示任务可以闲置
```
### React事件优先级转换为Scheduler优先级
```javascript
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
    ...
    switch (lanesToEventPriority(nextLanes)) {
      case DiscreteEventPriority:
        schedulerPriorityLevel = ImmediateSchedulerPriority;
        break;
      case ContinuousEventPriority:
        schedulerPriorityLevel = UserBlockingSchedulerPriority;
        break;
      case DefaultEventPriority:
        schedulerPriorityLevel = NormalSchedulerPriority;
        break;
      case IdleEventPriority:
        schedulerPriorityLevel = IdleSchedulerPriority;
        break;
      default:
        schedulerPriorityLevel = NormalSchedulerPriority;
        break;
    }
}
```
lanesToEventPriority函数就是上面Lane优先级转换为React事件优先级的函数，先将lane的优先级转换为React事件的优先级，然后再根据React事件的优先级转换为Scheduler的优先级。

## 如何为React不同的事件添加不同的优先级
第一次渲染页面的时候，也就是mount阶段。
```javascript
const root = document.getElementById('root');
ReactDOM.createRoot(root).render(<div>Hello World</div>);
```
调用createRoot方法创建根节点后，会为root这个节点做事件委托：
```javascript
export function createRoot(
  container: Container,
  options?: CreateRootOptions,
): RootType {
    ...
    
  // 创建容器-Fiber根节点
  const root = createContainer(
    container,
    ConcurrentRoot,
    hydrate,
    hydrationCallbacks,
    isStrictMode,
    concurrentUpdatesByDefaultOverride,
  );
  
  // 在root容器上添加事件监听，做事件委托
  listenToAllSupportedEvents(rootContainerElement);
  
}
```
会对所有支持的事件做一个优先级分类，并赋予这些事件不同的优先级：
```javascript
export function createEventListenerWrapperWithPriority(
  targetContainer: EventTarget,
  domEventName: DOMEventName,
  eventSystemFlags: EventSystemFlags,
): Function {
  // 根据不同的事件做优先级分类
  const eventPriority = getEventPriority(domEventName);

  // 根据优先级分类，设置事件触发时的优先级
  let listenerWrapper;
  switch (eventPriority) {
    case DiscreteEventPriority:
      listenerWrapper = dispatchDiscreteEvent;
      break;
    case ContinuousEventPriority:
      listenerWrapper = dispatchContinuousEvent;
      break;
    case DefaultEventPriority:
    default:
      listenerWrapper = dispatchEvent;
      break;
  }
  return listenerWrapper.bind(
    null,
    domEventName,
    eventSystemFlags,
    targetContainer,
  );
}
```
首先会调用getEventPriority方法，这个方法内部主要是将不同的事件区分为不同的优先级：
```javascript
export function getEventPriority(domEventName: DOMEventName): * {
  switch (domEventName) {
    case 'cancel':
    case 'click':
    case 'copy':
    case 'dragend':
    case 'dragstart':
    case 'drop':
    ...
    case 'focusin':
    case 'focusout':
    case 'input':
    case 'change':
    case 'textInput':
    case 'blur':
    case 'focus':
    case 'select':
      // 同步优先级
      return DiscreteEventPriority;
    case 'drag':
    case 'mousemove':
    case 'mouseout':
    case 'mouseover':
    case 'scroll':
    ...
    case 'touchmove':
    case 'wheel':
    case 'mouseenter':
    case 'mouseleave':
      // 连续触发优先级
      return ContinuousEventPriority;
   ...
    default:
      return DefaultEventPriority;
  }
}
```
通过上面的方法可以了解到，React将用户的一些行为做了处理，点击、input输入等都设置为同步优先级，主要是当执行这些操作的时候，需要立即得到反馈，如果操作没有反馈，就会给用户卡顿的感觉。

接下来会根据获取到的优先级分类，设置事件触发时拥有相对应优先级的回调函数，例如设置dispatchContinuousEvent，相对应回调函数中都调用了同一个方法setCurrentUpdatePriority，
并且都设置了当前事件相对应的事件优先级的值。
```javascript
function dispatchContinuousEvent(
  domEventName,
  eventSystemFlags,
  container,
  nativeEvent,
) {
  ...
  setCurrentUpdatePriority(ContinuousEventPriority);
}
```

## Lane是如何在React中工作的
当我们触发一个点击事件，调用setState产生一个更新任务的时候：
```javascript
Component.prototype.setState = function(partialState, callback) {
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};
```
首先调用setState函数发起更新，setState内部调用了enqueueSetState函数:
```javascript
enqueueSetState(inst, payload, callback) {
    const fiber = getInstance(inst); //获取当前组件对应的fiber节点
    const eventTime = requestEventTime(); // 获取当前事件触发的时间
    const lane = requestUpdateLane(fiber); // 获取到当前事件对应的Lane优先级

    // 创建更新对象，将需要更新的内容挂载到payload上
    const update = createUpdate(eventTime, lane); 
    update.payload = payload;
    if (callback !== undefined && callback !== null) {
      update.callback = callback;
    }

    // 将更新对象添加进更新队列中
    enqueueUpdate(fiber, update, lane);
    const root = scheduleUpdateOnFiber(fiber, lane, eventTime);
    ...
}
```

### 获取事件的优先级
首先获取到当前需要更新的组件的fiber对象，然后调用了requestUpdateLane函数获取到了当前事件的优先级，我们来看一下requestUpdateLane函数内部是如何获取到事件优先级的：
```javascript
export function requestUpdateLane(fiber: Fiber): Lane {
  // 获取到当前渲染的模式：sync mode（同步模式） 或 concurrent mode（并发模式）
  const mode = fiber.mode;
  if ((mode & ConcurrentMode) === NoMode) {
    // 检查当前渲染模式是不是并发模式，等于NoMode表示不是，则使用同步模式渲染
    return (SyncLane: Lane);
  } else if (
    !deferRenderPhaseUpdateToNextBatch &&
    (executionContext & RenderContext) !== NoContext &&
    workInProgressRootRenderLanes !== NoLanes
  ) {
    // workInProgressRootRenderLanes是在任务执行阶段赋予的需要更新的fiber节点上的lane的值
    // 当新的更新任务产生时，workInProgressRootRenderLanes不为空，则表示有任务正在执行
    // 那么则直接返回这个正在执行的任务的lane，那么当前新的任务则会和现有的任务进行一次批量更新
    return pickArbitraryLane(workInProgressRootRenderLanes);
  }

  // 检查当前事件是否是过渡优先级
  // 如果是的话，则返回一个过渡优先级
  // 过渡优先级的分配规则：
  // 产生的任务A给它分配为TransitionLanes的第一位：TransitionLane1 = 0b0000000000000000000000001000000
  // 现在又产生了任务B，那么则从A的位置向左移动一位： TransitionLane2 = 0b0000000000000000000000010000000
  // 后续产生的任务则会一次向后移动，直到移动到最后一位
  // 过渡优先级共有16位:                         TransitionLanes = 0b0000000001111111111111111000000
  // 当所有位都使用完后，则又从第一位开始赋予事件过渡优先级
  const isTransition = requestCurrentTransition() !== NoTransition;
  if (isTransition) {
    if (currentEventTransitionLane === NoLane) {
      currentEventTransitionLane = claimNextTransitionLane();
    }
    return currentEventTransitionLane;
  }

  // 在react的内部事件中触发的更新事件，比如：onClick等，会在触发事件的时候为当前事件设置一个优先级，可以直接拿来使用
  const updateLane: Lane = (getCurrentUpdatePriority(): any);
  if (updateLane !== NoLane) {
    return updateLane;
  }

  // 在react的外部事件中触发的更新事件，比如：setTimeout等，会在触发事件的时候为当前事件设置一个优先级，可以直接拿来使用
  const eventLane: Lane = (getCurrentEventPriority(): any);
  return eventLane;
}
```
#### 非concurrent模式
首先会检查当前的渲染模式是否是concurrent模式，如果不是concurrent模式则都会使用同步优先级做渲染：
```javascript
if ((mode & ConcurrentMode) === NoMode) {
    return (SyncLane: Lane);
}
```
#### concurrent模式
如果是并发模式，则会接着检查当前是否有任务正在执行，workInProgressRootRenderLanes是在初始化workInProgress树时，将当前执行的任务的优先级赋值给了workInProgressRootRenderLanes，如果workInProgressRootRenderLanes不为空，那么则直接返回这个正在执行的任务的lane，当前新的任务则会和现有的任务进行一次批量更新：
```javascript
if (
    !deferRenderPhaseUpdateToNextBatch &&
    (executionContext & RenderContext) !== NoContext &&
    workInProgressRootRenderLanes !== NoLanes
  ) {
    return pickArbitraryLane(workInProgressRootRenderLanes);
  }
```
如果上面都不是，则会判断当前事件是否是过渡优先级，如果是，则会分配过渡优先级中的一个位置。

过渡优先级分配规则是：分配优先级时，会从过渡优先级的最右边开始分配，后续产生的任务则会依次向左移动一位，直到最后一个位置被分配后，后面的任务会从最右边第一个位置再开始做分配：

当前产生了一个任务A，那么会分配过渡优先级的最右边第一个位置：
```javascript
TransitionLane1 = 0b0000000000000000000000001000000
```
现在又产生了任务B，那么则从A的位置向左移动一位：
```javascript
TransitionLane2 = 0b0000000000000000000000010000000
```
后续产生的任务则会依次向左移动一位，过渡优先级共有16位:
```javascript
TransitionLanes = 0b0000000001111111111111111000000
```
当最左边的1的位置被分配后，则又从最右边第一位1的位置开始赋予事件过渡优先级。

如果不是过渡优先级的任务，则接着往下找，可以看到接下来调用了getCurrentUpdatePriority函数，记得我们最开始讲到过，当项目初次渲染的时候，会在root容器上做事件委托并将所有支持的事件做优先级分类，当事件触发时会调用setCurrentUpdatePriority函数设置当前事件的优先级。调用getCurrentUpdatePriority函数也就获取到了事件触发时设置的事件优先级。获取到的事件优先级不为空的话，则会直接返回该事件的优先级。
```javascript
const updateLane: Lane = (getCurrentUpdatePriority(): any);
  if (updateLane !== NoLane) {
    return updateLane;
  }
```

如果上面都没有找到事件优先级，则是会调用getCurrentEventPriority来获取React的外部事件的优先级，比如：在setTimeout中调用了setState方法：
```javascript
const eventLane: Lane = (getCurrentEventPriority(): any);
return eventLane;
```
最后将找到的事件的优先级返回。

### 使用事件的优先级
现在我们已经看到是如果获取到事件的优先级了，那么是如果使用Lane的呢？我们接下来看。

在enqueueSetState中，通过requestUpdateLane获取到了lane，接着创建一个更新对象，将事件的lane添加到更新对象上：
```javascript
const update = createUpdate(eventTime, lane); 

export function createUpdate(eventTime: number, lane: Lane): Update<*> {
  const update: Update<*> = {
    eventTime, //更新事件触发的时间
    lane, // 事件更新的优先级

    tag: UpdateState, // 类型：更新，替换，强制更新等
    payload: null, // 需要更新的内容
    callback: null, // 更新回调 setState的第二个参数

    next: null, // 下一个更新对象
  };
  return update;
}
```
接着将需要更新的内容挂载到payload上，将更新回调函数挂载到更新对象的callback属性上：

```javascript
update.payload = payload;
if (callback !== undefined && callback !== null) {
  update.callback = callback;
}
```
然后将更新对象添加到当前组件对应的fiber节点上的更新队列中：

```javascript
enqueueUpdate(fiber, update, lane);
```

更新队列是一个循环链表结构。

接着会调用scheduleUpdateOnFiber，做好调度任务前的准备，我们主要看其中几个重要的地方：

```javascript
export function scheduleUpdateOnFiber(
  fiber: Fiber,
  lane: Lane,
  eventTime: number,
): FiberRoot | null {
  // 检查是否做了无限循环更新，比如：在render函数中调用了setState，如果是则会报错提示
  checkForNestedUpdates();
  ...

  // 收集需要更新的子节点的lane，存放在父fiber上的childLanes上
  // 更新当前fiber节点的lannes,表示当前节点需要更新
  // 从当前需要更新的fiber节点向上遍历，遍历到根节点（root fiber）并更新每个fiber节点上的childLanes属性
  // childLanes有值表示当前节点下有子节点需要更新
  const root = markUpdateLaneFromFiberToRoot(fiber, lane);
  if (root === null) {
    return null;
  }

  // 将当前需要更新的lane添加到fiber root的pendingLanes属性上，表示有新的更新任务需要被执行
  // 通过计算出当前lane的位置，并添加事件触发时间到eventTimes中
  markRootUpdated(root, lane, eventTime);
  ...

  ensureRootIsScheduled(root, eventTime);
  ...
  return root;
}
```

我们看到函数内部调用了markUpdateLaneFromFiberToRoot，这个函数主要的作用是更新当前fiber节点的lannes,表示当前节点需要更新，然后收集需要更新的子节点的lane，存放在父fiber上的childLanes属性上。在后面做更新时，会根据fiber节点上lannes*判断当前fiber节点是否需要更新，根据childLanes判断当前fiber的子节点是否需要更新。我们来看一下markUpdateLaneFromFiberToRoot内部是如何实现的：
```javascript
function markUpdateLaneFromFiberToRoot(
  sourceFiber: Fiber,
  lane: Lane,
): FiberRoot | null {
  // 更新当前节点的lanes，表示当前节点需要更新
  sourceFiber.lanes = mergeLanes(sourceFiber.lanes, lane);
  let alternate = sourceFiber.alternate;
  if (alternate !== null) {
    alternate.lanes = mergeLanes(alternate.lanes, lane);
  }
 ...
  // 从当前需要更新的fiber节点向上遍历直到根fiber节点（root fiber），更新每个fiber节点的childLanes
  // 在之后会通过childLanes来判断当前fiber节点下是否有子节点需要更新
  let node = sourceFiber;
  let parent = sourceFiber.return;
  while (parent !== null) {
    parent.childLanes = mergeLanes(parent.childLanes, lane);
    alternate = parent.alternate;
    if (alternate !== null) {
      alternate.childLanes = mergeLanes(alternate.childLanes, lane);
    } else {
      if (__DEV__) {
        if ((parent.flags & (Placement | Hydrating)) !== NoFlags) {
          warnAboutUpdateOnNotYetMountedFiberInDEV(sourceFiber);
        }
      }
    }
    node = parent;
    parent = parent.return;
  }
  if (node.tag === HostRoot) {
    const root: FiberRoot = node.stateNode;
    return root;
  } else {
    return null;
  }
}
```
首先将新任务的lane合并到当前fiber节点上的lanes属性，表示当前fiber需要更新，如果fiber节点对应的alternate不为空的话，表示正在更新，并且会同步更新alternate上的lanes。

```javascript
// 更新当前节点的lanes，表示当前节点需要更新
sourceFiber.lanes = mergeLanes(sourceFiber.lanes, lane);
let alternate = sourceFiber.alternate;
if (alternate !== null) {
    alternate.lanes = mergeLanes(alternate.lanes, lane);
}
```
接下来则是从当前更新的节点向上遍历至fiber根节点（root fiber），更新每个fiber节点上的childLanes属性，表示当前fiber下的子节点需要更新：

```javascript
  // 从当前需要更新的fiber节点向上遍历直到根fiber节点（root fiber），更新每个fiber节点的childLanes
  // 在之后会通过childLanes来判断当前fiber节点下是否有子节点需要更新
  let node = sourceFiber;
  let parent = sourceFiber.return;
  while (parent !== null) {
    parent.childLanes = mergeLanes(parent.childLanes, lane);
    alternate = parent.alternate;
    if (alternate !== null) {
      alternate.childLanes = mergeLanes(alternate.childLanes, lane);
    }
    node = parent;
    parent = parent.return;
  }
```

markUpdateLaneFromFiberToRoot执行完毕，紧接着调用了markRootUpdated函数，这个函数的作用是将当前需要更新的lane添加到fiber root的pendingLanes属性上，表示有新的更新任务需要被执行，然后将事件触发时间记录在eventTimes属性上：

```javascript
export function markRootUpdated(
  root: FiberRoot,
  updateLane: Lane,
  eventTime: number,
) {
  // 将当前需要更新的lane添加到fiber root的pendingLanes属性上
  root.pendingLanes |= updateLane;

  if (updateLane !== IdleLane) {
    root.suspendedLanes = NoLanes;
    root.pingedLanes = NoLanes;
  }

  // 假设updateLane为：0b000100
  // eventTimes是这种形式的：[-1, -1, -1, 44573.3452, -1， -1]
  // 用一个数组去储存eventTime，-1表示空位，非-1的位置和lane中1的位置相同
  const eventTimes = root.eventTimes;
  const index = laneToIndex(updateLane);
  eventTimes[index] = eventTime;
}
```
eventTimes是31位长度的Array，对应Lane使用31位的二进制。

假设updateLane为：0b000100 那么它在eventTimes中则是这种形式的：
```javascript
[-1, -1, 44573.3452, -1， -1...]
```
markRootUpdated调用完成后，紧接着调用了ensureRootIsScheduled函数，准备开始任务的调度。

ensureRootIsScheduled是一个比较重要的函数，里面存在了高优先级任务插队和任务饥饿问题，以及批量更新的处理。那么我们来看一下该函数中是如何处理这些问题的。

#### 任务饥饿问题
```javascript
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  ...
  // 为当前任务根据优先级添加过期时间
  // 并检查未执行的任务中是否有任务过期，有任务过期则expiredLanes中添加该任务的lane
  // 在后续任务执行中以同步模式执行，避免饥饿问题
  markStarvedLanesAsExpired(root, currentTime);
  ...
}
```

关于任务饥饿问题的处理，主要逻辑在markStarvedLanesAsExpired函数中，它主要的作用是为当前任务根据优先级添加过期时间，并检查未执行的任务中是否有任务过期，有任务过期则在expiredLanes中添加该任务的lane，在后续该任务的执行中以同步模式执行，避免饥饿问题：

```javascript
export function markStarvedLanesAsExpired(
  root: FiberRoot,
  currentTime: number,
): void {

  const pendingLanes = root.pendingLanes;
  const suspendedLanes = root.suspendedLanes;
  const pingedLanes = root.pingedLanes;
  const expirationTimes = root.expirationTimes;

  // 将要执行的任务会根据它们的优先级生成一个过期时间
  // 当某个任务过期了，则将该任务的lane添加到expiredLanes过期lanes上
  // 在后续执行任务的时候，会通过检查当前任务的lane是否存在于expiredLanes上,
  // 如果存在的话，则会将该任务以同步模式去执行，避免任务饥饿问题
  // ps: 什么饥饿问题？
  // 饥饿问题是指当执行一个任务时，不断的插入多个比该任务优先级高的任务，那么
  // 这个任务会一直得不到执行
  let lanes = pendingLanes;
  while (lanes > 0) {
    // 获取当前lanes中最左边1的位置
    // 例如：
    // lanes = 28 = 0b0000000000000000000000000011100
    // 以32位正常看的话，最左边的1应该是在5的位置上
    // 但是lanes设置了总长度为31，所以我们可以也减1，看作在4的位置上
    // 如果这样不好理解的话，可以看pickArbitraryLaneIndex中的源码：
    // 31 - clz32(lanes), clz32是Math中的一个API，获取的是最左边1前面的所有0的个数
    const index = pickArbitraryLaneIndex(lanes);

    // 上面获取到最左边1的位置后，还需要获取到这个位置上的值
    // index = 4
    // 16 = 10000 = 1 << 4
    const lane = 1 << index;

    // 获取当前位置上任务的过期时间，如果没有则会根据任务的优先级创建一个过期时间
    // 如果有则会判断任务是否过期，过期了则会将当前任务的lane添加到expiredLanes上
    const expirationTime = expirationTimes[index];
    if (expirationTime === NoTimestamp) {
      if (
        (lane & suspendedLanes) === NoLanes ||
        (lane & pingedLanes) !== NoLanes
      ) {
        expirationTimes[index] = computeExpirationTime(lane, currentTime);
      }
    } else if (expirationTime <= currentTime) {
      root.expiredLanes |= lane;
    }

    // 从lanes中删除lane, 每次循环删除一个，直到lanes等于0
    // 例如：
    // lane = 16 =  10000
    // ~lane =      01111
    // lanes = 28 = 11100
    // lanes = 12 = 01100 = lanes & ~lane
    lanes &= ~lane;
  }
}
```

然后使用index获取到相对应expirationTimes中的过期时间，如果过期时间为空则会根据当前优先级生成一个过期时间，优先级越高过期时间越小。然后将过期时间添加到相应的位置。

expirationTimes与eventTimes一样也是31位长度的Array，对应Lane使用31位的二进制。

```javascript
expirationTimes[index] = computeExpirationTime(lane, currentTime);
```
如果当前位置有过期时间，则会检查是否过期，如果过期则将当前lane添加到expiredLanes上，在后续执行该任务的时候使用同步渲染，避免任务饥饿的问题。

```javascript
if (expirationTime <= currentTime) {
  root.expiredLanes |= lane;
}
```

接着会将当前计算完成的lane从lanes中删除，每次循环删除一个，直到lanes等于0：
```javascript
 lanes &= ~lane;
```

#### 任务插队
```javascript
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  const existingCallbackNode = root.callbackNode;
 
  // 为当前任务根据优先级添加过期时间
  // 并检查未执行的任务中是否有任务过期，有任务过期则expiredLanes中添加该任务的lane
  // 在后续任务执行中以同步模式执行，避免饥饿问题
  markStarvedLanesAsExpired(root, currentTime);

  // 获取优先级最高的任务的优先级
  const nextLanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );

  // 如果nextLanes为空则表示没有任务需要执行，则直接中断更新
  if (nextLanes === NoLanes) {
    if (existingCallbackNode !== null) {
      cancelCallback(existingCallbackNode);
    }
    root.callbackNode = null;
    root.callbackPriority = NoLane;
    return;
  }

  // nextLanes获取的是所有任务中优先级最高任务的lane
  // 那么与当前现有的任务的优先级比较，只会有两种结果：
  // 1.与现有的任务优先级一样，那么则会中断当前新任务向下的执行，重用之前现有的任务
  // 2.新任务的优先级大于现有的任务优先级，那么则会取消现有的任务的执行，优先执行优先级高的任务

  // 与现有的任务优先级一样的情况
  if (
    existingCallbackPriority === newCallbackPriority
  ) {
    return;
  }

  // 新任务的优先级大于现有的任务优先级
  // 取消现有的任务的执行
  if (existingCallbackNode != null) {
    cancelCallback(existingCallbackNode);
  }

  // 开始调度任务
  // 判断新任务的优先级是否是同步优先级
  // 是则使用同步渲染模式，否则使用并发渲染模式（时间分片）
  let newCallbackNode;
  if (newCallbackPriority === SyncLane) {
    ...
    newCallbackNode = null;
  } else {
    ...
    newCallbackNode = scheduleCallback(
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root),
    );
  }

  root.callbackPriority = newCallbackPriority;
  root.callbackNode = newCallbackNode;
}
```
处理完饥饿问题，接下来调用了getNextLanes获取到所有任务中优先级最高的任务的lane。

```javascript
export function getNextLanes(root: FiberRoot, wipLanes: Lanes): Lanes {
  const pendingLanes = root.pendingLanes;

  //没有剩余任务的时候，跳出更新
  if (pendingLanes === NoLanes) {
    return NoLanes;
  }
  ...
}
```
首先判断pendingLanes是否为空。根据之前的代码，每个产生的任务都会将它们各自的优先级添加到fiber root的pendingLanes的属性上，也就是说pendingLanes上保存了所有将要执行的任务的lane，如果pendingLanes为空，那么则表示任务全部执行完成，也就不需要更新了，直接跳出。

可以看到ensureRootIsScheduled中对于getNextLanes返回空的处理：

```javascript
// 如果nextLanes为空则表示没有任务需要执行，则直接中断更新
if (nextLanes === NoLanes) {
    // existingCallbackNode不为空表示有任务使用了concurrent模式被scheduler调用，但是还未执行
    // nextLanes为空了则表示没有任务了，就算这个任务执行了但是也做不了任何更新，所以需要取消掉
    if (existingCallbackNode !== null) {
      // 使用cancelCallback会将任务的callback置为null
      // 在scheduler循环taskQueue时，会检查当前task的callback是否为null
      // 为null则从taskQueue中删除，不会执行
      cancelCallback(existingCallbackNode);
    }
    root.callbackNode = null;
    root.callbackPriority = NoLane;
    return;
}
```

回过头继续看getNextLanes中的代码：

```javascript
// 在将要处理的任务中检查是否有未闲置的任务，如果有的话则需要先执行未闲置的任务，不能执行挂起任务
  // 例如：
  // 当前pendingLanes为: 17 = 0b0000000000000000000000000010001 
  // NonIdleLanes          = 0b0001111111111111111111111111111
  // 结果为:                = 0b0000000000000000000000000010001 = 17  
  const nonIdlePendingLanes = pendingLanes & NonIdleLanes;

  //检查是或否还有未闲置且将要执行的任务
  if (nonIdlePendingLanes !== NoLanes) {
    //检查未闲置的任务中除去挂起的任务，是否还有未被阻塞的的任务，有的话则需要
    //从这些未被阻塞的任务中找出任务优先级最高的去执行
    // & ~suspendedLanes 相当于从 nonIdlePendingLanes 中删除 suspendedLanes
    const nonIdleUnblockedLanes = nonIdlePendingLanes & ~suspendedLanes;
    if (nonIdleUnblockedLanes !== NoLanes) {
      nextLanes = getHighestPriorityLanes(nonIdleUnblockedLanes);
    } else {
      // nonIdleUnblockedLanes（未闲置且未阻塞的任务）是未闲置任务中除去挂起的任务剩下来的
      // 如果nonIdleUnblockedLanes为空，那么则从剩下的，也就是挂起的任务中找到优先级最高的来执行
      const nonIdlePingedLanes = nonIdlePendingLanes & pingedLanes;
      if (nonIdlePingedLanes !== NoLanes) {
        nextLanes = getHighestPriorityLanes(nonIdlePingedLanes);
      }
    }
  } else {
    // The only remaining work is Idle.
    // 剩下的任务都是闲置的
    // 找出未被阻塞的任务，然后从中找出优先级最高的执行
    const unblockedLanes = pendingLanes & ~suspendedLanes;
    if (unblockedLanes !== NoLanes) {
      nextLanes = getHighestPriorityLanes(unblockedLanes);
    } else {
      // 进入到这里，表示目前的任务中已经没有了未被阻塞的任务
      // 需要从挂起的任务中找出任务优先级最高的执行
      if (pingedLanes !== NoLanes) {
        nextLanes = getHighestPriorityLanes(pingedLanes);
      }
    }
  }
```
如果pengdingLanes不为空，那么则会从pengdingLanes中取出未闲置将要处理的lanes，例如：

```javascript
当前pendingLanes为:    = 0b0100000000000000000000000010001 
```
最左边的1的位置为闲置位置，代表了闲置任务，闲置任务优先级最低，需要处理完所有其它优先级的任务后，再处理闲置任务。

```javascript
NonIdleLanes          = 0b0001111111111111111111111111111
```

NonIdleLanes表示了所有未闲置的1的位置，使用&符号运算（同位比较，值都为1，则结果为1，否则为0），取出未闲置的任务：

```javascript
结果为:                = 0b0000000000000000000000000010001 = 17
```
如果有未闲置的lanes，那么会优先找到未闲置lanes中未被阻塞的lane，如果没找到，则会从挂起的lanes中找到优先级最高的lane。

如果没有未闲置的lanes，则会从闲置的lanes中优先找未被阻塞的lane，如果没找到，则从闲置lanes中找到所有挂起的lanes，从中找出优先级最高的lane。

以上都没找到的话，则会返回一个空，跳出更新：

```javascript
//从pendingLanes中找不到有任务，则返回一个空
if (nextLanes === NoLanes) {
    return NoLanes;
}
```
如果当前有任务正在执行，那么wipLanes就不等于空，需要将新的任务的优先级与正在执行的任务的优先级进行比较。如果新任务比正在执行的任务的优先级低，那么则不会去管它，继续渲染，反之，新任务的优先级比正在执行的任务高，那么则取消当前任务，先执行新任务：
```javascript
  // wipLanes是正在执行任务的lane，nextLanes是本次需要执行的任务的lane
  // wipLanes !== NoLanes：wipLanes不为空，表示有任务正在执行
  // 如果正在渲染，突然新添加了一个任务，但是这个新任务比正在执行的任务的优先级低，那么则不会去管它，继续渲染
  // 如果新任务的优先级比正在执行的任务高，那么则取消当前任务，执行新任务
  if (
    wipLanes !== NoLanes &&
    wipLanes !== nextLanes &&
    (wipLanes & suspendedLanes) === NoLanes
  ) {
    const nextLane = getHighestPriorityLane(nextLanes);
    const wipLane = getHighestPriorityLane(wipLanes);
    if (
      nextLane >= wipLane ||
      (nextLane === DefaultLane && (wipLane & TransitionLanes) !== NoLanes)
    ) {
      return wipLanes;
    }
  }
```

再看ensureRootIsScheduled中是如何处理的：

```javascript
  const existingCallbackNode = root.callbackNode;
  ...
  const newCallbackPriority = getHighestPriorityLane(nextLanes);
  const existingCallbackPriority = root.callbackPriority;

  // nextLanes获取的是所有任务中优先级最高任务的lane
  // 那么与当前现有的任务的优先级比较，只会有两种结果：
  // 1.与现有的任务优先级一样，那么则会中断当前新任务向下的执行，重用之前现有的任务
  // 2.新任务的优先级大于现有的任务优先级，那么则会取消现有的任务的执行，优先执行优先级高的任务


  // 与现有的任务优先级一样的情况
  if (
    existingCallbackPriority === newCallbackPriority
  ) {
    return;
  }

  // 新任务的优先级大于现有的任务优先级
  // 取消现有的任务的执行
  if (existingCallbackNode != null) {
    cancelCallback(existingCallbackNode);
  }
```
现在获取到了所有任务中优先级最高的lane，和现有任务的优先级existingCallbackPriority，那么nextLanes与当前现有的任务的优先级比较，只会有两种结果：
- 与现有的任务优先级一样，那么则会中断当前新任务向下的执行，重用之前现有的任务
- 新任务的优先级大于现有的任务优先级，那么则会取消现有的任务的执行，优先执行优先级高的任务，实现高优先级任务插队

#### 任务调度开始

接下来则是开始创建任务，执行任务调度：

```javascript
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  ...

  // 开始调度任务
  // 判断新任务的优先级是否是同步优先级
  // 是则使用同步渲染模式，否则使用并发渲染模式（时间分片）
  let newCallbackNode;
  if (newCallbackPriority === SyncLane) {
    ...
    newCallbackNode = null;
  } else {
    let schedulerPriorityLevel;
    // lanesToEventPriority函数将lane的优先级转换为React事件的优先级，然后再根据React事件的优先级转换为Scheduler的优先级
    switch (lanesToEventPriority(nextLanes)) {
      case DiscreteEventPriority:
        schedulerPriorityLevel = ImmediateSchedulerPriority;
        break;
      case ContinuousEventPriority:
        schedulerPriorityLevel = UserBlockingSchedulerPriority;
        break;
      case DefaultEventPriority:
        schedulerPriorityLevel = NormalSchedulerPriority;
        break;
      case IdleEventPriority:
        schedulerPriorityLevel = IdleSchedulerPriority;
        break;
      default:
        schedulerPriorityLevel = NormalSchedulerPriority;
        break;
    }
    //将react与scheduler连接，将react产生的事件作为任务使用scheduler调度
    newCallbackNode = scheduleCallback(
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root),
    );
  }

  root.callbackPriority = newCallbackPriority;
  root.callbackNode = newCallbackNode;
}
```

判断新任务的优先级是否是同步优先级，是则使用同步渲染模式，否则使用并发渲染模式使用scheduler调度任务, 在使用并发模式时，会将lane的优先级转换为React事件的优先级，然后再根据React事件的优先级转换为Scheduler的优先级，Scheduler会根据它自己的优先级给任务做时间分片。


## 任务执行时Lane是如何工作的
当Scheduler开始任务调度，会循环taskQueue执行任务（详细请看[1-React源码解析之Scheduler](./1-React源码解析之Scheduler.md)），那么便会执行performConcurrentWorkOnRoot函数。
```javascript
function performConcurrentWorkOnRoot(root, didTimeout) {
  ...
  
  // 从所有待执行的任务中，找出优先级最高的任务
  let lanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );
  if (lanes === NoLanes) {
    // Defensive coding. This is never expected to happen.
    return null;
  }

  // shouldTimeSlice函数根据lane的优先级，决定是使用并发模式还是同步模式渲染(解决饥饿问题)
  // didTimeout判断当前任务是否是超时
  let exitStatus =
    shouldTimeSlice(root, lanes) &&
    (disableSchedulerTimeoutInWorkLoop || !didTimeout)
      ? renderRootConcurrent(root, lanes)
      : renderRootSync(root, lanes);
  ...
  return null;
}
```
可以看到在这个函数中又调用了一次getNextLanes方法，为什么这边又要调用一次？结合上下文来分析，在执行任务前，可能又产生了一个新的任务，
这个新的任务的优先级如果比将要执行的任务的优先级低，则不管继续渲染，但是如果比将要执行的任务的优先级高，那么则需要先执行这个优先级高的任务。
在每次执行任务的时候调用getNextLanes方法，就是为了在任何时候都要保证执行的任务是优先级最高的。

### 解决任务饥饿问题

```javascript
let exitStatus =
    shouldTimeSlice(root, lanes) &&
    (disableSchedulerTimeoutInWorkLoop || !didTimeout)
      ? renderRootConcurrent(root, lanes)
      : renderRootSync(root, lanes);
```
这个判断就是解决任务饥饿问题的关键。

didTimeout表示当前任务是否过期，如果过期则会进入同步模式执行。

接着我们来看一下shouldTimeSlice函数。
```javascript
export function shouldTimeSlice(root: FiberRoot, lanes: Lanes) {

  // 检查当前任务的lane是否在已过期的lanes中
  // 如果在，则为了防止饥饿问题，则会返回false，执行同步渲染
  if ((lanes & root.expiredLanes) !== NoLanes) {
    return false;
  }

  // 检查当前是否开启并发模式和当前使用的渲染模式是否是并发模式
  // 如果是则返回true，使用并发渲染
  if (
    allowConcurrentByDefault &&
    (root.current.mode & ConcurrentUpdatesByDefaultMode) !== NoMode
  ) {
    return true;
  }

  // 检查当前lane是否与SyncDefaultLanes有交集
  // 如果有，则会启用同步渲染模式 ，反之则使用并发模式渲染
  // InputContinuousHydrationLane  InputContinuousLane  DefaultHydrationLane  DefaultLane
  // 这四个lane都是需要使用同步模式执行的
  const SyncDefaultLanes =
    InputContinuousHydrationLane |
    InputContinuousLane |
    DefaultHydrationLane |
    DefaultLane;

  return (lanes & SyncDefaultLanes) === NoLanes;
}

```
记得我们之前在说过关于任务饥饿问题的处理，主要逻辑在markStarvedLanesAsExpired函数中，它主要的作用是为当前任务根据优先级添加过期时间，并检查未执行的任务中是否有任务过期，有任务过期则在expiredLanes中添加该任务的lane，在后续该任务的执行中以同步模式执行，避免饥饿问题。

在shouldTimeSlice中会检查当前任务的lane是否在已过期的expiredLanes中，如果在，则为了防止饥饿问题，则会返回false，执行同步渲染。

如果不在，表示当前任务没有过期，则会再检查当前是否开启并发模式和当前使用的渲染模式是否是并发模式：
```javascript
if (
    allowConcurrentByDefault &&
    (root.current.mode & ConcurrentUpdatesByDefaultMode) !== NoMode
  ) {
    return true;
  }

```
满足条件则返回true，使用并发模式。

如果不满足，则会检查当前lane是否与SyncDefaultLanes有交集：
```javascript
const SyncDefaultLanes =
    InputContinuousHydrationLane |
    InputContinuousLane |
    DefaultHydrationLane |
    DefaultLane;
    
return (lanes & SyncDefaultLanes) === NoLanes;

```
SyncDefaultLanes中的四个lane都是使用同步渲染模式执行的，如果当前lane与这个四个中的一个一样，那么则会启用同步渲染模式 ，反之则使用并发模式渲染。

当任务使用同步模式执行时是无法被打断的，直到执行完成。那么任务饥饿问题也相应的被解决了。

我们默认使用concurrent模式来执行任务。

### 状态更新
使用concurrent模式执行任务，接下来便会执行renderRootConcurrent：
```javascript
function renderRootConcurrent(root: FiberRoot, lanes: Lanes) {
  ...
  prepareFreshStack(root, lanes);

  do {
    try {
      workLoopConcurrent();
      break;
    } catch (thrownValue) {
      handleError(root, thrownValue);
    }
  } while (true);
 ...

```
这个函数中，我们主要看两个地方，一个是prepareFreshStack方法，一个是workLoopConcurrent函数。

我们先来看prepareFreshStack方法：
```javascript
function prepareFreshStack(root: FiberRoot, lanes: Lanes) {
  root.finishedWork = null;
  root.finishedLanes = NoLanes;

  ...
  workInProgressRoot = root;
  workInProgress = createWorkInProgress(root.current, null);
  workInProgressRootRenderLanes = subtreeRenderLanes = workInProgressRootIncludedLanes = lanes;
  ...
  workInProgressRootSkippedLanes = NoLanes;
  workInProgressRootUpdatedLanes = NoLanes;
  workInProgressRootPingedLanes = NoLanes;

}

```
这个方法的主要作用是创建workInProgress树:
```javascript
workInProgress = createWorkInProgress(root.current, null);

```
接着将当前任务的优先级赋值给workInProgressRootRenderLanes、subtreeRenderLanes、workInProgressRootIncludedLanes这三个变量。

workInProgressRootRenderLanes表示当前是否有任务正在执行，有值则表示有任务正在执行，反之则没有任务在执行。

subtreeRenderLanes表示需要更新的fiber节点的lane的集合，在后面更新fiber节点的时候会根据这个值判断是否需要更新。

接着便进入遍历fiber树，更新fiber节点的步骤：
```javascript
  workLoopConcurrent();
          |
          v
  performUnitOfWork(workInProgress);

```

workLoopConcurrent函数主要是调用了performUnitOfWork函数，我们直接看performUnitOfWork函数：
```javascript
function performUnitOfWork(unitOfWork: Fiber): void {
 ...
  const current = unitOfWork.alternate;
 ...
  let next;
  next = beginWork(current, unitOfWork, subtreeRenderLanes);
 ...
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  if (next === null) {
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }
  ...
  ReactCurrentOwner.current = null;
}

```
performUnitOfWork函数中调用了beginWork函数，beginWork函数的作用就是判断当前fiber节点是否需要更新：
```javascript

function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
    ...
    if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;
    // didReceiveUpdate  表示是否有新的props更新，有则会设置为true，没有则是false
    if (
      oldProps !== newProps ||
      hasLegacyContextChanged() ||
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      didReceiveUpdate = true;
    } else {
      // checkScheduledUpdateOrContext函数检查当前fiber节点上的lanes是否存在于renderLanes中
      // 存在则说明当前fiber节点需要更新，不存在则不需要更新则复用之前的节点
      const hasScheduledUpdateOrContext = checkScheduledUpdateOrContext(
        current,
        renderLanes,
      );
      if (
        !hasScheduledUpdateOrContext &&
        (workInProgress.flags & DidCapture) === NoFlags
      ) {
        // No pending updates or context. Bail out now.
        didReceiveUpdate = false;
        // 复用之前的节点
        return attemptEarlyBailoutIfNoScheduledUpdate(
          current,
          workInProgress,
          renderLanes,
        );
      }
      if ((current.flags & ForceUpdateForLegacySuspense) !== NoFlags) {
        didReceiveUpdate = true;
      } else {
        didReceiveUpdate = false;
      }
    }
  } else {
    didReceiveUpdate = false;
  }
  ...
}

```
我们先看这段代码，先是判断了当前fiber上老的props与新的props是否相同，不相同则需要更新，则不需要判断当前fiber节点上的lanes是否在renderLanes上，相同表示不需要更新，则调用checkScheduledUpdateOrContext方法来判断是否需要更新：
```javascript
function checkScheduledUpdateOrContext(
  current: Fiber,
  renderLanes: Lanes,
): boolean {
  const updateLanes = current.lanes;
  if (includesSomeLane(updateLanes, renderLanes)) {
    return true;
  }
 
  if (enableLazyContextPropagation) {
    const dependencies = current.dependencies;
    if (dependencies !== null && checkIfContextChanged(dependencies)) {
      return true;
    }
  }
  return false;
}

```
可以看到checkScheduledUpdateOrContext方法中判断updateLanes和renderLanes是否有交集，如果有则返回true，没有则返回false。

回过头我们再往下看，当checkScheduledUpdateOrContext返回false，表示不需要更新，则会调用attemptEarlyBailoutIfNoScheduledUpdate函数复用之前的节点。

返回true则会根据当前fiber节点上的tag判断组件是什么类型，我们以ClassComponent为例：
```javascript
switch (workInProgress.tag) {
    ...
    case ClassComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateClassComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes,
      );
    }
    ...
}

```
接着会执行updateClassComponent方法，这里我们就不详细解析了，把这些都放到fiber解析的时候再写。

进入updateClassComponent方法后，因为我们是做的更新操作所以会调用updateClassInstance方法：

```javascript
function updateClassComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  nextProps: any,
  renderLanes: Lanes,
) {
    ...
    shouldUpdate = updateClassInstance(
      current,
      workInProgress,
      Component,
      nextProps,
      renderLanes,
    );
    ...
}

```
updateClassInstance方法主要是做了根据更新对象上的lane判断是否需要做更新状态，以及调用一些渲染前的生命周期：getDerivedStateFromProps，componentWillUpdate等。

那我们来看一下管lane的部分，是如何做更新判断的。

在updateClassInstance中会调用一个叫processUpdateQueue的方法, 我们来看一下关于lane的部分：
```javascript
export function processUpdateQueue<State>(
  workInProgress: Fiber,
  props: any,
  instance: any,
  renderLanes: Lanes,
): void {
    ...
    let update = firstBaseUpdate;
    do {
      const updateLane = update.lane;
      const updateEventTime = update.eventTime;
      // 判断更新对象上挂载的lane是否在renderLanes上，如果在，则表明当前更新对象需要做更新操作
      // 如果不在，则说明不需要，则直接重用跳过该更新对象
      if (!isSubsetOfLanes(renderLanes, updateLane)) {
        const clone: Update<State> = {
          eventTime: updateEventTime,
          lane: updateLane,

          tag: update.tag,
          payload: update.payload,
          callback: update.callback,

          next: null,
        };
        if (newLastBaseUpdate === null) {
          newFirstBaseUpdate = newLastBaseUpdate = clone;
          newBaseState = newState;
        } else {
          newLastBaseUpdate = newLastBaseUpdate.next = clone;
        }
        newLanes = mergeLanes(newLanes, updateLane);
      } else {
        if (newLastBaseUpdate !== null) {
          const clone: Update<State> = {
            eventTime: updateEventTime,
            lane: NoLane,

            tag: update.tag,
            payload: update.payload,
            callback: update.callback,

            next: null,
          };
          newLastBaseUpdate = newLastBaseUpdate.next = clone;
        }
        newState = getStateFromUpdate(
          workInProgress,
          queue,
          update,
          newState,
          props,
          instance,
        );
        const callback = update.callback;
        if (
          callback !== null &&
          update.lane !== NoLane
        ) {
          workInProgress.flags |= Callback;
          const effects = queue.effects;
          if (effects === null) {
            queue.effects = [update];
          } else {
            effects.push(update);
          }
        }
      }
      update = update.next;
      if (update === null) {
        pendingQueue = queue.shared.pending;
        if (pendingQueue === null) {
          break;
        } else {
          const firstPendingUpdate = ((lastPendingUpdate.next: any): Update<State>);
          lastPendingUpdate.next = null;
          update = firstPendingUpdate;
          queue.lastBaseUpdate = lastPendingUpdate;
          queue.shared.pending = null;
        }
      }
    } while (true);
    ...
}

```
这段代码主要是在循环取出当前fiber节点上的updateQueue中的更新对象，然后根据更新对象上挂载的lane与renderLanes比较，判断更新对象上的lane是否存在于renderLanes上，如果存在，则表示当前更新对象需要更新，如果不存在，则会重用之前的状态，跳过该更新对象。

### 消耗lane
当循环遍历完workInProgress树，那么则开始进入commit阶段，我们来看一下关键的代码：
```javascript
function commitRootImpl(root, renderPriorityLevel) {
  ...
  let remainingLanes = mergeLanes(finishedWork.lanes, finishedWork.childLanes);
  markRootFinished(root, remainingLanes);
  ...
}

```
首先将finishedWork.lanes和finishedWork.childLanes进行合并操作，获取到剩下还需要做更新的lanes，然后调用markRootFinished清空掉已经执行完成的lanes的数据，将剩下的lanes重新挂载到pendingLanes上，准备下一次的执行：
```javascript
export function markRootFinished(root: FiberRoot, remainingLanes: Lanes) {
  // 从pendingLanes中删除还未执行的lanes，那么就找到了已经执行过的lanes
  const noLongerPendingLanes = root.pendingLanes & ~remainingLanes;

  // 将剩下的lanes重新挂载到pendingLanes上，准备下一次的执行
  root.pendingLanes = remainingLanes;

  root.suspendedLanes = 0;
  root.pingedLanes = 0;

  // 从expiredLanes， mutableReadLanes， entangledLanes中删除掉已经执行的lanes
  root.expiredLanes &= remainingLanes;
  root.mutableReadLanes &= remainingLanes;

  root.entangledLanes &= remainingLanes;

  if (enableCache) {
    const pooledCacheLanes = (root.pooledCacheLanes &= remainingLanes);
    if (pooledCacheLanes === NoLanes) {
      root.pooledCache = null;
    }
  }

  const entanglements = root.entanglements;
  const eventTimes = root.eventTimes;
  const expirationTimes = root.expirationTimes;

  // 取出已经执行的lane，清空它们所有的数据
  // eventTimes中的事件触发时间，expirationTimes中的任务过期时间等
  let lanes = noLongerPendingLanes;
  while (lanes > 0) {
    const index = pickArbitraryLaneIndex(lanes);
    const lane = 1 << index;

    entanglements[index] = NoLanes;
    eventTimes[index] = NoTimestamp;
    expirationTimes[index] = NoTimestamp;

    lanes &= ~lane;
  }
}

```
当commit阶段完成后，当前任务执行成功，最后还会再调用一次ensureRootIsScheduled函数，目的就是为了保证remainingLanes如果不为空的话，则继续执行剩下的任务。
