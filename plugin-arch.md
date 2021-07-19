# openreplay 的插件架构

主程序有一个`use`方法，用来注册插件：
```js
class App {
    constructor() {
        
    }
    
    use(fn) {
        return fn(this.app, this.options)
    }
}
```

可以看到，插件是一个函数，该函数可接受2个参数：app实例和构造app实例的options选项。

## track-axios 插件分析
代码简化后是这样的：
```js
export default function (options) {
    return (app) => {
        options.instance.interceptors.request.use(onFulfilled, onRejected)
        options.instance.interceptors.response.use(onFulfilled, onRejected)
    }
}
```

可以看到该插件只接受一个参数，也就是app实例，有了这个app实例，我们就可以通过它上面的方法`app.send`来将消息发送给webworker了。

注册该插件的过程就是在axios实例上面注册两个拦截器，一个请求拦截器，一个响应拦截器。

拦截器最终会调用一个`sendFetchMessage`函数，简化后的代码如下：
```js
const sendFetchMessage = (response) => {
    let requestData = JSON.stringify(response.config.data)
    let responseData = JSON.stringify(response.data)
    
    const fullURL = response.config.baseURL + '..' // 这里是获取该请求的完整url
    
    app.send(
        Messages.Fetch(
            response.config.method,
            fullURL,
            requestData,
            responseData,
            response.status,
        )
    )
}
```
该函数就是将axios请求通过Fetch消息提交给webworker，用来记录由axios发出的请求日志。


## tracker-fetch 插件分析
代码简化后是这样的：
```js
export default function(options) {
    return (app) => {
        if (app === null) {
            return window.fetch
        }
        
        return async (input, init) => {
            if (typeof input !== 'string') {
                return window.fetch(input, init)
            }
            
            const startTime = performance.now()
            const response = await window.fetch(input, init)
            const duration = performance.now() - startTime
            
            const r = response.clone()
            r.text().then(text => app.send(
                Messages.Fetch(
                    init.method,
                    input,
                    init.body,
                    text,
                    r.status,
                    duration,
                )
            ))
        }
    }
}
```
这个插件使用起来有点特殊，因为插件fn的返回值仍然是一个函数，返回的这个函数要么是`fetch`本身，要么是`fetch`的简单包装。作用就是用该函数发起的请求，会将请求数据通过`Fetch`消息记录下来，类似于`tracker-axios`插件。

使用时是下面这样的：
```js
import Tracker from '@openreplay/tracker'
import trackerFetch from '@openreplay/tracker-fetch'

const tracker = new Tracker({
    projectKey: 'YOUR_PROJECT_KEY',
})
tracker.start()

export const fetch = tracker.use(trackerFetch({
    sessionTokenHeader: 'X-Session-ID',
    failuresOnly: true,
}))

// 使用该fetch函数发出的请求，会被记录下来
fetch('https://my.api.io/resource').then(response => response.json())
```


## tracker-assist 插件分析

这个插件还没有readme文档，估计是新增的

代码简化后如下：
```js
import Peer from 'peerjs'

export default function(options) {
    return function(app, appOptions) {
        app.attachStartCallback(function () {
            const peerID = `${app.projectKey}-${app.getSessionID()}`
            const peer = new Peer(peerID, {
                host: app.getHost(),
                port: location.protocol === 'http:' ? 80 : 443,
                path: '/assist',
            })
            
            peer.on('connection', function (conn) {})
            
            peer.on('call', function (call) {})
        })
    }
}
```

从上面的简化代码可以看出来，这个插件是向app实例注册一个成功启动的回调，然后在这个回调函数里面使用`peerjs`这个库去做一些事情。因此，这个插件必须在启动app之前注册。

使用示例：
```js
import Tracker from '@openreplay/tracker'
import trackerAssist from '@openreplay/tracker-assist'

const tracker = new Tracker({
    projectKey: 'YOUR_PROJECT_KEY',
})
tracker.use(trackerAssist({
    confirmText: 'You have a call. Do you want to answer?',
    confirmStyle: {},
}))
tracker.start()
```

进一步分析之前，我们需要先熟悉下`peerjs`这个库是干啥的。

> PeerJS provides a complete, configurable, and easy-to-use peer-to-peer API built on top of WebRTC, supporting both data channels and media streams.

可参考下面这几篇文章理解一下`peerjs`这个库：
- [Taming WebRTC with PeerJS: Making a Simple P2P Web Game](https://www.toptal.com/webrtc/taming-webrtc-with-peerjs)
- [Building an Internet-Connected Phone with PeerJS](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Build_a_phone_with_peerjs)
