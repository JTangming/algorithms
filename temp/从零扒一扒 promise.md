## 从零扒一扒 promise

在开发过程中，很多时候都在熟练的使用 Promise/A+ 规范，然而有时在使用的时候发现并不是很了解它的底层实现，下面扒一扒它的实现。

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
Promise 解决的就是异步任务处理问题，处理异步任务问题最简单的办法就是异步回调，简单举例如下（假设有一个异步任务 asyncJob1，它执行完成后要执行 asyncJob2）：

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

### Promise 原型0 -- 最简陋的代码
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

### Promise 原型1 -- 支持多个回调
有时我们会希望在某件事完成之后，可以同时做其他的多件事情，为此我们修改 Promise，增加回调队列：

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

于是在job1后可以添加多个回调

```js
var job1 = asyncJob1('monkey');
job1.then(function (result) {
    console.log('we are done1:', result);
});

job1.then(function (result) {
    console.log('we are done2:', result);
});
```

这样之后可能还不够，因为如果另外一个回调是异步处理的话，可能就没法得到结果了。比如

```js
var job1 = asyncJob1('monkey');
job1.then(function (result) {
    console.log('we are done1:', result);
});

setTimeout(function () {
    job1.then(function (result) { // then调用时已经报错了
        console.log('we are done2:', result); // 此处没有打印
    });
}, 1000);
```

我们在then中增加一个判断，如果已经resolve过了，则直接执行回调：这样处理后上面的'done2'就可以输出了

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

### Promise原型2 -- 安全性
以上的Promise返回后，外部可以直接访问then、resolve这两个方法，然而外部应该只关心then，resolve方法不应该暴露出去，防止外部调用resolve修改了Promise的状态。于是我们把代码修整如下：

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

以上只列出了修改的代码，可以看出这个改动很小，其实就是给then封装多了一层，调用的方式就变成如下：

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

### Promise原型3 -- 链式调用
截到目前为止，我们的promise原型还不能实现链式调用，比如这样调用的话，第二个then就会报错

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

看着是不是很拗口？我也觉得，我们先来看看代码实现：

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
                    // 如果回调的结果返回的是promise(有then方法), 则调用then方法并将resolve方法传入
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
                // 创建一个新的defer并返回, 并且将defer和callback同时添加到当前的pending中。
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

我们来看看'monkey'是如何级联传递下去的：

1. 首先在asyncJob1中接收了'monkey'，然后返回一个promise（标记为pJob1）
2. pJob1的then被调用，并接受cbForJob1作为回调函数，然后又返回一个新的promise（标记为ppJob1）
3. 接着ppJob1的then被调用，并接受了cbForJob2作为回调函数，它也会返回一个新的promise，不过这个promise没有被继续使用，可以忽略
4. 然后job1完成：100ms后将'monkey'设置为'monkey with job1'作为job1Result，并以该值resolve了pJob1，即作为cbForJob1的回调函数参数，调用了cbForJob1（注意它的返回值，在第6步会用到）
5. asyncJob2被调用，接受job1Result作为参数，然后返回一个promise（标记为pJob2）
6. 此时返回的promise（pJob2）就会被第4步的resolve使用，将ppJob1的resolve方法作为pJob2的then方法的参数传入
7. 然后pJob2完成：100ms后将'monkey with job1'设置为'monkey with job1 with job2'作为job2Result，并以该值resolve了pJob2，**注意pJob2的pending回调其实是ppJob1的resolve方法**，所以pJob2的resolve调用时，其实就调用了ppJob1的resolve，也就调用了cbForJob2这个回调函数（见第3步），'monkey'这个值就这样被级联传递下去了。

看到这里如果还觉得晕的话，可以尝试在示例代码中添加上一些日志输出（源码中注释的输出部分），看一下它们的执行顺序以及resolve的值是如何传递的，就容易明白了

### Promise原型4 -- 错误分支
以上的Promise都是只有成功的resolve调用，我们平时在使用的Promise都能接受3个回调：resolve、reject、progress。在ajax调用时一般都要处理成功（resolve）和失败（reject）的情况，progress相对用得较少，这里只是为了说明原理，只实现resolve和reject。

为了实现可以reject，我们要引入一个promise的状态，记录它是被resolve还是reject过

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
                // 如果回调的结果返回的是promise(有then方法), 则调用then方法并将resolve方法传入
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
                // 创建一个新的defer并返回, 并且将defer和callback同时添加到当前的pending中。
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

为了简单起见，reject的代码和resolve差不多，可以抽取一下减少多余的代码

### Promise原型5 -- 融入异步
在上面的所有调用中，resolve或reject里的回调调用都是同步的，这取决于回调的实现。如果回调本身是同步的，就可能会出问题。比如按上面的promise4的代码，把job的调用中的setTimeout去掉，就会得不到结果：

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

所以我们要确保这些调用都是异步的，这里只是简单地用setTimeout来示意处理，这样之后像上面的调用也有结果输出了。

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
                // 创建一个新的defer并返回, 并且将defer和callback同时添加到当前的pending中。
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

### 总结

本文尝试从0开始说明promise的由来及原理，promise的核心是在异步调用，以及链式调用（分别在上面的promise5、promise3示例），但是直到第5个迭代promise5也只是一个很基础的版本，没有异常处理、progress回调等等，离我们平常用的promise还差很远。
本人才疏学浅，文字和代码也比较简陋，如有错误请各位指正。