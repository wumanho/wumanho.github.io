---
weight: 2
title: "【NodeJS】后端开放第三方接口基本设计"
summary: "记录 Node 后端开放第三方接口设计与实现"
date: 2022-10-25T19:59:14+08:00
categories: ["技术"]
tags: ["JavaScript","Node.js"]
draft: false
---

## 前言

近期公司一个用 Node 做后端的大前端项目需要开放一些接口给第三方平台接入使用，于是便开始了解一些接口开放的规范，记录一下实现过程。

在进行了一些简单的调研后，发现业界实现这种开放接口的方式都是类似的，于是选择了最适合我们业务的一种设计方式：**动态签名**。

&nbsp;

## 第三方系统接入

第三方系统需要调用我们平台的 API 时，需要先进行接入的操作，由于我们的系统已经实现了**基于多租户的资源隔离**，所以就顺理成章地使用租户的方式来进行系统间的对接。

而如果本身没有类似多租户的设计的话，也没有什么关系，「接入」的本质其实就是为了得到两个关键的参数，`appId` 以及 `secret`：

* appId：为申请接入的第三方平台生成的唯一凭证
* secret：为申请接入的第三方平台生成的密钥，用与加密

有了 `appId` 和 `secret` 就保证了最基本的安全问题，那就是**调用者的合法性**。

如果需要更进一步的限制，可以设计`IP白名单`或`域名白名单`方案，在用户申请接入时，要求用户填写自己服务端的访问 IP 或者域名，平台方只允许白名单中的地址进行访问。

```sql
-- ----------------------------
--  Table structure for `sys_third_party_info`
-- ----------------------------
DROP TABLE IF EXISTS `sys_third_party_info`;
CREATE TABLE `sys_third_party_info` (
  `id` varchar(128) NOT NULL,
  `app_id` varchar(128) DEFAULT NULL,
  `status` int DEFAULT NULL,
  `secret` varchar(128) DEFAULT NULL,
  `app_name` varchar(128) DEFAULT NULL,
  `logo` varchar(255) DEFAULT NULL,
  `create_by` varchar(128) DEFAULT NULL,
  `update_by` varchar(128) DEFAULT NULL,
  `remark` varchar(255) DEFAULT NULL,
  `create_time` timestamp NULL DEFAULT NULL,
  `update_time` timestamp NULL DEFAULT NULL,
  `delete_time` timestamp NULL DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `app_id` (`app_id`),
  KEY `idx_sys_tenant_delete_time` (`delete_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

&nbsp;

## 调用规则

在保证了调用的合法性之后，则需要考虑一下调用时数据在网络上传输的安全性，可以从两方面考虑，分别是 `对数据的保密`，以及 `对数据的防篡改`。

例如，提供一个第三方接口 `getOrders()` 用于获取订单，那么参数可以这样设计，将参数分为两部分，分别是 `公共参数` 以及 `业务参数`：

公共请求参数：

| 字段标识 |                             说明                             |
| :------: | :----------------------------------------------------------: |
|  appId   |               开发者在开放平台申请获取的应用ID               |
|  paras   | 使用 secret 对所有传入参数采用 XXTea 加密，并且按照接口详细规范中定义的参数（除 appId、sign）拼接，不要求参数顺序。 |
|   sign   |              对 appId、paras 进行签名，要求顺序              |

业务请求参数：

| 字段标识  |        说明        | 必填 |
| :-------: | :----------------: | :--: |
| timeStamp |    时间戳，毫秒    |  Y   |
| startTime | 过滤订单的开始时间 |  N   |
|  endTime  | 过滤订单的结束时间 |  N   |
|  keyWord  |     订单关键字     |  N   |

&nbsp;

## 数据传输加密

先来看第一个问题，如何对数据进行加密传输，同时在平台接收到参数之后可以解析成 JS 代码，这个时候就需要一个`可逆的加密算法`，例如 `XXTea`。

`XXTea` 是初代微型加密算法的二次增强版，特点是性能不错且实现简单，下面的代码演示了如何通过`xxtea-node`对加解密操作进行封装：

```javascript
const xxtea = require('xxtea-node');

const XXTea = {
    // 加密
    encrypt(plainText, secret) {
        if (!plainText) {
            return '';
        }
        let array = xxtea.encrypt(xxtea.toBytes(plainText.toString()), xxtea.toBytes(secret));
        let buffer = Buffer.from(array);
        let encryptedText = buffer.toString('hex');
        return encryptedText;
    },
    // 解密
    decrypt(encryptedText, secret) {
        if (!encryptedText) {
            return '';
        }
        let buffer = new Buffer(encryptedText, 'hex');
        let array = new Uint8Array(buffer);
        try {
            return xxtea.toString(xxtea.decrypt(array, xxtea.toBytes(secret)));
        } catch (e) {
            return '';
        }
    }
}

module.exports = XXTea;
```

封装好之后就可以直接用于对参数进行加密：

```javascript
const treeMap = require('./treemap')
const xxtea = require('./xxtea.js')
const APP_ID = '35c7b102'; // 平台方颁发的唯一接入凭证
const APP_SECRET = '6e1d88c3nqq95f9f82tt941309b68b1a402233f8' // 平台方颁发的密钥

const param = {
    timeStamp: '1666687690537',
    startTime: '2022-10-20 18:00:00',
    endTime: '2022-10-24 18:00:00',
    keyWord: '扫地机器人'
}
// 将参数转为 treeMap 格式：a=value&b=value1...
const treemap = treeMap(param)
// 用 APP_SECRET 对参数进行加密
const paras = xxtea.encrypt(treemap, APP_SECRET).toUpperCase()
```

&nbsp;

## 数据传输防篡改

再来看第二个问题，对于防篡改，则要求要求调用方传入公共参数 `sign` 来实现。

公共参数中的 `sign` 是除了 `sign` 本身以外的所有参数通过 `不可逆加密算法` 计算得出的摘要结果。

平台方在接收到请求之后，对参数以约定好的方式进行签名，然后将签名结果与调用方传入的 `sign` 签名进行比对，就可以得知数据是否被篡改。

下面的代码演示了如何通过`HmacSHA1`对签名操作进行封装：

```javascript
const crypto = require('crypto');

// HMAC 算法集合
const HMAC = {
  // HMAC_SHA1 加密算法
  HmacSHA1(plainText, secret) {
    return crypto.createHmac('sha1', secret).update(plainText).digest('hex').toUpperCase(); //16进制  
  }
}

module.exports = HMAC;
```

对上面的加密参数进行签名操作，需要注意的是，在签名时要求调用方`必须固定顺序`，不然得出的签名结果就会不同，导致不对不通过：

```javascript
const treeMap = require('./treemap')
const xxtea = require('./xxtea.js')
const APP_ID = '35c7b102'; // 平台方颁发的唯一接入凭证
const APP_SECRET = '6e1d88c3nqq95f9f82tt941309b68b1a402233f8' // 平台方颁发的密钥

const param = {
    timeStamp: '1666687690537',
    startTime: '2022-10-20 18:00:00',
    endTime: '2022-10-24 18:00:00',
    keyWord: '扫地机器人'
}
// 将参数转为 treeMap 格式：a=value&b=value1...
const treemap = treeMap(param)
// 用 APP_SECRET 对参数进行加密
const paras = xxtea.encrypt(treemap, APP_SECRET).toUpperCase()
// 签名
const sign = HMAC.HmacSHA1(APP_ID + paras, APP_SECRET).toUpperCase()
```

&nbsp;

## 服务端鉴权

服务端在接收到请求后进行鉴权，需要做以下几件事情：

* 根据传入的 appId 获取对应的 secret
* 对参数进行签名，比对客户端签名
* 解密参数
* 后续安全性校验（基于时间戳的 API 重放攻击防御）

```javascript
/**
* 鉴权辅助函数封装
* @param appId 租户ID
* @param encryptedParas 加密的参数
* @param clientSign 客户端签名
* @returns 解析后的参数
*/
async authenticate(appId, encryptedParas, clientSign) {
  const { secret } = await Tenant.findOne({ where: { appId: appId } }, appId)
  // 签名校验
  const inValidate = this.signValidating(appId, encryptedParas, clientSign, secret)
  if (inValidate) {
    return { decryptedParas: null, msg: '签名校验不通过' }
  }
  const treeMap = xxtea.decrypt(encryptedParas, secret)
  return this.parseQuery(treeMap)
},
```

```javascript
/**
* 校验签名
* @param appId 租户ID
* @param paras 加密的参数
* @param clientSign 客户端签名
* @param secret 租户密钥
* @returns boolean
*/
signValidating(appId, paras, clientSign, secret) {
  const serverSign = HMAC.HmacSHA1(appId + paras, secret).toUpperCase()
  return serverSign !== clientSign
}
```

```javascript
/**
* 解析 treemap
* @param treemap a=value1&b=value2&…
* @returns {{decryptedParas,msg}} treemap 参数解析后的 js 对象，提示消息
*/
parseQuery(treemap) {
  const decryptedParas = treemap.split('&').reduce((source, c) => {
  const paramMap = c.split('=')
  return {
    ...source,
    [paramMap[0]]: paramMap[1] || ''
   }
 }, {})
  // 时间戳是必传项
  if (!decryptedParas['timeStamp']) {
    return { decryptedParas: null, msg: '缺少必要参数:时间戳' }
  }
  // 防止 API 的重放攻击 https://help.aliyun.com/document_detail/50041.html
  const paramsTime = moment(Number(decryptedParas['timeStamp']))
  const now = moment(Date.now())
  const timeDiff = now.diff(paramsTime, 'minutes')
  if (timeDiff >= 15) {
    return { decryptedParas: null, msg: '请求时间超出允许的范围' }
  }
  return { decryptedParas, msg: '' }
}
```

&nbsp;

## 总结

至此，基于动态签名的开放接口鉴权设计就完成了。
