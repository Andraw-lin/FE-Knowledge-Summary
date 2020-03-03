# Javascript 中浮点数的精度问题

在日常开发中，常常会遇到前端需要计算业务，由于精度问题，后端一般都不会放心滴将计算都交给前端来算，特别是涉及到金钱方面的业务（少0.1元还好，少100元可就大亏呀...）。为此，前端同事一般都不敢背这锅，直接交给后端全盘接手业务。🤣

既然谈到精度问题，那究竟所谓的精度问题又是如何的呢？我们接着探究下去。



## 目录

1. [精度问题](#js1)
2. [二进制和十进制间互转](#js2)
3. [如何解决精度问题？](#js3)



## 精度问题

相信大家在刚学习 js 时，就有听说过千万别使用 js 来进行计算，因为 js 计算起来可是有误差？

那么为什么 js 在计算时就有误差呢？

答案就是今天的主题——浮点数的精度问题。

在讲解之前，我们先来看看一些常出现浮点数精度导致的计算出现误差。

```javascript
// 加法 =====================
// 0.1 + 0.2 = 0.30000000000000004
// 0.7 + 0.1 = 0.7999999999999999
// 0.2 + 0.4 = 0.6000000000000001
// 2.22 + 0.1 = 2.3200000000000003
 
// 减法 =====================
// 1.5 - 1.2 = 0.30000000000000004
// 0.3 - 0.2 = 0.09999999999999998
 
// 乘法 =====================
// 19.9 * 100 = 1989.9999999999998
// 19.9 * 10 * 10 = 1990
// 1306377.64 * 100 = 130637763.99999999
// 1306377.64 * 10 * 10 = 130637763.99999999
// 0.7 * 180 = 125.99999999999999
// 9.7 * 100 = 969.9999999999999
// 39.7 * 100 = 3970.0000000000005
 
// 除法 =====================
// 0.3 / 0.1 = 2.9999999999999996
// 0.69 / 10 = 0.06899999999999999
```

上面的🌰，都是借用一下网上提供的一些常见由于浮点数精度问题导致计算有误差。

按照官方的解释，**Javascript 遵循的是[IEEE 754 二进制浮点数算术标准](https://baike.baidu.com/item/IEEE 754/3869922?fr=aladdin)中64位双精度浮点数。**

根据这个标准，**Javascript 开发中定义数据一般都是十进制，因此在计算过程中，都会将所有十进制数值转换成二进制数值来进行计算**。

那么，又如何理解64位双精度浮点数呢？我们来看个结构图就清楚啦。

![64位双精度浮点数](https://raw.githubusercontent.com/Andraw-lin/keep-Learning/master/asset/64%E4%BD%8D%E5%8F%8C%E7%B2%BE%E5%BA%A6%E6%B5%AE%E7%82%B9%E6%95%B0.jpg)

现在就来简单阐述一下上面结构图的提到的词汇。

- 符号部分：占1位，使用 s 表示，其中0开头表示为正数，1开头表示为负数。
- 指数部分：占11位，使用 e 表示，用于存储指数部分。
- 尾数部分：占52位，使用 f 表示，**真正用于存储有效数字部分（必须得敲黑板！）**。

**符号部分用于决定一个数值的正负，指数部分则决定了一个数值的大小，尾数部分决定一个数值的精度**。一个有效数字的描述在正常情况下，则是直接由符号位和尾数部分所决定，因此 **Javascript 提供的有效数字为 53 个二进制位（即符号位+52位尾数）**。

正是由于有效数字为 53 个二进制位，所以数值也被限制在 -(2^53 - 1) ~ 2^53 - 1 之间（即Number.MAX_SAFE_INTEGER === 9007199254740991 到 Number.MIN_SAFE_INTEGER === -9007199254740991之间）。

一旦超出这个数值范围会怎样？答案就是截取。

而**二进制超出范围时截取方式也是直接导致了 Javascript 中计算的精度问题**。

让我们来看个🌰，你就知道了。

```javascript
console.log(90071992547409910) // 90071992547409900
```

由上面的栗子就可以清楚看到，由于数值 90071992547409910 直接超出了最大范围 9007199254740991 ，最终将会进行截取，而截取后输出的结果就为 90071992547409900。



## 二进制和十进制间互转

大概了解 Javascript 精度问题的引起原因后，现在就来简单回顾一下，数值是如何转化为二进制的。毕竟这都是大学里学过的知识，现在还得来简单列一下，避免看到二进制就懵逼哈哈。😄（当然你可以直接跳过哈...）

二进制转化为十进制，采用的是**按权相加**方式。

```javascript
1100 = 1 * 2^0 + 1 * 2^1 + 0 * 2^2 + 0 * 2^3 = 3
// 二进制1100相当于十进制3
```

十进制整数部分转化为二进制，采用的是**除2取余，逆序排列**方式。借用一个图

![十进制整数部分转化为二进制](https://www.runoob.com/wp-content/uploads/2018/11/210-2.png)

十进制小数部分转化为二进制，采用的是**乘2取整，顺序排列**方式。继续借用一个图

![十进制小数部分转化为二进制](https://www.runoob.com/wp-content/uploads/2018/11/210-3.png)

那么，现在就拿`0.1 + 0.2`举个例子。先看看0.1和0.2转化为二进制后的数值。

```javascript
0.1 -> 0.0001100110011001...(无限)
0.2 -> 0.0011001100110011...(无限)
```

可以看到，**0.1和0.2转化为二进制后，后面都是循环无限的，就算它们进行相加后也还是循环无限的**。那么问题来了，**有效数值范围是53位，一旦超过就会被截取**，这也直接导致截取到的结果不是我们想要的。

```javascript
0.1 + 0.2 === 0.30000000000000004
```



## 如何解决精度问题？

既然 Javascript 由于遵循 IEEE 754 二进制浮点运算的标准，那么前端就真的无法解决所谓的精度问题了吗？答案却是否定的。我们就来看看究竟有哪些方法可以解决？

1. 引用第三方库。

   一般情况下，都是尽量不需要前端计算，为此复杂的计算问题都交给服务器进行计算。同样地，我们**将计算问题都可以交给第三方库处理，是一种比较方便的处理方案**。

   - **Math.js**

     Math.js 是专门为 JavaScript 和 Node.js 提供的一个广泛的数学库，提供集成解决方案来处理不同的数据类型。

   - **decimal.js**

     为 JavaScript 提供十进制类型的任意精度数值。

   - **big.js**

     一个小型，快速的JavaScript库，用于任意精度的十进制算术运算。

2. 使用 toFixed 方法。

   相信大部分童鞋都使用过 toFixed 方法处理精度问题，toFixed 方法接受一个参数，用于保留小数点后几位小数，得到的数字会以**四舍五入**的方式进行转化。

   ```javascript
   console.log((0.1 + 0.2).toFixed(1)) // 0.3
   ```

   但是 toFixed 方法会存在某些小问题，在 IE 上测试是正常的，一旦到 chrome，可能就会有一些小毛病出来了。看下面这个🌰：

   ```javascript
   console.log((1.25).toFixed(1)) // 1.3 正确
   console.log((1.225).toFixed(2)) // 1.23 正确
   console.log((1.2225).toFixed(3)) // 1.222 错误
   console.log((1.22225).toFixed(4)) // 1.2223 正确
   console.log((1.222225).toFixed(5))  // 1.22222 错误
   console.log((1.2222225).toFixed(6)) // 1.222222 错误
   ```

   可以看到，**toFixed 方法虽好，但在处理某些浮点数时，不同浏览器也会得到不一样的结果（即精度丢失）**。

   为此，网上有些大神们也是推荐直接重写数值的 toFixed 方法。

   ```javascript
   Number.prototype.toFixed = function(len) {
     if(len > 20 || len < 0) {
       throw new Error('toFixed method must length in 0 ~ 20')
     }
     const num = Number(this) // 将.123转化为0.123
     if(isNaN(num) || num >= Math.pow(10, 21)) { // 判断是否为NaN，以及不能超过1e+21
       return num.toString()
     }
     if(len === undefined || len === 0) { // 判断参数是否不合法
       return Math.round(num).toString()
     }
     const result = num.toString() // 转化数值为字符串
     const numArr = result.split('.') // 将字符串根据.号转换为数组
     const intNum = numArr[0] // 获取整数部分
     const deciNum = numArr[1] // 获取小数部分
     if(numArr.length < 2) {
       return `${intNum}.`.padEnd(len)
     }
     if(deciNum.length <= len) {
     	return `${intNum}.${deciNum}`.padEnd(len - deciNum.length)
     } else {
       const newDeciNum = deciNum.slice(0, len - 1)
       return `${intNum}.${newDeciNum}`
     }
   }
   ```

   重写的 toFixed ，就是将数值进行字符串的一系列操作得到需要的结果。

3. 将浮点数转化为字符串取整处理。

   相信很多童鞋都会使用过，将浮点数取整后，再进行运算，最后再除以相应的倍数得到最终结果。可别说，这种算法的效果也是杠杠的。🤔

   ```javascript
   console.log((0.1 * 10 + 0.2 * 10) / 10) // 0.3
   ```

   既然这算法这么好用，那么为啥不推广呢？细究下去，会发现依然存在问题。

   ```javascript
   console.log((35.41 * 100 + 0.1 * 10) / 100) // 35.419999999999995
   ```

   可以看到，问题依然还是会存在，那么究竟如何处理是好？答案就是**字符串处理**。

   简单来说，就是**将浮点数值使用字符串处理方式将小数点去掉，变成一个整数形式，再进行运算，运算得到的结果再除以相应的倍数即可**。这样就可以有效避免以上的坑了。

   ```javascript
   var floatObj = (function() {
     var isIntegar = obj => Math.floor(obj) === obj // 判断数值是否为整数
     var toIntegar = floatNum => { // 将数值转化为字符串处理
       var ret = { rate: 1, num: 0 }
       if(isIntegar(floatNum)) {
         ret.num = floatNum
         return ret
       }
       var floatStr = floatNum.toString()
       var deciNum = floatStr.split('.')[1]
       var integarNum = Number(floatStr.replace('.', ''))
   		ret.num = integarNum
       ret.rate = Math.pow(10, deciNum.length)
       return ret
     }
     var add = (a, b) => { // 两个浮点数相加
       var floatA = toIntegar(a)
       var floatB = toIntegar(b)
       var { rate: rateA, num: numA } = floatA
       var { rate: rateB, num: numB } = floatB
       var resultRate = rateA > rateB ? rateA : rateB
       var resultNum
       if(rateA === rateB) {
         resultNum = numA + numB
       } else if (rateA > rateB) {
         resultNum = numA + numB * (rateA / rateB)
       } else {
         resultNum = numA * (rateB / rateA) + numB
       }
       return resultNum / resultRate
     }
     var subtract = (a, b) => { // 两个浮点数相减
       var floatA = toIntegar(a)
       var floatB = toIntegar(b)
       var { rate: rateA, num: numA } = floatA
       var { rate: rateB, num: numB } = floatB
       var resultRate = rateA > rateB ? rateA : rateB
       var resultNum
       if(rateA === rateB) {
         resultNum = numA - numB
       } else if (rateA > rateB) {
         resultNum = numA - numB * (rateA / rateB)
       } else {
         resultNum = numA * (rateB / rateA) - numB
       }
       return resultNum / resultRate
     }
     var multiply = (a, b) => { // 两个浮点数相乘
       var floatA = toIntegar(a)
       var floatB = toIntegar(b)
       var { rate: rateA, num: numA } = floatA
       var { rate: rateB, num: numB } = floatB
       var resultNum = (numA * numB) / (rateA * rateB)
       return resultNum
     }
     var divide = (a, b) => { // 两个浮点数相除
       var floatA = toIntegar(a)
       var floatB = toIntegar(b)
       var { rate: rateA, num: numA } = floatA
       var { rate: rateB, num: numB } = floatB
       var resultNum = (numA / numB) * (rateB / rateA)
       return resultNum
     }
     return { add, subtract, multiply, divide }
   })()
   ```

   

   



































