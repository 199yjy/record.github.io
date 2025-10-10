# Promise

Promise特点：
- 执行了resolve，Promise状态会变成fulfilled
- 执行了reject，Promise状态会变成rejected
- Promise状态不可逆，第一次成功就永久为fulfilled，第一次失败就永远状态为rejected
- Promise中有throw的话，就相当于执行了reject
- then接收两个回调，一个是成功回调，一个是失败回调
- 当Promise状态为fulfilled执行成功回调，为rejected执行失败回调
- 如resolve或reject在定时器里，则定时器结束后再执行then
- then支持链式调用，下一次then执行受上一次then返回值的影响

思路：
- Promise的初始状态是pending，状态一旦变为fulfilled成功或者rejected失败，就不可更改
- 对resolve和reject绑定this：确保resolve和reject的this指向永远指向当前的MyPromise实例，防止随着函数执行环境的改变而改变
- 捕捉到错误直接执行reject
- 链式调用：
    - then接收两个回调(成功，失败)，当Promise状态为fulfilled执行成功回调，为rejected执行失败回调
    - 如resolve或reject在定时器里，则定时器结束后再执行then
        - 如果state=成功，执行成功的回调
        - 如果state=失败，执行失败的回调方法
        - 如果pending，暂时保存回调任务
    - then支持链式调用，下一次then执行受上一次then返回值的影响

```js
class MyPromise {
    constructor(executor) {
        // 初始化值
        this.initValue()
        // 初始化this指向
        this.initBind()
        try {
            // 执行传进来的函数
            executor(this.resolve, this.reject)
        } catch (e) {
            // 如果捕捉到错误直接执行reject
            this.reject(e)
        }
    }

    initBind() {
        // 初始化this
        this.resolve = this.resolve.bind(this)
        this.reject = this.reject.bind(this)
    }

    initValue() {
        // 初始化值
        this.PromiseResult = null // 终值
        this.PromiseState = 'pending' // 状态

        this.onFulfilledCallbacks = [] // 保存成功回调
        this.onRejectedCallbacks = [] // 保存失败回调
    }

    resolve(value) {
        // state是不可变的
        if (this.PromiseState !== 'pending') return
        // 如果执行resolve，状态变为fulfilled
        this.PromiseState = 'fulfilled'
        // 终值为传进来的值
        this.PromiseResult = value
        // 执行保存的成功回调
        while (this.onFulfilledCallbacks.length) {
            this.onFulfilledCallbacks.shift()(this.PromiseResult)
        }
    }

    reject(reason) {
        // state是不可变的
        if (this.PromiseState !== 'pending') return
        // 如果执行reject，状态变为rejected
        this.PromiseState = 'rejected'
        // 终值为传进来的reason
        this.PromiseResult = reason
        // 执行保存的失败回调
        while (this.onRejectedCallbacks.length) {
            this.onRejectedCallbacks.shift()(this.PromiseResult)
        }
    }

    then(onFulfilled, onRejected) {
        // 接收两个回调 onFulfilled, onRejected

        // 参数校验，确保一定是函数
        onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : val => val
        onRejected = typeof onRejected === 'function' ? onRejected : reason => { throw reason }

        var thenPromise = new MyPromise((resolve, reject) => {
            const resolvePromise = cb => {
                setTimeout(() => {
                    try {
                        const x = cb(this.PromiseResult)
                        if (x === thenPromise) {
                            throw new Error('不能返回自身')
                        }
                        if (x instanceof MyPromise) {
                            // 如果返回值是Promise
                            // 如果返回值是promise对象，返回值为成功，新promise就是成功
                            // 如果返回值是promise对象，返回值为失败，新promise就是失败
                            // 谁知道返回的promise是失败成功？只有then知道
                            x.then(resolve, reject)
                        } else {
                            // 非Promise就直接成功
                            resolve(x)
                        }
                    } catch (err) {
                        // 处理报错
                        reject(err)
                        throw new Error(err)
                    }
                })
            }

            if (this.PromiseState === 'fulfilled') {
                // 如果当前为成功状态，执行第一个回调
                resolvePromise(onFulfilled)
            } else if (this.PromiseState === 'rejected') {
                // 如果当前为失败状态，执行第二个回调
                resolvePromise(onRejected)
            } else if (this.PromiseState === 'pending') {
                // 如果状态为待定状态，暂时保存两个回调
                // 如果状态为待定状态，暂时保存两个回调
                this.onFulfilledCallbacks.push(resolvePromise.bind(this, onFulfilled))
                this.onRejectedCallbacks.push(resolvePromise.bind(this, onRejected))
            }
        })

        // 返回这个包装的Promise
        return thenPromise

    }
}
```
# race

特点：
- 接收一个Promise数组，数组中如有非Promise项，则此项当做成功
- 哪个Promise最快得到结果，就返回那个结果，无论成功失败
思路：
- 循环Promise数组，得到结果就resolve
```js
static race(promiseArr) {
    return new MyPromise((resolve, reject) => {
        promiseArr.forEach(promise => {
            if (promise instanceof MyPromise) {
                promise.then(res => {
                    resolve(res)
                }, err => {
                    reject(err)
                })
            } else {
                resolve(promise)
            }
        })
    })
}
```

# allSettled方法

allSettled会等待所有 Promise完成，并返回一个包含所有结果的对象数组
成功 fulfilled
失败 rejected

思路

- 入参：一个Promise数组 数组中如有非Promise项，则此项当做成功
- 返回值：一个 Promise集合成数组后返回 {status: '', value: ''}
- 实现：遍历数组，每个 Promise 执行 then 方法，将结果添加到数组中，最后返回数组

```js
static allSettled(promiseArr) {
    return new Promise((resolve, reject) => {
        const result = []
        let count = 0
        const addData = (status, value, i) => {
            result[i] = {
                status,
                value
            }
            count++
            if (count === promiseArr.length) {
                resolve(result)
            }
        }
        promiseArr.forEach((promise, i) => {
            if (promise instanceof MyPromise) {
                promise.then(res => {
                    addData('fulfilled', res, i)
                }, err => {
                    addData('rejected', err, i)
                })
            } else {
                addData('fulfilled', promise, i)
            }
        })
    })
}
```