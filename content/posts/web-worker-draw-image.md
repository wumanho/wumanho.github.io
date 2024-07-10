---
weight: 2
title: "前端使用 Web Worker 和 OffscreenCanvas 异步生成图片"
summary: "本篇博客将会介绍前端如何使用 Web Worker API 结合离屏 canvas 在不占用浏览器主线程的情况下生成图片"
date: 2024-07-09T20:24:07+08:00
draft: false
categories: ["前端"]
tags: ["OffscreenCanvas","Web Worker","Vue","Vue3","异步"]
---

## 前言

想象一下有这样一个需求：给你一张图片以及一组矩形坐标，需要在前端将矩形绘制到原图上，**并生成一张新的图片，最后将原图和对比图，都展示出来。**

这个需求如果每次需要处理的图片只有一张，那么可以用 canvas API 来完成，但如果将**数量**修改一下，变成：

给你**一堆**图片，每张图片对应一组矩形坐标，需要在前端绘制，并将原图和对比图都展示出来。

在这种情况下，图片数量不确定，如果还是简单使用 canvas API 来进行实现，可能会由于数量太大而导致主线程被长时间阻塞，最终影响交互体验。

所以，将这部分绘制的工作交给 Web Worker 是非常合适的，起码在图片生成的过程中，主线程可以继续执行业务代码而不至于造成整体卡顿。

### 例：原图

![source](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/webworkerdraw/source.jpeg)

### 例：修改后

![edit](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/webworkerdraw/draw.jpg)

&nbsp;

## 如何使用 Web Worker

要使用 Web Worker，只需要先创建一个 Worker 对象，在参数中传入一个包含 Worker 脚本的文件地址，作为提供给 Worker 执行的代码即可。

```javascript
const worker = new Worker('imgDrawer.js');
```

不过在生产项目中，由于文件最终会被打包，为了避免可能会发生的资源路径错误问题，建议使用 `import.meta.url` 的方式去引用文件：

```javascript
// worker 路径
const workerPath = new URL("./worker/imgDrawer.js", import.meta.url).href;
const worker = new Worker(workerPath);
```

这种引用方式是 ESM 的原生功能，可以放心使用。

主线程和 Worker 之间通过 `postMessage` 传递消息，进行线程间通讯，以及数据交互工作

例如，我需要往 Worker 中传入一个 id 数据，我只需要这样做：

```javascript
// main.js
const id = uuid();
worker.postMessage(id);
```

在 Worker 脚本中，则通过 `onMessage` 的方式，监听主线程发过来的消息：

```javascript
// imgDrawer.js
onmessage = handleMessage

/**
 * 处理消息
 * @param {*} msg
 */
function handleMessage(msg) {
  console.log('主线程的消息:', msg.data);
}
```

如果需要在 Worker 线程往主线程发送消息，交互方式是一模一样的，在 Worker 中，我们可以直接使用 `postMessage` 发送消息：

```javascript
// imgDrawer.js
postMessage('ok');
```

在主线程中，用 `worker.onmessage` 来接收：

```javascript
// main.js
worker.onmessage = function(event) {
  console.log('worker 的消息:', event.data);
};
```

&nbsp;

## 为什么要使用 OffscreenCanvas

首先我们需要明确的一点是，我们通过 `postMessage` 传递到 Worker 的数据，都需要经过克隆，而不是单纯的地址传递。

换句话说，`postMessage` 传递的数据**仅支持**可以被[结构化克隆算法](https://developer.mozilla.org/zh-CN/docs/Web/API/Web_Workers_API/Structured_clone_algorithm)法处理的值或 JavaScript 对象。

在实际开发过程中，有很多非常常用的数据无法被传递到 Worker 中，例如 Vue3 中的响应式对象，还有 DOM 节点等。

既然 DOM 节点无法被传递，那就意味着 canvas DOM 无法被传递，所以使用 `OffscreenCanvas` 就是为了解决这个问题。

**`OffscreenCanvas`** 提供了一个可以脱离屏幕渲染的 canvas 对象。它在窗口环境和 Web Worker 环境均有效。

&nbsp;

## 需求实现：初始化 Worker 和 OffscreenCanvas

知道了这些基础用法之后，我们就可以着手开发我们的需求，这次 demo 就用到两个文件：

```bash
├── App.vue
├── worker
│   ├── imgDrawer.js
```

Vue 文件作为我们的主线程，里面包含业务代码，imgDrawer.js 则是提供给 Worker 执行的脚本文件。

由于 canvas DOM 不需要展示在屏幕上，所以在初始化时创建出来就可以了，不需要再额外写一个 DOM 元素：

```vue
<script setup>
  import { onMounted, onBeforeUnmount, toRaw } from "vue";    
  // 绘制标注的 canvas 元素
  let tempCanvas = null;
  // worker 对象
  let worker = null;
  // worker 路径
  const workerPath = new URL("./worker/imgDrawer.js", import.meta.url).href;
  // 初始化
  onMounted(() => {
    // 创建一个 OffscreenCanvas
    tempCanvas = document.createElement("canvas").transferControlToOffscreen();
    worker = new Worker(workerPath);
    // 传给 worker
    worker.postMessage({ canvas: tempCanvas }, [tempCanvas]);
    // 注册回调，获取 worker 给出来的消息
    worker.onmessage = handleWorkerMessage;
  });
  /**
   * 处理 worker 返回的事件
   * @param {Object} msg 消息对象
   */
  function handleWorkerMessage(msg) {
  }
</script>
```

在 Worker 脚本中，我们需要处理这些从主线程发送过来的消息。

在线程间通讯的时候，一个消息可能会包含多个不同的任务，我们可以设计一种结构，根据不同的动作，作出不同的处理，例如：

```javascript
worker.postMessage({
  action: "draw",
  data: {
  	id: '123',
    file:file  
  },
});
```

这样，在传递消息的时候，就可以通过 `action`字段来执行对应的操作了。

首先第一步，我们要将主线程初始化好的 OffscreenCanvas 对象保存起来：

```javascript
// imgDrawer.js
onmessage = handleMessage
// canvas 对象
let canvas = null

/**
 * 处理消息
 * @param {*} msg
 */
function handleMessage(msg) {
  if (msg.data.canvas) {
    canvas = msg.data.canvas
  } else {
    handleActions(msg.data)
  }
}

/**
 * 处理动作
 * @param {Object} msg
 */
function handleActions(msg) {}
```

&nbsp;

## 需求实现：绘制矩形

假设，我们获取到的数据结构如下：

```javascript
const file = {
    id:'', // 唯一 id
    dataUrl:'', // 图片路径
    bboxList: [[50,60,100,120],[240,160,120,140]] // 坐标二维数组
}
const fileList = [file,file,file]
```

我们只需要遍历 `fileList`，将每个 `file` 对象丢给 Worker 处理

但这里我们会遇到第一个麻烦的点

前面说到 Worker 里面并不支持操作 DOM，所以我们没有办法在 Worker 脚本中为 canvas 加载图片

而需要先将图片图片在主线程中创建出来，再将图片以 `ImageBitmap` 的格式传递到 Worker 中，才能被 canvas 加载

```vue
<script setup>
function sendActionToWorker() {
  fileList.forEach((file) => {
      drawMarks(file);
  });
}
// 处理绘制逻辑    
function drawMarks(file) {
    const imgUrl = file.dataUrl;
    const image = new Image();
    // 必须要设置，否则无法生成图片
    image.crossOrigin = "Anonymous";   
    image.onload = () => {
      createImageBitmap(image).then((imgBitmap) => {
        // 将 ImageBitmap 传递到 Worker
        worker.postMessage({ imgBitmap, id: file.id }, [imgBitmap]);
        // 发送绘制请求  
        worker.postMessage({
          action: "draw",
          data: {
            file: toRaw(file), // 这里也是一个坑，如果你的对象是一个响应式对象，记得 toRaw
            width: image.width,
            height: image.height
          },
        });
      });
    };
    // 加载图片
    image.src = imgUrl;
}
</script>
```

在 Worker 中，处理这些数据和进行绘制动作。

由于 Worker 是并发处理的，所以我们可以创建一个 Map 对象， 将 `file` 的唯一 id 和它对应的 ImageBitmap 对应起来：

```javascript
const imgMap = new Map()

/**
 * 处理消息
 * @param {*} msg
 */
function handleMessage(msg) {
  // 保存 canvas 对象
  if (msg.data.canvas) {
    canvas = msg.data.canvas
  } else if (msg.data.imgBitmap) {
    imgMap.set(msg.data.id, msg.data.imgBitmap) // 保存 ImageBitMap
  } else {
    handleActions(msg.data)
  }
}

/**
 * 处理动作
 * @param {Object} msg
 */
function handleActions(msg) {
  const { action, data } = msg
  switch (action) {
    case 'draw': {
      const ctx = canvas.getContext('2d')
      const { file, width, height } = data
      canvas.width = width
      canvas.height = height
      // 获取对应的图片，加载到 ctx
      const img = imgMap.get(file.id)
      ctx.drawImage(img, 0, 0)
      // 创建一个新对象，用于返回
      const markFile = {
        ...file,
        sourceId: file.id,
        id: file.id + '_mark'
      }
      // 标记边框颜色
      ctx.strokeStyle = 'red'
      // 标记线宽
      ctx.lineWidth = 6
      // 遍历所有框坐标，画框
      for (let idx = 0; idx < file.bboxList.length; idx++) {
        const element = file.bboxList[idx]
        ctx.strokeRect(...element)
      }
      // 这里为了兼容性，判断了一下调用哪个方法
      canvas[
        canvas.convertToBlob
          ? 'convertToBlob'
          : 'toBlob' // Firefox
      ]().then((blob) => {
        // 标注好之后新的 url
        const newImageDataURL = new FileReaderSync().readAsDataURL(blob)
        markFile['dataUrl'] = newImageDataURL
        markFile['bboxList'] = null
        // 通知主线程  
        self.postMessage({ action: 'done', markFile: markFile })
      })
      break
    }
  }
}
```

到这里还有第二个坑，正常情况下我们通过 `document.createElement("canvas")` 创建出来的普通 canvas 对象是有 `toDataUrl` 方法的

但是 OffscreenCanvas 却没有，惊不惊喜，所以我们还需要先使用 `canvas.convertToBlob` 将 canvas 转成 blob

再用 FileReader 读取进来，`new FileReaderSync().readAsDataURL(blob)` 多进行一次转换操作才行。

最后，在主线程中保存这些处理好的图片对象，进行后续的业务处理，这个需求就完成了：

```vue
<script setup>
/**
* 处理 worker 返回的事件
* @param {Object} msg 消息对象
*/
function handleWorkerMessage(msg) {
  const data = msg.data;
  if (data.action && typeof data.action === "string") {
    switch (data.action) {
      case "done": {
        const { markFile } = data
	    // 处理剩下的业务逻辑...
        break;
      }
    }
  }
}
</script>    
```

&nbsp;

## 总结

之所以写这篇博客记录，是因为在很多场景下，我们前端必须要在这种类似的场景下，对大量的数据进行处理

这个时候想要将主线程释放出来，借助 Web Worker 是必不可少的

但是 Worker 本身又存在非常多的限制，使用起来要绕过很多的坑

所以写一篇博客简单记录一下踩过的坑

此外，这篇博客中所展示的 demo 代码仅作为参考，如果需要用在生产系统中的话，可能还需要额外进行一些优化操作。

&nbsp;

（完）
