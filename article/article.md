---
theme: hydrogen
highlight: androidstudio
---

# 前言

最近开始阅读`react-router`的源码，一开始接触到一个名为`history.js`的第三方库，后面慢慢发现`react-router`的设计中很大程度依赖于这个库，因此我先把注意力转移到`history.js`上进行学习。下面就写一下我对这个库的学习总结。

# 作用

关于`history.js`的作用，可以引述官方的介绍来说明：

> `history.js`可以令你更轻松地管理在`JavaScript`环境下运行的会话历史（`session history`）。一个`history`实例抽象了各种环境中的差异，且提供了最简洁的`API`去管理会话中的历史栈（`history stack`）、导航（`navigate`）以及持久化的状态（`persist state`）。

通常我们在管理页面路由方面有两种模式：`hash`和`history`。如果要我们手动去实现的话，`hash`需要通过用`window.onhashchange`监听`location.hash`的变化实现。`history`需要通过`window.history`和`window.onpopstate`实现。而`history.js`提供了针对两种模式下的简洁的管理方式， 我们只需要根据项目的需求获取`hash`或`history`模式下的**管理实例**。调用这个**管理实例**下统一的`API`(如`push`、`replace`等)就可以做到对`url`的动态改变。

而且，`history.js`也弥补了原生的[window.history]一些不尽人意的效果，例如`popstate`事件的触发条件：

> 调用`history.pushState()`或者`history.replaceState()`不会触发`popstate`事件. `popstate`事件只会在浏览器某些行为下触发, 比如点击后退、前进按钮(或者调用`history.back()`、`history.forward()`、`history.go()`方法)

接下来，我会采用一边介绍`API`一边分析源码的方式来揭秘`history.js`的原理。

# 用法以及源码分析

## 实例化

### 用法

首先，我们需要获取一个**会话历史管理实例**（`history instance`）来执行后续的操作。`history.js`提供了三个对应不同模式的方法来创建**会话历史管理实例**：

- **createBrowserHistory**:用于创建`history`模式下的**会话历史管理实例**
- **createHashHistory**:用于创建`hash`模式下的**会话历史管理实例**
- **createMemoryHistory**:用于无浏览器环境下，例如`React Native`和测试用例

接下来我们先以`history`模式进行分析，获取**会话历史管理实例**的代码如下：

```js
// 有两种
// 1. 通过函数创建实例
import { createBrowserHistory } from 'history';
const history = createBrowserHistory();

// 如果要创建管理iframe历史的实例，可以把iframe.window作为形参传入该函数上
const iframeHistory = createBrowserHistory({
  window: iframe.contentWindow
});

// 2. 通过导入获取对应的单例
import history from 'history/browser';
```

### 源码分析

```typescript
export function createBrowserHistory(
  options: BrowserHistoryOptions = {}
): BrowserHistory {
  let { window = document.defaultView! } = options;
  let globalHistory = window.history;

  // ...
  let history: BrowserHistory = {
    get action() {
      return action;
    },
    get location() {
      return location;
    },
    createHref,
    push,
    replace,
    go,
    back() {
      go(-1);
    },
    forward() {
      go(1);
    },
    listen(listener) {
      
    },
    block(blocker) {
      
    }
  };

  return history; 
}
```

其实`createBrowserHistory`内部就是生成`push`、`replace`、`listen`等方法，然后挂到一个刚生成的普通对象的属性上，最后把这个对象返回出去，因此**会话历史管理实例**其实就是一个包含多个属性的纯对象而已。

对于第二种获取实例的方式，我们可以看一下`history/browser`的代码：

```ts
import { createBrowserHistory } from 'history';

export default createBrowserHistory();
```

其实就是用`createBrowserHistory`创建了实例，然后把该实例导出，这样子就成为了单例。

## 获取location

### 用法

```js
// 可通过history.location获取当前路由地址信息
let location = history.location;
```

`location`是一个纯对象，代表当前路由地址，包含以下属性：

- `pathname`：等同于`window.location.pathname`
- `search`：等同于`window.location.search`
- `hash`：等同于`window.location.hash`
- `state`：当前路由地址的状态，类似但不等于`window.history.state`
- `key`：代表当前路由地址的唯一值

### 源码分析

我们看一下`hisoory.location`的相关源码

```ts
const readOnly: 
  // ts类型定义
  <T extends unknown>(obj: T) => T 
  /**
   *  __DEV__在开发环境下为true
   *  在开发环境下使用Object.freeze让obj只读，
   *  不可修改其中的属性和新增属性
   */
  = __DEV__? obj => Object.freeze(obj): obj => obj;
```

```ts
export function createBrowserHistory(): BrowserHistory {
  // 获取index和location的方法
  // index代表该历史在历史栈中的第几个
  function getIndexAndLocation(): [number, Location] {
    let { pathname, search, hash } = window.location;
    let state = globalHistory.state || {};
    return [
      state.idx,
      readOnly<Location>({
        pathname,
        search,
        hash,
        state: state.usr || null,
        key: state.key || 'default'
      })
    ];
  }

  let [index, location] = getIndexAndLocation();

  let history: BrowserHistory = {
    get location() {
      return location;
    },
    // ...
  }

  return history;
}
```

从`getIndexAndLocation`中可以看出`location`中的`pathname`、`search`、`hash`取自`window.location`。`state`和`key`分别取自`window.history.state`中的`key`、`usr`。至于`window.history.state`中的`key`、`usr`是怎么生成的，我们留在后面介绍`push`方法时介绍。

## listen

### 用法

`history.listen`用于监听当前路由（`history.location`）变化。作为形参传入的回调函数会在当前路由变化时执行。示例代码如下：

```js
// history.listen执行后返回一个解除监听的函数，该函数执行后解除对路由变化的监听
let unlisten = history.listen(({ location, action }) => {
  console.log(action, location.pathname, location.state);
});

unlisten();
```

`history.listen`中传入的回调函数中，形参是一个对象，解构该对象可以获取两个参数：
 - `location`：等同于`history.location`
 - `action`：描述触发路由变化的行为，字符串类型，有三个值：
    1. `"POP"`: 代表路由的变化是通过`history.go`、`history.back`、`history.forward`以及浏览器导航栏上的前进和后退键触发。
    2. `"PUSH"`：代表路由的变化是通过`history.push`触发的。
    3. `"REPLACE"`：代表路由的变化是通过`history.replace`触发的。
    此值可以通过`history.action`获取
 

### 源码分析

```ts
// 基于观察者模式创建一个事件中心
function createEvents<F extends Function>(): Events<F> {
  let handlers: F[] = [];

  return {
    get length() {
      return handlers.length;
    },
    // 用于注册事件的方法，返回一个解除注册的函数
    push(fn: F) {
      handlers.push(fn);
      return function() {
        handlers = handlers.filter(handler => handler !== fn);
      };
    },
    // 用于触发事件的方法
    call(arg) {
      handlers.forEach(fn => fn && fn(arg));
    }
  };
}

let listeners = createEvents<Listener>();

let history: BrowserHistory = {
  // ...
  listen(listener) {
    return listeners.push(listener);
  },
}
```

至于为啥回调函数的形参可以解构出`action`和`location`。可以看下面介绍`push`的源码分析。

## push

### 用法

用过`Vue-Router`和`React-Router`都知道，`push`就是把新的历史条目添加到历史栈上。而`history.push`也是同样的效果，实例代码如下：

```js
// 改变当前路由
history.push('/home');
// 第二参数可以指定history.location.state
history.push('/home', { some: 'state' });
// 第一参数可以是字符串，也可以是普通对象，在对象中可以指定pathname、search、hash属性
history.push({
  pathname: '/home',
  search: '?the=query'
}, {
  some: state
});
```

### 源码分析

```ts
// 上面说到history.push的第一参数除了字符串还可以是含部分特定属性的对象
// 此函数的作用在于把此对象转化为url字符串
export function createPath({
  pathname = '/',
  search = '',
  hash = ''
}: PartialPath) {
  return pathname + search + hash;
}

// 根据传入的to的数据类型创建链接字符串
function createHref(to: To) {
  return typeof to === 'string' ? to : createPath(to);
}

// 此函数用于根据传入的location获取新的history.state和url
function getHistoryStateAndUrl(
  nextLocation: Location,
  index: number
): [HistoryState, string] {
  return [
    {
      usr: nextLocation.state,
      key: nextLocation.key,
      idx: index
    },
    createHref(nextLocation)
  ];
}

function push(to: To, state?: State) {
  // Action是一个TypeScript中枚举类型的数据，有三个值：Push、Pop、Replace，
  // 分别对应字符串'PUSH'、'POP'、'PUSH'
  // Action用于描述触发路由改变行为，与history.action一样
  let nextAction = Action.Push;
  // 根据传入的to生成下一个location
  let nextLocation = getNextLocation(to, state);
  // 用于重试
  function retry() {
    push(to, state);
  }
  // allowTx主要与history.block相关，该API在后面的章节会介绍，
  // 我们这里先默认它为true然后往下阅读
  if (allowTx(nextAction, nextLocation, retry)) {
    // 根据新的location生成新的historyState和url
    // 注意此处index + 1，会赋值于historyState.idx，
    // 在globalHistory.pushState更新state后，再次调用getIndexAndLocation时，
    // 由于getIndexAndLocation是根据globalHistory.state生成index和location的，
    // 因此得出的index会比之前的多1
    let [historyState, url] = getHistoryStateAndUrl(nextLocation, index + 1);
    // 之所以用try~catch包裹着是因为iOS中限制pushState的调用次数为100次
    try {
      globalHistory.pushState(historyState, '', url);
    } catch (error) {
      // 刷新页面，该写法等同于window.location.href = url
      // 官方源码注释：此处操作会失去state，但没有明确的方法警告他们因为页面会被刷新
      window.location.assign(url);
    }
    // 触发listeners执行注册其中的回调函数
    applyTx(nextAction);
  }
}

function applyTx(nextAction: Action) {
  // 更新history.action
  action = nextAction;
  // 更新index和location，在history.pushState过后得出的index比之前的多1
  [index, location] = getIndexAndLocation();
  // 触发listeners中注册的回调函数的执行
  // 此处传入的参数{ action, location }会作为形参传入到回调函数中
  listeners.call({ action, location });
}
```

## replace

### 用法

与`Vue-Router`和`React-Router`类似，`replace`用于把当前历史栈中的历史条目替换成新的历史条目。传入的参数和`history.push`一样。这里就不展示代码实例了。

### 源码分析

```ts
function replace(to: To, state?: State) {
  // 定义行为为"REPLACE"
  let nextAction = Action.Replace;
  let nextLocation = getNextLocation(to, state);
  function retry() {
    replace(to, state);
  }

  if (allowTx(nextAction, nextLocation, retry)) {
    // 注意此处getHistoryStateAndUrl的第二参数为index，而push的是index+1
    // 是因为replace只是生成新的历史条目替代现在的历史条目
    let [historyState, url] = getHistoryStateAndUrl(nextLocation, index);

    globalHistory.replaceState(historyState, '', url);

    applyTx(nextAction);
  }
}
```

`history.replace`和`history.push`的源码基本一致，只是在`index`的更新和调用`window.history`的原生`API`方面有所不同，我这里就重复了。

## block

### 用法

`history.block`是相比于原生的`window.history`中新增的一个很好的特性。它能够在当前路由即将变化之前，执行回调函数从而阻塞甚至拒绝更改路由。看一下示例代码：

```js
let unblock = history.block(tx => {
  if (window.confirm(`你确定要离开当前页面吗?`)) {
    // 移除阻塞函数，这里必须要移除，不然该阻塞函数会一直执行
    unblock();
    // 继续执行路由更改操作
    tx.retry();
  }
});
```

页面效果如下：

![block.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/90d0942129e444959415646bd5c3807b~tplv-k3u1fbpfcp-watermark.image?)

### 源码分析

```ts
// 此方法在beforeunload事件触发时执行，beforeunload事件是在浏览器窗口关闭或刷新的时候才触发的
// 该方法中做了两件事：
//    1. 调用event.preventDefault()
//    2. 设置event.returnValue为字符串值，或者返回一个字符串值
// 此时达成的效果是，页面在刷新或关闭时会被弹出的提示框阻塞，直至用户与提示框进行交互
function promptBeforeUnload(event: BeforeUnloadEvent) {
  // 阻止默认事件
  event.preventDefault();
  // 官方注释：Chrome和部分IE需要设置返回值
  event.returnValue = '';
}
// 这里可以看出，blockers和listeners一样是一个观察者模式的事件中心
let blockers = createEvents<Blocker>();

const BeforeUnloadEventType = 'beforeunload';

let history = {
  // ...
  block(blocker) {
    // 往blockers中注册回调函数blocker
    let unblock = blockers.push(blocker);
    // blockers中被注册回调函数时，则监听'beforeunload'以阻止默认操作
    if (blockers.length === 1) {
      window.addEventListener(BeforeUnloadEventType, promptBeforeUnload);
    }

    return function() {
      unblock();
      // 当blockers中注册的回调函数数量为0时，移除事件监听
      // 官方注释：移除beforeunload事件监听以让页面文档document在pagehide事件中是可回收的
      if (!blockers.length) {
        window.removeEventListener(BeforeUnloadEventType, promptBeforeUnload);
      }
    };
  }
}
```

`history.block`的源码中，只编写了关于`beforeunload`事件监听的注册和注销。而`beforeunload`事件是在浏览器窗口关闭或刷新的时候才触发的，`beforeunload`事件触发后执行传入的回调函数`promptBeforeUnload`，此时会达到下面的效果：

![beforeunload-block.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9e463825f73f40718bdb6e5d79b85018~tplv-k3u1fbpfcp-watermark.image?)

要注意一点，上面的效果图中的弹窗内容是不可自定义的，根据`history.block`源码中的内容可以推断出，`history.block`中的形参`blocker`是不会参与**手动刷新页面**或者**跳转页面**时的阻塞交互的。`history.block`做到的是，在`blockers`阻塞事件中心的回调函数不为空时，进行上述两个操作（**手动刷新页面**或者**跳转页面**）时，就会按照上面的效果图进行阻塞，而阻塞事件中心的回调函数为空时，则不会阻塞。

这么一说，`history.block`中传入的回调函数`blocker`只会影响到`history.push`、`history.replace`、`history.go`这些`history库`中定义的用于改变当前路由的`API`。这里我们先研究`history.block`怎么影响到`history.push`和`history.replace`的。至于如何影响`history.go`的就放在下面介绍`history.go`时说明。

首先再次看一下`history.push`中的源码：

```js
function push(to: To, state?: State) {
    let nextAction = Action.Push;
    let nextLocation = getNextLocation(to, state);
    function retry() {
        push(to, state);
    }
    // 上一次说到allowTx是用于处理history.block中注册的回调函数的。
    // 这里可以看出，allowTx返回true时才会往历史栈添加新的历史条目（globalHistory.pushState）
    if (allowTx(nextAction, nextLocation, retry)) {
        let [historyState, url] = getHistoryStateAndUrl(nextLocation, index + 1);
        try {
            globalHistory.pushState(historyState, '', url);
        } catch (error) {
            window.location.assign(url);
        }
        applyTx(nextAction);
    }
}

function allowTx(action: Action, location: Location, retry: () => void) {
    return (
        !blockers.length || 
        // 此处(xxx,false)的格式用于保证无论xxx返回什么，最后该条件的值还是false
        (blockers.call({ action, location, retry }), false) 
    );
}
```
从`allowTx`可知，只有`blockers`中注册的的回调函数数量为0（`blockers.length`）时，`allowTx`才会返回`true`。当在开头**用法**中的示例代码是这么写的：

```js
let unblock = history.block(tx => {
  if (window.confirm(`你确定要离开当前页面吗?`)) {
    unblock();
    tx.retry();
  }
});
```

当`window.confirm`形成的弹出框被用户点击确定后，会进入`if`语句块，从而执行`unlock`和`retry`。`unlock`用于注销该注册函数，`retry`用于再次执行路由更改的操作。在`retry`执行后，会再次调用`push`函数，然后再次进入`allowTx`函数，然后执行被`history.block`注册的阻塞函数。

在存在多个被注册的阻塞函数，且`history.push`被调用时，会有以下流程：

1. 进入`history.push`函数
2. 进入`allowTx`函数，有两种情况：
    - `blockers`中含被阻塞回调函数时：执行其中一个注册的阻塞回调函数，然后到下面第3步👇
    - `blockers`中不含被阻塞回调函数时，`allowTx`函数返回true，然后到下面第4步👇
3. 阻塞回调函数中调用`unlock`把自己从`blockers`中移除，然后调用`retry`回到第1步，一直重复1～3直到`blockers`中的阻塞函数被清空。
4. 调用`globalHistory.pushState`更改路由

**注意：实现以上流程需要我们在每个被注册的阻塞回调函数中必须写调用`unlock`和`retry`的逻辑。**
## go

### 用法

类似的，我们可以推理出`history.go`的作用是基本当前历史条目在历史栈中的位置去前进或者后退到附近的历史条目上。如：

```js
// 下面两条语句实现的效果都一样，都是后退到前面的历史条目上
history.go(-1);
history.back();
```

### 源码分析

```ts
function go(delta: number) {
  globalHistory.go(delta);
}

let history: BrowserHistory = {
  // ...
  go,
  back() {
    go(-1);
  },
  forward() {
    go(1);
  },
}
```

从源码看出，`go`、`back`、`forward`其实就是间接性地调用`globalHistory.go`。***那么现在问题来了，使用`go`可以触发在`history.listen`和`history.block`中注册的回调函数吗？***

答案是可以的，我们看一下源码中下面这部分的代码：

```ts
const PopStateEventType = 'popstate';

window.addEventListener(PopStateEventType, handlePop);

function handlePop() {
    if (blockedPopTx) {
        blockers.call(blockedPopTx);
        blockedPopTx = null;
    } else {
        let nextAction = Action.Pop;
        let [nextIndex, nextLocation] = getIndexAndLocation();

        if (blockers.length) {
            if (nextIndex != null) {
                let delta = index - nextIndex;
                if (delta) {
                    // Revert the POP
                    blockedPopTx = {
                        action: nextAction,
                        location: nextLocation,
                        retry() {
                        go(delta * -1);
                        }
                    };

                    go(delta);
                }
            } else {
                // Trying to POP to a location with no index. We did not create
                // this location, so we can't effectively block the navigation.
                warning(
                    false,
                    // TODO: Write up a doc that explains our blocking strategy in
                    // detail and link to it here so people can understand better what
                    // is going on and how to avoid it.
                    `You are trying to block a POP navigation to a location that was not ` +
                        `created by the history library. The block will fail silently in ` +
                        `production, but in general you should do all navigation with the ` +
                        `history library (instead of using window.history.pushState directly) ` +
                        `to avoid this situation.`
                );
            }
        } else {
            applyTx(nextAction);
        }
    }
}
```
