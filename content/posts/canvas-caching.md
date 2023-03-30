---
weight: 2
title: "浅析 Fabric.js 基于离屏 canvas 的缓存策略"
date: 2023-03-30T16:00:24+08:00
draft: false
categories: ["前端"]
tags: ["离屏Canvas","Canvas","缓存"]
---

## 前言

近期在实现关于 canvas 的需求的时候，借助了经典的 canvas 库 Fabric.js，Fabric.js 主要帮助用户简化对于二维 canvas 的操作，提供了很多清晰易用的 API。

在阅读文档和学习源码的过程中发现了 Fabric 中一个有意思的优化手段，那就是它的缓存策略。

Fabric 的缓存策略在这篇 [Fabric 官方文档](http://fabricjs.com/fabric-object-caching) 中有详细描述，在当前的版本中，缓存的粒度是每个画布上的对象，而不是整个 canvas。

当开启了缓存策略之后，在画布上添加的每一个对象都会先在一个离屏 canvas 上进行绘制，再通过 drawImage 操作复制到主画布上，以减少在主画布上的绘制操作，达到优化的目的。

&nbsp;

## 什么是离屏 canvas

实际上现在离屏 canvas 一共有两种含义。

### offscreenCanvas

一种是真正意义上的离屏 canvas，通过浏览器提供的类`new OffscreenCanvas()`实现，它有着跟 canvas 非常接近的 api，不同的是 offscreenCanvas 不会挂载在 DOM 树上，由于跟 DOM 树解耦，所以 offscreenCanvas 的计算任务是通过 `WebWorker` 处理的。

这样的好处非常明显，WebWorker 的计算不会占用主线程，在解放主线程之后，尽管主线程在执行复杂的计算任务甚至被阻塞，都不会对 offscreenCanvas 的渲染造成影响。

正因为如此，Fabricjs 团队也对通过 offscreenCanvas 提升 Fabric 的性能进行讨论和 POC，但目前还没有对应的实现。

对于 offscreenCanvas 的性能和场景测试，可以参考[这个网站](https://devnook.github.io/OffscreenCanvasDemo/index.html)。

&nbsp;

### cacheCanvas DOM

注意 cacheCanvas DOM 不是一个非常正式或者官方的名字，只是我自己对这个概念的理解随便起的。

cacheCanvas DOM 就是 Fabric 目前正在使用的缓存策略，尽管仍然需要一个跟 DOM 绑定的 canvas 对象，但这个 canvas 对象是离屏绘制的。

跟实时绘制不一样的是，离屏绘制不需要占用浏览器的渲染流程，当需要渲染到主画布上的时候，通过 drawImage API 一次性输出就可以了。

&nbsp;

## Fabric.js 中的实现

上文提到，Fabricjs 的缓存粒度是每个对象，所以在[源码](https://github.com/fabricjs/fabric.js/blob/master/src/shapes/Object/Object.ts)中，可以从 `FabricObject` 这个类里面找到相关的代码。

`FabricObject` 被其他常用的对象例如 Rect、Circle 等继承，是它们的父类，在对象的 `render()` 方法中，可以找到渲染缓存的逻辑：

```typescript
/**
   * Renders an object on a specified context （渲染对象到指定的 context）
   * @param {CanvasRenderingContext2D} ctx Context to render on
   */
  render(ctx: CanvasRenderingContext2D) {
    /** ====省略部分代码==== **/
      
    // 判断是否需要缓存  
    if (this.shouldCache()) {
      this.renderCache();        // 操作缓存
      (this as TCachedFabricObject).drawCacheOnCanvas(ctx);        // 渲染到主画布
    } else {
      this._removeCacheCanvas();
      this.dirty = false;
      this.drawObject(ctx);     // 直接操作主画布
    }
    ctx.restore();
  }
```

在 renderCache 方法中，创建了一个缓存 canvas DOM：

```typescript
  renderCache(options?: any) {
    options = options || {};
    if (!this._cacheCanvas || !this._cacheContext) {
      this._createCacheCanvas();  // 这里主要是通过 document.createElement('canvas') 创建了一个 canvas dom
    }
    if (this.isCacheDirty() && this._cacheContext) {
        // drawObject 方法是通用的，但注意这里的 ctx 是一个 cacheCanvas
      this.drawObject(this._cacheContext, options.forClipping);
      this.dirty = false;
    }
  }
```

drawObject 是一个通用的方法，对传入的第一个参数作为进行绘制：

```typescript
 /**
   * Execute the drawing operation for an object on a specified context(在指定的 context 执行渲染操作)
   * @param {CanvasRenderingContext2D} ctx Context to render on
   * @param {boolean} forClipping apply clipping styles
   */
  drawObject(ctx: CanvasRenderingContext2D, forClipping?: boolean) {
    /** ====省略部分代码==== **/
    
    // 进行绘制  
    this._render(ctx);
    this._drawClipPath(ctx, this.clipPath);
    this.fill = originalFill;
    this.stroke = originalStroke;
  }
```

_render 方法在 `FabricObject` 父类中之作占位，具体的实现逻辑由子类覆写：

```typescript
  _render(ctx: CanvasRenderingContext2D) {
    // placeholder to be overridden
  }
```

drawCacheOnCanvas 方法将 cache 中的图形一次性复制到主画布：

```typescript
  drawCacheOnCanvas(this: TCachedFabricObject, ctx: CanvasRenderingContext2D) {
    // 如果限制了 cache size，cache 中的对象可能会被缩放，这里我猜就是在处理这件事的
    ctx.scale(1 / this.zoomX, 1 / this.zoomY); 
    // 通过 drawImage 一次性复制到主画布  
    ctx.drawImage(
      this._cacheCanvas,
      -this.cacheTranslationX,
      -this.cacheTranslationY
    );
  }
```

&nbsp;

## 实际性能测试

Fabric 所使用的缓存策略对于性能提升，取决于画布中`每一个对象绘制的复杂程度`，当然还有就是对象的数量。

如果我们需要绘制的只是一些简单的圆形、矩形，那么实际上缓存策略所带来的性能提升是非常有限的，甚至是反过来产生了额外的开销。

但是，如果这些圆形、矩形绘制的时候有一些额外的细节，或者是更复杂的对象例如 svg，那这个时候就比较可以体现出性能差异了。

### 简单图形测试

点击[查看例子](https://codesandbox.io/s/sharp-chaplygin-11vm85?file=/index.html)，在这个例子中，我们生成了 2000 个简单的小球，可以通过按钮切换是否开启缓存，来观察性能的变化。

是否开启缓存的关键代码是这个方法：

```javascript
paint() {
          if (!this.useCache) {
              // 如果没有开启缓存，直接绘制到画布上
            ctx.save();
            ctx.lineWidth = BORDER_WIDTH;
            ctx.beginPath();
            ctx.strokeStyle = this.color;
            ctx.arc(this.x, this.y, this.r, 0, 2 * Math.PI);
            ctx.stroke();
            ctx.restore();
          } else {
              // 否则将复制 cacheCanvas 中的内容
            ctx.drawImage(
              this.cacheCanvas,
              this.x - this.r,
              this.y - this.r,
              this.cacheCanvas.width,
              this.cacheCanvas.height
            );
          }
        }
```

通过切换开启按钮可以发现，在这个情况下，是否开启缓存影响并不是非常大。

{{< admonition tip "小提示" >}}以下动画效果截屏基于 gif，在录制时就可能会有些许掉帧，实际效果应该查看上文的例子。{{< /admonition >}}

### 开启缓存效果

![开启缓存](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/canvas-cache/easy-cache.gif)

### 关闭缓存效果

![关闭缓存](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/canvas-cache/easy-nocache.gif)

&nbsp;

## 复杂图形测试

点击[查看例子](https://codesandbox.io/s/cool-breeze-dwmb8d?file=/src/index.js)，在这个例子中，绘制一个圆形的复杂程度增加了，在这个情况下，我们只生成了 1000 个小球，在不开启缓存的情况下，就已经可以感受到明显掉帧了。

{{< admonition tip "小提示" >}}以下动画效果截屏基于 gif，在录制时就可能会有些许掉帧，实际效果应该查看上文的例子。{{< /admonition >}}

### 开启缓存效果

![开启缓存](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/canvas-cache/complex-cache.gif)

### 关闭缓存效果

![关闭缓存](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/canvas-cache/complex-nocache.gif)
