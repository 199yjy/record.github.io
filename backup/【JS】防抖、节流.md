防抖、节流都是用来控制高频率触发事件的执行次数的，但它们的适用场景和核心思想不同。

| 特性|防抖 | 节流 |
|------|------|------|
|执行时机|最后一次触发|固定间隔时间|
|场景|只关心最终结果|持续触发时需要定期反馈|
|例子|电梯关门|公交车定时发车|
## 防抖
把一连串高频触发的事件 合并成一次，只有在最后一次触发结束后，等待一段时间才真正执行。如果在等待时间内又触发了事件，就重新计时。
- 用户输入完最后一个字，才去请求接口
- 窗口大小调整：用户停止拖拽调整窗口大小后，才重新计算布局

```js
/**
 * @param {Function} fn - 需要防抖的函数
 * @param {number} delay - 延迟时间（毫秒）
 * @returns {Function}
 */
function debounce(fn, delay = 300) {
  let timer = null
  return function (...args) {
    clearTimeout(timer) // 清除上一次的定时器
    timer = setTimeout(() => {
      fn.apply(this, args) // 保证 this 和参数不丢失
    }, delay)
  }
}
```
## 节流
第一次执行后，后面在规定时间内的触发都忽略，直到下一个时间间隔才执行。
- 页面滚动：监听 scroll 事件，做吸顶导航、无限加载等
- 用户点击按钮：用户在时间范围内，点击最后一次为准

思路：
- 记录上次执行时间
- 每次触发时检查当前时间与上次执行时间的差值
- 若差值大于等于设定间隔，则执行函数并更新上次执行时间
```js
/**
 * @param {Function} fn - 需要节流的函数
 * @param {number} interval - 时间间隔（毫秒）
 * @returns {Function}
 */
function throttle(fn, interval = 300) {
  let lastTime = 0
  return function (...args) {
    const now = Date.now()
    if (now - lastTime >= interval) {
      fn.apply(this, args)
      lastTime = now
    }
  }
}
```