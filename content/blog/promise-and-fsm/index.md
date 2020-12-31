---
title: 状态机与Promise
date: "2020-12-31T20:00:00.000Z"
description: "基于状态机介绍了Promise的原理，并从零实现了Promise"
---

> *本文是关于现实Promise，如果你还不了解promise的使用。请先阅读 [MDN关于Promise的文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise)。*
>
> 文章基于自己对Promise的理解，如有错误还请指正。
>
> *在class内实现方法容易因为this的指向问题导致一些错误，同时绑定this或者在constructor里使用箭头函数会增加内存占用，所以除了`then`, `catch`, `finally`，其它辅助方法均在class外实现。*

首先 Promise的本质是有限状态机(finite-state machine)。它持有一系列的状态。对于每个**状态**，状态机都将接收各种**事件**然后执行**转换**并在转换后执行相对应的**动作**。在Promise中，**状态**即为PEDING，FULFILLED，REJECTED；**事件**为各种回调函数；**转换**为fulfill，reject方法；**动作**则是fulfill，reject在执行完状态转换之后做的事。

![promise-state-machine](https://mdn.mozillademos.org/files/8633/promises.png)

#### 状态机实现的Promise

有了这些概念之后，我们先用[javascript-state-machine](https://github.com/jakesgordon/javascript-state-machine)来实现一个基本的Promise。

```javascript
const StateMachine = require('javascript-state-machine');

const makePromise = (fn) => {
  let result;
  // 储存then的回调函数，内容为{onFulfilled, onRejected}
  let handlers = [];
  const promise = new StateMachine({
    // *状态*
    init: 'PENDING',
    // *转换*
    transitions: [
      { name: 'fulfill', from: 'PENDING', to: 'FULFILLED' },
      { name: 'reject', from: 'PENDING', to: 'REJECTED' },
    ],
    // *动作*，转换后执行的操作
    methods: {
      onFulfill: function () {
        handlers.forEach((h) => h.onFulfilled(result));
        handlers = null;
      },
      onReject: function () {
        handlers.forEach((h) => h.onRejected(result));
        handlers = null;
      },
    },
  });
  const schedule = (onFulfilled, onRejected) => {
    // 实现异步
    setTimeout(() => {
      if (promise.is('PENDING')) {
        handlers.push({ onFulfilled, onRejected });
      } else if (promise.is('FULFILLED')) {
        onFulfilled(result);
      } else {
        onRejected(result);
      }
    }, 0);
  };
  promise.then = (onFulfilled, onRejected) => {
    // 链式调用
    return makePromise((resolve, reject) => {
      schedule(
        (value) => {
          resolve(onFulfilled(value));
        },
        (reason) => {
          reject(onRejected(reason));
        }
      );
    });
  };
  // 改变值，实现值传递
  const fulfill = (value) => {
    result = value;
    promise.fulfill();
  };
  const reject = (reason) => {
    result = reason;
    promise.reject();
  };
  fn(fulfill, reject);
  return promise;
};

const p = makePromise((resolve, reject) => {
  setTimeout(() => {
    resolve('hello promise');
  }, 2000);
});
p.then((v) => {
  console.log(v);
  return v + ' again';
}).then(console.log);

```

以上代码已经能反应Promise的基本结构，除了一些细节没有实现，希望能帮助理解Promise的工作原理。整个Promise简单来说就是一个状态机，在不同状态时做出不同的相应。

**PENDING**：fulfill还未被执行，表示这个Promise的工作还未开始或者未结束。在此时调用then，会把then的`onFulfilled`和`onRejected`保存在内部的队列中。

**FULFILLED**：fulfill被执行，此时状态已不会再改变，表示这个Promise的工作已经完成，所以在fulfill被执行时，先改变内部储存的结果值`result`，再执行内部队列中的每一个`onFulfilled`。在执行这个onFulfilled的过程中，可能会执行另一个Promise(then返回的Promise)的fulfill，从而递归式地、dfs地执行。

**REJECTED**：与FULFILLED相同，唯一区别就是出现错误，reject被调用，value变成reject的错误，并调用内部队列的onFulfilled。

即然从始至终都是fulfilled来实现状态转变，那么为什么需要**resolve**方法呢？

这是因为then的回调函数中的返回值有可能是一个Promise。按照规范，如果返回一个Promise，那么需要等这个Promise的完成工作，并且用它fulfilled的值来作为结果。

#### 从零开始实现Promise

以上介绍了大部分关于Promise的内部工作原理，下面我们开始根据状态机的逻辑从零开始实现Promise。

##### 1. 状态

定义状态，内部储存以供传递的值和pending时then的添加的回调函数队列subscribers。

```javascript
const PENDING = 0;
const FULFILLED = 1;
const REJECTED = 2;

class MyPromise {
  constructor(resolver) {
    this._result = undefined;
    this._state = PENDING;
    this._subscribers = [];
  }
}
```

##### 2. 状态转换和动作

实现状态转换函数，即`fulfill`和`reject`。同时转换结束之后需要执行对应subscribers中的函数并清空。

```javascript
function fulfill(promise, value) {
  if (promise._state !== PENDING) return;

  promise._result = value;
  promise._state = FULFILLED;
  promise._subscribers.forEach((s) => handle(promise, s));
  promise._subscribers = null;
}

function reject(promise, reason) {
  if (promise._state !== PENDING) return;

  promise._result = reason;
  promise._state = REJECTED;
  promise._subscribers.forEach((s) => handle(promise, s));
  promise._subscribers = null;
}
```

`handle`用来在PENDING阶段向subscribers增加then中的`onFulfilled`和`onRejected`，在其它时候执行对应的内容。根据状态工作。

```javascript
// handler: {onFulfilled, onRejected}
function handle(promise, handler) {
  if (promise._state === PENDING) {
    promise._subscribers.push(handler);
  } else if (
    promise._state === FULFILLED &&
    typeof handler.onFulfilled === 'function'
  ) {
    handler.onFulfilled(promise._result);
  } else if (
    promise._state === REJECTED &&
    typeof handler.onRejected === 'function'
  ) {
    handler.onRejected(promise._result);
  }
}
```

##### 3. resolve

如前文所说，如果返回的是Promise或者类Promise (Thenable)，需要等待它完成工作，然后再次调用resolve来改变这个Promise的状态和结果值。如果resolve的对象不是thenable，则直接继续执行fulfill改变状态。

```javascript
function resolve(promise, value) {
  // 避免无限循环
  if (promise === value) {
    reject(promise, new TypeError("Can't resolve self"));
  }
  try {
    if (isThenable(value)) {
      const thenable = value;
      thenable.then(
        (value) => {
          resolve(promise, value);
        },
        (reason) => {
          reject(promise, reason);
        }
      );
    } else {
      fulfill(promise, value);
    }
  } catch (e) {
    reject(promise, e);
  }
}
```

辅助方法来判断这个value是不是thenable。

```javascript
function isThenable(value) {
  const t = typeof value;
  if (value !== null && (t === 'object' || t === 'function')) {
    const then = value.then;
    if (typeof then === 'function') {
      return true;
    }
  }
  return false;
}
```

##### 4. then

首先我们都知道then的callback应该在微任务中执行，所以用`schedule`方法把添加到微任务中。(以下只模拟了浏览器和node)

```javascript
function schedule(promise, onFulfilled, onRejected) {
  if (typeof queueMicrotask === 'function') {
     queueMicrotask(() => {
      handle(promise, { onFulfilled, onRejected });
    });
  } else if (process) {
    process.nextTick(() => {
    	handle(promise, { onFulfilled, onRejected });
  	});
  } else {
    setTimeout(() => {
    	handle(promise, { onFulfilled, onRejected });
  	}, 0);
  }
}
```

`then`方法需要创建一个新的Promise来实现链式调用，此处实现和前文用直接用状态机实现的Promise大同小异，核心在于包装`onFulfilled`和`onRejected`，使其绑定新创建的Promise。之后在微任务中根据状态把这一对handler放入schedule或者直接运行，可以直接由handle完成。

这里有一个小细节，判断`onFulfilled`和`onRejected`是不是function，如果不是就直接调用resolve或者reject实现值穿透。

```javascript
function then(onFulfilled, onRejected) {
  const Constructor = this.constructor;
  const parent = this;
  return new Constructor((resolve, reject) => {
    schedule(
      parent,
      (result) => {
        if (typeof onFulfilled === 'function') {
          try {
            return resolve(onFulfilled(result));
          } catch (ex) {
            return reject(ex);
          }
        } else {
          return resolve(result);
        }
      },
      (error) => {
        if (typeof onRejected === 'function') {
          try {
            return resolve(onRejected(error));
          } catch (ex) {
            return reject(ex);
          }
        } else {
          return reject(error);
        }
      }
    );
  });
}
```

顺便实现以下`catch`和`finally`。

```javascript
MyPromise.prototype.catch = function (onRejected) {
  return this.then(null, onRejected);
};

MyPromise.prototype.finally = function (fn) {
  const Constructor = this.constructor;
  const onFinally = (callback) => Constructor.resolve(fn()).then(callback);
  return this.then(
    (value) => onFinally(() => value),
    (reason) => onFinally(() => Promise.reject(reason))
  );
};
```

 ##### 5. 静态方法

Promise.resolve有幂等性，所以当参数是一个Promise的时候直接返回。

```javascript
MyPromise.resolve = function (value) {
  const Constructor = this;
  if (value instanceof Constructor) return value;
  return new Constructor((resolve, reject) => {
    resolve(value);
  });
};

MyPromise.reject = function (error) {
  const Constructor = this;
  return new Constructor((resolve, reject) => {
    reject(error);
  });
};
```

`all`，`race`，`allSettled`实现的方式都差不多，只要用`Promise.resolve`包装一下，然后在then的回调函数里判断一下result的长度。用`Promise.resolve`是为了避免值不是Promise的情况。

```JavaScript 
MyPromise.all = function (entries) {
  const Constructor = this;
  return new Constructor((resolve, reject) => {
    const l = entries.length;
    const result = [];
    let numOfResolved = 0;

    entries.forEach((p, i) => {
      Constructor.resolve(p).then(
        (value) => {
          result[i] = value;
          if (++numOfResolved === l) resolve(result);
        },
        (error) => {
          reject(error);
        }
      );
    });
  });
};

MyPromise.race = function (entries) {
  const Constructor = this;
  return new Promise((resolve, reject) => {
    for (let i = 0, l = entries.length; i < l; i++) {
      Constructor.resolve(entries[i]).then(resolve, reject);
    }
  });
};

MyPromise.allSettled = function (entries) {
  const Constructor = this;
  const onFulfilled = (value) => ({ status: 'fulfilled', value });
  const onRejected = (reason) => ({ status: 'rejected', reason });
  return Constructor.all(
    entries.map((p) => Constructor.resolve(p).then(onFulfilled, onRejected))
  );
};
```



> 参考资料：
>
> - [Wiki有限状态机](https://en.wikipedia.org/wiki/Finite-state_machine)
>
> - [javascript-state-machine](https://github.com/jakesgordon/javascript-state-machine)
>
> - [MDN关于Promise的文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise)
>
> - [Forbes Lindesay的Promise实现教程](https://www.promisejs.org/implementing/)
>
> - [polyfill of the ES6 Promise](https://github.com/stefanpenner/es6-promise)