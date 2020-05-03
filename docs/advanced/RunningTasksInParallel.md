# 同时执行多个任务

> 1. 上一章的无阻塞和这一章的同时不是一个概念，上一章的无阻塞类似于直接创建几个 Promise，没有 await，几个任务同时执行，且几个 Promise 后面的动作依然被执行，这一章的同时类似于 await Promise.all，几个任务同时执行，但是仍然阻塞后面的流程，要等待几个任务都有返回了，才继续执行
> 2. all 函数就是执行并行任务的

`yield` 指令可以很简单的将异步控制流以同步的写法表现出来，但与此同时我们将也会需要同时执行多个任务，我们不能直接这样写：

```javascript
// 错误写法，effects 将按照顺序执行
const users = yield call(fetch, '/users'),
      repos = yield call(fetch, '/repos')
```

由于第二个 effect 将会在第一个 `call` 执行完毕才开始。所以我们需要这样写：

```javascript
import { call } from 'redux-saga/effects'

// 正确写法, effects 将会同步执行
const [users, repos] = yield all([
  call(fetch, '/users'),
  call(fetch, '/repos')
])
```

当我们需要 `yield` 一个包含 effects 的数组， generator 会被阻塞直到所有的 effects 都执行完毕，或者当一个 effect 被拒绝 （就像 `Promise.all` 的行为）。
