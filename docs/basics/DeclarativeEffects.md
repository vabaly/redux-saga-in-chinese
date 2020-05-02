# 声明式 Effects

> 1. Effect 都是对象，相当于一条条指令，middleware 会根据指定做正确的事情，这个开发者不用管
> 2. 是对象的原因是为了方便测试，这个思想非常好，事件都成为了描述性的对象。
> 3. 为什么描述性对象是可行的？因为测试的时候，我们其实并不需要知道请求结果是否回来了，因为这是不可控的，每次情况都可能不同，所以也是没必要的，我们只需要知道是否发起了请求，也就是是否调用了请求函数，而描述性对象成功产生了，就说明会正确调用函数了。
> 4. 主要用的 副作用 创建器是 call, apply, cps
> 5. 为什么不直接测试请求？首先，单元测试应该是可以重复运行的，独立的，外部环境不影响测试结果，测试也不影响外部环境，如果是在测试中实际请求了，那么可能会改变服务器或数据库的状态，这样不可取；其次，好的测试应该写明实际值和期望值是什么，关于请求回来的数据是根本不能给出期望值的，如果说是模拟一个请求函数，写死返回值，那么不就是成功调用了模拟函数而已吗？既然终究是要测试是否直接调用了模拟函数，那用描述性对象来描述调用情况，即调用的函数名和参数，真正的调用执行封装进黑盒里，这个黑盒本身能够保证（也就是我们信任它）有相关描述情况就能做到执行函数，这样一来，不就能够给出期望值了吗？不就有描述更清晰的一个测试了吗？


在 `redux-saga` 的世界里，Sagas 都用 Generator 函数实现。我们从 Generator 里 yield 纯 JavaScript 对象以表达 Saga 逻辑。
我们称呼那些对象为 *Effect*。Effect 是一个简单的对象，这个对象包含了一些给 middleware 解释执行的信息。
你可以把 Effect 看作是发送给 middleware 的指令以执行某些操作（调用某些异步函数，发起一个 action 到 store，等等）。

你可以使用 `redux-saga/effects` 包里提供的函数来创建 Effect。

这一部分和接下来的部分，我们将介绍一些基础的 Effect。并见识到这些 Effect 概念是如何让 Sagas 很容易地被测试的。

Sagas 可以多种形式 yield Effect。最简单的方式是 yield 一个 Promise。

举个例子，假设我们有一个监听 `PRODUCTS_REQUESTED` action 的 Saga。每次匹配到 action，它会启动一个从服务器上获取产品列表的任务。

```javascript
import { takeEvery } from 'redux-saga/effects'
import Api from './path/to/api'

function* watchFetchProducts() {
  yield takeEvery('PRODUCTS_REQUESTED', fetchProducts)
}

function* fetchProducts() {
  const products = yield Api.fetch('/products')
  console.log(products)
}
```

在上面的例子中，我们在 Generator 中直接调用了 `Api.fetch`（在 Generator 函数中，`yield` 右边的任何表达式都会被求值，结果会被 yield 给调用者）。

`Api.fetch('/products')` 触发了一个 AJAX 请求并返回一个 Promise，Promise 会 resolve 请求的响应，
这个 AJAX 请求将立即执行。看起来简单又地道，但...

假设我们想测试上面的 generator：

```javascript
const iterator = fetchProducts()
assert.deepEqual(iterator.next().value, ??) // 我们期望得到什么？
```

我们想要检查 generator yield 的结果的第一个值。在我们的情况里，这个值是执行 `Api.fetch('/products')` 这个 Promise 的结果。
在测试过程中，执行真正的服务（real service）是一个既不可行也不实用的方法，所以我们必须 *模拟（mock）* `Api.fetch` 函数。
也就是说，我们需要将真实的函数替换为一个假的，这个假的函数并不会真的发送 AJAX 请求而只会检查是否用正确的参数调用了 `Api.fetch`（在我们的情况里，正确的参数是 `'/products'`）。

模拟使测试更加困难和不可靠。另一方面，那些只简单地返回值的函数更加容易测试，因此我们可以使用简单的 `equal()` 来检查结果。
这是编写最可靠测试用例的方法。

关于测试原则这一方面的内容可以看[这篇文章](https://75.team/post/5-questions-every-unit-test-must-answer.html)

> `equal()` 函数从本质上反映了每个单元测试必须有的两个要素，尽管如今很多单元测试并没有做到，那就是：
> 1. 函数调用产生的真实值是什么？
> 2. 你期望的值是什么？
> 如果你的测试没有反应这两个要素，那么它就是一个不严谨的单元测试

实际上我们需要的只是保证 `fetchProducts` 任务 yield 一个调用正确的函数，并且函数有着正确的参数。

相比于在 Generator 中直接调用异步函数，**我们可以仅仅 yield 一条描述函数调用的信息**。也就是说，我们将简单地 yield 一个看起来像下面这样的对象：

```javascript
// Effect -> 调用 Api.fetch 函数并传递 `./products` 作为参数
{
  CALL: {
    fn: Api.fetch,
    args: ['./products']  
  }
}
```

换句话说，Generator 将会 yield 包含 *指令* 的文本对象（plain Objects），`redux-saga` middleware 将确保执行这些指令并将指令的结果回馈给 Generator。
这样的话，在测试 Generator 时，所有我们需要做的就是，将 yield 后的对象作一个简单的 `deepEqual` 来检查它是否 yield 了我们期望的指令。

出于这样的原因，`redux-saga` 提供了一个不一样的方式来执行异步调用。

```javascript
import { call } from 'redux-saga/effects'

function* fetchProducts() {
  const products = yield call(Api.fetch, '/products')
  // ...
}
```

我们使用了 `call(fn, ...args)` 这个函数。**与前面的例子不同的是，现在我们不立即执行异步调用，相反，`call` 创建了一条描述结果的信息**。
就像在 Redux 里你使用 action 创建器，创建一个将被 Store 执行的、描述 action 的纯文本对象。
`call` 创建一个纯文本对象描述函数调用。`redux-saga` middleware 确保执行函数调用并在响应被 resolve 时恢复 generator。

这让你能容易地测试 Generator，就算它在 Redux 环境之外。因为 `call` 只是一个返回纯文本对象的函数而已。

```javascript
import { call } from 'redux-saga/effects'
import Api from '...'

const iterator = fetchProducts()

// expects a call instruction
assert.deepEqual(
  iterator.next().value,
  call(Api.fetch, '/products'),
  "fetchProducts should yield an Effect call(Api.fetch, './products')"
)
```

现在我们不需要模拟任何东西了，一个简单的相等测试就足够了。

这些 *声明式调用（declarative calls）* 的优势是，我们可以通过简单地遍历 Generator 并在 yield 后的成功的值上面做一个 `deepEqual` 测试，
就能测试 Saga 中所有的逻辑。这是一个真正的好处，因为复杂的异步操作都不再是黑盒，你可以详细地测试操作逻辑，不管它有多么复杂。

`call` 同样支持调用对象方法，你可以使用以下形式，为调用的函数提供一个 `this` 上下文：

```javascript
yield call([obj, obj.method], arg1, arg2, ...) // 如同 obj.method(arg1, arg2 ...)
```

`apply` 提供了另外一种调用的方式：

```javascript
yield apply(obj, obj.method, [arg1, arg2, ...])
```

`call` 和 `apply` 非常适合返回 Promise 结果的函数。另外一个函数 `cps` 可以用来处理 Node 风格的函数
（例如，`fn(...args, callback)` 中的 `callback` 是 `(error, result) => ()` 这样的形式，`cps` 表示的是延续传递风格（Continuation Passing Style））。

举个例子：

```javascript
import { cps } from 'redux-saga'

const content = yield cps(readFile, '/path/to/file')
```

当然你也可以像测试 call 一样测试它：

```javascript
import { cps } from 'redux-saga/effects'

const iterator = fetchSaga()
assert.deepEqual(iterator.next().value, cps(readFile, '/path/to/file') )
```

`cps` 同 `call` 的方法调用形式是一样的。

完整列表的 declarative effects 可在这里找到： [API reference](https://redux-saga.js.org/docs/api/#effect-creators)