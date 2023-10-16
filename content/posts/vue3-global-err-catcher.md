---
weight: 2
title: "简易 Vue 通用错误处理 hooks 封装"
summary: "将代码中同步和异步函数调用错误捕获都封装到 hooks 里面"
date: 2022-12-10T11:24:22+08:00
draft: false
categories: ["前端"]
tags: ["Vue.js",Vue3","hooks","错误处理"]
---

## 前言

为了统一错误处理的规范，将常见的错误处理封装到一个 hook 里面方便在业务代码中调用。

本次封装只会考虑两种常见错误：

* JavaScript 语法异常及运行时错误
* 异步请求异常

&nbsp;

## 代码运行时错误处理

运行时错误处理提供两种处理方式以应对不同的使用场景：

* 辅助函数返回一个新的函数供用户自行调用
* 辅助函数直接调用传入的函数

&nbsp;

### bind 方式

```javascript
export function useErrHelper() {
    
  const bindWithErr = (fn) => {
    return (...args) => {
      try {
        return fn && fn(...args)
      } catch (err) {
        console.error('捕获到了错误: ', err)
      }
    }
  }
  
  return {
  	bindWithErr
  }
}
```

### bind 用法

这个辅助函数会将用户传入的参数包裹到 `try catch` 中执行，并返回一个新的函数，让用户自行调用。

适合用于注册未页面 handler 的回调函数。

用一个自己的业务场景举例：一个下拉框组件的回调参数可能会包含一个字符串数学表达式，而我们需要将这个表达式执行，并避免执行错误时程序崩溃：

```html
<!-- 例，伪代码 -->
<el-select @change="setCamera"></el-select>
```

```javascript
/**
 * 设置相机参数
 * @param {* Object} val 相机参数对象
 */
const setCamera = bindWithErr((val) => {
  if (!val) return
  cameraCondition.value = {
    cmos: eval(val.cmos),
    shutterSpeed: eval(val.shutterSpeed),
  }
})
```

&nbsp;

### call 方式

```javascript
export function useErrHelper() {
   
  const callWithErr = (fn, ...args) => {
    try {
      return fn && fn(...args)
    } catch (err) {
      console.error('捕获到了错误: ', err)
    }
  }
  
  return {
  	callWithErr
  }
}
```

### call 用法

就是直接调用用户传入的参数，比较简单粗暴，不展开解析。

&nbsp;

## 异步错误处理

```javascript
export function useErrHelper() {
    /**
   * 异步调用错误捕获
   * @param {* Function} promise 异步函数，一般是接口调用
   * @param {* String} successMsg 成功时的提示信息，如果不需要提示，就不要传
   * @param {* String} failMsg 失败时的提示信息，如果不需要提示或者需要使用后端返回的信息，就不要传（建议不传）
   * @returns 返回 null 或者  promise 的结果
   */
  const awaitWithErr = async (promise, options = { successMsg: '', failMsg: '' }) => {
    const [err, res] = await promise.then((data) => [null, data]).catch((err) => [err, null])
    if (err || res.data.code >= 500) {
      let msg
      if (options.failMsg) {
        msg = options.failMsg
      } else {
        msg = err ? err.message : res.data.msg
      }
      ElMessage({ showClose: false, message: msg, type: 'error' })
      return null
    } else if (res.data.code == 200) {
      if (options.successMsg) ElMessage({ showClose: false, message: options.successMsg, type: 'success' })
      return res
    }
  }
    return {
    awaitWithErr
  }
}
```

### 用法

* 这个辅助函数将 ElementUI 的提示函数封装在里面，减少了业务组件中不必要的代码，并且会根据是否传入成功或错误信息，判断是否需要进行提示。

* `500`和`200`是后端定义的业务状态码，可以根据实际业务代码进行替换，或直接抽取到 options 中作为参数传入。
* 注意该辅助函数的返回结果为 `null` 或者 `promise 异步数据`，也就是说在业务组件中，只需要判断返回值是否为空即可。

```javascript
// 不需要任何提示的接口调用，如列表等分页接口
const res = await awaitWithErr(queryList(keyWord))
if (!res) {
    // 异步错误时 do something ...
}
```

```javascript
// 需要提示的接口调用，如编辑等操作
const res = await awaitWithErr(deleteById(id), { successMsg: '删除成功' })
if (res) {
  // 操作成功时 do something ...
}
```

&nbsp;

### allSettled 并发版本

```javascript
  // 并发异步调用辅助函数（allSettled 版本）
  const allSettledWithErr = async (promises, options = { successMsg: '', failMsg: '' }) => {
    const results = await Promise.allSettled(promises)
    const ress = results.filter((result) => result.status === 'fulfilled')
    const result = results.map((result) => {
      if (result.status === 'fulfilled') {
        return result.value
      } else {
        console.error(result.reason, '异步调用错误信息')
        return null
      }
    })
    if (ress.length) {
      if (options.successMsg) ElMessage({ showClose: false, message: options.successMsg, type: 'success' })
      return result
    } else {
      if (options.failMsg) {
        ElMessage({ showClose: false, message: options.failMsg, type: 'error' })
      }
      return null
    }
  }
```

### 用法

并发版本用法跟普通版本一致，简化了并发请求时的冗余处理。

&nbsp;

（完）

