# openreplay-analysis
openreplay代码分析

## 项目介绍

### OpenReplay
[OpenReplay](https://github.com/openreplay/openreplay) 是一款web会话录制与回放的整套解决方案，包含前端tracker、dashboard以及后端数据处理于一体，并可一键部署到kubernetes中。

### OpenReplay-Analysis (本项目)
理清OpenReplay项目中的代理逻辑，方面进行二次开发及修复bug。


## 目录结构

```
- openreplay
  - api 前端dashboard的接口，python实现
  - backend 后端数据收集与处理，golang实现
  - ee 企业版相关代码
  - frontend 前端dashboard，react实现
  - scripts 部署脚本
  - sourcemap-uploader 暂未分析
  - tracker 前端tracker
```

## 接口

### tracker 中涉及的接口

只有3个，相对比较直观：
1. /v1/web/not-started：客户端api不满足要求，无法录制(主程序调用)
2. /v1/web/start：客户端开始录制(主程序调用，返回的token会传给webworker，后续的数据上传需要)
3. /v1/web/i：发送录制数据给后端(webworker调用)

这3个接口的实现位于 `backend/services/http/main.go` 中

### frontend 中涉及的接口

接口较多，后续会进行分析

比如，
1. /api/account
2. /api/client
3. /api/...

接口代码位于 `api/chalicelib/blueprints/bg_core.py` 中


## tracker实现逻辑

tracker是前端数据收集器，由核心和插件组成，这里主要分析核心的代码，也就是`tracker/tracker`部分。

这部分代码的目录结构如下：
```
- tracker/src
  - main
    - app app核心
    - modules 各个模块
  - messages 消息格式定义
  - webworker worker线程，用来发送数据
```

`messages`下面定义了`tracker`支持的所有消息格式及编码实现，比如`boolean/uint/int/string`数据都会编码到一个`Uint8Array`中，具体编码方式可以查看`/tracker/src/messages/writer.ts`。

`messages/index.ts`中定义了所有的消息格式及对应的编码方法
```
80: BatchMeta(pageNo, firstIndex, timestamp)
0: Timestamp(timestamp)
4: SetPageLocation(url, referrer, navigationStart)
5: SetViewportSize(width, height)
6: SetViewportScroll(x, y)
7: CreateDocument()
8: CreateElementNode(id, parentID, index, tag, svg)
9: CreateTextNode(id, parentID, index)
10: MoveNode(id, parentID, index)
11: RemoveNode(id)
12: SetNodeAttribute(id, name, value)
13: RemoveNodeAttribute(id, name)
14: SetNodeData(id, data)
16: SetNodeScroll(id, x, y)
17: SetInputTarget(id, label)
18: SetInputValue(id, value, mask)
19: SetInputChecked(id, checked)
20: MouseMove(x, y)
21: MouseClick(id, hesitationTime, label)
22: ConsoleLog(level, value)
23: PageLoadTiming(requestStart, responseStart, responseEnd, domContentLoadedEventStart,domContentLoadedEventEnd, loadEventStart, loadEventEnd, firstPaint, firstContentfulPaint)
24: PageRenderTiming(speedIndex, visuallyComplete, timeToInteractive)
25: JSException(name, message, payload)
27: RawCustomEvent(name, payload)
28: UserID(id)
29: UserAnonymousID(id)
30: Metadata(key, value)
37: CSSInsertRule(id, rule, index)
38: CSSDeleteRule(id, index)
39: Fetch(method, url, request, response, status, timestamp, duration)
40: Profiler(name, duration, args, result)
41: OTable(key, value)
42: StateAction(type)
44: Redux(action, state, duration)
45: Vuex(mutation, state)
46: MobX(type, payload)
47: NgRx(action, state, duration)
48: GraphQL(operationKind, operationName, variables, response)
49: PerformanceTrack(frames, ticks, totalJSHeapSize, usedJSHeapSize)
53: ResourceTiming(timestamp, duration, ttfb, headerSize, encodedBodySize, decodedBodySize, url, initiator)
54: ConnectionInformation(downlink, type)
55: SetPageVisibility(hidden)
59: LongTask(timestamp, duration, context, containerType, containerSrc, containerId, containerName)
60: SetNodeAttributeURLBased(id, name, value, baseURL)
61: SetCSSDataURLBased(id, data, baseURL)
63: TechnicalInfo(type, value)
64: CustomIssue(name, payload)
65: PageClose()
67: CSSInsertRuleURLBased(id, rule, index, baseURL)
```

### webworker

webworker主要用来发送收集到的数据给服务器(调用/v1/web/i接口)，与主程序通过postMessage进行通信。

主程序向webworker发送的消息有：
```
1. this.worker.postMessage(null)
发送`null`数据给 webworker，在`beforeunload`和`mouseleave`事件触发时调用，表示将已经发往 webworker 的数据马上提交给服务器
注：这里是不是会丢失一些数据呢？更好的做法是调用 this.commit()
也有可能是 this.commit() 花费的时间过多，不适合在这个场景下调用？

注：commit() 只是把数据发给 webworker，但不会立即上传给服务器，因为 commit() 的数据需要依赖 webworker 内部的定时器去上传，而这里需要马上上传数据，所以不能使用 commit() 方法，而是必须直接调用 send()

2. this.worker.postMessage(messages)
`commit()`私有方法调用，表示将收集到的数据发送给 webworker，准备发往服务器，具体发送时间由 webworker 内部的定时器决定

3. this.worker.postMessage({ingestPoint, pageNo, startTimestamp, connAttempCount, connAttempGap})
`tracker.start()`内部调用，传递一些配置数据给 webworker，同时启动 webworker 内部的定时器，周期性(20秒)的调用 `send()` 方法上传数据

4. this.worker.postMessage({ token })
`tracker.start()`内部调用，传递一些配置数据给 webworker，token 来自于`/v1/web/start/`接口返回

5. this.worker.postMessage('stop')
`tracker.stop()`内部调用，主程序会给 webworker 发送一个'stop'字符串，表示停止
```

webworker向主程序发送的消息有：
```
1. self.postMessage(null)
webworker中出现不可恢复的错误(比如达到重试次数还没有上传成功，或者接口出现500类错误)，需要停止

2. self.postMessage('restart')
出现授权失败或页面不可见超过5分钟时触发
```

webworker内部的实现细节有：
1. writer作为消息缓存区，大小设置为`4 * 1e5`，也就是`400kB`，表示每批数据提交最多为`400kB`。注意内部有队列
2. 内部定时器为20秒，如果不通过`this.worker.postMessage(null)`立即上传的话，所有传给webworker的数据都按照这个定时器进行上传。
3. 主程序传给webworker的所有消息，都会被编码到内部缓冲区writer中，具体编码方法参考每个`Message.encode()`方法


### 主程序

#### app实例构造流程

我们在使用 OpenReplay 的时候，通常是下面这样：
```js
const tracker = new Tracker({
  projectKey: '',
  ingestPoint: 'https://ingest.openreplay.com/',
  obscureInputNumbers: false,
  obscureInputEmails: false,
  defaultInputMode: 0,
  onStart() {},
});
tracker.start()
```

首先是实例化一个`Tracker`实例，逻辑如下：
1. 检测客户端api是否满足要求，比如`Map`、`Set`、`MutationObserver`、`performance`、`Blob`、`Worker`等api是否存在，因为tracker代码依赖这些api。
如果客户端不满足这些要求，则直接调用`/v1/web/not-started`接口告知服务器。
   
2. `new App(projectKey, sessionToken, options)`实例化一个`App`实例
App内部有3个核心组件：Nodes/Observer/Ticker
在这一步中，还会初始化好webworker，并设置好worker与主程序的通信方式(绑定message事件)

3. 初始化各个模块，比如`Viewport`模块、`CSSRules`模块、`Input`模块、`Scroll`模块等。
并且把`tracker`实例保存在`window.__OPENREPLAY__`变量上。


接着，调用`tracker.start()`启动tracker实例
tracker的start方法是封装了app.start方法，调用start方法时，主程序通过postMessage将配置数据传给worker来启动worker内部的定时器开始工作，同时启动app内部的三个组件：observer/ticker开始工作。
再然后，主程序调用`/v1/web/start`接口获取到本次会话的token，传给worker，后续worker内部调用`/v1/web/i`接口时需要通过这个token进行认证。

至此，tracker启动成功。

我们接下来分析app内部的3个核心组件: Nodes、Observer、Ticker

#### Ticker

我们在`new App()`的时候，调用了如下代码：
```js
this.ticker = new Ticker(this)
this.ticker.attach(() => this.commit())
```

`Ticker`的构造器很简单，如下：
```js
class Ticker {
    constructor(app) {
        this.app = app
        this.callbacks = []
    }
    attach(callback) {
        this.callbacks.unshift(callback)
    }
    start() {
        if (this.timer === null) {
            this.timer = setInterval(() => {
                this.callbacks.forEach(cb => cb())
            }, 30)
        }
    }
    stop() {
        if (this.timer !== null) {
            clearInterval(this.timer)
            this.timer = null
        }
    }
}
```

然后，我们在启动`tracker`的时候调用了
```js
this.ticker.start();
```

可以看出来，`Ticker`功能很简单，就是一个定时器，每隔30毫秒执行一遍绑定的回调函数。在我们的代码里面，我们只在ticker上面绑定了一个回调函数，那就是`() => this.commit()`，而这个`commit`的代码如下：
```js
function commit() {
    if (this.worker && this.messages.length) {
        this.messages.unshift(new Timestamp())
        this.worker.postMessage(this.messages)
        this.messages.length = 0
    }
}
```
也就是说，每隔30毫秒，我们就把app实例上面收集的所有消息，提交给worker线程，然后我们可以通过`this.worker.postMessage(null)`主动告诉worker提交这些数据，或者等待worker内部的计时器(20秒)触发提交。

#### Nodes

nodes的相关代码只在实例化app的时候有如下代码：
```js
this.nodes = new Nodes(this.options.node_id)
```

然后在`app.stop()`的时候有如下代码：
```js
this.nodes.clear()
```

我们简化下`Nodes`相关代码，如下：
```js
class Nodes {
    constructor(node_id) {
        this.node_id = node_id
        this.nodes = []
        this.nodeCallbacks = []
        this.elementListeners = new Map()
    }
    registerNode(node) {
        
    }
    unregisterNode(node) {
        
    }
    getID(node) {}
    getNode(id) {}
}
```
这个组件主要用于处理dom树中的节点与id的一一对应，对于每一个被跟踪的dom节点，都会被绑定一个id值。这个id值会作为内部各种数组容器的下标，用来表示对应的节点。根元素`html`的id值为0，后续节点根据注册的先后顺序依次自增。

#### Observer

同样，我们在`new App()`的时候调用
```js
this.observer = new Observer(this, this.options);
```
在启动tracker的时候调用
```js
this.observer.observe();
```

简化`Observer`代码如下：
```js
class Observer {
    constructor(app, opts) {
        this.app = app
        this.options = opts
        this.observer = new MutationObserver((mutations) => {
            for (const mutation of mutations) {
                
            }
            this.comitNodes()
        })
        this.comited = []
        this.recents = []
        this.indexes = [0]
        this.attributesList = []
        this.textSet = new Set()
        this.textasked = new Set()
    }
    observe() {
        this.observer.observe(document, {
            childList: true,
            attributes: true,
            characterData: true,
            subtree: true,
            attributeOldValue: false,
            characterDataOldValue: false,
        })
        this.app.send(new CreateDocument())
    }
    _commitNode(id, node) {}
}
```
`Observer`用于跟踪整颗dom树的变化，并把变化以消息的形式记录下来发送给`this.app.messages`中，最终通过webworker发送给服务器。

这个类里面的核心应该是`_commitNode(id, node)`方法。

## 模块

各个模块的功能就是不断地给app发送消息，然后app将这些消息同步到webworker，由worker提交给服务器。服务器在后端去处理这些消息

