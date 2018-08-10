# 前言
本文的主要目的是阅读源码的过程中做下笔记和分享给有需要的小伙伴，可能会有纰漏和错误，请读者自行判断，头一次写阅读代码的文章，可能写得有点乱，有什么问题欢迎一起探讨一起进步。

React的版本为16.4，主分支的代码，只贴出部分关键代码，完整代码请到Github查看。

## 同系列文章
因为每次更新文章都要更新一遍目录太麻烦，所以本系列的文章都整合到Github，有兴趣阅读更多React源码阅读文章的同学可以用力点[这里这里这里](https://github.com/MrWindlike/React-Source-Reading)，如果觉得文章写得还行的话，对你有小小的帮助，希望可以给颗小星星给我，谢谢😜

# 虚拟DOM的初始化

## React.createElement
在阅读源码前，我们先提出一个问题，React是如何将虚拟DOM转换为真实的DOM呢？有问题以后我们才会更有目标的阅读代码，下面我们就带着这个问题去思考。

在平时工作中我们常常用JSX语法来创建React元素实例，但他们最后都会通过打包工具编译成原生的JS代码，通过React.createElement来创建。例如：
```javascript
//  class ReactComponent extends React.Component {
//      render() {
//          return <p className="class">Hello React</p>;
//      }
//  }
//  以上代码会编译为：
class ReactComponent extends React.Component {
    render() {
        React.createElement(
          'p',
          { className: 'class'},
          'Hello React'
        )
    }
}

//  <ReactComponent someProp="prop" />
React.createElement(ReactComponent, { someProp: 'prop' }, null);

```
这样我们就可以创建得到React元素实例。
先来看看createElement的主要源码（部分代码省略）：
```javascript
function createElement(type, config, children) {
  let propName;

  const props = {};

  let key = null;
  let ref = null;
  let self = null;
  let source = null;

  if (config != null) {
    if (hasValidRef(config)) {
      //  如果有ref，将他取出来
      ref = config.ref;
    }
    if (hasValidKey(config)) {
      //  如果有key，将他取出来
      key = '' + config.key;
    }

    self = config.__self === undefined ? null : config.__self;
    source = config.__source === undefined ? null : config.__source;
    
    for (propName in config) {
      if (
        hasOwnProperty.call(config, propName) &&
        !RESERVED_PROPS.hasOwnProperty(propName)
      ) {
        //  将除ref，key等这些特殊的属性放到新的props对象里
        props[propName] = config[propName];
      }
    }
  }

  //  获取子元素
  const childrenLength = arguments.length - 2;
  if (childrenLength === 1) {
    props.children = children;
  } else if (childrenLength > 1) {
    const childArray = Array(childrenLength);
    for (let i = 0; i < childrenLength; i++) {
      childArray[i] = arguments[i + 2];
    }
    props.children = childArray;
  }

  //  添加默认props
  if (type && type.defaultProps) {
    const defaultProps = type.defaultProps;
    for (propName in defaultProps) {
      if (props[propName] === undefined) {
        props[propName] = defaultProps[propName];
      }
    }
  }
  
  return ReactElement(
    type,
    key,
    ref,
    self,
    source,
    ReactCurrentOwner.current,
    props,
  );
}
```
```javascript
const ReactElement = function(type, key, ref, self, source, owner, props) {
  //  最终得到的React元素
  const element = {
    // This tag allows us to uniquely identify this as a React Element
    $$typeof: REACT_ELEMENT_TYPE,

    // Built-in properties that belong on the element
    type: type,
    key: key,
    ref: ref,
    props: props,

    // Record the component responsible for creating this element.
    _owner: owner,
  };

  return element;
};
```
是不是很简单呢，主要是把我们传进去的东西组成一个React元素对象，而type就是我们传进去的组件类型，他可以是一个类、函数或字符串（如'div'）。
## ReactDom.render

虽然我们已经得到创建好的React元素，但React有是如何把React元素转换为我们最终想要的DOM呢？就是通过ReactDom.render函数啦。
```javascript
ReactDom.render(
  React.createElement(App),  
  document.getElementById('root')
 );
```
ReactDom.render的定义：

```javascript
render(
    element: React$Element<any>,
    container: DOMContainer,
    callback: ?Function,
  ) {
    return legacyRenderSubtreeIntoContainer(
      null,  /* 父组件 */
      element,  /* React元素 */
      container,  /* DOM容器 */
      false,
      callback,
    );
  }
```
legacyRenderSubtreeIntoContainer先获取到React根容器对象（只贴部分代码）：
```javascript
...
root = container._reactRootContainer = legacyCreateRootFromDOMContainer(
  container,
  forceHydrate,
);
...
```
因代码过多只贴出通过legacyCreateRootFromDOMContainer最终得到的React根容器对象：
```javascript
const NoWork = 0;

{
    _internalRoot: {
      current: uninitializedFiber,  // null
      containerInfo: containerInfo,  //  DOM容器
      pendingChildren: null,

      earliestPendingTime: NoWork,
      latestPendingTime: NoWork,
      earliestSuspendedTime: NoWork,
      latestSuspendedTime: NoWork,
      latestPingedTime: NoWork,

      didError: false,

      pendingCommitExpirationTime: NoWork,
      finishedWork: null,
      context: null,
      pendingContext: null,
      hydrate,
      nextExpirationTimeToWorkOn: NoWork,
      expirationTime: NoWork,
      firstBatch: null,
      nextScheduledRoot: null,
    },
    render: (children: ReactNodeList, callback: ?() => mixed) => Work,
    legacy_renderSubtreeIntoContainer: (
      parentComponent: ?React$Component<any, any>,
      children: ReactNodeList,
      callback: ?() => mixed
    ) => Work,
    createBatch: () => Batch
}
```
在初始化React根容器对象root后，调用root.render开始虚拟DOM的渲染过程。
```javascript
DOMRenderer.unbatchedUpdates(() => {
  if (parentComponent != null) {
    root.legacy_renderSubtreeIntoContainer(
      parentComponent,
      children,
      callback,
    );
  } else {
    root.render(children, callback);
  }
});
```

因代码量过大，就不逐一贴出详细代码，只贴出主要的函数的调用过程。

```javascript
root.render(children, callback) -> 
DOMRenderer.updateContainer(children, root, null, work._onCommit) -> 
updateContainerAtExpirationTime(
    element,
    container,
    parentComponent,
    expirationTime,
    callback,
) ->
scheduleRootUpdate(current, element, expirationTime, callback) ->
scheduleWork(current, expirationTime) ->
requestWork(root, rootExpirationTime) ->
performWorkOnRoot(root, Sync, false) ->
renderRoot(root, false) -> 
workLoop(isYieldy) ->
performUnitOfWork(nextUnitOfWork: Fiber) => Fiber | null ->
beginWork(current, workInProgress, nextRenderExpirationTime)
```
### Fiber
Fiber类型：
```javascript
type Fiber = {|
  tag: TypeOfWork,
  key: null | string,

  // The function/class/module associated with this fiber.
  type: any,
  return: Fiber | null,

  // Singly Linked List Tree Structure.
  child: Fiber | null,
  sibling: Fiber | null,
  index: number,
  ref: null | (((handle: mixed) => void) & {_stringRef: ?string}) | RefObject,

  memoizedProps: any, // The props used to create the output.
  updateQueue: UpdateQueue<any> | null,
  memoizedState: any,
  
  mode: TypeOfMode,

  effectTag: TypeOfSideEffect,
  nextEffect: Fiber | null,
  firstEffect: Fiber | null,
  lastEffect: Fiber | null,
  
  expirationTime: ExpirationTime,

  alternate: Fiber | null,

  actualDuration?: number,
  actualStartTime?: number,
  selfBaseTime?: number,
  treeBaseTime?: number,

  _debugID?: number,
  _debugSource?: Source | null,
  _debugOwner?: Fiber | null,
  _debugIsCurrentlyTiming?: boolean,
|};
```
### beginWork
按以上函数调用过程，我们来到beginWork函数，它的作用主要是根据Fiber对象的tag来对组件进行mount或update：
```javascript
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderExpirationTime: ExpirationTime,
): Fiber | null {
  if (enableProfilerTimer) {
    if (workInProgress.mode & ProfileMode) {
      markActualRenderTimeStarted(workInProgress);
    }
  }

  if (
    workInProgress.expirationTime === NoWork ||
    workInProgress.expirationTime > renderExpirationTime
  ) {
    return bailoutOnLowPriority(current, workInProgress);
  }

  //  根据组件类型来进行不同处理
  switch (workInProgress.tag) {
    case IndeterminateComponent:
      //  不确定的组件类型
      return mountIndeterminateComponent(
        current,
        workInProgress,
        renderExpirationTime,
      );
    case FunctionalComponent:
      //  函数类型的组件
      return updateFunctionalComponent(current, workInProgress);
    case ClassComponent:
      //  类类型的组件，我们这次主要看这个
      return updateClassComponent(
        current,
        workInProgress,
        renderExpirationTime,
      );
    case HostRoot:
      return updateHostRoot(current, workInProgress, renderExpirationTime);
    case HostComponent:
      return updateHostComponent(current, workInProgress, renderExpirationTime);
    case HostText:
      return updateHostText(current, workInProgress);
    case TimeoutComponent:
      return updateTimeoutComponent(
        current,
        workInProgress,
        renderExpirationTime,
      );
    case HostPortal:
      return updatePortalComponent(
        current,
        workInProgress,
        renderExpirationTime,
      );
    case ForwardRef:
      return updateForwardRef(current, workInProgress);
    case Fragment:
      return updateFragment(current, workInProgress);
    case Mode:
      return updateMode(current, workInProgress);
    case Profiler:
      return updateProfiler(current, workInProgress);
    case ContextProvider:
      return updateContextProvider(
        current,
        workInProgress,
        renderExpirationTime,
      );
    case ContextConsumer:
      return updateContextConsumer(
        current,
        workInProgress,
        renderExpirationTime,
      );
    default:
      invariant(
        false,
        'Unknown unit of work tag. This error is likely caused by a bug in ' +
          'React. Please file an issue.',
      );
  }
}
```
### updateClassComponent
updateClassComponent的作用是对未初始化的类组件进行初始化，对已经初始化的组件更新重用
```javascript
function updateClassComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  renderExpirationTime: ExpirationTime,
) {
  const hasContext = pushLegacyContextProvider(workInProgress);
  let shouldUpdate;
  if (current === null) {
    if (workInProgress.stateNode === null) {
      //  如果还没创建实例，初始化
      constructClassInstance(
        workInProgress,
        workInProgress.pendingProps,
        renderExpirationTime,
      );
      mountClassInstance(workInProgress, renderExpirationTime);

      shouldUpdate = true;
    } else {
      //  如果已经创建实例，则重用实例
      shouldUpdate = resumeMountClassInstance(
        workInProgress,
        renderExpirationTime,
      );
    }
  } else {
    shouldUpdate = updateClassInstance(
      current,
      workInProgress,
      renderExpirationTime,
    );
  }
  return finishClassComponent(
    current,
    workInProgress,
    shouldUpdate,
    hasContext,
    renderExpirationTime,
  );
}
```

### constructClassInstance
实例化类组件：
```javascript
function constructClassInstance(
  workInProgress: Fiber,
  props: any,
  renderExpirationTime: ExpirationTime,
): any {
  const ctor = workInProgress.type;  //  我们传进去的那个类
  const unmaskedContext = getUnmaskedContext(workInProgress);
  const needsContext = isContextConsumer(workInProgress);
  const context = needsContext
    ? getMaskedContext(workInProgress, unmaskedContext)
    : emptyContextObject;

  const instance = new ctor(props, context);  //  创建实例
  const state = (workInProgress.memoizedState =
    instance.state !== null && instance.state !== undefined
      ? instance.state
      : null);
  adoptClassInstance(workInProgress, instance);

  if (needsContext) {
    cacheContext(workInProgress, unmaskedContext, context);
  }

  return instance;
}
```
### adoptClassInstance
```javascript
function adoptClassInstance(workInProgress: Fiber, instance: any): void {
  instance.updater = classComponentUpdater;
  workInProgress.stateNode = instance;  //  将实例赋值给stateNode属性
}
```

### mountClassInstance
下面的代码就有我们熟悉的componentWillMount生命周期出现啦，不过新版React已经不建议使用它。
```javascript
function mountClassInstance(
  workInProgress: Fiber,
  renderExpirationTime: ExpirationTime,
): void {
  const ctor = workInProgress.type;

  const instance = workInProgress.stateNode;
  const props = workInProgress.pendingProps;
  const unmaskedContext = getUnmaskedContext(workInProgress);

  instance.props = props;
  instance.state = workInProgress.memoizedState;
  instance.refs = emptyRefsObject;
  instance.context = getMaskedContext(workInProgress, unmaskedContext);

  let updateQueue = workInProgress.updateQueue;
  if (updateQueue !== null) {
    processUpdateQueue(
      workInProgress,
      updateQueue,
      props,
      instance,
      renderExpirationTime,
    );
    instance.state = workInProgress.memoizedState;
  }

  const getDerivedStateFromProps = workInProgress.type.getDerivedStateFromProps;
  if (typeof getDerivedStateFromProps === 'function') {
    //  React新的生命周期函数
    applyDerivedStateFromProps(workInProgress, getDerivedStateFromProps, props);
    instance.state = workInProgress.memoizedState;
  }

  if (
    typeof ctor.getDerivedStateFromProps !== 'function' &&
    typeof instance.getSnapshotBeforeUpdate !== 'function' &&
    (typeof instance.UNSAFE_componentWillMount === 'function' ||
      typeof instance.componentWillMount === 'function')
  ) {
    //  如果没有使用getDerivedStateFromProps而使用componentWillMount，兼容旧版
    callComponentWillMount(workInProgress, instance);
    updateQueue = workInProgress.updateQueue;
    if (updateQueue !== null) {
      processUpdateQueue(
        workInProgress,
        updateQueue,
        props,
        instance,
        renderExpirationTime,
      );
      instance.state = workInProgress.memoizedState;
    }
  }

  if (typeof instance.componentDidMount === 'function') {
    workInProgress.effectTag |= Update;
  }
}
```
### finishClassComponent
调用组件实例的render函数获取需渲染的子元素，并把子元素进行处理为Fiber类型，处理state和props：
```javascript
function finishClassComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  shouldUpdate: boolean,
  hasContext: boolean,
  renderExpirationTime: ExpirationTime,
) {
  markRef(current, workInProgress);

  const didCaptureError = (workInProgress.effectTag & DidCapture) !== NoEffect;

  if (!shouldUpdate && !didCaptureError) {
    if (hasContext) {
      invalidateContextProvider(workInProgress, false);
    }

    return bailoutOnAlreadyFinishedWork(current, workInProgress);
  }

  const ctor = workInProgress.type;
  const instance = workInProgress.stateNode;

  ReactCurrentOwner.current = workInProgress;
  let nextChildren;
  if (
    didCaptureError &&
    (!enableGetDerivedStateFromCatch ||
      typeof ctor.getDerivedStateFromCatch !== 'function')
  ) {
    nextChildren = null;

    if (enableProfilerTimer) {
      stopBaseRenderTimerIfRunning();
    }
  } else {
    if (__DEV__) {
      ...
    } else {
    //  调用render函数获取子元素
      nextChildren = instance.render();
    }
  }

  workInProgress.effectTag |= PerformedWork;
  if (didCaptureError) {
    reconcileChildrenAtExpirationTime(
      current,
      workInProgress,
      null,
      renderExpirationTime,
    );
    workInProgress.child = null;
  }
  //  把子元素转换为Fiber类型
  //  如果子元素数量大于一（即为数组）的时候，
  //  返回第一个Fiber类型子元素
  reconcileChildrenAtExpirationTime(
    current,
    workInProgress,
    nextChildren,
    renderExpirationTime,
  );
  //  处理state
  memoizeState(workInProgress, instance.state);
  //  处理props
  memoizeProps(workInProgress, instance.props);

  if (hasContext) {
    invalidateContextProvider(workInProgress, true);
  }

  //  返回Fiber类型的子元素给beginWork函数，
  //  一直返回到workLoop函数（看上面的调用过程）
  return workInProgress.child;
}
```

### workLoop
我们再回头看下workLoop和performUnitOfWork函数（看上面的函数调用过程），当benginWork对不同类型组件完成相应处理返回子元素后，workLoop继续通过performUnitOfWork来调用benginWork对子元素进行处理，从而遍历虚拟DOM树：
```javascript
function workLoop(isYieldy) {
  if (!isYieldy) {
    while (nextUnitOfWork !== null) {
      //  遍历整棵虚拟DOM树
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    }
  } else {
    while (nextUnitOfWork !== null && !shouldYield()) {
      //  遍历整棵虚拟DOM树
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    }

    if (enableProfilerTimer) {
      pauseActualRenderTimerIfRunning();
    }
  }
}
```
### performUnitOfWork
这个函数很接近把虚拟DOM转换为真实DOM的代码啦，当遍历完成一颗虚拟DOM的子树后（beginWork返回null，即已没有子元素），调用completeUnitOfWork函数开始转换：
```javascript
function performUnitOfWork(workInProgress: Fiber): Fiber | null {
  const current = workInProgress.alternate;

  startWorkTimer(workInProgress);

  let next;
  if (enableProfilerTimer) {
    if (workInProgress.mode & ProfileMode) {
      startBaseRenderTimer();
    }

    next = beginWork(current, workInProgress, nextRenderExpirationTime);

    if (workInProgress.mode & ProfileMode) {
      recordElapsedBaseRenderTimeIfRunning(workInProgress);
      stopBaseRenderTimerIfRunning();
    }
  } else {
    next = beginWork(current, workInProgress, nextRenderExpirationTime);
  }

  if (next === null) {
    next = completeUnitOfWork(workInProgress);
  }

  ReactCurrentOwner.current = null;

  return next;
}
```
### completeUnitOfWork
此函数作用为先将当前Fiber元素转换为真实DOM节点，然后在看有无兄弟节点，若有则返回给上层函数处理完后再调用此函数进行转换；否则查看有无父节点，若有则转换父节点。

```javascript
function completeUnitOfWork(workInProgress: Fiber): Fiber | null {
  while (true) {
    const current = workInProgress.alternate;

    const returnFiber = workInProgress.return;
    const siblingFiber = workInProgress.sibling;

    if ((workInProgress.effectTag & Incomplete) === NoEffect) {
      //  调用completeWork转换虚拟DOM
      let next = completeWork(
        current,
        workInProgress,
        nextRenderExpirationTime,
      );
      stopWorkTimer(workInProgress);
      resetExpirationTime(workInProgress, nextRenderExpirationTime);

      if (next !== null) {
        stopWorkTimer(workInProgress);
        return next;
      }

      //  处理完当前节点后
      if (siblingFiber !== null) {
        //  如果有兄弟节点，则将其返回
        return siblingFiber;
      } else if (returnFiber !== null) {
        //  没有兄弟节点而有父节点，则处理父节点
        workInProgress = returnFiber;
        continue;
      } else {
        return null;
      }
    } else {
      ...
  }
  
  return null;
}
```

### completeWork
根据Fiber的类型进行处理和抛出错误，我们主要看HostComponent类型。对HostComponent类型的处理主要是更新属性，然后通过createInstance创建DOM节点，并添加进父节点。
```javascript
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderExpirationTime: ExpirationTime,
): Fiber | null {
  const newProps = workInProgress.pendingProps;

  if (enableProfilerTimer) {
    if (workInProgress.mode & ProfileMode) {
      recordElapsedActualRenderTime(workInProgress);
    }
  }

  switch (workInProgress.tag) {
    ...
    case HostComponent: {
      popHostContext(workInProgress);
      const rootContainerInstance = getRootHostContainer();
      const type = workInProgress.type;
      if (current !== null && workInProgress.stateNode != null) {
        //  更新属性
        const oldProps = current.memoizedProps;
        const instance: Instance = workInProgress.stateNode;
        const currentHostContext = getHostContext();
        const updatePayload = prepareUpdate(
          instance,
          type,
          oldProps,
          newProps,
          rootContainerInstance,
          currentHostContext,
        );

        updateHostComponent(
          current,
          workInProgress,
          updatePayload,
          type,
          oldProps,
          newProps,
          rootContainerInstance,
          currentHostContext,
        );

        if (current.ref !== workInProgress.ref) {
          markRef(workInProgress);
        }
      } else {
        if (!newProps) {
          ...
          return null;
        }

        const currentHostContext = getHostContext();
        let wasHydrated = popHydrationState(workInProgress);
        if (wasHydrated) {
          if (
            prepareToHydrateHostInstance(
              workInProgress,
              rootContainerInstance,
              currentHostContext,
            )
          ) {
            markUpdate(workInProgress);
          }
        } else {
          //  创建并返回DOM元素
          let instance = createInstance(
            type,
            newProps,
            rootContainerInstance,
            currentHostContext,
            workInProgress,
          );

          //  将此DOM节点添加进父节点
          appendAllChildren(instance, workInProgress);

          if (
            finalizeInitialChildren(
              instance,
              type,
              newProps,
              rootContainerInstance,
              currentHostContext,
            )
          ) {
            markUpdate(workInProgress);
          }
          workInProgress.stateNode = instance;
        }

        if (workInProgress.ref !== null) {
          // If there is a ref on a host node we need to schedule a callback
          markRef(workInProgress);
        }
      }
      return null;
    }
    ...
  }
}
```
### createInstance

```javascript
function createInstance(
  type: string,
  props: Props,
  rootContainerInstance: Container,
  hostContext: HostContext,
  internalInstanceHandle: Object,
): Instance {
  let parentNamespace: string;
  if (__DEV__) {
    ...
  } else {
    parentNamespace = ((hostContext: any): HostContextProd);
  }
  const domElement: Instance = createElement(
    type,
    props,
    rootContainerInstance,
    parentNamespace,
  );
  precacheFiberNode(internalInstanceHandle, domElement);
  updateFiberProps(domElement, props);
  return domElement;
}
```

### createElement
```
function createElement(
  type: string,
  props: Object,
  rootContainerElement: Element | Document,
  parentNamespace: string,
): Element {
  let isCustomComponentTag;

  const ownerDocument: Document = getOwnerDocumentFromRootContainer(
    rootContainerElement,
  );
  let domElement: Element;
  let namespaceURI = parentNamespace;
  if (namespaceURI === HTML_NAMESPACE) {
    namespaceURI = getIntrinsicNamespace(type);
  }
  if (namespaceURI === HTML_NAMESPACE) {

    if (type === 'script') {
      const div = ownerDocument.createElement('div');
      div.innerHTML = '<script><' + '/script>'; // eslint-disable-line
      const firstChild = ((div.firstChild: any): HTMLScriptElement);
      domElement = div.removeChild(firstChild);
    } else if (typeof props.is === 'string') {
      domElement = ownerDocument.createElement(type, {is: props.is});
    } else {
      domElement = ownerDocument.createElement(type);
    }
  } else {
    domElement = ownerDocument.createElementNS(namespaceURI, type);
  }

  return domElement;
}
```

# 总结
到此为止，“React是如何将虚拟DOM转换为真实DOM”的问题就被解决啦。我们来回顾一下。
1.  首先我们要通过React.createElement函数来将我们定义好的组件进行转换为React元素
2.  将创建好的React元素通过调用ReactDom.render来进行渲染
3.  ReactDom.render调用后先创建根对象root，然后调用root.render
4.  然后经过若干函数调用，来到workLoop函数，它将遍历虚拟DOM树，将下一个需要处理的虚拟DOM传给performUnitOfWork，performUnitOfWork再将虚拟DOM传给beginWork后，beginWork根据虚拟DOM的类型不同进行相应处理，并对儿子进行处理为Fiber类型，为Fiber类型虚拟DOM添加父节点、兄弟节点等待细节，已方便遍历树。
5.  beginWork处理完后返回需要处理的子元素再继续处理，直到没有子元素（即返回null），此时performUnitOfWork调用completeUnitOfWork处理这颗虚拟DOM子树，将其转换为真实DOM。
6.  最后所有的虚拟DOM都将转为真实DOM。

除此之外还有很多细节等待我们去研究，比如说React是如何更新和删除虚拟DOM的？setState的异步实现原理是怎样的？以后再写文和大家分析和探讨。


