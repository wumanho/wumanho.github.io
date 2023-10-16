---
weight: 2
title: "持久化存储 leaflet geoman 图层指南"
summary: "本篇博客将会介绍如何将一个 Leaflet geoman 绘制的图形入库并还原进行二次编辑"
date: 2023-04-27T21:10:12+08:00
draft: false
categories: ["技术"]
tags: ["leaflet","webgis","gis"]
---

## Leaflet-Geoman 介绍

[Leaflet-Geoman](https://geoman.io/leaflet-geoman) 是一个对 leaflet 的扩展或者说插件，可以让用户对 leaflet 对象进行自由编辑，包括了自由绘制、缩放旋转拖拽等等功能。

&nbsp;

## 需求

在使用 leaflet-geoman 提供的绘制功能在地图上绘制一个 polygon 之后，需要将 polygon 持久化保存到数据库中，并允许重新加载以及再次进行编辑。

&nbsp;

## 保存入库

要将整个 geoman 对象存入数据库是不现实的，因为如果在控制台打印一下就会发现它自身拥有的字段和属性非常多，即使在数据库中以 json 字段存储，也会产生过多数据。

![控制台信息](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/leaflet/consle.png)

所以思路就是只保存关键信息，下次加载时再通过这些关键信息将图形重新加载出来。

简单版：

```javascript
 /**
   * 生成用于保存到数据库的 geoJSON 数据
   * @returns featureCollection geoJSON
   */
  function generateGeoJson() {
    const fg = L.featureGroup()
    const layers = map.pm.getGeomanDrawLayers()
    layers.forEach(function (layer) {
      fg.addLayer(layer)
    })
    return fg.toGeoJSON()
  }
```

详细版：

```javascript
function generateGeoJson() {
  var layers = findLayers(map)

  var geo = {
    type: 'FeatureCollection',
    features: []
  }
  layers.forEach(function (layer) {
    var geoJson = JSON.parse(JSON.stringify(layer.toGeoJSON()))
    if (!geoJson.properties) {
      geoJson.properties = {}
    }

    geoJson.properties.options = JSON.parse(JSON.stringify(layer.options))

    if (layer.options.radius) {
      var radius = parseFloat(layer.options.radius)
      if (radius % 1 !== 0) {
        geoJson.properties.options.radius = radius.toFixed(6)
      } else {
        geoJson.properties.options.radius = radius.toFixed(0)
      }
    }

    if (layer instanceof L.Rectangle) {
      geoJson.properties.type = 'rectangle'
    } else if (layer instanceof L.Circle) {
      geoJson.properties.type = 'circle'
    } else if (layer instanceof L.CircleMarker) {
      geoJson.properties.type = 'circlemarker'
    } else if (layer instanceof L.Polygon) {
      geoJson.properties.type = 'polygon'
    } else if (layer instanceof L.Polyline) {
      geoJson.properties.type = 'polyline'
    } else if (layer instanceof L.Marker) {
      geoJson.properties.type = 'marker'
    }

    geo.features.push(geoJson)
  })
  return geo
}

function findLayers(map) {
  let layers = []
  map.eachLayer((layer) => {
    if (
      layer instanceof L.Polyline ||
      layer instanceof L.Marker ||
      layer instanceof L.Circle ||
      layer instanceof L.CircleMarker
    ) {
      layers.push(layer)
    }
  })
  // 过滤没有 geoman 实例的图层
  layers = layers.filter((layer) => !!layer.pm)
  // 过滤没必要的临时数据
  layers = layers.filter((layer) => !layer._pmTempLayer)
  return layers
}

```

&nbsp;

## 加载图层

加载方法：

```javascript
function importGeoJSON(feature) {
  const geoLayer = L.geoJSON(feature, {
    style: function (feature) {
      return feature.properties.options
    },
    pointToLayer: function (feature, latlng) {
      switch (feature.properties.type) {
        case 'marker':
          return new L.Marker(latlng)
        case 'circle':
          return new L.Circle(latlng, feature.properties.options)
        case 'circlemarker':
          return new L.CircleMarker(latlng, feature.properties.options)
      }
    }
  })

  geoLayer.getLayers().forEach((layer) => {
    if (layer._latlng) {
      var latlng = layer.getLatLng()
    } else {
      var latlng = layer.getLatLngs()
    }
    switch (layer.feature.properties.type) {
      case 'rectangle':
        new L.Rectangle(latlng, layer.options).addTo(map)
        break
      case 'circle':
        console.log(layer.options)
        new L.Circle(latlng, layer.options).addTo(map)
        break
      case 'polygon':
        new L.Polygon(latlng, layer.options).addTo(map)
        break
      case 'polyline':
        new L.Polyline(latlng, layer.options).addTo(map)
        break
      case 'marker':
        new L.Marker(latlng, layer.options).addTo(map)
        break
      case 'circlemarker':
        new L.CircleMarker(latlng, layer.options).addTo(map)
        break
    }
  })
}
```

启用编辑：

```javascript
 function editPolygon(layer) {
    layer.pm.enable({
      allowSelfIntersection: false   // 禁止交叉
    })
  }
```

&nbsp;

（完）

### 参考

[示例代码](https://jsfiddle.net/falkedesign/4fycojbu/)





