# redux-saga 的 fork model

> 1. fork 并不是直接执行 Promise，而是类似于下面这样：
>    ```ts
>    function a () {
>       const b = new Promise(...)
>       const c = new Promise(...)
>       //... 其他逻辑
>       return Promise.all([b, c])
>    }
>    ```
>    fork 在一个函数中是非阻塞的，但如果调用这个函数的另一个函数用了 `call`，也会等待所有 fork 任务执行完毕，也就是成阻塞的了
> 2. spawn 则彻底的分离了所有的任务，彼此没有联系，就像在函数里面随意创建 promise，却从来不在 .then 里面创建

在 `redux-saga` 的世界里，你可以使用 2 个 Effects 在后台动态地 fork task

- `fork` 用来创建 *attached forks* => 关联的后台任务
- `spawn` 用来创建 *detached forks* => 分离的后台任务

## Attached forks (using `fork`)

Attached forks 通过以下规则继续附加在它们的 parent

### 结论

- Saga 只会在这之后终止：
  - 它终止了自己的指令
  - 所有附加的 forks 本身被终止

假如我们有以下情形：

```js
import { delay } from 'redux-saga'
import { fork, call, put } from 'redux-saga/effects'
import api from './somewhere/api' // app specific
import { receiveData } from './somewhere/actions' // app specific

function* fetchAll() {
  const task1 = yield fork(fetchResource, 'users')
  const task2 = yield fork(fetchResource, 'comments')
  yield call(delay, 1000)
}

function* fetchResource(resource) {
  const {data} = yield call(api.fetch, resource)
  yield put(receiveData(data))
}

function* main() {
  yield call(fetchAll)
}
```

`call(fetchAll)` 将在这之后终止:

- `fetchAll` 本身终止了, 意味着这 3 个 effect 都会被执行. 由于 `fork` effects 是非阻塞的, task 将被阻塞在 `call(delay, 1000)`

- 这 2 个被 fork 的 task 终止, 意思是在 fetch 所需的资源之后放入对应的 `receiveData` action

所以整个 task 将阻塞直到一个 1000 毫秒的 delay 被传送，并且 task1 和 task2 完成了他们的任务。

比方说，1000 毫秒的 delay 和这两个 task 还沒有完成，然后 fetchAll 在终止整个 task 之前，将一直等待直到所有被 fork 的 task 完成。

细心的读者可能会注意到 `fetchAll` saga 可以使用平行 Effect 来重写。

```js
function* fetchAll() {
  yield all([
    call(fetchResource, 'users'),     // task1
    call(fetchResource, 'comments'),  // task2,
    call(delay, 1000)
  ])
}
```

事实上，被附加的 fork 与平行 Effect 共享相同的语意：

- 在平行情況下我们执行 task
- 在所有被 launch 的 task 终止后，parent 将会终止

这也适用于其他语意（错误和取消传播）。你可以简单地把它考虑作为一个*动态平行* Effect，来理解附加 fork 的行为。

## Error 传播

按照同样的比喻，让我们来详细的检查在平行的 Effect 中是如何处理错误的

例如，假设我们有这么一个 Effect：

```js
yield all([
  call(fetchResource, 'users'),
  call(fetchResource, 'comments'),
  call(delay, 1000)
])
```

一旦其中一个子 Effect 失敗，上方的 Effect 就会失敗。
此外，未捕获的错误将会造成平行 Effect 取消所有其他 pending 中的 Effect。例如，如果 `call(fetchResource, 'users')` 发出了一个未捕获的错误，平行 Effect 将会取消其他两个 task（如果它们依然在 pending），然后从失败的调用中，以错误终止它本身。

类似于被 attach 的 forks，Saga 会在以下情况马上终止：

- 指令的 main body 拋出了一个错误

- 一个未捕获的错误通过其中一个被 attach 的 forks 抛出

所以在先前的例子中：

```js
//... imports

function* fetchAll() {
  const task1 = yield fork(fetchResource, 'users')
  const task2 = yield fork(fetchResource, 'comments')
  yield call(delay, 1000)
}

function* fetchResource(resource) {
  const {data} = yield call(api.fetch, resource)
  yield put(receiveData(data))
}

function* main() {
  try {
    yield call(fetchAll)
  } catch (e) {
    // handle fetchAll errors
  }
}
```


如果在这样的情況下，例如 `fetchAll` 被阻塞在 `call(delay, 1000)` Effect，假如 `task1` 失败了，然后整个 `fetchAll` task 将会因此失敗

- 取消所有其他在 pending 的 task。包含：
  - *main task*（`fetchAll` 的本身）：取消的意思是，取消目前的 `call(delay, 1000)` 的 Effect
  - 其他被 fork 的 task 仍然在 pending。例如我们例子中的 `task2`。

- 在 `main` 的 `catch` 内将会捕获由 `call(fetchAll)` 抛出的错误

注意，因为我们使用了一个阻塞的 call，所以我们只能从 `main` 內部 catch `call(fetchAll)` 的错误，而且我们不能直接从 `fetchAll` 捕获错误。这是首要的原则，**你不能从被 fork 的 task 捕获错误**。在一个被附加的 fork 中的错误将会导致被 fork 的 parent 被终止（就像没有方法在一个平行 Effect *内*捕捉错误一样，只能从外部通过被阻塞的平行 Effect）。

## Cancellation

取消 Saga 导致:

- *main task* 意思是当 Saga 被阻塞时，取消当前的 Effect

- 所有被附加的 fork 仍然继续执行

**WIP**

## Detached forks (using `spawn`)

Detached forks live in their own execution context. A parent doesn't wait for detached forks to terminate. Uncaught
errors from spawned tasks are not bubbled up to the parent. And cancelling a parent doesn't automatically cancel detached
forks (you need to cancel them explicitly).

In short, detached forks behave like root Sagas started directly using the `middleware.run` API.

## Detached forks（使用 `spawn`）

被分离的 fork 存活在它们本身的执行上下文中。parent 不会等待被分离的 fork 终止。从被 spawn 的 task 抛出的未捕获的错误不会冒泡到 parent，而且取消一个 parent 不会自动取消被分离的 fork（你需要明确地取消它们）。

简单来说，被分离的 fork 的行为像是使用 `middleware.run` API 直接启动 root Saga。

**WIP**
