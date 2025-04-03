---
weight: 2
title: "【Cesium】实现上下垂直的卷帘分割效果"
summary: "本篇博客将会以修改源代码的方式，实现 Cesium 实体和模型上下垂直的卷帘分割效果"
date: 2025-03-08T12:10:12+08:00
draft: false
categories: ["技术"]
tags: ["gis","Cesium","Cesium 源码","GLSL","垂直分割"]
---

## 前言

本篇博客将会尝试通过修改源码的方式，扩展 Cesium 的 `SpilitDirection` 可选项，最终实现对于 Primitive 和 ImageryLayer 等元素的上下垂直分割以及左右水平分割显示模式切换的功能。

&nbsp;

而目前，Cesium 仅支持左右水平分割的模式，可以参考官方用例 [3D Tiles Compare](https://sandcastle.cesium.com/?src=3D%20Tiles%20Compare.html)，本篇博客代码基于 Cesium `1.123.1` 版本进行修改。

&nbsp;

最终效果，两种模式无缝切换，实现对于两个三维模型的分割：

![模式切换](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/splitter.gif)

&nbsp;

## 关键属性分析

先从上述的官方用例中，看看目前已经支持的水平分割模式，需要应用到哪些属性，作为切入点，来分析和定位需要重点关注或者修改的源代码部分。

```javascript
// 官方用例
try {
  // left 三维模型
  const left = await Cesium.Cesium3DTileset.fromIonAssetId(69380);
  viewer.scene.primitives.add(left);
  // 设置 left 的 splitDirection 为 Cesium.SplitDirection.LEFT
  left.splitDirection = Cesium.SplitDirection.LEFT;

  viewer.zoomTo(left);
  // right 三维模型
  const right = await Cesium.createOsmBuildingsAsync();
  viewer.scene.primitives.add(right);
  // 设置 right 的 splitDirection 为 Cesium.SplitDirection.RIGHT
  right.splitDirection = Cesium.SplitDirection.RIGHT;
} catch (error) {
  console.log(`Error loading tileset: ${error}`);
}
```

在这段实例代码中可以看到，我们需要关注的关键属性一共有两个，一个是`splitDirection`，另一个是`Cesium.SplitDirection`。

&nbsp;

`splitDirection` 这个属性存在于 cesium 的几乎所有实体和图元类中，包括 Billboard、Point、ImageryLayer、PointCloud 等等。

&nbsp;

为实体和图元设置该属性，可以决定他们在渲染时处于哪一侧。

&nbsp;

`Cesium.SplitDirection` 是一个枚举，包含了三个属性：

```javascript
// 源码位置：packages/engine/Source/Scene/SplitDirection.js
const SplitDirection = {
  /**
   * 将元素显示在左边
   * @type {number}
   * @constant
   */
  LEFT: -1.0,

  /**
   * 总是显示元素
   * @type {number}
   * @constant
   */
  NONE: 0.0,

  /**
   * 将元素显示在右边
   * @type {number}
   * @constant
   */
  RIGHT: 1.0,
};
export default Object.freeze(SplitDirection);
```

如果仅仅设置了图元的 `splitDirection` 属性，是看不到任何效果的，因为接下来还有一个关键属性 `splitPosition`。

&nbsp;

他的默认值是 0.0，在官方用例中的这段代码，就是用来监听分割器元素的移动，将分割器 left 坐标归一化为 0.0 - 1.0 范围内的值，然后修改 scene 中的 `splitPosition` 属性的：

```javascript
// 官方用例
function move(movement) {
  if (!moveActive) {
    return;
  }
  const relativeOffset = movement.endPosition.x;
  const splitPosition =
    (slider.offsetLeft + relativeOffset) / slider.parentElement.offsetWidth;
  slider.style.left = `${100.0 * splitPosition}%`;
  viewer.scene.splitPosition = splitPosition;
}
```

这个 `splitPosition` 最终会作用到 glsl 代码的渲染逻辑中：

```glsl
// 源码位置：packages/engine/Source/Shaders/GlobFS.glsl

#ifdef APPLY_SPLIT
    float splitPosition = czm_splitPosition;
    // Split to the left
    if (split < 0.0 && gl_FragCoord.x > splitPosition) {
       alpha = 0.0;
    }
    // Split to the right
    else if (split > 0.0 && gl_FragCoord.x < splitPosition) {
       alpha = 0.0;
    }
#endif
```

总结一下，关键属性一共有三个：

* splitDirection 属性

* Cesium.SplitDirection 枚举

* splitPosition

&nbsp;

## 修改着色器逻辑

`GlobFS.glsl` 中的 `APPLY_SPLIT` 宏是实现对于 `ImageryLayer` 分割功能的主要逻辑，我们上面提到的关键属性，将会在这个宏里面得到体现。

  &nbsp;

如果对  cesium 中的 glsl，甚至是 glsl 本身都还不太了解的话，可能会看不懂，我们先详细分解一下这个宏：

&nbsp;

### czm_splitPosition：

在 cesium 中，以 `czm_` 作为前缀的 uniform 代表是 cesium 自动管理和提供的，uniform 是 glsl 中一种类似环境变量的存在，是我们向 glsl 传参的重要方式之一。

&nbsp;

`czm_splitPosition` 其实就是我们在 `viewer.scene.splitPosition = splitPosition` 这里赋予的值，cesium 内部对他进行了处理，并作为内置 uniform 提供。

&nbsp;

### gl_FragCoord：

这个 gl_FragCoord 是 glsl 本身内置的变量，表示正在处理的片段的像素坐标。

&nbsp;

### split 变量

split 变量可以简单理解为我们为实体设置的 `splitDirection` 属性，他的值要么是 0.0 要么是 1.0。

&nbsp;

可以看到，这段着色器逻辑里面包含了我们上述提到的所有关键变量，所以我们可以直接从这里着手开始修改。

&nbsp;

我们实现垂直分割，核心目标是让元素处于分割线上下两端，我们可以复用原来的逻辑，只是加一个分支：

```glsl
#ifdef APPLY_SPLIT
    float splitPosition = czm_splitPosition;
    float splitMode = czm_splitMode;
    if (splitMode > 0.0) {
        // Split to the left
        if (split < 0.0 && gl_FragCoord.x > splitPosition) {
            alpha = 0.0;
        }
        // Split to the right
        else if (split > 0.0 && gl_FragCoord.x < splitPosition) {
            alpha = 0.0;
        }
    }
    else if (splitMode < 0.0) {
        // Split to the top
        if (split < 0.0 && gl_FragCoord.y < splitPosition) {
            alpha = 0.0;
        }
        // Split to the bottom
        else if (split > 0.0 && gl_FragCoord.y > splitPosition) {
            alpha = 0.0;
        }
    }
#endif
```

我们增加了一个 uniform`czm_splitMode`，然后基于这个 uniform 做判断，决定元素是上下分割，还是左右分割。

&nbsp;

参考 `APPLY_SPLIT` 对于其他会用到 `splitPosition` 的实体，我们也要对应的去修改他们的渲染逻辑：

```glsl
// 源码位置：packages/engine/Source/Shaders/Model/ModelSplitterStageFS.glsl

void modelSplitterStage()
{
    // Don't split when rendering the shadow map, because it is rendered from
    // the perspective of a totally different camera.
#ifndef SHADOW_MAP
    if (czm_splitMode > 0.0) {
        if (model_splitDirection < 0.0 && gl_FragCoord.x > czm_splitPosition) discard;
        if (model_splitDirection > 0.0 && gl_FragCoord.x < czm_splitPosition) discard;
    } 
    else if (czm_splitMode < 0.0) {
        if (model_splitDirection < 0.0 && gl_FragCoord.y > czm_splitPosition) discard;
        if (model_splitDirection > 0.0 && gl_FragCoord.y < czm_splitPosition) discard;
    }
#endif
}
```

Billboard：

```glsl
// 源码位置：packages/engine/Source/Shaders/BillboardCollectionFS.glsl

void main()
{
    // 添加 czm_splitMode 判断 
    if (czm_splitMode > 0.0) {
        if (v_splitDirection < 0.0 && gl_FragCoord.x > czm_splitPosition) discard;
        if (v_splitDirection > 0.0 && gl_FragCoord.x < czm_splitPosition) discard;
    } 
    else if (czm_splitMode < 0.0) {
        if (v_splitDirection < 0.0 && gl_FragCoord.y > czm_splitPosition) discard;
        if (v_splitDirection > 0.0 && gl_FragCoord.y < czm_splitPosition) discard;
    }
        
    vec4 color = texture(u_atlas, v_textureCoordinates);
    
	/* 省略下面 ...*/
}
```

PointPrimitive：

```glsl
// 源码位置：packages/engine/Source/Shaders/PointPrimitiveCollectionFS.glsl
void main()
{
    // 添加 czm_splitMode 判断 
    if (czm_splitMode > 0.0) {
        if (v_splitDirection < 0.0 && gl_FragCoord.x > czm_splitPosition) discard;
        if (v_splitDirection > 0.0 && gl_FragCoord.x < czm_splitPosition) discard;
    } 
    else if (czm_splitMode < 0.0) {
        if (v_splitDirection < 0.0 && gl_FragCoord.y > czm_splitPosition) discard;
        if (v_splitDirection > 0.0 && gl_FragCoord.y < czm_splitPosition) discard;
    }

    // The distance in UV space from this fragment to the center of the point, at most 0.5.
    float distanceToCenter = length(gl_PointCoord - vec2(0.5));
    
    /* 省略下面 ...*/
}    
```

Splitter 对象：

```javascript
// 源码位置：packages/engine/Source/Scene/Splitter.js
const Splitter = {
  /**
   * Given a fragment shader string, returns a modified version of it that
   * only renders on one side of the screen or the other. Fragments on the
   * other side are discarded. The screen side is given by a uniform called
   * `czm_splitDirection`, which can be added by calling
   * {@link Splitter#addUniforms}, and the split position is given by an
   * automatic uniform called `czm_splitPosition`.
   */
  modifyFragmentShader: function modifyFragmentShader(shader) {
    shader = ShaderSource.replaceMain(shader, "czm_splitter_main");
    shader +=
      // czm_splitPosition is not declared because it is an automatic uniform.
      "uniform float czm_splitDirection; \n" +
      "uniform float czm_splitMode; \n" +
      "void main() \n" +
      "{ \n" +
      // Don't split when rendering the shadow map, because it is rendered from
      // the perspective of a totally different camera.
      "#ifndef SHADOW_MAP\n" +
      "if (czm_splitMode > 0.0) {\n" +
      "    if (czm_splitDirection < 0.0 && gl_FragCoord.x > czm_splitPosition) discard; \n" +
      "    if (czm_splitDirection > 0.0 && gl_FragCoord.x < czm_splitPosition) discard; \n" +
      "} else if (czm_splitMode < 0.0) {  \n" +
      "    if (czm_splitDirection < 0.0 && gl_FragCoord.y > czm_splitPosition) discard; \n" +
      "    if (czm_splitDirection > 0.0 && gl_FragCoord.y < czm_splitPosition) discard; \n" +
      "} \n" +
      "#endif\n" +
      "    czm_splitter_main(); \n" +
      "} \n";

    return shader;
  },
}    
```

&nbsp;

接下来，我们的工作就是从 cesium 的整个渲染过程中，加入这个 `splitMode` 变量。

&nbsp;

## 创建 spilitMode 枚举

遵循 ceisum 的规范，创建 `/packages/engine/Srouce/Scene/SplitMode.js` 枚举文件：

```javascript
const SplitMode = {
  
  LEFT_RIGHT: 1.0,

  TOP_BOTTOM: -1.0,
};
export default Object.freeze(SplitMode);
```

文件声明了两个枚举值，代表当前模式是上下垂直分割，还是左右水平分割。

&nbsp;

### 添加到 frameState 

在 `/packages/engine/Srouce/Scene/frameState.js` 中，添加 `splitMode` 变量。 

&nbsp;

`frameState` 这个类代表了每一帧渲染时的状态信息，在 cesium 进行渲染时，可以从 `frameState` 中获取渲染一帧所需的所有上下文信息。

```javascript
// 源码位置 /packages/engine/Srouce/Scene/frameState .js
import SplitMode from "./SplitMode.js";

/* 省略上面 ...*/
this.splitPosition = 0.0;
// 添加这一句
this.splitMode = SplitMode.LEFT_RIGHT;
```

&nbsp;

### 添加到场景类

在场景类中添加一个 getter 和 setter，方便用户获取和修改：

```javascript
// 源码位置 /packages/engine/Srouce/Scene/Scene.js

/**
   * Gets or sets the position of the splitter within the viewport.  Valid values are between 0.0 and 1.0.
   * @memberof Scene.prototype
   *
   * @type {number}
   */
  splitPosition: {
    get: function () {
      return this._frameState.splitPosition;
    },
    set: function (value) {
      this._frameState.splitPosition = value;
    },
  }
	// 添加 splitMode 
  splitMode: {
    get: function () {
      return this._frameState.splitMode;
    },
    set: function (value) {
      this._frameState.splitMode = value;
    },
  },
```

&nbsp;

### 添加到 UniformState

`UniformState` 这个类的主要职责是管理、计算和提供渲染过程中 glsl 着色器所需的各种 uniform 变量的值，我们在 glsl 代码中用到的各种内置 uniform 也是从这里进行处理的。

```javascript
// 源码位置 /packages/engine/Srouce/Renderer/UniformState.js
import SplitMode from "../Scene/SplitMode.js";
/* 省略上面 ...*/
this._splitPosition = 0.0;
// 添加
this._splitMode = SplitMode.LEFT_RIGHT;

/* 省略中间 ...*/
  /**
   * The splitter position to use when rendering with a splitter. This will be in pixel coordinates relative to the canvas.
   * @memberof UniformState.prototype
   * @type {number}
   */
  splitPosition: {
    get: function () {
      return this._splitPosition;
    },
  },
  // 添加 getter
  splitMode: {
    get: function() {
        return this._splitMode;
    }
  },  
```

另外，我们在对于 `splitPosition` 进行处理的代码也要进行修改。

```javascript
  // Convert the relative splitPosition to absolute pixel coordinates
  this._splitPosition =
    frameState.splitPosition * frameState.context.drawingBufferWidth;
```

上面这段代码中，cesium 将 `splitPosition` 的归一化坐标相对位置（0-1）转换为实际的像素坐标，在这里我们根据从 `frameState` 获取到的当前分割模式，加上垂直分割的逻辑的判断：

```javascript
  // Convert the relative splitPosition to absolute pixel coordinates
const splitMode = frameState.splitMode;
// 从 frameState 中获取 splitMode 赋值给 uniform
this._splitMode = splitMode;
// 坐标转换
this._splitPosition = splitMode === SplitMode.LEFT_RIGHT ? frameState.splitPosition * frameState.context.drawingBufferWidth : frameState.splitPosition * frameState.context.drawingBufferHeight;
```

&nbsp;

## 添加到 AutomaticUniforms

AutomaticUniforms 这个类跟 UniformState 强相关，主要用于将 UniformState 中的字段跟 glsl 着色器之间建立映射和关联关系。

```javascript
  /**
   * An automatic GLSL uniform representing the splitter position to use when rendering with a splitter.
   * This will be in pixel coordinates relative to the canvas.
   *
   * @example
   * // GLSL declaration
   * uniform float czm_splitPosition;
   */
  czm_splitPosition: new AutomaticUniform({
    size: 1,
    datatype: WebGLConstants.FLOAT,
    getValue: function (uniformState) {
      return uniformState.splitPosition;
    },
  }),
  // 添加内置 uniform
  czm_splitMode: new AutomaticUniform({
    size: 1,
    datatype: WebGLConstants.FLOAT,
    getValue: function(uniformState) {
        return uniformState.splitMode;
    }
})
```

&nbsp;

## 扩展 Cesium.SplitDirection 枚举

最后，为了代码的可读性和合理性，我们最好把 `Cesium.SplitDirection` 进行扩展，添加两个可选项：

```javascript
// 源码位置：packages/engine/Source/Scene/SplitDirection.js
const SplitDirection = {
  /**
   * 将元素显示在左边
   * @type {number}
   * @constant
   */
  LEFT: -1.0,
    
  /**
   * 将元素显示在上边
   * @type {number}
   * @constant
   */
  TOP: -1.0,      

  /**
   * 总是显示元素
   * @type {number}
   * @constant
   */
  NONE: 0.0,

  /**
   * 将元素显示在右边
   * @type {number}
   * @constant
   */
  RIGHT: 1.0,
  
  /**
   * 将元素显示在下边
   * @type {number}
   * @constant
   */
  BOTTOM: 1.0,   
};
export default Object.freeze(SplitDirection);
```

至此，整个改造就完成了。

&nbsp;

## 总结

在这篇博客中，我们对 Cesium 源码进行了部分重构，扩展了原本的左右分割模式，实现了上下垂直分割。

由于 Cesium 源码非常复杂，写这篇博客也是主要出于学习的目的，可能并不具备部署到生产环境的条件，请注意甄别。

