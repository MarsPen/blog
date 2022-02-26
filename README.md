## 手写
01 🍎 手写函数柯里化
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

02 🍎 实现symbol polyfill
```js
题解：如果浏览器不支持情况下 写出让代码让浏览器支持symbol
```

03 🍎 实现 `Promise.allSettled()`



