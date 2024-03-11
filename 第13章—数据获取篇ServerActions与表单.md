## 前言
表单是 Web 应用中一个常见的组成部分。Next.js 提供了强大的 Server Actions 功能来处理表单数据的创建和更新。

本章我们会先单独讲讲 Server Actions，为大家介绍如何开启和使用 Server Actions，然后再来讲解 Server Actions 与表单如何结合，以及表单处理中的常见问题该如何解决。
## 1. Server Actions
### 1.1. 效果概述
Server Actions 是 Next.js 内置的关于服务端数据更改的解决方案。简单的来说，Server Actions 正如它的名字，指的是可以在服务端运行的函数，但它可以在客户端被调用，就像正常的函数一样，而无须手动创建一个 API 路由。

为了更加直观的了解 Server Actions 的应用场景，让我们以表单提交为例。传统我们写一个表单提交，代码大致如下：

```javascript
import { FormEvent } from 'react'

export default function Page() {
  async function oSubmit(event) {
    event.preventDefault()

    const formData = new FormDate(event.currentTarget)
    const response = await fetch('/api/submit', {
      method: 'POST',
      body: formData,
    })

    const data = await response.json()
    // ...
  }
  return (
    <form onSubmit={onSubmit}>
      <input type="text" name="name" />
      <button type="submit">Submit</button>
    </form>
  )
}
```

我们首先会创建一个名为 `/api/submit`的 API 路由，然后用该 API 处理提交的数据，由此实现客户端与服务端通信。

这是使用 Next.js Pages Router 时的解决方案。但是使用 App Routers，借助 Server Actions，实现代码可以改为：

```javascript
// page.js
export default function Page({ params }) {
  async function onSubmit() {
    'use server';
    // ...
  }
 
  return (
    <form action={onSubmit}>
      <input type="text" name="name" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

无须创建 API 进行交互，直接定义一个 onSubmit 异步函数进行调用即可。不过要注意的是，这次使用的不再是 onSubmit 事件，而是 form 的 action 属性。

回顾一下基础的 HTML 知识，action 属性和 onsubmit 事件都与表单提交行为有关。onsubmit 用于表单提交时执行 JavaScript，一般用于验证数据格式。action 属性表示处理表单提交的 URL，浏览器会将表单数据封装成一个 HTTP 请求，将其发送到 action 属性指定的地址。如果 obsubmit 事件处理程序返回 false，浏览器将取消表单提交，也就不会触发 action 指定地址的数据发送。

```html
<form action="submit.php" onsubmit="return validateForm()">
  <input type="text" name="username">
  <button type="submit">提交</button>
</form>

<script>
function validateForm() {
  if (/* 验证不通过 */) {
    return false;
  }
  return true;
}
</script>
```

使用 Server Actions 还会带来一个好处，就是因为借助的是 HTML 的 action 属性，这使得即使 JavaScript 没有加载完毕或是禁用 JavaScript，表单依然是可以正常使用的。这就实现了渐进式功能增强。

除此之外，传统开发应用的时候，往往一个路由一个表单，因为提交表单的时候，页面正常会刷新，但使用 Server Action，支持一个路由多个表单，且浏览器不会在提交表单的时候刷新。这样就可以在一次网络请求中实现数据和 UI 更新。
### 1.2. 开启使用
现在让我们正式的开始学习使用 Server Actions 吧。

使用 Server Actions，你需要先通过 experimental 的 `serverActions` 配置项开启：

```javascript
// next.config.js
module.exports = {
  experimental: {
    serverActions: true,
  },
}
```

Next.js v14 以后，Server Actions 默认可用，使用 Next.js v14及以后的版本就不用开启了。

Server Actions 可以在两个地方被定义：

1. 使用它的组件内部（仅限服务端组件）
2. 在一个单独的文件（客户端组件和服务端组件），目的在于实现复用。你也可以在一个文件里定义多个 Server Action

让我们逐个介绍。
### 1.3. 服务端组件中使用
在服务端组件中创建 Server Actions，你需要定义一个异步函数，该函数的函数体顶部使用 `"use server"`指令。`"use server"`是为了确保该函数只会在服务端被执行。示例代码如下：

```javascript
// app/page.js
export default function ServerComponent() {
  async function myAction() {
    'use server'
    // ...
  }
}
```

要注意：该函数需要使用可序列化的参数以及可序列化的返回值。其原因在服务端组件一节也有讲过，函数返回的结果会被序列化后发送给客户端。
### 1.4. 客户端组件中使用
如果想在客户端组件中使用 Server Actions，第一种方式是导入。首先在一个单独的文件中创建 Server Action，该文件顶部需要一个 `"use server"`指令。然后在客户端组件导入该 Server Action，示例如下：

```javascript
'use server'
// app/actions.js
export async function myAction() {
  // ...
}
```

注意：当使用这种顶层的 'use server' 指令的时候，下面所有的导出都会被认为是 Server Actions，所以你可以在一个文件里定义多个 Server Action。

```javascript
'use client'
// app/client-component.jsx
import { myAction } from './actions'
 
export default function ClientComponent() {
  return (
    <form action={myAction}>
      <button type="submit">Add to Cart</button>
    </form>
  )
}
```

第二种方式是作为 props 传递给客户端组件，示例代码如下：

```javascript
<ClientComponent updateItem={updateItem} />
```

```javascript
'use client'
// app/client-component.jsx
export default function ClientComponent({ updateItem }) {
  return (
    <form action={updateItem}>
      <input type="text" name="name" />
      <button type="submit">Update Item</button>
    </form>
  )
}
```
### 1.5. 绑定参数
你可以使用 `bind` 方法为 Server Actions 绑定参数。这会创建一个新的 Server Action，其中部分参数被绑定。这个技巧有的时候会很实用，示例如下：

```javascript
'use client'
 // app/client-component.jsx
import { updateUser } from './actions'
 
export function UserProfile({ userId }) {
  const updateUserWithId = updateUser.bind(null, userId)
 
  return (
    <form action={updateUserWithId}>
      <input type="text" name="name" />
      <button type="submit">Update User Name</button>
    </form>
  )
}
```

`updateUser` Server Action 的代码如下：

```javascript
'use server'
// app/actions.js
export async function updateUser(userId, formData) {
  // ...
}
```
### 1.6. 调用
现在我们已经知道如何开启和创建 Server Action 了，而关于如何调用，Server Actions 有三种调用方式：

第一种方式是使用 `action`，React 的 `action` prop 支持在 `<form>` 元素上调用一个 Server Action。在这里就不举例了，前面的例子都是这种形式。

第二种方式是使用 `formAction`，React 的 `formAction` prop 允许在 `<form>` 中处理 `<button>`, `<input type="submit">`, 和 `<input type="image">`元素。示例代码如下：

```javascript
<form action={handleSubmit}>
    <input type="submit" name="name" formAction={handleName} />
    <button type="submit">Submit</button>
</form>
```

不过说起 formaction，这其实是 HTML5 中的属性，formaction 属性只能作用于具有提交功能的按钮（也就是`<button>`、`<input type="submit">`、`<input type="image">`）。如果通过具有 formaction 属性的按钮提交表单，数据发送的地址会是 formaction 指定的地址而非 form 上的 action 指定的地址。

第三种方式是直接在事件处理程序（比如 onClick）中调用，不过使用这种方式就不具备渐进式增强的特性了，举个例子：

```javascript
'use client'
 
import { incrementLike } from './actions'
import { useState } from 'react'
 
export default function LikeButton({ initialLikes }) {
  const [likes, setLikes] = useState(initialLikes)
 
  return (
    <>
      <p>Total Likes: {likes}</p>
      <button
        onClick={async () => {
          const updatedLikes = await incrementLike()
          setLikes(updatedLikes)
        }}
      >
        Like
      </button>
    </>
  )
}
```

Next.js 会推荐搭配使用 [useTransition](https://react.dev/reference/react/useTransition) 或者是 [useOptimistic](https://react.dev/reference/react/useOptimistic) 这种 React hook。示例代码如下：

```javascript
'use client'

import { useTransition } from 'react'
import { addTodo } from '@/actions/addTodo';

export default function CourseComment() {
  let [isPending, startTransition] = useTransition()

  return (
    <button onClick={() => startTransition(() => { 
      addTodo()})
    }>
      Add Todo
    </button>
  )
}

```
### 1.7. 关于渐进式增强
渐进式增强下，即使没有 JavaScript 或者 JavaScript 被禁用，`<form>` 依然可以正常运行。这就使得在表单的 JavaScrpt 尚未加载完毕或者加载失败的时候，用户也能与表单交互并提交数据。

React Actions 支持渐进式增强，这又具体分两种情况：

1. 如果在服务端组件调用 Server Action，那表单就是可以在没有 JavaScript 的时候正常运行
2. 如果在客户端组件调用 Server Action，表单依然是可交互的，但是该 Action 会被放到一个队列中，直到该表单完全可交互（水合）。React 会提高该 action 的优先级，所以它依然会很快发生。
### 1.8. 大小限制
默认情况下，发送到 Server Action 的请求体最大是 1M 大小，这是为了防止在解析大量数据时消耗过多的服务器资源。

然而，你可以通过 `serverActionsBodySizeLimit` 配置项来修改这个限制。它可以是一个关于字节的数字或是支持字节的任何格式化字符串，，比如 `1000`, `'500kb'` 或者是 `'3mb'`。

```javascript
// next.config.js
module.exports = {
  experimental: {
    serverActions: true,
    serverActionsBodySizeLimit: '2mb',
  },
}
```
## 2. 使用常见问题
### 2.1. 参考示例
Next.js 提供了 forms 与 Server Actions 的示例代码，我们可以通过该命令行创建：

```bash
npx create-next-app@latest -e next-forms
```

这是[该示例代码源码地址](https://github.com/vercel/next.js/tree/canary/examples/next-forms)，官方也做了对应的[视频介绍](https://www.youtube.com/watch?v=dDpZfOQBMaU)。

一个基础的示例代码如下：

```javascript
// app/page.jsx
export default function Home() {
  async function handleFormAction(formData) {
    'use server';
    const name = formData.get('name');
  }

  return (
    <form action={handleFormAction}>
      <input type="text" name="name" />
      <button type="submit">Submit</button>
    </form>
  )
}
```

从这个例子中，我们看到，使用 Server Actions，我们可以很方便的定义服务端处理函数，也可以很方便的从表单数据中取值。

我们也可以结合 React 的 [useFormState](https://react.dev/reference/react-dom/hooks) hook，当然这个 hook 还在实验中。

```javascript
'use client'

import { experimental_useFormState as useFormState } from 'react-dom'

export default function Home() {

  async function createTodo(prevState, formData) {
    return prevState.concat(formData.get('todo'));
  }

  const [state, formAction] = useFormState(createTodo, [])

  return (
    <form action={formAction}>
      <input type="text" name="todo" />
      <button type="submit">Submit</button>
      <p>{state.join(',')}</p>
    </form>
  ) 
}

```

使用 useFormState 的时候要注意，该组件需要在客户端组件中使用。
### 2.2. 重新验证
Server Actions 允许你按需重新验证数据。你可以使用 [revalidatePath](https://juejin.cn/book/7307859898316881957/section/7309079586296791050#heading-12)：

```javascript
'use server'
// app/actions.js
import { revalidatePath } from 'next/cache'
 
export default async function submit() {
  await submitForm()
  revalidatePath('/')
}
```

或者使用 [revalidateTag](https://juejin.cn/book/7307859898316881957/section/7309079586296791050#heading-23)：

```javascript
'use server'
 // app/actions.js
import { revalidateTag } from 'next/cache'
 
export default async function submit() {
  await addPost()
  revalidateTag('posts')
}
```
### 2.3. 重定向
如果你希望 Server Action 结束之后重定向到其他路由，你可以使用 [redirect](https://juejin.cn/book/7307859898316881957/section/7309079651500949530#heading-44) 到一个相对或者固定地址。

```javascript
'use server'
 // app/actions.js
import { redirect } from 'next/navigation'
import { revalidateTag } from 'next/cache'
 
export default async function submit() {
  const id = await addPost()
  revalidateTag('posts') // Update cached posts
  redirect(`/post/${id}`) // Navigate to new route
}
```
### 2.4. 表单验证
Next.js 推荐基本的表单验证使用 HTML 元素自带的验证如 `required`、`type="email"`等。

对于更高阶的服务端数据验证，可以使用 [zod](https://zod.dev/) 这样的 schema 验证库来验证表单数据的结构：

```javascript
// app/actions.js
import { z } from 'zod'
 
const schema = z.object({
  // ...
})
 
export default async function submit(formData) {
  const parsed = schema.parse({
    id: formData.get('id'),
  })
  // ...
}
```
### 2.5. 显示加载状态
当表单提交数据的时候，可以使用 `useFormStatus`hook 来显示加载状态。`useFormStatus` hook 只能用于 form 元素的子级。

关于 `useFormStatus`的具体使用用法，可以参考 [React 官方文档中的介绍](https://react.dev/reference/react-dom/hooks/useFormStatus)。

示例代码如下：

```javascript
'use client'
// app/submit-button.jsx
import { useFormStatus } from 'react-dom'
 
export function SubmitButton() {
  const { pending } = useFormStatus()
 
  return (
    <button type="submit" aria-disabled={pending}>
      {pending ? 'Adding' : 'Add'}
    </button>
  )
}
```

`<SubmitButton />`可以被用在使用 Server Action 的表单中：

```javascript
// app/page.jsx
import { SubmitButton } from '@/app/submit-button'
 
export default async function Home() {
  return (
    <form action={...}>
      <input type="text" name="field-name" />
      <SubmitButton />
    </form>
  )
}
```
### 2.6. 错误处理
Server Action 可以返回可序列化的对象。举个例子，当一个条目创建失败，你需要处理这个错误：

```javascript
'use server'
// app/actions.js
export async function createTodo(prevState, formData) {
  try {
    await createItem(formData.get('todo'))
    return revalidatePath('/')
  } catch (e) {
    return { message: 'Failed to create' }
  }
}
```

然后你就可以在客户端组件中，读取这个值并显示这个错误信息：

```javascript
'use client'
// app/add-form.jsx
import { useFormState, useFormStatus } from 'react-dom'
import { createTodo } from '@/app/actions'
 
const initialState = {
  message: null,
}
 
function SubmitButton() {
  const { pending } = useFormStatus()
 
  return (
    <button type="submit" aria-disabled={pending}>
      Add
    </button>
  )
}
 
export function AddForm() {
  const [state, formAction] = useFormState(createTodo, initialState)
 
  return (
    <form action={formAction}>
      <label htmlFor="todo">Enter Task</label>
      <input type="text" id="todo" name="todo" required />
      <SubmitButton />
      <p aria-live="polite" className="sr-only">
        {state?.message}
      </p>
    </form>
  )
}
```

当然你也可以结合 `error.jsx` 展示错误时的 UI：

```javascript
'use client'
// error.jsx
export default function Error() {
  return (
    <h2>error</h2>
  )
}
```

```javascript
// page.jsx
import { experimental_useFormState as useFormState } from 'react-dom'

function AddForm() {
  async function serverActionWithError() {
    'use server';   
    throw new Error(`This is error is in the Server Action`);
  }

  return (
    <form action={serverActionWithError}>
      <button type="submit">Submit</button>
    </form>
  ) 
}

export default AddForm
```

这样当 Server Action 发生错误的时候，就会展示错误 UI。
### 2.7. 乐观更新
所谓乐观更新，举个例子，当用户点击一个点赞按钮的时候，传统的做法是等待接口返回成功时再更新 UI，乐观更新是先更新 UI，同时发送数据请求，至于数据请求后的错误处理，则根据自己的需要自定义实现。

React 提供了 `useOptimistic` 这个 hook，结合 Server Actions 使用的示例代码如下：

```javascript
'use client'
// app/page.jsx
import { experimental_useOptimistic as useOptimistic } from 'react'
import { send } from './actions'
 
export function Thread({ messages }) {
  const [optimisticMessages, addOptimisticMessage] = useOptimistic(
    messages,
    (state, newMessage) => [...state, { message: newMessage }]
  )
 
  return (
    <div>
      {optimisticMessages.map((m) => (
        <div>{m.message}</div>
      ))}
      <form
        action={async (formData) => {
          const message = formData.get('message')
          addOptimisticMessage(message)
          await send(message)
        }}
      >
        <input type="text" name="message" />
        <button type="submit">Send</button>
      </form>
    </div>
  )
}
```
### 2.8. 设置 Cookies
你可以在 Server Action 中使用 [cookies](https://juejin.cn/book/7307859898316881957/section/7309079651500949530#heading-7) 函数设置 cookie：

```javascript
'use server'
// app/actions.js
import { cookies } from 'next/headers'
 
export async function create() {
  const cart = await createCart()
  cookies().set('cartId', cart.id)
}
```
### 2.9. 读取 Cookies
依然是使用 [cookies](https://juejin.cn/book/7307859898316881957/section/7309079651500949530#heading-7) 函数：

```javascript
'use server'
// app/actions.js
import { cookies } from 'next/headers'
 
export async function read() {
  const auth = cookies().get('authorization')?.value
  // ...
}
```
### 2.10. 删除 Cookies
还是使用  [cookies](https://juejin.cn/book/7307859898316881957/section/7309079651500949530#heading-7) 函数：

```javascript
'use server'
// app/actions.js
import { cookies } from 'next/headers'
 
export async function delete() {
  cookies().delete('name')
  // ...
}
```
## 参考链接

1. [Data Fetching: Fetching, Caching, and Revalidating](https://nextjs.org/docs/app/building-your-application/data-fetching/fetching-caching-and-revalidating)
2. [Data Fetching: Data Fetching Patterns](https://nextjs.org/docs/app/building-your-application/data-fetching/patterns)
3. [Data Fetching: Forms and Mutations](https://nextjs.org/docs/app/building-your-application/data-fetching/forms-and-mutations)
4. [Functions: Server Actions](https://nextjs.org/docs/app/api-reference/functions/server-actions)
5. [https://makerkit.dev/blog/tutorials/nextjs-server-actions](https://makerkit.dev/blog/tutorials/nextjs-server-actions)
