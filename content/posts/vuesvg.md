---
weight: 2
title: "Vue 封装 icon 组件"
summary: "基于 SVG雪碧图 和 svg-sprite-loader 封装全局 icon 组件"
date: 2020-04-24T10:52:44+08:00
draft: false
categories: ["前端"]
tags: ["JavaScript","Vue","SVG-Sprite"]
---

通过 SVG `symbol` 的方式引入 icon 是一种全新的使用方式，应该说这才是未来的主流，同时也是 [iconfont](https://www.iconfont.cn/) 推荐的引用方法，工作原理是利用一种叫做`SVG雪碧图`的 SVG 集合，关于`SVG雪碧图`的前置知识，请参考张鑫旭大佬的[这篇文章](https://www.zhangxinxu.com/wordpress/2014/07/introduce-svg-sprite-technology/)。

在这个基础上，只要做一些简单的封装，就可以在项目中方便地引用 SVG icon。

## 安装 svg-sprite-loader

```code
yarn add svg-sprite-loader -D
```

想要轻松地实现雪碧图，可以借助一个 webpack loader ：[svg-sprite-loader](https://github.com/JetBrains/svg-sprite-loader) 

这个 loader 的主要作用是将引入的图像转换成 SVG `symbol`元素，例如：

```vue
// index.vue

<script>
//引入 svg，打印看看
import cart from "@/assets/icons/cart.svg"
console.log(cart);
    
export default {
  name: "index"
}
</script>
```

这个时候把我们导入的 SVG 图标打印出来，可以看到已经转换成一个`symbol`元素，默认 id 跟文件名相同：

![控制台](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/vuesvg/symbol01.png)

然而需要实现这种自动转换的效果，只是引入 loader 还不够，**需要对 loader 进行配置**。

&nbsp;

## 配置 svg-sprite-loader

使用 Vue 推荐的 [webpack-chain](https://github.com/Yatoo2018/webpack-chain/tree/zh-cmn-Hans) 的方式来配置一下 loader，具体配置可以参考：

```javascript
// vue.config.js

const path = require("path")
module.exports = {
  lintOnSave: false,
  chainWebpack: config =>{
    const dir = path.resolve(__dirname,"src/assets/icons") //指定项目保存 icon 的文件夹
    config.module
        .rule("svg-sprite") //指定规则
        .test(/\.svg$/) //正则匹配，指定.svg后缀
        .include.add(dir).end() //注意这里需要 end() 回退一下，再加 use
        .use("svg-sprite-loader").loader("svg-sprite-loader").options({extract:false}).end()
    config.plugin("svg-sprite").use(require("svg-sprite-loader/plugin"),[{plainSprite:true}])
    config.module.rule("svg").exclude.add(dir) //其他规则排除指定的目录，避免冲突
  }
}
```

配置完成后，就可以引入 svg 并使用 symbol 元素将 icon 展示在页面中了

```vue
<template>
  <div>
    <svg>
      <use xlink:href="#cart"/>
    </svg>
    购物车
  </div>
</template>

<script>
//引入
import cart from "@/assets/icons/cart.svg"
console.log(cart);
export default {
  name: 'index'
}
</script>
```

效果：

![icon效果](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/vuesvg/cart.png)

&nbsp;

## 组件化 

利用 loader 可以方便高效地引用 SVG icon，但仍然存在一些工程化问题，例如每次使用时，需要手动引入：

```vue
<script>
//每个 icon 都需要单独引入
import cart from "@/assets/icons/cart.svg"
import tag from "@/assets/icons/tag.svg"
console.log(cart);
console.log(tag);
</script>
```

以及需要写 svg 和 use 标签展示

```vue
<template>
  <div>
    <svg>
      <use xlink:href="#cart"/>
    </svg>
      <svg>
      <use xlink:href="#tag"/>
    </svg>
  </div>
</template>
```

显然这些都是可以优化的地方，解决办法是抽取一个全局 icon 组件。

```vue
// MyIcon.vue

<template>
  <svg class="icon">
    <use :xlink:href="name"/>
  </svg>
</template>

<script>
//利用require.context 实现自动引入icons
const importAll = (requireContext) => requireContext.keys().forEach(requireContext);
try {
  importAll(require.context('../assets/icons', true, /\.svg$/));
} catch (e) {
  console.log(e);
}
    
export default {
  name: 'MyIcon',
  props:["name"]
};
</script>

<style lang="scss" scoped>
.icon{
  
}
</style>
```

这样，在使用时就可以更加简单：

```vue
<template>
  <div>
    <MyIcon name="#tag"/>
    标签
  </div>
</template>

<script>
import MyIcon from "@/components/MyIcon";
export default {
  name: 'index',
  components: {MyIcon}
}
</script>

<style lang="scss" scoped>

</style>
```

效果：

![tag效果](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/vuesvg/tag.png)

&nbsp;

（完）

&nbsp;