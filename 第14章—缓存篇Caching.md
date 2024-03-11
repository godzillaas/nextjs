## 前言

本章我们将介绍 Next.js 的缓存机制，学习本篇有助于理解 Next.js 的工作原理，但这不是使用 Next.js 的必要知识。因为 Next.js 会自动根据你使用的 API 做好缓存管理。

缓存的重要性不言而喻，可以优化应用性能和降低开销。Next.js 中有四种缓存机制，这是概览：

| 机制                        | 缓存内容               | 使用地方 | 目的               | 期间        |
| ------------------------- | ------------------ | ---- | ---------------- | --------- |
| 请求记忆（Request Memoization） | 函数返回值              | 服务端  | 在 React 组件树中复用数据 | 每个请求的生命周期 |
| 数据缓存（Data Cache ）         | 数据                 | 服务端  | 跨用户请求和部署复用数据     | 持久（可重新验证） |
| 完整路由缓存（Full Route Cache）  | HTML 和 RSC payload | 服务端  | 降低渲染成本、提高性能      | 持久（可重新验证） |
| 路由缓存（Router Cache）        | RSC payload        | 客户端  | 减少导航时的服务端请求      | 用户会话或基于时间 |

默认情况下，Next.js 会尽可能多的使用缓存以提高性能和降低成本。像路由默认会采用静态渲染，数据请求的结果默认会被缓存。下图是构建时静态路由渲染以及首次访问静态路由的原理图：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e176beb2d77848f88f7790d9eb7ab720~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1600\&h=1179\&s=418691\&e=png\&b=0d0d0d)

在这张图中， 打包构建 `/a`时，因为路由中的请求都是首次，所以都会 `MISS`，从数据源获取数据后，在请求记忆和数据缓存中都保存了一份（`SET`），并将生成的 RSC Payload 和 HTML 也在服务端保存了一份（`SET`），当客户端访问 `/a` 的时候，命中服务端缓存的 RSC Payload 和 HTML，并将 RSC Payload 在客户端保存一份（`SET`）。

缓存行为是会发生变化的，具体取决的因素有很多，比如路由是动态渲染还是静态渲染，数据是缓存还是未缓存，请求是在初始化访问中还是后续导航中。随着本篇内容的展开，我们会有更加深入的了解。

## 1. 请求记忆（Request Memoization）

### 1.1. 工作原理

React 拓展了 [fetch API](https://nextjs.org/docs/app/building-your-application/caching#fetch)，当有相同的 URL 和参数的时候，React 会自动将请求结果缓存。也就是说，即时你在组件树中的多个位置请求一份相同的数据，但数据获取只会执行一次。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dcd22d593060474fa2a7437337469aa0~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1600\&h=857\&s=666460\&e=png\&b=141414)

这样当你跨路由（比如跨布局、页面、组件）时，你不需要在顶层请求数据，然后将返回结果通过 props 一一转发，直接在需要数据的组件中请求数据即可，不用担心对同一数据发出多次请求造成的性能影响。

```javascript
// app/page.js
async function getItem() {
  // 自动缓存结果
  const res = await fetch('https://.../item/1')
  return res.json()
}
 
// 函数调用两次，但只会执行一次请求
const item = await getItem() // cache MISS
 
const item = await getItem() // cache HIT
```

这是请求记忆的工作原理图：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5b9746cb977d49888c846ea230f99fa1~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1600\&h=800\&s=149814\&e=png\&b=0e0e0e)

在这种图中，当渲染 `/a` 路由的时候，由于是第一次请求，会触发缓存 `MISS`，函数被执行，请求结果会被存储到内存中（缓存`SET`），当下一次相同的调用发生时，会触发缓存 `HIT`，数据直接从内存中取出。

它背后的原理想必大家也想到了，就是[函数记忆](https://juejin.cn/post/6844903494256705543)，《JavaScript 权威指南》中就有类似的函数：

```javascript
function memoize(f) {
    var cache = {};
    return function(){
        var key = arguments.length + Array.prototype.join.call(arguments, ",");
        if (key in cache) {
            return cache[key]
        }
        else return cache[key] = f.apply(this, arguments)
    }
}
```

### 1.2. 持续时间

当路由渲染完毕，储存的数据就会清除重置。所以 React 的请求记忆只持续到 React 组件树渲染完成，它的存在只是为了避免组件树渲染的时候多次请求造成的性能影响。

### 1.3. 重新验证

由于请求记忆只会在渲染期间应用，因此也无须重新验证。

### 1.4. 退出方式

如果你不想在 fetch 请求使用记忆化，你可以借助 [AbortController](https://developer.mozilla.org/zh-CN/docs/Web/API/AbortController)这个 Web API，具体使用方式如下：

```javascript
const { signal } = new AbortController()
fetch(url, { signal })
```

使用请求记忆时，要注意：

*   这是 React 的特性，并非 Next.js 的特性。 React 和 Next.js 都做了请求缓存，React 的方案叫做“请求记忆化”，Next.js 的方案叫做“数据缓存”（这是接下来要讲的内容），两者其实有很多不同。
*   请求记忆只适合用于用 `GET` 方法的 `fetch` 请求。
*   记忆化只应用于 React 组件树，也就是说你在 `generateMetadata`、`generateStaticParams`、布局、页面和其他服务端组件中使用 fetch 会触发请求记忆化，但是在路由处理程序中使用则不会触发，因为这就不在 React 组件树中了。
*   如果你不能使用 fetch 请求，但是又想实现记忆化，可以借助 React 的 cache 函数：

```javascript
// utils/get-item.ts
import { cache } from 'react'
import db from '@/lib/db'
 
export const getItem = cache(async (id: string) => {
  const item = await db.item.findUnique({ id })
  return item
})
```

## 2. 数据缓存（Data Cache）

### 2.1. 工作原理

Next.js 有自己内置的数据缓存方案，可以跨服务端请求和构建部署存储数据。之所以能够实现，是因为 Next.js 也拓展了 fetch API，在 Next.js 中，每个请求都可以设置自己的缓存方式。不过与 React 的请求记忆不同的是，请求记忆因为只用于组件树渲染的时候，所以不用考虑数据缓存更新的情况，但 Next.js 的数据缓存方案更为持久，则需要考虑这个问题。

默认情况下，使用 `fetch` 的数据请求都会被缓存，这个缓存是持久的，它不会自动被重置。你可以使用 `fetch` 的 `cache` 和 `next.revalidate` 选项来配置缓存行为：

```javascript
fetch(`https://...`, { cache: 'force-cache' | 'no-store' })
```

```javascript
// 最多 1 小时后重新验证
fetch(`https://...`, { next: { revalidate: 3600 } })
```

这是 Next.js 数据缓存的原理图：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d1fe452584b349bd94efe4cf658c729f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1600\&h=661\&s=138430\&e=png\&b=101010)

让我们解释一下：当渲染的时候首次调用，请求记忆和数据缓存都会 MISS，从而执行请求，返回的结果在请求记忆和数据缓存中都会存储一份。

当再次调用的时候，因为添加了 `{cache: 'no-store'}`参数，请求参数不同，请求记忆会  MISS，而这个参数会导致数据缓存跳过，所以依然是执行请求，因为配置了要跳过，所以数据缓存也不会缓存返回的结果，请求记忆则会正常做缓存处理。

### 2.2. 持续时间

数据缓存在传入请求和部署中都保持不变，除非重新验证或者选择退出。

### 2.3. 重新验证

Next.js 提供了两种方式更新缓存：

一种是**基于时间的重新验证（Time-based revalidation）**，即经过一定时间并有新请求产生后重新验证数据，适用于不经常更改且新鲜度不那么重要的数据。

一种是**按需重新验证（On-demand revalidation）**，根据事件手动重新验证数据。按需重新验证又可以使用基于标签（tag-based）和基于路径（path-based）\*\*两种方法重新验证数据。适用于需要尽快展示最新数据的场景。

#### 基于时间

如果你使用基于时间的重新验证，那你需要使用 `fetch` 的 `next.revalidate` 选项设置缓存的时间（注意它是以秒为单位）。

```javascript
// 每小时重新验证
fetch('https://...', { next: { revalidate: 3600 } })
```

如果你无法使用 fetch，那你可以借助路由段配置项来配置该路由所有的 fetch 请求：

```javascript
// layout.jsx / page.jsx / route.js
// 退出数据缓存
export const revalidate = 0
```

这是基于时间的重新验证原理图：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c2a97c76ad24bfe98708d452cd27211~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1600\&h=1252\&s=260753\&e=png\&b=0e0e0e)

通过这种图，你可以发现：并不是 60s 后该请求会自动更新，而是 60s 后再有请求的时候，会进行重新验证，60s 后的第一次请求依然会返回之前的缓存值，但 Next.js 将使用新数据更新缓存。60s 后的第二次请求会使用新的数据。

#### 按需更新

如果你使用按需重新验证，数据可以根据路径（`revalidatePath`）和 缓存标签（`revalidateTag`） 按需更新。

`revalidatePath` 用在路由处理程序或 Server Actions 中，用于手动清除特定路径中的缓存数据：

```javascript
revalidatePath('/')
```

`revalidateTag`依赖的是 Next.js 的缓存标签系统，当使用 fetch 请求的时候，声明一个标签，然后在路由处理程序或是 Server Actions 中重新验证具有某一标签的请求：

```javascript
// 使用标签
fetch(`https://...`, { next: { tags: ['a', 'b', 'c'] } })
```

```javascript
// 重新验证具有某一标签的请求
revalidateTag('a')
```

这是按需更新的原理图：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cac0e8d8c8854c9586c72dbd7981169a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1600\&h=1082\&s=189982\&e=png\&b=0e0e0e)

你会发现，这跟基于时间的重新验证有所不同。第一次调用请求的时候，正常缓存数据。当触发按需重新验证的时候，将会从缓存中删除相应的缓存条目。下次请求的时候，又相当于第一次调用请求，正常缓存数据。

### 2.4. 退出方式

如果你想要退出数据缓存，有两种方式：

一种是将 `fetch` 的 `cache` 选项设置为 `no-store`，示例如下，每次调用的时候都会重新获取数据。

```javascript
fetch(`https://...`, { cache: 'no-store' })
```

一种是使用[路由段配置项](https://juejin.cn/book/7307859898316881957/section/7309079033223446554)，它会影响该路由段中的所有数据请求。

```javascript
export const dynamic = 'force-dynamic'
```

## 3. 完整路由缓存（Full Route Cache）

### 3.1. 工作原理

Next.js 在构建的时候会自动渲染和缓存路由，这样当访问路由的时候，可以直接使用缓存中的路由而不用从零开始在服务端渲染，从而加快页面加载速度。

那你可能要问，缓存路由是个什么鬼？我听过缓存数据，但是路由怎么缓存呢？让我们复习下 Next.js 的渲染原理：

Next.js 使用 React 的 API 来编排渲染。当渲染的时候，渲染工作会根据路由段和 Suspense 拆分成多个 chunk，每个 chunk 分为两步进行渲染：

1.  React 会将服务端组件渲染成一种特殊的数据格式，我们称之为 React Server Component Payload，简写为 RSC payload。比如一个服务端组件的代码为：

```javascript
<div>
  Don’t give up and don’t give in.
  <ClientComponent />
</div>
```

React 会将其转换为如下的 Payload：

```javascript
["$","div",null,{"children":["Don’t give up and don’t give in.", ["$","$L1",null,{}]]}]
1:I{"id":123,"chunks":["chunk/[hash].js"],"name":"ClientComponent","async":false}
```

这个格式针对流做了优化，它们可以以流的形式逐行从服务端发送给客户端，客户端可以逐行解析 RSC Payload，渐进式渲染页面。

当然这个 RSC payload 代码肯定是不能直接执行的，它包含的更多是信息：

1.  服务端组件的渲染结果
2.  客户端组件的占位和引用文件
3.  从服务端组件传给客户端组件的数据

比如这个 RSC Payload 中的 `$L1` 表示的就是 ClientComponent，客户端会在收到 RSC Payload 后，解析下载 ClientComponent 对应的 bundle 地址，然后将执行的结果渲染到 `$L1` 占位的位置上。

2.  Next.js 会用 RSC payload 和客户端组件 JavaScript 指令在服务端渲染 HTML。

这张图生动的描述了这个过程：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb119007336543c288845f29c425e014~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1600\&h=888\&s=171366\&e=png\&b=0d0d0d)

Next.js 的完整路由缓存，缓存的就是服务端编译后 RSC Payload 和 HTML。

而路由在构建的时候是否会被缓存取决于它是静态渲染还是动态渲染。静态路由默认都是会被缓存的，动态路由因为只能在请求的时候被渲染，所以不会被缓存。这张图展示了静态渲染和动态渲染的差异：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/27d1e45c1c5849d2892dfad986832acc~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1600\&h=1314\&s=351817\&e=png\&b=0c0c0c)

在这种图中，静态路由 `/a`有完整路由缓存， 动态路由 `/b` 跳过了完整路由缓存。但这并不影响客户端的路由缓存，所以在后续的请求中都命中了路由缓存。

### 3.2. 持续时间

完整路由缓存默认是持久缓存的，这意味着渲染输出是可以跨用户请求复用的。

### 3.3. 失效

有两种方式可以使完整路由缓存失效：

*   重新验证数据：重新验证数据缓存将使完整路由缓存失效，毕竟渲染输出依赖于数据
*   重新部署：数据缓存是可以跨部署的，但完整路由缓存会在重新部署中被清除

### 3.4. 退出方式

退出完整路由缓存的方式就是将其改为动态渲染：

*   使用动态函数：使用动态函数后会改为动态渲染，此时数据缓存依然可以用
*   使用路由段配置：`dynamic = 'force-dynamic'`或 `revalidate = 0` 这会跳过完整路由缓存和数据缓存，也就是说，每次请求时都会重新获取数据并渲染组件。此时路由缓存依然可以用，毕竟它是客户端缓存
*   退出数据缓存：如果路由中有一个 fetch 请求退出了缓存，则会退出完整路由缓存。这个特定的 fetch 请求会在每次请求时重新获取，其他  fetch 请求依然会使用数据缓存。Next.js 允许这种缓存和未缓存数据的混合。

## 4. 路由缓存（Router Cache）

### 4.1. 工作原理

Next.js 有一个存放在内存中的客户端缓存，它会在用户会话期间按路由段存储 RSC Payload。这就是路由缓存。工作原理图如下：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8964c9aa0318409db0dd74ed86d18dd6~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1600\&h=1375\&s=312833\&e=png\&b=0b0b0b)

原理图很好理解，当访问 `/a`的时候，因为是首次访问（`MISS`），将 `/（layout）`和 `/a(page)`放在路由缓存中（`SET`），当访问与 `/a`共享布局的 `/b`的时候，使用路由缓存中的 `/（layout）`，然后将 `/b(page)`放在路由缓存中（`SET`）。再次访问 `/a`的时候，直接使用路由缓存中（`HIT`）的 `/(layout)`和 `/b(page)`。

不止如此，当用户在路由之间导航，Next.js 会缓存访问过的路由段并预获取用户可能导航的路由（基于视口内的 `<Link>` 组件）。这会为用户带来更好的导航体验：

1.  即时前进和后退导航，因为访问过的路由已经被缓存，并且预获取了新路由
2.  导航不会导致页面重载，并且会保留 React 的状态和浏览器状态

让我们根据原理图写个 demo 验证一下：

```javascript
// app/layout.js
import Link from "next/link";

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        <div>
          <Link href="/a">Link to /a</Link>
            

          <Link href="/b">Link to /b</Link>
        </div>
        {children}
      </body>
    </html>
  )
}
```

两个路由的代码类似：

```javascript
// app/a/page.js | app/b/page.js
export default function Page() {
  return (
    <h1>Component X</h1>
  )
}
```

当首次访问 `/a`的时候，因为 Link 组件的 `/a` 和 `/b` 都在视口内，所以会预加载 `/a` 和  `/b` 的 RSC Payload：

![截屏2023-11-28 上午11.13.19.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/73175e3be51847c7bfb65e00c32da591~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1298\&h=1028\&s=150651\&e=png\&b=ffffff)

得益于预加载和缓存，无论是导航还是前进后退都非常顺滑：

![1114.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a18d51d4ff7448d81b2e883b223b73f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=927\&h=570\&s=131584\&e=gif\&f=32\&b=fefefe)

### 4.2. 持续时间

路由缓存存放在浏览器的临时缓存中，有两个因素决定了路由缓存的持续时间：

*   Session，缓存在导航期间会持续存在，当页面刷新的时候会被清除
*   自动失效期：单个路由段会在特定时长后自动失效，如果路由是静态渲染，持续 5 分钟，如果是动态渲染，持续 30s，比如上面的 demo 中如果等 5 分钟后再去点击，就会重新获取新的 RSC Payload

通过添加 `prefetch={true}`或者在动态渲染路由中调用 `router.prefetch`，可以进入缓存 5 分钟。（Link 组件的 prefetch 默认就为 true）

### 4.3. 失效

有两种方法可以让路由缓存失效：

*   在 Server Action 中
    *   通过 `revalidatePath` 或 `revalidateTag` 重新验证数据
    *   使用  `cookies.set` 或者 `cookies.delete` 会使路由缓存失效，这是为了防止使用 cookie 的路由过时（如身份验证）
*   调用 `router.refresh` 会使路由缓存失效并发起一个重新获取当前路由的请求

### 4.4. 退出方式

无法退出路由缓存，你可以通过给 `<Link>` 组件的 `prefetch` 传递 `false` 来退出预获取，但依然会临时存储路由段 30s，这是为了实现嵌套路由段之间的即时导航。访问过的路由也会被缓存。

### 4.5. 区别

路由缓存和完整路由缓存的区别：

1.  路由缓存发生在用户访问期间，将 RSC Payload 暂时存储在浏览器，导航期间都会持续存在，页面刷新的时候会被清除。而完整路由缓存则会持久的将 RSC Payload 和 HTML 缓存在服务器上
2.  完整路由缓存仅缓存静态渲染的路由，路由缓存可以应用于静态和动态渲染的路由

### 4.6. 渲染原理

最后，让我们结合完整路由缓存和路由缓存完整的看下 Next.js 的渲染原理：

#### 服务端 React 渲染

Next.js 使用 React 的 API 来编排渲染。当渲染的时候，渲染工作会根据路由段和 Suspense 拆分成多个 chunk，每个 chunk 分为两步进行渲染：

1.  React 会将服务端组件渲染成一种特殊的数据格式，我们称之为 React Server Component Payload，简写为 RSC payload
2.  Next.js 会用 RSC payload 和客户端组件 JavaScript 指令在服务端渲染 HTML

#### 服务端缓存（完整路由缓存）

Next.js 将 HTML 和 RSC payload 缓存在服务端

#### 客户端解析

在请求时，客户端：

1.  加载 HTML，快速展示一个不可交互的页面
2.  加载 RSC Payload 用于构建服务端组件树，并更新 DOM
3.  加载 JavaScript 指令让应用程序具有交互性

#### 客户端缓存（路由缓存）

将用到的 RSC Payload 缓存在客户端以改善用户导航体验

#### 后续导航

在后续导航的时候，先检查路由缓存中是否有对应的 RSC Payload，没有再向服务端发送请求获取 RSC Payload，并将结果存储在路由缓存中

## 参考链接

1.  [Building Your Application: Caching | Next.js](https://nextjs.org/docs/app/building-your-application/caching)
