## 前言
本文主要介绍一些React的设计思想和相关概念，不管是想要阅读源码还是想深入了解React的同学看过来呀。欢迎指出错误，一起探讨一起进步。

## 同系列文章
因为每次更新文章都要更新一遍目录太麻烦，所以本系列的文章都整合到Github，有兴趣阅读更多React源码阅读文章的同学可以用力点[这里这里这里](https://github.com/MrWindlike/React-Source-Reading)，如果觉得文章写得还行的话，对你有小小的帮助，希望可以给颗小星星给我，谢谢😜

## 异步还是同步？
我们先来探索一下经常被人们问到的一个问题吧，React中的```setState```是异步还是同步的呢？诶，先别急着回答，让我们看个栗子思考下下吧，当点击一次按钮时，下面的输出结果和最终显示的结果你觉得是什么呢？
```js
class ReactComponent extends React.Component {
  state = {
    count: 0,
  }
  
  add = () => {
    this.setState({ count: this.state.count + 1 });
    this.setState({ count: this.state.count + 1 });
    console.log(this.state.count);

    setTimeout(() => {
      this.setState({ count: this.state.count + 1 });
      this.setState({ count: this.state.count + 1 });
      console.log(this.state.count);
    }, 0);
  }
  
  render() {
    return (
      <div>
        <p>{this.state.count}</p>
        <button onClick={this.add}>+</button>
      </div>
    );
  }
}

ReactDOM.render(
  <ReactComponent/>,
  document.getElementById('root')
);
```
想好没有，下面来公布答案啦，噔噔噔噔~
```js
...
  add = () => {
    this.setState({ count: this.state.count + 1 });
    this.setState({ count: this.state.count + 1 });
    console.log(this.state.count);  // 0

    setTimeout(() => {
      this.setState({ count: this.state.count + 1 });
      this.setState({ count: this.state.count + 1 });
      console.log(this.state.count);  // 3
    }, 0);
  }
```
最终显示的结果也是3，怎么样，是不是很吃惊呢。会产生这样的结果是因为```setState```在React合成事件中是异步的，他会把多次的状态更新整合为一次，对于同一个状态的多次更新会覆盖，只执行最后一次更新，所以第一个输出的结果就为0啦。而在一些异步和原生的DOM事件中，React暂时还未做优化处理，所以是同步更新的，在上面的```setTimeout```里，执行回调时状态已经更新完一次，所以这时的```this.state.count```为1，再执行两次同步相加后的结果为3。

所以通过这个栗子我们以后回答这个问题的时候不要傻傻地回答是异步还是同步啦，要因不同的情况而定，但在React的17版本可能会全部处理为异步。

## 源码阅读技巧

在阅读源码前，我们先来学习一下阅读源码的方法吧，方便小伙伴们自己有兴趣可以更深入的阅读。阅读源码的最好方法肯定是设断点啦，这样即减少阅读错误的几率，还大大提高效率，那我们要怎样开启调试模式呢？我这里使用的编辑器为VS Code，使用其他编辑器的话请自行参考摸索下。

先配置VS Code的配置文件：
```
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "attach",
      "name": "Attach",
      "port": 9229
    }
  ]
}
```
我们可以在```package.json```在找到一句话：
```
"debug-test": "cross-env NODE_ENV=development node --inspect-brk node_modules/.bin/jest --config ./scripts/jest/config.source.js --runInBand"
```
Windows下需要把这句改为：
```
"debug-test": "cross-env NODE_ENV=development node --inspect-brk node_modules/jest/bin/jest.js --config ./scripts/jest/config.source.js --runInBand",
```
只要运行```yarn debug-test```再按F5就可以开启测试调试模式啦。
可是这样的话会运行所有的测试用例，这显然不是我们想要的，我们只要运行涉及我们想要阅读源码的那一块就可以了，所以我通常这样运行```yarn debug-test <测试用例名>```，例如```yarn debug-test ChangeEventPlugin```，这样就可以只运行我们想要跑的用例，最好自己写一个想要的测试用例，如果不懂怎么写的话可以点波关注，以后有时间会写下文章（这波广告植入是不是毫无痕迹😃）。

下面就让我们愉快地设置断点阅读代码吧

## 异步实现原理
我们一起来迷失，啊不对，遨游在浩瀚的源码中吧，看看```setState```的异步是如何实现的

我们都知道类组件都是通过继承```React.Component```来实现的，所以我们先去那里看看：
```
function Component(props, context, updater) {
  this.props = props;
  this.context = context;
  // If a component has string refs, we will assign a different object later.
  this.refs = emptyObject;
  // We initialize the default updater but the real one gets injected by the
  // renderer.
  this.updater = updater || ReactNoopUpdateQueue;
}

Component.prototype.setState = function(partialState, callback) {
  ...
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};
```

可以看到```setState```里调用```this.updater```的一个方法，而通过一句注释我们可以发现```this.updater```是通过动态注入的，线索从这里就断开啦。然后我费尽千辛万苦终于找到```updater```的定义：
```js
const classComponentUpdater = {
  ...
  enqueueSetState(inst, payload, callback) {
    const fiber = ReactInstanceMap.get(inst);
    const currentTime = requestCurrentTime();
    const expirationTime = computeExpirationForFiber(currentTime, fiber);

    const update = createUpdate(expirationTime);
    update.payload = payload;
    if (callback !== undefined && callback !== null) {
      if (__DEV__) {
        warnOnInvalidCallback(callback, 'setState');
      }
      update.callback = callback;
    }

    enqueueUpdate(fiber, update, expirationTime);  // 加入更新队列
    scheduleWork(fiber, expirationTime);  // 开始安排更新工作
  },
  ...
};
```
因为阅读源码的过程太过复杂，就不一一和大家详细讲啦，只列出主要的实现代码。
```js
function requestWork(root: FiberRoot, expirationTime: ExpirationTime) {
  addRootToSchedule(root, expirationTime);

  if (isRendering) {
    // Prevent reentrancy. Remaining work will be scheduled at the end of
    // the currently rendering batch.
    return;
  }

  if (isBatchingUpdates) {
    // 如果是合并更新的话进入这里，合成事件里的更新是合并更新
    if (isUnbatchingUpdates) {
      // ...unless we're inside unbatchedUpdates, in which case we should
      // flush it now.
      nextFlushedRoot = root;
      nextFlushedExpirationTime = Sync;
      performWorkOnRoot(root, Sync, false);
    }
    return;
  }

  // TODO: Get rid of Sync and use current time?
  if (expirationTime === Sync) {
    performSyncWork();
  } else {
    scheduleCallbackWithExpirationTime(expirationTime);
  }
}
```
因为我们知道合成事件里的```setState```的多次更新会合并成一次更新，所以```setState```运行时会跑到第二个```if```，然后```return```回去，先不执行更新，所以这就是为什么在执行完```setState```后状态并没有立刻更新的原因。

为了让大家可以更好地理解，下面就用一段伪代码大概来解释一下吧
```js
function interactiveUpdates(callback) {
    isBatchingUpdates = true;  // 先把合成更新标识符设为真
    
    // 执行事件的回调函数，如果里面有调用到setState
    // 则会发生上面所说的情况，先把更新加入更新队列
    // 再先返回不执行更新
    callback();  
    
    isBatchingUpdates = false;
    performSyncWork();  // 开始更新
}
```
这大概就是```setState```异步的实现原理，当然源码比这复杂的要多的多。

## 总结
```setState```既是异步的也是同步的，由不同情况下决定，所以使用时要小心，我建议都把他当成异步的来使用，合并更新是通过回调和更新队列来实现。

