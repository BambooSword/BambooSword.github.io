---
layout:     post
title:      又见 koa 2 之 Middleware
date:       2017-12-03
author:     BambooSword
header-img: img/post-bg-ios10.jpg
catalog: true
tags:
    - Node
    - JS
    - koa
---


温馨提醒：阅读本文到最后，你将有拯救世界的实力！！！

上篇  [初见Koa 2](http://www.jianshu.com/p/65d3e0f5b757), 我们简单介绍了一下koa 2 的基本用法，现在咱们深入讲解一下 koa 的 Middleware.
## 类比图
 middleware（ 中间件） 始终是Koa中的一个核心概念。在middleware中，中间件由
外而内的相互嵌套，执行完毕之后再由内到外的依次执行回调。他的执行顺序就像穿过下图的洋葱：![如果你一层一层的剥开我的心](http://upload-images.jianshu.io/upload_images/2455149-b6398c0c389a0da7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其实 这张图也就是上篇文章最后一段示例代码的执行顺序，这里我就不举例了，这是基本的概念，想看上篇代码的请点传送门 [初见Koa 2](http://www.jianshu.com/p/65d3e0f5b757)。
## 深入理解 Middleware
我们都知道koa 是 *promise-based*, 如果我们的 Middleware 中省略 next()，会出现什么结果呢？
```
const Koa = require('koa')
const app = new Koa()
// Middleware 1
app.use(async (ctx, next) => {
  ctx.status = 200
  console.log('Setting status')
  // Call the next middleware, wait for it to complete
 // await next()
})
// Middleware 2
app.use((ctx) => {
  console.log('Setting body')
  ctx.body = 'Hello from Koa'
})
app.listen(3002, () => console.log('Koa app listening on 3002'))
``` 
运行上面代码，我们会发现 打开本地的3002端口页面 控制台的log只打印出了 `Setting status`,而页面则返回了 `OK`, 这说明Koa 完成了请求，但是没有返回任何body内容，只设置了status code(状态码)。所以说，第二个 middleware 没有被执行。
Koa 还有一个重要的特性。那就是，如果执行了 next(), 就必须等待它完成。下面这个例子很好的说明了这一点：
```
// Simple Promise delay
function delay (ms) {
  return new Promise((resolve) => {
    setTimeout(resolve, ms)
  })
}
app.use(async (ctx, next) => {
  ctx.status = 200
  console.log('Setting status')
  next() // forgot await!
})
app.use(async (ctx) => {
  await delay(1000) // simulate actual async behavior
  console.log('Setting body')
  ctx.body = 'Hello from Koa'
})
```
运行代码发生了什么呢？控制台的log打印出了 
```
Setting status
Setting body
```
而页面则返回了 `OK`, 虽然我们调用了next(), 却没有返回内容，这何解呢？原来Koa 在 Promise chain is resolved 时就 认为请求结束了，这就说明在我们设置 `ctx.body` 之前，响应的内容已经发送给了客户端。
  另外要说的一点是，如果你想只用用原生的  `Promise.then()` 来代替风骚的 `async-await` ，则需要一个条件：middleware 需要返回一个 promise 对象。
```
app.use((ctx, next) => {
  ctx.status = 200
  console.log('Setting status')
  // need to return here, not using async-await
  return next()
})
```
一个运用原生 promise 更好的例子
```
// We don't call `next()` because
// we don't want anything else to happen.
app.use((ctx) => {
  return delay(1000).then(() => {
    console.log('Setting body')
    ctx.body = 'Hello from Koa'
  })
})
```
## Koa middleware   VS Express
让我们来比较一下 Express 和 Koa, 在 Express 里，一个
 middleware 只能在调用 `next()`前做一些事情，一旦调用了`next()`, 请求便再也不会染指这个 middleware , 这有点让人失望。于是人们不得不用一些精巧的方法，例如 watching the response stream for when headers get written， 但是对于我们劳苦大众来说 这种方法，感觉就有点 尴尬了（用不了，用不好，不好用...）。
举个栗子：如果我们想记录请求所用的时间，并把它放到 `X-ResponseTime` 头部，需要我们记录“before calling next” 和“after calling next” 这两点。在 Express中，只能用监听流的技术来实现，尴尬~
但是在Koa 里怎么实现呢？
```
async function responseTime (ctx, next) {
  console.log('Started tracking response time')
  const started = Date.now()
  await next()
  // once all middleware below completes, this continues
  const ellapsed = (Date.now() - started) + 'ms'
  console.log('Response time is:', ellapsed)
  ctx.set('X-ResponseTime', ellapsed)
}
app.use(responseTime)
app.use(async (ctx, next) => {
  ctx.status = 200
  console.log('Setting status')
  await next()
})
app.use(async (ctx) => {
  await delay(1000)
  console.log('Setting body')
  ctx.body = 'Hello from Koa'
})
```
看，只需要一个 `responseTime ` 方法，8行代码便能‘了却君王天下事，赢得生前身后名’，没有了烦人的人流技术，sorry, 是 监听流技术(⊙o⊙)，只有风骚靓丽的 async-await, 哈哈。
让我们打开控制台，看看`X-ResponseTime`有没有写进去呢？看下图，嗯，答案肯定的。

![chrome 控制台](http://upload-images.jianshu.io/upload_images/2455149-5a4a92b1d978a1ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
所以说呢，与 Express 相比 Koa 能很好的掌控middleware 的一举一动，分分钟解决身份验证，报错处理等问题，啧啧，几乎要帅到没朋友了。

## Error handling （报错处理）
Koa 得益于 Middleware 的 Promise chain, Koa 总会把 `next()` 给我们包裹在 promise 中， 所以我们甚至不用担心这个是 async 还是 sync 错误，可以一并处理。
 报错处理一般运行在 middleware 的顶部， 因为这样它就能‘包裹住’每一个middleware, 这也就意味着任何在middleware抛出的错误都会被捕捉到！（感受到这令人恐怖的力量了吗？）
```
app.use(async (ctx, next) => {
  try {
    await next()
  } catch (err) {
    ctx.status = 400
    ctx.body = `Uh-oh: ${err.message}`
    console.log('Error handler:', err.message)
  }
})
app.use(async (ctx) => {
  if (ctx.query.greet !== 'world') {
    throw new Error('can only greet "world"')
  }
  
  console.log('Sending response')
  ctx.status = 200
  ctx.body = `Hello ${ctx.query.greet} from Koa`
})
```
让我们在chrome运行：
```
http://localhost:3002?greet=jeff
Uh-oh: can only greet "world"
```
再看一下控制台输出了什么：
```
Error handler: can only greet "world"
```
如此畅快、好用的处理机制，我就想问一句，还有谁，还有谁？？？！！！

## 终章
因其秉承 ‘大道至简，大音希声’的原则，Koa 只授心法，帮你打通任督二脉，但并不传武艺。可是只有心法，没有上乘的武学造诣，我们好像还是拯救不了世界啊！~


![武功秘籍](http://upload-images.jianshu.io/upload_images/2455149-30261318c4111c41.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



------少侠，请留步，我看你骨骼惊奇，眉宇间透露着一股正气，简直百年一见的练武奇才啊，既然已经有人打通了你的任督二脉，如果你再习得盖世武功，那还不飞龙上天？！正所谓，我不入地狱谁入地狱，惩奸除恶，维护世界和平的任务任务就交给你了，我这里有本武林秘籍，是无价之宝，我看与你有缘，就收你一个赞，传授于你！！！

[koa-compress](https://github.com/koajs/compress)
[koa-respond](https://github.com/jeffijoe/koa-respond)
[kcors](https://github.com/koajs/cors)
[koa-convert](https://github.com/koajs/convert)
[koa-bodyparser](https://github.com/koajs/bodyparser)
[koa-compose](https://github.com/koajs/compose)
[koa-router](https://github.com/alexmingoia/koa-router)

------慢着，少侠，在修炼之前我先要提醒你，目前并不是所有的 middleware 框架 都支持 Koa 2， 但是我们有 `koa-convert` 专治各种不服，让其能很好的为你服务，妙哉！
***

感谢阅读，但愿本文能对你有所帮助，如果有帮助，请点赞或关注，我会有更加深入的文章推送的~

## 参考链接：
[Mastering Koa Middleware](https://medium.com/@Jeffijoe/mastering-koa-middleware-f0af6d327a69)
[koa2进阶学习笔记](https://chenshenhai.github.io/koa2-note/)
[Learn-Koa2](https://github.com/ecmadao/Learn-Koa2) 