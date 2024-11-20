---
weight: 2
title: "Cesium 在不显示地形的情况下获取地形高度"
summary: "介绍如何使用奇技淫巧在 Cesium 不显示地形的情况下获取地形高度"
date: 2024-11-20T11:16:02+08:00
draft: false
categories: ["技术"]
tags: ["GIS","Cesium","地形高"]
---

## 前言

由于业务需要，要求可以动态对 Cesium 地形进行开启和关闭，同时在获取地形高时，还能获取到实际地形高，用于计算模型（Model Entity）等场景的实际飞行高度。

所以有了这篇博客，记录一下是如何实现的。

&nbsp;

## 动态设置地形

要动态设置地形可以通过修改 viewer 的 terrainProvider 来实现：

```javascript
/* 设置地形 */
async function setTerrain(viewer, terrainPath) {
  viewer.terrainProvider = await Cesium.CesiumTerrainProvider.fromUrl(terrainPath)
}
/* 清除地形 */
function clearTerrain(viewer){
  viewer.terrainProvider = new Cesium.EllipsoidTerrainProvider()
}
```

&nbsp;

## 实现思路

由于需求比较奇怪，也不知道有没有什么推荐的最佳实践，就用了一些邪门歪道来实现。

我们在进行地形高查询的时候，一般都是这样获取 provider：

```javascript
// 获取 provider
const terrainProvider = this._viewer.terrainProvider
const graphic = Cesium.Cartographic.fromCartesian(cartesian)
const height = await Cesium.sampleTerrainMostDetailed(terrainProvider, [graphic])
```

所以只要在初始化时，用一个额外的字段保存 provider，然后在查询的时候，用这个 provider 来查询，就可以了：

```javascript
/* 设置用于查询的地形 */
async function initQueryPrivider(viewer, terrainPath) {
  viewer['terrainProviderForQuery'] = await Cesium.CesiumTerrainProvider.fromUrl(terrainPath)
}
```

查询的时候：

```javascript
// 获取 provider
const terrainProvider = this._viewer.terrainProviderForQuery
const graphic = Cesium.Cartographic.fromCartesian(cartesian)
const height = await Cesium.sampleTerrainMostDetailed(terrainProvider, [graphic])
```

&nbsp;

（完）
