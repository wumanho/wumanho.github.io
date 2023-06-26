---
weight: 2
title: "【Fabric.js】常用形状交互式即时绘制指南(上)"
date: 2023-06-20T13:39:22+08:00
draft: false
categories: ["前端"]
tags: ["绘制图形",Canvas","fabric.js"]
---

## 前言

本篇博客主要介绍使用 fabric.js 在画布中绘制常见的图形，主要包括有：

* 绘制直线
* 绘制折线
* 绘制矩形
* 绘制圆形
* 绘制点
* 绘制多边形
* 放置可编辑文本对象

出于通用性考虑，本次封装将会通过面向对象的方式，所以限于篇幅，本次主题将会分成上下两篇。

**上篇**主要实现类的封装，初始化及通用函数的实现。

**下篇**则是具体的图形绘制实现。



### 效果展示

展示效果采用公司已经实现的业务系统（顺便装个B），不过本篇博客**主要聚焦在图形的绘制上**，且会**在 demo 项目中对代码进行重新实现**并且省略非关键代码。

工具栏上的其余功能及红外温度计算等代码由于涉及商业，也不会在本篇博客中涉及到。

![演示](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/fabricjs/drawing.gif)

&nbsp;

## 基础对象

出于通用性考虑，本次封装将会通过面向对象的方式，所有绘制操作都在 `IFabric` 类的内部实现。

```javascript
import { fabric } from 'fabric'
import { Emitter } from './Emitter.js'

// 设置选中样式
fabric.Object.prototype.set({
  transparentCorners: false,
  borderColor: '#51B9F9',
  cornerColor: '#FFF',
  borderScaleFactor: 2.5,
  cornerStyle: 'circle',
  cornerStrokeColor: '#0E98FC',
  borderOpacityWhenMoving: 0.6
})

class IFabric extends Emitter{
  constructor(container) {
    super()
    // 创建实例
    this._canvas = new fabric.Canvas(container, {
      fireRightClick: true,  // 监听右键事件
      stopContextMenu: true  // 取消右键默认事件
    })
    // 设置选中时不会置顶图形
    this._canvas.set('preserveObjectStacking', true)
    // 禁止多选
    this._canvas.set('selection', false)  
  }
    
  /**
   * 绘制图形入口
   * @param {* String} shape 形状
   */
  handleDraw(shape) {
    // 指针样式
    this._canvas.defaultCursor = "crosshair";
    switch (shape) {
      case "rect":
        this._drawRect();
        break;
      case "circle":
        this._drawCircle();
        break;
      case "text":
        this._canvas.defaultCursor = "text";
        this._genText();
        break;
      case "polyLine":
        this._drawPolyline();
        break;      
      case "point":
        this._drawPoint();
        break;
      case "line":
        this._drawLine();
        break;
      case "polygon":
        this._drawPolygon();
    }
  }

  _drawPolygon() {}

  _drawLine() {}

  _drawRect() {}

  _genText() {}

  _drawPoint() {}

  _drawCircle() {}
    
  _drawPolyline() {}  
}

export default IFabric
```

`Emitter` 用于实现对象的事件派发和监听，基于开源库 [tiny-emitter](https://github.com/scottcorgan/tiny-emitter/blob/master/index.js)。

右键事件可以用来做取消绘制之类的交互，或者用于实现自己的右键菜单，所以建议把默认事件取消了。



## 图形添加方法

为了方便使用以及在业务代码中加入自己的逻辑，我们可以将添加图形的动作抽取成一系列的方法。

举个例子，当我们添加一个矩形到画布中时，可以将当前添加的状态设置为矩形`rect`，用作业务逻辑判断，添加其他图形也是一样的：

```javascript
class IFabric extends Emitter{
  constructor(container, url) {
      /** ... **/
    // 记录当前添加的图形类型  
    this._activeShape = ''
  }
    
  get activeShape() {
    return this._activeShape
  }
    
  // 添加一个矩形到画布中
  addRectToCanvas(options) {
    this._activeShape = 'rect'
    new fabric.Rect(options)
  }

  // 添加一个圆形到画布中
  addCircleToCanvas(options) {
    this._activeShape = 'circle'
    new fabric.Circle(options)
  }

  // 添加点到画布中
  addCrossToCanvas(path, options) {
    this._activeShape = "cross";
    new fabric.Path(path, options);
  }

  // 添加多边形到画布中
  addPolygonToCanvas(points, options) {
    this._activeShape = 'polygon'
    new fabric.Polygon(points, options)
  }

  // 添加直线到画布中
  addLineToCanvas(linePath, options) {
    this._activeShape = 'line'
    new fabric.Line(linePath, options)
  }
  
  // 添加折线到画布中
  addPolyLineToCanvas(points, options) {
    this._activeShape = "polyline";
    new fabric.Polyline(points, options);
  }  

  // 添加文本到画布中
  addTextToCanvas(text, options) {
    this._activeShape = 'text'
    new fabric.IText(text, options)
  }
}
```

### 统一添加方法

在创建对象之后，需要将图形添加到画布中，我们可以实现一个统一的添加方法，在添加到画布中的时候，加上业务逻辑。

例如：在添加之前，为每个对象创建一个 uuid，将 uuid 返回给调用者。添加标识对象类型的属性`logType`，并且维护一个对象数据，将所有对象保存起来供业务使用：

```javascript
import { v4 as uuid } from "uuid";

class IFabric extends Emitter {
  constructor(container) {
    /** ... **/
    this._elements = [];
  }

  get elements() {
    return this._elements;
  }
  // 添加一个矩形到画布中
  addRectToCanvas(options) {
    this._activeShape = "rect";
    const rect = new fabric.Rect(options);
    return this._addElementToCanvas(rect);
  }

  // 添加一个圆形到画布中
  addCircleToCanvas(options) {
    this._activeShape = "circle";
    const circle = new fabric.Circle(options);
    return this._addElementToCanvas(circle);
  }

  // 添加点到画布中
  addCrossToCanvas(path, options) {
    this._activeShape = "cross";
    const cross = new fabric.Path(path, options);
    return this._addElementToCanvas(cross);
  }

  // 添加多边形到画布中
  addPolygonToCanvas(points, options) {
    this._activeShape = "polygon";
    const polygon = new fabric.Polygon(points, options);
    return this._addElementToCanvas(polygon);
  }

  // 添加直线到画布中
  addLineToCanvas(linePath, options) {
    this._activeShape = "line";
    const line = new fabric.Line(linePath, options);
    return this._addElementToCanvas(line);
  }
    
  // 添加折线到画布中
  addPolyLineToCanvas(points, options) {
    this._activeShape = "polyline";
    const polyLine = new fabric.Polyline(points, options);
    return this._addElementToCanvas(polyLine);
  }  

  // 添加文本到画布中
  addTextToCanvas(text, options) {
    this._activeShape = "text";
    const content = new fabric.IText(text, options);
    return this._addElementToCanvas(content);
  }

  // 添加已创建的对象到画布中  
  _addElementToCanvas(el) {
    // 创建一个 uuid
    const uid = uuid();
    el.uid = uid;
    // 为每个对象添加一个标识类型的字段
    if (!el.logType) {
      el.logType = this._activeShape;
    }
    this._canvas.add(el);
    this._elements.push(el);
    return uid;  
  }
}
```

### 对象关联事件

最后，在添加对象时，可以顺便将对象关联上事件：

```javascript {hl_lines=["17"]}
import { v4 as uuid } from "uuid";

class IFabric extends Emitter {  

  /** ... **/  

  // 添加已创建的对象到画布中  
  _addElementToCanvas(el) {
    // 创建一个 uuid
    const uid = uuid();
    el.uid = uid;
    if (!el.logType) {
      el.logType = this._activeShape;
    }
    this._canvas.add(el);
    this._elements.push(el);
    this.hookEvents(el);
    return uid;  
  }

  // 为添加到画布的对象关联选择、移动、缩放事件
  hookEvents(target) {
    target.on({
      moving: this.emitEvent.bind(this),
      scaling: this.emitEvent.bind(this),
      selected: () => {
        this.emit("object-selected", target.uid);
      },
    });
  }

  // 选择对象、移动和缩放都会进入此方法
  emitEvent() {
    const activeEl = this._canvas.getActiveObject();
    this.emit("shape-change", activeEl);
  }    
}
```

&nbsp;

## 公共操作方法

有了以上的准备，我们就可以扩展出很多公共的操作方法，方便调用者进行操作了。

举个例子，当用户要将所有文字元素重新更换一个颜色，那么可以提供一个`resetFontColor`方法：

```javascript
class IFabric extends Emitter {
  // 设置文字颜色
  setFontSize(color) {
    // 遍历所有元素
    this._elements.forEach((el) => {
      // 找到 type 为 text 的元素
      if (el.logType === 'text') {
        el.set({
          fill: color
        })
      }
    })
  }
}
```

根据 id 获取元素：

```javascript
class IFabric extends Emitter {
  
  getCanvasElementById(uid) {
    return this._elements.find((el) => el.uid === uid);
  }
}
```

移除指定元素：

```javascript
class IFabric extends Emitter {
  
  removeElementById(uid) {
    const el = this.getCanvasElementById(uid);
    this._canvas.remove(el);
    this._elements = this._elements.filter((el) => el.uid !== uid);
  }
}
```

&nbsp;

## 总结

至此，我们便拥有了一个易于扩展的类，为后续的实现打好了基础。

在下一篇博客中，会对本次主题最关键的图形绘制功能进行一一实现。

&nbsp;

（上篇完）
