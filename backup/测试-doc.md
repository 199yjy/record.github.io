# 【异步】JS 实现带并发的异步任务调度器
作用：控制异步任务的并发数量，避免因同时发起过多请求或任务导致服务器压力过大或系统资源耗尽，从而保证系统稳定性和性能。

核心：
- 控制同时执行的异步任务数量，当任务数超过限制时，
- 需要将多余任务放入队列等待，直到有任务完成后再从队列中取出新任务执行.

设计思路：
状态管理：用 running 记录当前正在执行的任务数，queue 存储等待执行的任务（包含任务函数及结果回调）。
任务添加：调用 add 方法时，若当前运行数未达上限，直接执行任务；否则将任务放入队列等待。
任务调度：每当一个任务完成（成功或失败），减少 running 计数，并自动从队列中取出下一个任务执行，确保并发数始终不超过限制。

```js
class Scheduler{
    constructor(limit) {
        this.limit = limit; //最大并发数
        this.running = 0; //当前正在执行的任务数
        this.queue = []; //等待执行的任务队列
    }
    add(promiseCreator){
        return new Promise((resolve, reject) => {
            this.queue.push({
                promiseCreator,
                resolve,
                reject
            })
            this.run()
        })
    }
    run(){
        //若当前运行数已达上限，或队列中无等待任务，直接返回
        if (this.running >= this.limit || this.queue.length === 0) {
            return;
        }
        //从队列取出第一个任务
        const { promiseCreator, resolve, reject } = this.queue.shift();
        this.running++; //增加当前运行数
        //执行异步任务
        promiseCreator()
        .then((result) => {
            resolve(result); //将任务结果传递给add返回的Promise
        })
        .catch((error) => {
            reject(error); //任务出错时传递错误
        })
        .finally(() => {
            this.running--; //任务完成（无论成败），减少运行数
            this.run(); //递归调用run，继续执行队列中的下一个任务
        });
    }
}

const timeout = (time) => new Promise(resolve => {
    setTimeout(resolve, time)
})
const scheduler = new Scheduler(n)
const addTask = (time, order) => {
    scheduler.add(() => timeout(time)).then(() => {
        console.log(order)
    })
}

addTask(1000, '1')
addTask(500, '2')
addTask(300, '3')
addTask(400, '4')
//输出2 3 1 4
```