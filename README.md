## 手写题，每日一题

### Day01 手写函数柯里化

```js
function curry(func) {
  //此处补全
}
function sum(a, b, c) {
  return a + b + c;
}

let curriedSum = curry(sum);

alert(curriedSum(1, 2, 3)); // 6, still callable normally
alert(curriedSum(1)(2, 3)); // 6, currying of 1st arg
alert(curriedSum(1)(2)(3)); // 6, full currying
```

### Day02 实现 symbol polyfill

题解：如果浏览器不支持情况下 写出让代码让浏览器支持symbol

### Day03 实现 `Promise.allSettled()`
### Day04 实现 `Object.is()`
### Day05 请计算出时钟的时针和分针的角度（两个角度的较小者，四舍五入）。时间以HH:mm的格式传入。
```js
angle('12:00')
// 0

angle('23:30')
// 165
```
### Day07 实现 `Object.create`
### Day08 实现一个 `Event Emitter`
### Day09 实现一个可以拖拽的DIV
### Day10 实现 `Promise.any()`
### Day11 实现 `Promise.race()`
### Day12 实现 `Promise.any()`
### Day13 实现 Immer 的 produce()
### Day14 Virtual DOM V - JSX 2
### Day15 去除字符

















