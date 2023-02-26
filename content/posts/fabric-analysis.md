---
weight: 2
title: "基于 Fabric.js 实现红外照片热斑分析"
date: 2023-02-26T22:10:29+08:00
draft: false
categories: ["前端"]
tags: ["Vue3","计算机视觉","Canvas","fabricjs"]
---

## 前言

近期做了一个比较有意思的需求，虽然只是最初的版本，离完成还差很远，但还是想先记录一下。

目前的实现效果如下：

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/fabricjs/f-analysis.gif)

实现这部分看似简单的需求一共需要三端参与开发，分别是：

* 计算机视觉开发：负责计算出航拍照片中每个像素点的温度数据
* 后端开发：提供分析接口，调用视觉提供的库获取到分析数据返回给前端
* 前端开发：负责前端交互，从像素点矩阵中分析出所选范围内的最高热斑温度、平均温度及温差等值，最终将编辑后的照片、分析数据及 form 表单信息上传到后端入库，用于导出报告

&nbsp;

## 难点

由于是第一次跟计算机视觉开发合作，一开始并不知道需要怎么配合，后来发现前端可以从接口拿到的数据只有三个值：

```json
{
    imageWidth: 640,  // 照片的实际宽度
    imageHeight: 512, // 照片的实际高度
    temperature: "96,72,232,232..."  // 实际上是一个 width * height 长度的超大字符串
}
```

基于这三个值分析出所选区域内的温度数据，首先碰到的难点就是照片在各个屏幕上的自适应问题。

例如这个例子中，这张照片的实际大小是 640 * 512，但页面上的画布是根据屏幕大小自适应的，所以必然会对照片进行一定程度上的缩放，这就导致了用户在画布上所框选的范围是不准确的，计算出来的结果也是错误的。

要解决这个问题，就需要获取到照片的`缩放比`，通过缩放比来计算出用户在画布上选择的位置，所对应到的真实大小照片中的像素点位置，类似一个映射的关系，这样才能计算出准确的温度值数据。

&nbsp;

## 封装 Fabric.js

这类型的需求在交互上如果用传统前端的手段实现起来是比较麻烦的，所以选择了基于 Canvas 实现，再通过 fabricjs 提供的简单易用的 api 对 Canvas 进行操作。

封装一个工具类，数据部分：

```javascript
class UFIFabric {
    constructor(container, url) {
    // 创建实例
    this._canvas = new fabric.Canvas(container)
    // 设置选中时不会置顶图形
    this._canvas.set('preserveObjectStacking', true)
    // 用于初始化背景图片的 url
    this._url = url
    this._elements = []
    this._scale = {
      scaleX: 0,
      scaleY: 0
    }
    this._imgSize = {
      width: 0,
      height: 0
    }
    this._mouseEvent = null
  }
  // 图片实际大小
  get imgSize() {
    return this._imgSize
  }

  // 获取缩放
  get scale() {
    return this._scale
  }

  // 获取画布上的元素
  get elements() {
    return this._elements
  }

  // 获取 canvas 实例
  get canvas() {
    return this._canvas
  }
}
```

在页面上初始化 Fabric 实例：

```vue
// PhotoAnalysis.vue
<template>
	 <canvas id="container"></canvas>
</template>

<script setup>
import { nextTick, onMounted, ref, toRefs } from 'vue'
import UFIFabric from '@/utils/UFIFabric.js'

const props = defineProps({
  analysisListData: Array
})
const { analysisListData } = toRefs(props)    

// 挂载后获取 canvas dom 初始化
onMounted(() => {
  if (analysisListData.value.length === 0) {
      ElMessage({ message: '没有照片数据', type: 'warning' })
      return
  }
  initCanvas(analysisListData.value[0])
})

// canvas 实例    
const canvasBox = ref(null)
let fb = null
const initCanvas = (imgInfo) => {
  fb = new UFIFabric('container', imgInfo.imageUrl)
  // 获取 canvas 容器的宽高，用于设置 canvas 背景图片宽高  
  fb.init(canvasBox.value.clientWidth, canvasBox.value.clientHeight)
  // 注册鼠标事件回调到实例上
  fb.registerMouseEvent(getCanvasElementInfo)  
  handleImgChange(imgInfo)
}

const getCanvasElementInfo = () => {}
const handleImgChange = () => {}
</script>
```

&nbsp;

## 初始化 Canvas 背景图片和注册事件回调

在 Fabric 类中提供一个初始化的方法，在初始化中完成几件事：

* 设置宽高
* 挂载事件回调
* 设置自适应的背景图片

```javascript
class UFIFabric {
    // ...
  /**
   * 注册事件方法
   */
  registerMouseEvent(func) {
    this._mouseEvent = func
  }
  
  /**
   * 初始化画布，适应窗口大小
   * @param width 元素宽度
   * @param height 元素高度
   */
  init(width, height) {
    this.setSize(width, height)
    this._canvas.setZoom(1)
    this.hookEvents()
    this.setContainImageFromUrl(this._url, width, height)
  }
    
   /**
   * 挂载事件方法，当 target 为空时，代表挂载到画布上
   * @param target 元素宽度
   */
  hookEvents(target = null) {
    if (target) {
      target.on({
        moving: this.emitEvent.bind(this),
        scaling: this.emitEvent.bind(this)
      })
    } else {
    // 拖拽元素放置事件, 暂时只实现了矩形
    this._canvas.on('drop', (e) => {
      this.addRectToCanvas({
        left: e.e.layerX,
        top: e.e.layerY,
        width: 200,
        height: 120,
        stroke: '#5FF21E',
        strokeWidth: 3,
        fill: 'transparent',
        objectCaching: false
      })
      // 设置放置的对象为激活对象，然后激活事件
      this._canvas.setActiveObject(this._elements[this._elements.length - 1])
      this.emitEvent.call(this)
    })
   }
 }

  emitEvent() {
    const activeEl = this._canvas.getActiveObject()
    this._mouseEvent && this._mouseEvent(this._scale.scaleX, this._scale.scaleY, activeEl)
  }  
}
```

在设置背景图片时就可以获取到我们需要的关键参数缩放比例了，由于 Canvas 默认对于被污染过的画布会有跨域保护，导致在导出图片时会报错，解决这个问题只要在第三个参数中传入 `{ crossOrigin: 'anonymous' }` 即可

```javascript
class UFIFabric {
    // ...
    
  /**
   * 设置画布中的图片背景
   * 自动居中
   * 图片背景 *会* 自适应容器大小，副作用是会改变图片的大小
   * @param url 图片 url
   * @param width 图片宽度
   * @param height 图片高度
   */
  setContainImageFromUrl(url, width, height) {
    fabric.Image.fromURL(
      url,
      (oImg) => {
        // 记录图片的实际大小
        this._imgSize.width = oImg.width
        this._imgSize.height = oImg.height
        // 记录画布的缩放大小
        this._scale.scaleX = width / oImg.width
        this._scale.scaleY = height / oImg.height
        oImg.set({
          // 通过scale来设置图片大小，这里设置和画布一样大
          scaleX: this._scale.scaleX,
          scaleY: this._scale.scaleY,
          selectable: false,  // 背景图片不可选择
          hasControls: false  // 背景图片禁用选择器
        })
        this._canvas.setBackgroundImage(oImg)
        this._canvas.viewportCenterObject(oImg)
        this._canvas.renderAll()
      },
      { crossOrigin: 'anonymous' }
    )
  }
}
```

还有就是一些添加元素到画布中的方法

```javascript
class UFIFabric {
    //.....
  
   /**
   * 添加文本到画布中
   */  
  addTextToCanvas(text, options) {
    const content = new fabric.Text(text, options)
    return this._addElementToCanvas(content)
  }

  /**
   * 添加一个矩形到画布中
   */
  addRectToCanvas(options) {
    options.lockRotation = true // 禁止旋转
    const rect = new fabric.Rect(options)
    return this._addElementToCanvas(rect)
  }

  /**
   * 添加一个圆形到画布中
   */
  addCircleToCanvas(options) {
    options.lockRotation = true // 禁止旋转
    const circle = new fabric.Circle(options)
    return this._addElementToCanvas(circle)
  }
    
  /**
   * 1.对象设置 uuid
   * 2.关联事件
   * 4.添加元素到画布中
   * 4.记录画布元素到 elements 数组
   * @param el 元素
   */
  _addElementToCanvas(el) {
    const uid = uuid()
    el.uid = uid
    this.reset()
    this.hookEvents(el)
    this._canvas.add(el)
    this._elements.push(el)
    return uid
  }
    
  /**
   * 重置
   */
  reset() {
    this._canvas.clear()
    this.setContainImageFromUrl(this._url, this._width, this._height)
  }  
}
```

&nbsp;

## 获取照片信息

在用户点击下方图片的时候，除了将图片设置为 Canvas 的背景图片，还需要在这个时候通过接口获取到所选图片的数据，也就是之前提到的图片宽高和像素点温度信息。

由于像素点温度信息是一个简单粗暴的超大字符串，获取到之后为了方便计算，我们可以先根据图片宽高将他转为一个二维数组。

```vue
<script setup>
 //..省略前面代码..//
    
// 图片大小记录
const imageSize = {
  width: 0,
  height: 0
}
// 温度值数据
let temperatureData = []
let timer = null
const handleImgChange = async (imgInfo) => {
  // ** 防止快速切换图片做的简单控制 **  //
  if (timer) return
  timer = setTimeout(() => {
    timer = null
  }, 300)
  // ** end **  //
  markList.value = [] // 记录温度数据的列表
  activeImg.value = imgInfo.imageUrl
  fb.resetCanvas(imgInfo.imageUrl) // 实际上就是调用了一下 reset 方法并记录新的 url
  // 获取分析接口返回的数据
  const { data } = await analysisTemperature(imgInfo.imagePath)
  // 获取图片原始宽高
  imageSize.width = data.data.imageWidth || fb.imgSize.width
  imageSize.height = data.data.imageHeight || fb.imgSize.height
  if (data.data.imageTemperature) {
    temperatureData = data.data.imageTemperature.split(',')
    // 切割成行列组成的二维数组
    temperatureData = groupingArray(temperatureData, imageSize.width)
  } else {
    temperatureData = []
  }
}
</script>
```

附上二维数组切割的工具函数

```javascript
/**
 * @param source 原数组
 * @param target 子数组的大小
 * @returns [[],[]]
 */
const groupingArray = (source, target) => {
  if (!source) return []
  let index = 0
  let result = []
  while (index < source.length) {
    result.push(source.slice(index, (index += target)))
  }
  return result
}

export default { groupingArray }
```

&nbsp;

## 在回调中计算温度信息

万事俱备，现在只要实现我们之前注册的回调就可以了，这个回调在元素被拖拽、缩放以及放置在画布上的时候都会被触发。

```vue
<script setup>
  //..省略前面代码..//
    
/**
 * 计算热斑温度数值
 * @param scaleX *画布* 的 X 轴缩放比
 * @param scaleY *画布* 的 Y 轴缩放比
 * @param active 当前激活的对象
 */
const computedTemperature = (scaleX, scaleY, active) => {
  // 通过缩放比，计算元素所在画布的实际位置和大小
  // sx 和 sy 是对象的缩放比，需要跟参数中的画布缩放比区分
  const { realLeft, realWidth, realTop, realHeight } = getRealConditions(scaleX, scaleY, active)
  // 像素点行数
  const rows = temperatureData.slice(realTop, realTop + realHeight)
  // 扁平化像素矩阵
  const flatMatrix = rows
    .map((r) => {
      // 像素点列数
      return r.slice(realLeft, realLeft + realWidth)
    })
    .flat()
    .map((f) => Number(f))
  // 热斑温度
  const maxHeat = Math.max(...flatMatrix)
  const minHeat = Math.min(...flatMatrix)
  // 区域温度
  const average = (flatMatrix.reduce((cur, prev) => cur + prev, 0) / flatMatrix.length).toFixed(1)
  // 温差
  const diff = maxHeat - minHeat
  markList.value = [
    {
      heat: maxHeat.toFixed(0),
      area: average,
      diff: diff
    }
  ]
}

/**
 * 计算原比例图片的参数
 * @param scaleX *画布* 的 X 轴缩放比
 * @param scaleY *画布* 的 Y 轴缩放比
 * @param element 画布中的元素对象
 * @returns {{realHeight: number, realLeft: number, realWidth: number, realTop: number}}
 */
const getRealConditions = (scaleX, scaleY, element) => {
  const { top, left, width, height, scaleX: sx, scaleY: sy } = element
  let realLeft = Math.floor(left / scaleX)
  const realWidth = Math.floor((sx ? width * sx : width) / scaleX)
  let realTop = Math.floor(top / scaleY)
  const realHeight = Math.floor((sy ? height * sy : height) / scaleY)
  // 边界处理
  if (realTop < 0) realTop = 0
  if (realLeft < 0) realLeft = 0
  if (realTop >= imageSize.height) realTop = imageSize.height - 1
  if (realLeft >= imageSize.width) realLeft = imageSize.width - 1
  return {
    realLeft,
    realWidth,
    realTop,
    realHeight
  }
}
</script>
```

至此，基本的温度分析逻辑就完成了。

&nbsp;

## 以原始大小导出绘制后的图片

由于上传还需要考虑到接口调用，这里只演示一下导出的逻辑。

在绘制完成后，如果将图片重新缩放到原始比例直接导出，图片会变得非常模糊，特别当图片缩放比例非常大的时候。

所以只能用一个曲线救国的办法，将原图片设置到一个原始大小的隐藏临时 Canvas 容器上，根据绘制的元素信息重新在这个容器上绘制元素，将这个临时的容器作为导出的目标。

```vue
<template>
	 <canvas id="container"></canvas>
	 <canvas style="display:none" id="temp-canvas"></canvas>
</template>

<script setup>
  //..省略前面代码..//    
      
const saveCanvasImage = () => {
  fb.renderAll()
  // 创建临时画布，用于生成原比例图片，activeImg 是原图片的 url
  const tempFb = new UFIFabric('temp-canvas', activeImg.value)
  tempFb.init(imageSize.width, imageSize.height)
  // 遍历所有元素，添加到原比例图片上
  fb.elements.forEach((el) => {
    // 复用之前的计算方法，绘制图形到原始照片上  
    const { realLeft, realWidth, realTop, realHeight } = getRealConditions(
      fb.scale.scaleX,
      fb.scale.scaleY,
      el
    )
    tempFb.addRectToCanvas({
      left: realLeft,
      top: realTop,
      width: realWidth,
      height: realHeight,
      stroke: '#5FF21E',
      strokeWidth: 6,
      fill: 'transparent'
    })
  })
  // 下载  
  setTimeout(() => {
    const tempEl = document.getElementById('temp-canvas')
    const el = document.createElement('a')
    el.href = tempEl.toDataURL('image/png')
    el.download = '文件名称.png'
    const event = new MouseEvent('click')
    el.dispatchEvent(event)
    tempFb.clear() // 销毁临时对象  
  })
}    
</script>
```

对比一下，编辑器绘制的位置：

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/fabricjs/page-position.png)



导出后的原图：

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/fabricjs/source.png)

&nbsp;

（完）
