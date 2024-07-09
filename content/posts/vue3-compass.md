---
weight: 2
title: "【前端小组件】通过鼠标控制的旋转按钮"
summary: "一个简单好用的入门级小组件"
date: 2022-09-21T16:48:23+08:00
draft: false
categories: ["前端"]
tags: ["Vue3","css v-bind","入门"]
---

## 前言

迫于很久没更新博客，水一篇小文章，最终效果：

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/compass.gif)

是一个双向绑定 Input 框的旋转按钮，旋转按钮的图片可以根据业务场景替换，例如换成音量键之类的。

&nbsp;

## 基本思路

* 双向绑定：`v-model` 实现
* 鼠标点击时记录抓点位置，允许 move 事件回调
* move 事件回调中旋转图片
* 鼠标抬起时停止 move 事件回调

&nbsp;

## 代码实现



### 模板代码

模板代码需要注意的地方是这个按钮分为内外两圈，里面那圈是不需要跟着转的，所以`ref`要正确给到外面那圈的元素。

```vue
<template>
  <div
    class="compass-wrap"
    @mousedown="recordStartPoint"
    @mousemove="handleMouseMove"
    @mouseup="stopCompute"
    @mouseleave="stopCompute"
  >
    <div class="main-borad" ref="boardRef">
      <span class="n-text">N</span>
      <div class="borad"></div>
    </div>
    <div class="inner"></div>
  </div>
</template>
```



### 逻辑代码

组件初始化时，获取元素的圆心坐标，用于后续的计算

```vue
<script setup>
import { ref, onMounted, nextTick } from 'vue'
const props = defineProps({
  value: {
    type: Number,
    default: 0
  }
})
const emit = defineEmits(['update:value'])

// dom ref  
const boardRef = ref(null)
// 圆心坐标
let centerPoint = { x: 0, y: 0 }
// 获取圆心坐标
function getCenter() {
  const { x, y } = boardRef.value.getBoundingClientRect()
  const width = boardRef.value.offsetWidth
  const height = boardRef.value.offsetHeight
  centerPoint.x = x + width / 2
  centerPoint.y = y + height / 2
}
// 在 onMounted 调用
onMounted(() => {
  nextTick(() => {
    getCenter()
  })
})  
</script>
```

`recordStartPoint` 鼠标点击回调：

```vue
<script setup>
let startAngle = 0 // 鼠标落点角度
let isDragging = false // 是否开启 move 事件
function recordStartPoint(e) {
  startAngle = computeAngle(e)
  isDragging = true
}
  
/**
 * 复用函数：计算起始角度 & 旋转角度
 * @param e 鼠标事件
 */
function computeAngle(e) {
  // 鼠标坐标与圆心的弧度转角度
  const radian = Math.atan((e.clientY - centerPoint.y) / (e.clientX - centerPoint.x))
  let angle = (radian * 180) / Math.PI
  if (e.clientX < centerPoint.x) {
    angle += 180
  }
  return angle
}  
</script>
```

`handleMouseMove` 鼠标移动回调计算旋转角度：

```vue
<script setup>
let renderAngle = 0 // 实际用于渲染的旋转角度
const displayAngle = ref(0) // css v-bind 值
function handleMouseMove(e) {
  if (!isDragging) return
  const movingAngle = Math.round(startOffsetAngle + computeAngle(e) - startAngle)
  renderAngle = movingAngle < 0 ? (movingAngle + 360) % 360 : movingAngle % 360
  emit('update:value', renderAngle + '')
  displayAngle.value = renderAngle + 'deg'
}
</script>
```

`stopCompute` 鼠标抬起时回调，并停止计算指针移动

```vue
<script setup>
let startOffsetAngle = 0 // 记录指南针表盘当前的偏移角度
function stopCompute() {
  isDragging = false
  startOffsetAngle = renderAngle
}
</script>
```

暴露手动设置罗盘方向的方法给组件外部使用：

```vue
<script setup>
// 手动设置罗盘角度
function setCompassRotate(val) {
  startOffsetAngle = val
  displayAngle.value = val + 'deg'
}

defineExpose({
  setCompassRotate
})
</script>
```

增加初始化时设置罗盘方向的逻辑：

```vue
<script setup>
import { ref, onMounted, nextTick } from 'vue'
onMounted(() => {
  nextTick(() => {
    getCenter()
    // 新增
    if (props.value !== 0) {
      setCompassRotate(props.value)
    }
  })
})
</script>
```

&nbsp;

## CSS v-bind 绑定

将刚刚记录的旋转角度值直接通过 `v-bind` 绑定到 css 中即可

```vue
<style lang="less" scoped>
 .main-borad {
    transform: rotate(v-bind(displayAngle));
  }  
</style>
```

&nbsp;

（完）
