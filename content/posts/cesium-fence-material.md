---
weight: 2
title: "【Cesium】飞行区电子围栏"
summary: "本篇博客将会尝试还原大疆司空2平台的栅栏飞行区材质"
date: 2024-06-07T10:10:12+08:00
draft: false
categories: ["技术"]
tags: ["DJI","司空2","gis","Cesium","圆形 polyline"]
---

## 前言

本篇博客将会尝试还原 [大疆司空2](https://enterprise.dji.com/cn/flighthub-2) 平台的栅栏飞行区材质，并将材质应用到 Primitive 中。

## 司空2 效果

### 多边形和圆形飞行区栅栏效果

![zones](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/cesiumfz/zones.png)

### 圆形栅栏效果

![circle](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/cesiumfz/workzone.png)

### 编辑中效果

![edit](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/cesiumfz/editting.png)

&nbsp;

## 如何绘制圆形

目前 Cesium 支持的圆形 geometry 只有 ellipse，刚开始我也尝试过用 ellipse 来实现，但是栅栏效果是个 polyline 材质

当内部填充区域通过 ellipse 实现，但是外围通过 polyline 来包裹的话，要实现完全贴合的包围效果会比较难。

要以相对简单的方式实现完全贴合的包围效果，可以通过圆心坐标和半径，通过圆心坐标向外移动半径距离，得出作为填充区域的 polygon 和包围边框的 polyline 的坐标。

这样就可以得到一个完全包围的圆形效果：

```javascript
/**
 * 创建圆形 polygon 坐标
 * @param {Array} centerLatlng 圆心坐标(经纬度)
 * @param {Number} radius 半径
 */
export const generateCirclePolygon = (centerLatlng, radius) => {
  const point = turf.point([centerLatlng[0], centerLatlng[1]])
  const polylinePointsArray = []
  for (let i = 0; i <= 360; i = i + 1) {
    if (i % 3 === 0) continue
    const targetLatlng = turf.destination(point, radius, i, {
      units: 'meters'
    }).geometry.coordinates
    const resLatlng = [targetLatlng[0], targetLatlng[1], 0]
    polylinePointsArray.push(resLatlng)
  }
  return polylinePointsArray.filter((_, i) => i % 2 === 0)
}
```

&nbsp;

## 创建材质

```javascript
// 创建 polyline 栅栏材质
export function createFenceMaterial() {
  const source = `#ifdef GL_OES_standard_derivatives\n
      #extension GL_OES_standard_derivatives : enable\n
      #endif\n\n\n
      uniform vec4 color;
      uniform float dashLength;
      uniform float dashPattern;
      uniform float maskLength; 
      uniform float outlineWidth;
      uniform vec4 outlineColor;
      in float v_polylineAngle;
      in float v_width; // 线段宽度
      mat2 rotate(float rad) {    
        float c = cos(rad);    
        float s = sin(rad);    
        return mat2(    
          c, s,   
          -s, c    
        );
      }
      // 计算给定输入的材质
      // 栅栏形状的polyline是以虚线材质未基础，往虚线空隙里面填充外轮廓是透明的外轮廓线所绘制的
      czm_material czm_getMaterial(czm_materialInput materialInput)
      {    
        // 获取默认的材质    
        czm_material material = czm_getDefaultMaterial(materialInput);    
        // 外轮廓纹理部分    
        // 获取标准化的纹理坐标    
        vec2 st = materialInput.st;    
        //v_width是整个线宽，outlineWidth是轮廓线宽，两者差值的一半就是外轮廓线内部线条的宽度    
        float halfInteriorWidth =  0.5 * (v_width - outlineWidth) / v_width;    
        // 判断当前的纹理坐标是否在内部线条范围内，如果在范围内，b值将为1，否则为0。step(edge, x)函数会返回一个值，如果x < edge，返回0.0，否则返回1.0    
        float b = step(0.5 - halfInteriorWidth, st.t);    
        b *= 1.0 - step(0.5 + halfInteriorWidth, st.t); 
        //计算当前片元距离最近的颜色分隔线的距离，这个距离将被用于后面的抗锯齿处理。   
        float d1 = abs(st.t - (0.5 - halfInteriorWidth));   
        float d2 = abs(st.t - (0.5 + halfInteriorWidth));   
        float dist = min(d1, d2);  
        // 根据b的值选择颜色，如果b是1，则选择color，否则选择outlineColor。这就确定了当前片元的基础颜色   
        vec4 currentColor = mix(outlineColor, color, b);    
        // 使用czm_antialias()函数进行抗锯齿处理，输入参数包括轮廓色，内部色，当前色和距离    
        vec4 outColor = czm_antialias(outlineColor, color, currentColor, dist);    
        // 对经过抗锯齿处理后的颜色进行伽马校正    
        vec4 gapColor = czm_gammaCorrect(outColor);    
        // 虚线纹理部分   
        // 计算旋转后的片元位置。    
        vec2 pos = rotate(v_polylineAngle) * gl_FragCoord.xy;    
        // 计算当前片元在虚线模式中的位置。    
        float dashPosition = fract(pos.x / (dashLength * czm_pixelRatio));    
        // 计算对应虚线模式的索引位置。    
        float maskIndex = floor(dashPosition * maskLength);    
        // 通过使用虚线模式和索引位置，计算当前位置是否应有线条。    
        float maskTest = floor(dashPattern / pow(2.0, maskIndex));    
        // 如果当前位置应为虚线的间隔部分，则颜色设为gapColor，否则设为color。   
        vec4 fragColor = (mod(maskTest, 2.0) < 1.0) ? gapColor : color;    
        if (fragColor.a < 0.005) {        
          discard;    
        }    
        // 设置材质的颜色和透明度，后返回处理过后的材质    
        fragColor = czm_gammaCorrect(fragColor);    
        material.emission = fragColor.rgb;   
        material.alpha = fragColor.a;    
        return material;
      }`
  if (!Cesium.Material._materialCache.getMaterial('PolylineFenceMaterial')) {
    Cesium.Material._materialCache.addMaterial('PolylineFenceMaterial', {
      fabric: {
        type: PolylineFenceMaterial.materialType,
        uniforms: {
          color: new Cesium.Color(1, 1, 1, 1),
          outlineColor: new Cesium.Color(1, 1, 1, 1),
          dashLength: 10,
          dashPattern: 15,
          outlineWidth: 16,
          maskLength: 16
        },
        source: source
      },
      translucent: function (material) {
        return true
      }
    })
  }
}
```

&nbsp;

## 应用材质

```javascript
  /**
   * 添加贴地多段线
   * @param {* Object} cartesian  xyz 坐标
   */
  addGroundPolyline(cartesian) {
    // 创建折线Primitive
    const polyline = new Cesium.GeometryInstance({
      id: uuid(),
      geometry: new Cesium.GroundPolylineGeometry({
        positions: cartesian,
        loop: true,
        width: 14.0
      })
    })
    // 通过自定义材质创建 primitive
   viewer.scene.groundPrimitives.add(
      new Cesium.GroundPolylinePrimitive({
        geometryInstances: polyline,
        appearance: new Cesium.EllipsoidSurfaceAppearance({
          material: new Cesium.Material({
            fabric: {
              type: 'PolylineFenceMaterial',
              uniforms: {
                color: Cesium.Color.fromCssColorString('#fa0707'),
                outlineColor: new Cesium.Color(1, 1, 1, 0),
                dashLength: 10,
                dashPattern: 15,
                outlineWidth: 11,
                maskLength: 16
              }
            }
          })
        })
      })
    )
  }
```

&nbsp;

（完）
