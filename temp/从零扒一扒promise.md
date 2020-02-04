## 从零扒一扒 promise

在开发过程中，很多时候都在熟练的使用 Promise/A+ 规范，然而有时在使用的时候发现并不是很了解它的底层实现，下面扒一扒它的实现。

> 本文基于 AngularJs $q

### Promises/A+规范

> Promise 是 JS 异步编程中的重要概念，异步抽象处理对象，是目前比较流行 Javascript 异步编程解决方案之一。

**术语**
- 解决 (fulfill): 指一个 promise 成功时进行的一系列操作，如状态的改变、回调的执行。虽然规范中用 fulfill 来表示解决，但在后世的 promise 实现多以 resolve 来指代之。
- 拒绝（reject): 指一个 promise 失败时进行的一系列操作。
- 拒因 (reason): 也就是拒绝原因，指在 promise 被拒绝时传递给拒绝回调的值。
- 终值（eventual value）: 所谓终值，指的是 promise 被解决时传递给解决回调的值，由于 promise 有一次性的特征，因此当这个值被传递时，标志着 promise 等待态的结束，故称之终值，有时也直接简称为值（value）。
- Promise: promise 是一个拥有 then 方法的对象或函数，其行为符合本规范。
- thenable: 是一个定义了 then 方法的对象或函数，文中译作“拥有 then 方法”。
- 异常（exception）: 是使用 throw 语句抛出的一个值。

### 异步回调
Promise 解决的就是异步任务处理问题，简单举例如下（假设有一个异步任务 asyncJob1，它执行完成后要执行 asyncJob2）：

```js
// asyncJob2 作为参数传给 asyncJob1，在它完成某些操作后调用
asyncJob1(asyncJob2)

// asyncJob1 的伪代码
function asyncJob1(callback) {
    // some task to get the 'result'
    callback(result);
}
```

这样做的问题是 callback 的控制权在 asyncJob1 里面了，并且若有多个异步任务将会有回调地狱的问题，如：

```js
asyncJob1(param, function(job1Result) {
    asyncJob2(job1Result, function (job2Result) {
        asyncJob3(job2Result, function (job3Result) {
            alert('we are done');
            // ...
        }
    });
})
```

### 控制反转
稍稍把代码修改一下：

```js
function asyncJob1() {
    // some task to get the 'result'
    // ...
    return function (callback) {
        callback(result);
    }
}
```

那么调用的方式就是：

```js
asyncJob1()(function (job1Result) {
    // ...
});
```

写代码的时候，「同步」的写法语义更容易理解，即”干完某件事后，再处理另外一件事”，通过 then 方法来实现链式调用：

```js
function asyncJob1() {
    // some task to get the 'result'
    return {
        then: function (callback) {
            callback(result);
        }
    }
}
```

可按照如下例子来调用，这样看起来就更有序了。

```js
asyncJob1(param).then(function (job1Result) {
    return asyncJob2(job1Result);
}).then(function (job2Result) {
    return asyncJob3(job3Result);
}).then(function (job3Result) {
    // finally done
});
```

带着上面的思路，接下来实现一个简版的 Promise。

### Promise simple demo
上面的例子，都是函数执行完成后同步执行回调，看下面的例子：

```js
var Promise = function () {
    var _callback, _result;

    return {
        resolve: function (result) {
            _result = result;
            if (_callback) {
                _callback(result);
            }
            _callback = null;
        },
        then: function (callback) {
            _callback = callback;
        }
    }
};
```

于是上面的回调实现就可以变成：

```js
function asyncJob1(param) {
    var promise = Promise();
    setTimeout(function monkey() {
        var result = param + ' with job1';
        promise.resolve(result);
    }, 100);
    return promise;
}

asyncJob1('monkey').then(function (result) {
    console.log('we are done', result);
});
```

### Promise 支持多个回调
有时在某件事完成之后，可以同时做其他的多件事情，为此修改 Promise，增加回调队列：

```js
var Promise = function () {
    var _pending = [], _result;

    return {
        resolve: function (result) {
            _result = result;
            if (_pending) {
                for (var i = 0, l = _pending.length; i < l; i++) {
                    _pending[i](_result);
                }
            }
            _pending = null;
        },
        then: function (callback) {
            _pending.push(callback);
        }
    }
};
```

于是在 job1 后可以添加多个回调

```js
var job1 = asyncJob1('monkey');
job1.then(function (result) {
    console.log('we are done1:', result);
});

job1.then(function (result) {
    console.log('we are done2:', result);
});
```

这样之后可能还不够，因为如果另外一个回调是异步处理的话，可能就没法得到结果了，比如：

```js
var job1 = asyncJob1('monkey');
job1.then(function (result) {
    console.log('we are done1:', result);
});

setTimeout(function () {
    job1.then(function (result) { // then 调用时已经报错了
        console.log('we are done2:', result); // 此处没有打印
    });
}, 1000);
```

可以在 then 中增加一个判断，如果已经 resolve 过了，则直接执行回调：这样处理后上面的'done2'就可以输出了

```js
var Promise = function () {
    ...
    return {
        ...
        then: function (callback) {
            if (_pending) {
                _pending.push(callback);
            } else {
                callback(_result);
            }
        }
    }
};
```

### Promise 作用域安全性
以上的 Promise 返回后，外部可以直接访问 then、resolve 这两个方法，然而外部应该只关心 then，resolve 方法不应该暴露出去，防止外部调用 resolve 修改了 Promise 的状态。代码修整如下：

```js
var Deferred = function () {
    ...
    return {
        ...
        promise: {
            then: function (callback) {
                ...
            }
        }
    }
};
```

以上只列出了修改的代码，可以看出这个改动很小，其实就是给 then 封装多了一层，调用的方式就变成如下：

```js
function asyncJob1(param) {
    var defer = Deferred();
    setTimeout(function () {
        var result = param + ' with job1';
        defer.resolve(result);
    }, 100);
    return defer.promise;
}

asyncJob1('monkey').then(function (result) {
    console.log('we are done', result);
});
```

### Promise 链式调用
截到目前为止，promise 原型还不能实现链式调用，比如这样调用的话，第二个 then 就会报错

```js
asyncJob1('monkey').then(function (job1Result) {
    return asyncJob2(job1Result);
}).then(function (job2Result) {  // <-- 此处的then会报错
    console.log('we are done', job2Result);
});
```

链式调用是promise很重要的特性，为了实现链式调用，我们要实现：
- then方法也返回一个promise
- 返回的promise必须要用传给then方法函数的返回值，来设置（resolve）自己的result。
- 传递给then方法的函数，必须返回promise或值
    - 如果返回的是promise，则必须等待这个promise处理后，才设置result
    - 如果返回的是值，则直接设置result

先来看看代码实现：

```js
var Deferred = function () {
    var _pending = [], _result;

    return {
        resolve: function (result) {
            _result = result;
            if (_pending) {
                for (var item, r, i = 0, l = _pending.length; i < l; i++) {
                    item = _pending[i];
                    r = item[1](_result);
                    // 如果回调的结果返回的是 promise(有then方法), 则调用 then 方法并将 resolve 方法传入
                    if (r && typeof r.then === 'function') {
                        r.then.call(r, item[0].resolve);
                    } else {
                        item[0].resolve(_result);
                    }
                }
            }
            _pending = null;
        },
        promise: {
            then: function (callback) {
                // 创建一个新的 defer 并返回, 并且将 defer 和 callback 同时添加到当前的 pending 中
                var defer = Deferred();
                if (_pending) {
                    _pending.push([defer, callback]);
                } else {
                    callback(_result);
                }
                return defer.promise;
            }
        }
    }
};
```

执行以下代码，我们能得到：`we are all done! monkey with job1 with job2` 的输出

```js
asyncJob1('monkey').then(function cbForJob1(job1Result) {
    return asyncJob2(job1Result);
}).then(function cbForJob2(job2Result) {
    console.log('we are all done!', job2Result);
});
```

### Promise 错误分支
以上的 Promise 都是只有成功的 resolve 调用，在使用的 Promise 都能接受 2 个回调：resolve、reject。

为了实现可以 reject，需要引入一个 promise 的状态，记录它是被 resolve 还是 reject 过。

```js
var Deferred = function () {
    var _pending = [], _result, _reason;
    var _this = {
        resolve: function (result) {
            if (_this.promise.status !== 'pending') {
                return;
            }
            _this.promise.status = 'resolved';
            _result = result;
            for (var item, r, i = 0, l = _pending.length; i < l; i++) {
                item = _pending[i];
                r = item[1](_result);
                // 如果回调的结果返回的是 promise(有then方法), 则调用 then 方法并将 resolve 方法传入
                if (r && typeof r.then === 'function') {
                    r.then.call(r, item[0].resolve, item[0].reject);
                } else {
                    item[0].resolve(_result);
                }
            }
        },
        reject: function (reason) {
            if (_this.promise.status !== 'pending') {
                return;
            }
            _this.promise.status = 'rejected';
            _reason = reason;
            for (var item, r, i = 0, l = _pending.length; i < l; i++) {
                item = _pending[i];
                r = item[2](_reason);
                if (r && typeof r.then === 'function') {
                    r.then.call(r, item[0].resolve, item[0].reject);
                } else {
                    item[0].reject(_reason);
                }
            }
        },
        promise: {
            then: function (onResolved, onRejected) {
                // 创建一个新的 defer 并返回, 并且将 defer 和 callback 同时添加到当前的 pending 中
                var defer = Deferred();
                var status = _this.promise.status;
                if (status === 'pending') {
                    _pending.push([defer, onResolved, onRejected]);
                } else if (status === 'resolved') {
                    onResolved(_result);
                } else if (status === 'rejected') {
                    onRejected(_reason);
                }
                return defer.promise;
            },
            status: 'pending'
        }
    };

    return _this;
};
```

为了简单起见，reject的代码和resolve差不多，可以抽取一下减少多余的代码。

### Promise 融入异步
在上面的所有调用中，resolve 或 reject 里的回调调用都是同步的，这取决于回调的实现。如果回调本身是同步的，就可能会出问题。

比如按上面的 promise 的代码，把 job 的调用中的 setTimeout 去掉，就会得不到结果。

```js
function asyncJob1(param, isOk) {
    var defer = Deferred();
    //setTimeout(function () {
        var result = param + ' with job1';
        if (isOk) {
            defer.resolve(result);
        } else {
            defer.reject('job1 fail');
        }
    //}, 100);
    return defer.promise;
}

function asyncJob2(param, isOk) {
    var defer = Deferred();
    //setTimeout(function () {
        var result = param + ' with job2';
        if (isOk) {
            defer.resolve(result);
        } else {
            defer.reject('job2 fail');
        }
    //}, 100);
    return defer.promise;
}

asyncJob1('monkey', true).then(function (job1Result) {
    return asyncJob2(job1Result, true);
}).then(function (job2Result) {
    console.log('we are all done!', job2Result); // 无输出
});
```

**调用时机**

resolve 和 reject 只有在执行环境堆栈仅包含平台代码时才可被调用 注1

注1 这里的平台代码指的是引擎、环境以及 promise 的实施代码。实践中要确保 resolve 和 reject 方法异步执行，且应该在 then 方法被调用的那一轮事件循环之后的新执行栈中执行。

这个事件队列可以采用“宏任务（macro - task）”机制或者“微任务（micro - task）”机制来实现。

由于 promise 的实施代码本身就是平台代码（译者注：即都是 JavaScript），故代码自身在处理在处理程序时可能已经包含一个任务调度队列。

所以我们要确保这些调用都是异步的，这里只是简单地用 setTimeout 来示意处理，这样之后像上面的调用也有结果输出了。

```js
var Deferred = function () {
    var _pending = [], _result, _reason;
    var _this = {
        resolve: function (result) {
            if (_this.promise.status !== 'pending') {
                return;
            }
            _result = result;

            setTimeout(function () {
                processQueue(_pending, _this.promise.status = 'resolved', _result);
                _pending = null;
            }, 0);
        },
        reject: function (reason) {
            if (_this.promise.status !== 'pending') {
                return;
            }
            _reason = reason;

            setTimeout(function () {
                processQueue(_pending, _this.promise.status = 'rejected', null, _reason);
                _pending = null;
            }, 0);
        },
        promise: {
            then: function (onResolved, onRejected) {
                var defer = Deferred();
                var status = _this.promise.status;
                if (status === 'pending') {
                    _pending.push([defer, onResolved, onRejected]);
                } else if (status === 'resolved') {
                    onResolved(_result);
                } else if (status === 'rejected') {
                    onRejected(_reason);
                }
                return defer.promise;
            },
            status: 'pending'
        }
    };

    function processQueue(pending, status, result, reason) {
        var item, r, i, l, callbackIndex, method, param;
        if (status === 'resolved') {
            callbackIndex = 1;
            method = 'resolve';
            param = result;
        } else {
            callbackIndex = 2;
            method = 'reject';
            param = reason;
        }
        for (i = 0, l = pending.length; i < l; i++) {
            item = pending[i];
            r = item[callbackIndex](param);
            // 如果回调的结果返回的是promise(有then方法), 则调用then方法并将resolve方法传入
            if (r && typeof r.then === 'function') {
                r.then.call(r, item[0].resolve, item[0].reject);
            } else {
                item[0][method](param);
            }
        }
    }

    return _this;
};
```

以上仅仅是简版的 Promise，离我们平常用的promise还差很远，仅仅给自己带来的一些思考。