## 前言

本篇我们来实现右侧笔记预览部分。

## 笔记预览

当点击笔记的时候，可以查看该笔记的内容：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1376db7d8e4047c4bda9bf7c2dfca36f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2188\&h=1316\&s=237030\&e=png\&b=ffffff)

按照我们之前的设计，当点击左侧笔记列表的时候，会导航至对应的 `/note/xxxx`路由，所以我们再新建一个 `app/note/[id]/page.js`文件，代码如下：

```jsx
import Note from '@/components/Note'
import {getNote} from '@/lib/redis';

export default async function Page({ params }) {
  // 动态路由 获取笔记 id
  const noteId = params.id;
  const note = await getNote(noteId)

  // 为了让 Suspense 的效果更明显
  const sleep = ms => new Promise(r => setTimeout(r, ms));
  await sleep(5000);

  if (note == null) {
    return (
      <div className="note--empty-state">
        <span className="note-text--empty-state">
          Click a note on the left to view something! 🥺
        </span>
      </div>
    )
  }

  return <Note noteId={noteId} note={note} />
}

```

还记得上篇总结的最佳实践吗？用到数据请求的地方，考虑一下是否需要用 `Suspense` 和 `loading.js`。这里就是一个很适合用 `loading.js` 的地方，我们再添加一个 `app/note/[id]/loading.js`文件，代码如下：

```javascript
export default function NoteSkeleton() {
  return (
    <div
      className="note skeleton-container"
      role="progressbar"
      aria-busy="true"
    >
      <div className="note-header">
        <div
          className="note-title skeleton"
          style={{ height: '3rem', width: '65%', marginInline: '12px 1em' }}
        />
        <div
          className="skeleton skeleton--button"
          style={{ width: '8em', height: '2.5em' }}
        />
      </div>
      <div className="note-preview">
        <div className="skeleton v-stack" style={{ height: '1.5em' }} />
        <div className="skeleton v-stack" style={{ height: '1.5em' }} />
        <div className="skeleton v-stack" style={{ height: '1.5em' }} />
        <div className="skeleton v-stack" style={{ height: '1.5em' }} />
        <div className="skeleton v-stack" style={{ height: '1.5em' }} />
      </div>
    </div>
  )
}
```

我们在 `page.js`中引入了` <Note>` 组件，`components/Note.js`的代码如下：

```jsx
import dayjs from 'dayjs';
import NotePreview from '@/components/NotePreview'
import EditButton from '@/components/EditButton'
export default function Note({ noteId, note }) {
  const { title, content, updateTime } = note

  return (
    <div className="note">
      <div className="note-header">
        <h1 className="note-title">{title}</h1>
        <div className="note-menu" role="menubar">
          <small className="note-updated-at" role="status">
            Last updated on {dayjs(updateTime).format('YYYY-MM-DD hh:mm:ss')}
          </small>
            <EditButton noteId={noteId}>Edit</EditButton>
        </div>
      </div>
      <NotePreview>{content}</NotePreview>
    </div>
  )
}
```

这里我们把预览的部分又单独抽离成了一个 `<NotePreview>` 组件，之所以抽离，是考虑到在编辑界面复用。`components/NotePreview.js`的代码如下：

```javascript
import {marked} from 'marked'
import sanitizeHtml from 'sanitize-html'

const allowedTags = sanitizeHtml.defaults.allowedTags.concat([
  'img',
  'h1',
  'h2',
  'h3'
])
const allowedAttributes = Object.assign(
  {},
  sanitizeHtml.defaults.allowedAttributes,
  {
    img: ['alt', 'src']
  }
)

export default function NotePreview({ children }) {
  return (
    <div className="note-preview">
      <div
        className="text-with-markdown"
        dangerouslySetInnerHTML={{
          __html: sanitizeHtml(marked(children || ''), {
            allowedTags,
            allowedAttributes
          })
        }}
      />
    </div>
  )
}

```

其中，[marked](https://www.npmjs.com/package/marked) 是一个把 markdown 转换为 HTML 的库，[sanitize-html](https://www.npmjs.com/package/sanitize-html) 用于清理 HTML，比如删除一些不良的写法，转义特殊字符等。因为用到了这两个库，我们还需要安装一下：

```bash
npm i marked sanitize-html
```

此时页面已经可以正常运行了，效果如下：

![笔记预览.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a67f7d6f5bac4059ae96c59d3689b0d6~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1031\&h=642\&s=131947\&e=gif\&f=47\&b=f4f6f9)

## 原理解析

### RSC Payload

现在让我们多点击几次左侧的笔记列表，切换查看不同的笔记。你会发现，页面的地址虽然发生了变化，但页面并没有重新加载，但是页面的内容确实发生了变化，Next.js 是怎么实现的呢？

让我们查看下点击笔记时的请求（执行 `npm run start`时）：

![截屏2023-12-16 下午7.58.57.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1e367cd4bb7b4b06a9fd94555461ac2f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=3610\&h=1678\&s=675860\&e=png\&b=fdfdfd)

点击笔记的时候，我们请求了对应路由的地址，比如我们点击第一篇笔记的时候，这篇笔记的 `noteId` 为 `1702459188837`，发出的请求地址就对应为 `http://localhost:3000/note/1702459188837?_rsc=9ehs5`，Chrome 标注这条请求的 Type 是 fetch，返回的数据为：

```javascript
3:I[5613,[],""]
5:I[1778,[],""]
4:["id","1702459188837","d"]
0:["S5DEOJMw4dANsj-nNd4RK",[["children","note",["note",{"children":[["id","1702459188837","d"],{"children":["__PAGE__",{}]}]}],["note",{"children":[["id","1702459188837","d"],{"children":["__PAGE__",{},["$L1","$L2",null]]},["$","$L3",null,{"parallelRouterKey":"children","segmentPath":["children","note","children","$4","children"],"loading":["$","div",null,{"className":"note skeleton-container","role":"progressbar","aria-busy":"true","children":[["$","div",null,{"className":"note-header","children":[["$","div",null,{"className":"note-title skeleton","style":{"height":"3rem","width":"65%","marginInline":"12px 1em"}}],["$","div",null,{"className":"skeleton skeleton--button","style":{"width":"8em","height":"2.5em"}}]]}],["$","div",null,{"className":"note-preview","children":[["$","div",null,{"className":"skeleton v-stack","style":{"height":"1.5em"}}],["$","div",null,{"className":"skeleton v-stack","style":{"height":"1.5em"}}],["$","div",null,{"className":"skeleton v-stack","style":{"height":"1.5em"}}],["$","div",null,{"className":"skeleton v-stack","style":{"height":"1.5em"}}],["$","div",null,{"className":"skeleton v-stack","style":{"height":"1.5em"}}]]}]]}],"loadingStyles":[],"loadingScripts":[],"hasLoading":true,"error":"$undefined","errorStyles":"$undefined","errorScripts":"$undefined","template":["$","$L5",null,{}],"templateStyles":"$undefined","templateScripts":"$undefined","notFound":"$undefined","notFoundStyles":"$undefined","styles":null}]]},["$","$L3",null,{"parallelRouterKey":"children","segmentPath":["children","note","children"],"loading":"$undefined","loadingStyles":"$undefined","loadingScripts":"$undefined","hasLoading":false,"error":"$undefined","errorStyles":"$undefined","errorScripts":"$undefined","template":["$","$L5",null,{}],"templateStyles":"$undefined","templateScripts":"$undefined","notFound":"$undefined","notFoundStyles":"$undefined","styles":null}]],[null,"$L6"]]]]
6:[["$","meta","0",{"name":"viewport","content":"width=device-width, initial-scale=1"}],["$","meta","1",{"charSet":"utf-8"}],["$","link","2",{"rel":"icon","href":"/favicon.ico","type":"image/x-icon","sizes":"16x16"}]]
1:null
7:I[5250,["250","static/chunks/250-3c648b94097e3c7b.js","156","static/chunks/app/note/%5Bid%5D/page-5070a024863ac55b.js"],""]
2:["$","div",null,{"className":"note","children":[["$","div",null,{"className":"note-header","children":[["$","h1",null,{"className":"note-title","children":"ea molestias"}],["$","div",null,{"className":"note-menu","role":"menubar","children":[["$","small",null,{"className":"note-updated-at","role":"status","children":["Last updated on ","2023-12-13 05:19:48"]}],["$","$L7",null,{"href":"/note/edit/1702459188837","className":"link--unstyled","children":["$","button",null,{"className":"edit-button edit-button--outline","role":"menuitem","children":"Edit"}]}]]}]]}],["$","div",null,{"className":"note-preview","children":["$","div",null,{"className":"text-with-markdown","dangerouslySetInnerHTML":{"__html":"<p>et iusto sed quo iure</p>\n"}}]}]]}]

```

注：如果你用 Chrome 查看数据的时候，发现无法加载响应数据，那换成其他浏览器如火狐试试。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/61e4946be63d4c19a24db13eb39ff43f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1428\&h=324\&s=31290\&e=png\&b=282828)

这个数据就被称为 `React Server Components Payload`，简称 `RSC Payload`，其实你看这个地址的参数`?rsc=xxxx`其实就暗示了它跟 RSC 相关。查看返回的数据 ，你会发现这个数据很奇怪，既不是我们常见的 HTML、XML，也不是什么其他格式，这就是 React 定义的一种特殊的格式。

还记得[《 缓存篇 | Caching》](https://juejin.cn/book/7307859898316881957/section/7309077169735958565#heading-19)中讲到 RSC Payload 包含哪些信息吗：

1.  服务端组件的渲染结果
2.  客户端组件的占位位置和引用文件
3.  从服务端组件传给客户端组件的数据

比如以 `0:` 开头的那行，根据其中的内容，可以判断出渲染的是笔记加载时的骨架图。以 `2:`开头的那行，渲染的则是笔记的具体内容。

**使用这种格式的优势在于它针对流做了优化，数据是分行的，它们可以以流的形式逐行从服务端发送给客户端，客户端可以逐行解析 RSC Payload，渐进式渲染页面。**

比如客户端收到 `0:`开头的这行，于是开始渲染骨架图。收到 `7:`开头的这行，发现需要下载 `static/chunks/app/note/[id]/page-5070a024863ac55b.js`，于是开始请求该 JS 文件，查看刚才的请求，也确实请求了该文件。收到 `2:`开头的这行，于是开始渲染笔记的具体内容。

因为我们特地设置了请求时间大于 5s，所以 `2:`开头的那行数据返回的时候肯定比 `0:`晚了 `5s`以上，这条请求的时长也确实大于了 5s，这也应证了 RSC Payload 服务端是逐行返回，客户端是逐行解析、渐进式渲染的。

注：你可能发现，还有一个 404 的 RSC Payload 请求，它请求的地址是 `/note/edit/170245918883`，这是因为渲染出的 Edit 按钮用的是 Link 组件，Link 组件有预获取，所以触发了请求，但因为这个路由我们还没写，所以出现了 404 错误。其实跟本篇要讲的内容无关。

现在我们将 `http://localhost:3000/note/1702459188837?_rsc=9ehs5`这个地址在新标签页中打开，你会发现还是这个请求地址，返回的却是 HTML：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9a2f596cd23a475c834da5116271c260~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=3578\&h=1122\&s=553000\&e=png\&b=fdfdfd)

至于怎么实现的，想必你也想到了，两个请求虽然地址一样，但请求头不一样。这是返回 RSC Payload 的请求头：

![截屏2023-12-16 下午8.45.08.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9825f626460c464cb18f5a17dc736c6b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1670\&h=1210\&s=369427\&e=png\&b=282828)

这是返回 HTML 的请求头：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee4c16e1e23b4ba8a19ffcd8249dfd31~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=2530\&h=948\&s=300784\&e=png\&b=292929)

那在这个 HTML 中又是怎么实现初始加载骨架图，然后 5s 后加载笔记数据的呢？

这节在上一篇已经讲过了。简单的来说，该 HTML 也是流式传输，会先后返回骨架图和笔记数据：

```html
<!DOCTYPE html>
<html lang="en">
    <body>
        <div class="container">
            <div class="main">
                <section class="col note-viewer">
                    <!-- 笔记加载时的骨架图 -->
                    <template id="B:1"></template>
                    <div class="note skeleton-container" role="progressbar" aria-busy="true">
                        <div class="note-header">
                            <div class="note-title skeleton" style="height:3rem;width:65%;margin-inline:12px 1em"></div>
                            <div class="skeleton skeleton--button" style="width:8em;height:2.5em"></div>
                        </div>
                        <div class="note-preview">
                            <div class="skeleton v-stack" style="height:1.5em"></div>
                            <div class="skeleton v-stack" style="height:1.5em"></div>
                            <div class="skeleton v-stack" style="height:1.5em"></div>
                            <div class="skeleton v-stack" style="height:1.5em"></div>
                            <div class="skeleton v-stack" style="height:1.5em"></div>
                        </div>
                    </div>
                    <!--/$-->
                </section>
            </div>
        </div>
      	// ...
        <div hidden id="S:1">
            <div class="note">
                <div class="note-header">
                    <h1 class="note-title">ea molestias</h1>
                    <div class="note-menu" role="menubar">
                        <small class="note-updated-at" role="status">Last updated on 
                        <!-- -->
                        2023-12-13 05:19:48</small>
                        <a class="link--unstyled" href="/note/edit/1702459188837">
                            <button class="edit-button edit-button--outline" role="menuitem">Edit</button>
                        </a>
                    </div>
                </div>
                <div class="note-preview">
                    <div class="text-with-markdown">
                        <p>et iusto sed quo iure</p>
                    </div>
                </div>
            </div>
        </div>
        <script>
            // 替换内容
            $RC = function(b, c, e) { // ... }
            $RC("B:1", "S:1")
        </script>
        // ...
    </body>
</html>

```

我们接着讲 RSC Payload，那客户端获取到 RSC Payload 后还干了什么呢？其实就是根据 RSC Payload 重新渲染组件树，修改 DOM。但使用 RSC Payload 的好处在于组件树中的状态依然会被保持，比如左侧笔记列表的展开和收回就是一种客户端状态，当你新增笔记、删除笔记时，虽然组件树被重新渲染，但是客户端的状态依然会继续保持了。

这也被认为是 SSR 和 RSC 的最大区别，其实现的关键就在于服务端组件没有被渲染成 HTML，而是一种特殊的格式（RSC Payload）。这里让我们再复习下 SSR（传统的 SSR，想想 Pages Router 下的 SSR 实现） 和 RSC 的区别：

1.  RSC 的代码不会发送到客户端，但传统 SSR 所有组件的代码都会被发送到客户端
2.  RSC 可以在组件树中任意位置获取后端，传统 SSR 只能在顶层（getServerSideProps）访问后端
3.  服务器组件可以重新获取，而不会丢失其树内的客户端状态

注：这里虽然比较了 SSR 和 RSC，但并不是说明两者是冲突的，其实 SSR 和 RSC 是互补关系，是可以一起使用的，Next.js 中两者就是一起使用的。

### 路由缓存

现在让我们再多点击几次左侧的笔记列表，切换查看不同的笔记，你会发现有一件奇怪的事情（这个 GIF 有 30 多秒）：

![路由缓存.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7e1242d79fa149eb9dc99b75b1663b47~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1031\&h=642\&s=628380\&e=gif\&f=223\&b=f3f6f9)

点击切换不同的笔记，你会发现同样一条笔记，有时会触发数据的重新请求（出现了骨架图），但有的时候又没有，但有的时候又会重新出现（又出现了骨架图），这是为什么吗？

这就是 Next.js 提供的客户端路由缓存功能，客户端会缓存 RSC Payload 数据，所以当点击笔记后很快再次点击，这时就会从缓存中获取数据，那么问题来了，缓存的失效逻辑还记得吗？具体会缓存多久呢？我们在[缓存篇](https://juejin.cn/book/7307859898316881957/section/7309077169735958565#heading-20)中和大家讲过，回忆下基础知识：

路由缓存存放在浏览器的临时缓存中，有两个因素决定了路由缓存的持续时间：

*   **Session，缓存在导航期间会持续存在，当页面刷新的时候会被清除**
*   **自动失效期：单个路由段会在特定时长后自动失效，如果路由是静态渲染，持续 5 分钟，如果是动态渲染，持续 30s**

这个例子中因为我们用的是动态路由，是动态渲染，缓存持续 30s，所以首次点击笔记获取 RSC Payload 数据 30s 后再点击就会重新获取 RSC Payload。

小问题：以这个项目为例，如果点击笔记的时间算成 0s，因为请求时长大于 5s，假设 RSC Payload 在第 5s 完全返回，下次路由缓存失效重新获取的时间是大概在 30s 后还是 35s 后呢？

答案是 30s。以 RSC Payload 的返回时间为准，RSC Payload 是逐行返回的，所以点击的时候很快就有返回了。

## 总结

那么今天的内容就结束了，本篇的内容并不多，主要是通过笔记预览这个例子，在实际开发中加深对 RSC Payload 和路由缓存的理解。

本篇的代码我已经上传到[代码仓库](https://github.com/mqyqingfeng/next-react-notes-demo/tree/main)的 Day3 分支：<https://github.com/mqyqingfeng/next-react-notes-demo/tree/day3>，直接使用的时候不要忘记在本地开启 Redis。
