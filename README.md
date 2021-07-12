# openreplay-analysis
openreplay代码分析

## 项目介绍

### OpenReplay
[OpenReplay](https://github.com/openreplay/openreplay) 是一款web会话录制与回放的整套解决方案，包含前端tracker与后端数据处理于一体，可一键部署到kubernetes中。

### OpenReplay-Analysis (本项目)
理清OpenReplay项目中的代理逻辑，方面进行二次开发及修复bug。


## 目录结构

```
- openreplay
  - api 网站前端的接口，python实现
  - backend 数据后端收集与处理，golang实现
  - ee 企业版相关代码
  - frontend 网站前端，react实现
  - scripts 部署脚本
  - sourcemap-uploader 暂未分析
  - tracker 前端tracker代码
```

## 接口

### tracker 中涉及的接口

只有3个，相对比较直观：
1. /v1/web/not-started：客户端api不满足要求，无法录制
2. /v1/web/start：客户端开始录制
3. /v1/web/i：发送录制数据给后端

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
  - webworker webworker
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
tracker采用主程序与webworker通信的方式交互，交互方式有：

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

writer作为消息缓存区，大小设置为`4 * 1e5`，也就是`400kB`，表示每次数据提交最多为`400kB`
