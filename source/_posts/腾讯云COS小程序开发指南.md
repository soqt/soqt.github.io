---
layout: post
title: 腾讯云COS小程序开发指南
categories: [cloud]
description: 踩坑笔记
keywords: 腾讯云
date: 2019-12-06 10:43:00
tags: cos
hidden: true
---
小程序开发需要用到对象储存，因为使用腾讯云，所以腾讯云的COS也自然成为了go-to choice。就是开发过程有点坑，记录一下。

<!-- more -->

先附上腾讯云官方文档: [腾讯云COS](https://cloud.tencent.com/document/product/436/7751)
我们还需要看两个库：
微信COS SDK github: [cos-wx-sdk-v5](https://github.com/tencentyun/cos-wx-sdk-v5)
JS COS SDK github: [cos-js-sdk-v5](https://github.com/tencentyun/cos-js-sdk-v5)

## 用SDK开发小程序COS
实现腾讯云COS分两步，第一步需要现在后端建立起sts验证API，以防止secretKey泄露。
如果用SDK的话，可以参考 [cos-js-sdk-v5](https://github.com/tencentyun/cos-js-sdk-v5) 下面的server文档。


这时你会发现竟然有两个版本的sts，一脸懵逼。文档也没解释清楚他们的区别，但是你要用SDK的话，选择sts.js 或sts.php 就可以了。因为用sts-auth结合SDK会报错。


当你在复制粘贴后端代码的时候，一定注意把AllowPrefix 这一个字段修改为你想要上传的文件夹，比如 *** upload/\* ***。如果你想使用其他COS API，比如删除操作，记得把 policy['statement']['action']里面对应的功能uncomment掉。

在小程序里下载cos-wx-sdk-v5 到utils，然后对应引用就可以了，跟着demo走很简单。

## 不用SDK开发小程序COS
当你用了SDk，你就后悔了。

why？

因为cos-wx-sdk-v5这玩意竟然有600多kb？！小程序整个包大小不能超过2 MB，你告诉我我上传个破图片就要占我四分之一的容量，而且大多数人也就是用一个上传图片的功能吧？！

所以sdk这个东西，你早晚都要删掉。

这时你就要踩坑了。

还是先搞后端，之前的sts.js 的API不能用了，因为你要重新写一个sts-auth的API。

然后你就会发现，腾讯提供的github demo，其中有JS SDK, WX SDK, Node SDK... WTF?

而且每个SDK demo下面的server demo还都不是一致的？？？

经过几番折磨，我发现只有JS COS SDK github下面的auth-sts文件是对的: [auth-sts](https://github.com/tencentyun/cos-js-sdk-v5/blob/master/server/sts-auth.js)

打开他，复制粘贴会吧。别把底下的express server也粘贴了，看看代码再复制啊。

我把它抽离出来单独写了个 sts-autu-utlis, 代码如下:

```
// 临时密钥计算样例

const crypto = require('crypto');
const request = require('request');

const {
  cos: {
    SecretKey, SecretId, Region, Bucket,
  },
} = require('../../config');

// 配置参数
const config = {
  Url: 'https://sts.api.qcloud.com/v2/index.php',
  Domain: 'sts.api.qcloud.com',
  Proxy: '',
  SecretId, // 固定密钥
  SecretKey, // 固定密钥
  Bucket,
  Region,
  AllowPrefix: 'upload/*', // 这里改成允许的路径前缀，这里可以根据自己网站的用户登录态判断允许上传的目录，例子：* 或者 a/* 或者 a.jpg
};

// 缓存临时密钥
const tempKeysCache = {
  policyStr: '',
  expiredTime: 0,
};

const util = {
  // 获取随机数
  getRandom(min, max) {
    return Math.round(Math.random() * (max - min) + min);
  },
  // obj 转 query string
  json2str(obj, $notEncode) {
    const arr = [];
    Object.keys(obj).sort().forEach((item) => {
      const val = obj[item] || '';
      arr.push(`${item}=${$notEncode ? encodeURIComponent(val) : val}`);
    });
    return arr.join('&');
  },
  // 计算签名
  getSignature(opt, key, method) {
    const formatString = `${method + config.Domain}/v2/index.php?${util.json2str(opt)}`;
    const hmac = crypto.createHmac('sha1', key);
    const sign = hmac.update(Buffer.from(formatString, 'utf8')).digest('base64');
    return sign;
  },
};

// 拼接获取临时密钥的参数
const getTempKeys = function (callback) {
  // 判断是否修改了 AllowPrefix
  if (config.AllowPrefix === '_ALLOW_DIR_/*') {
    callback({ error: '请修改 AllowPrefix 配置项，指定允许上传的路径前缀' });
    return;
  }

  // 定义绑定临时密钥的权限策略
  const ShortBucketName = config.Bucket.substr(0, config.Bucket.lastIndexOf('-'));
  const AppId = config.Bucket.substr(1 + config.Bucket.lastIndexOf('-'));
  const policy = {
    version: '2.0',
    statement: [{
      action: [
        // // 这里可以从临时密钥的权限上控制前端允许的操作
        // 'name/cos:*', // 这样写可以包含下面所有权限

        // // 列出所有允许的操作
        // // ACL 读写
        // 'name/cos:GetBucketACL',
        // 'name/cos:PutBucketACL',
        // 'name/cos:GetObjectACL',
        // 'name/cos:PutObjectACL',
        // // 简单 Bucket 操作
        // 'name/cos:PutBucket',
        // 'name/cos:HeadBucket',
        // 'name/cos:GetBucket',
        // 'name/cos:DeleteBucket',
        // 'name/cos:GetBucketLocation',
        // // Versioning
        // 'name/cos:PutBucketVersioning',
        // 'name/cos:GetBucketVersioning',
        // // CORS
        // 'name/cos:PutBucketCORS',
        // 'name/cos:GetBucketCORS',
        // 'name/cos:DeleteBucketCORS',
        // // Lifecycle
        // 'name/cos:PutBucketLifecycle',
        // 'name/cos:GetBucketLifecycle',
        // 'name/cos:DeleteBucketLifecycle',
        // // Replication
        // 'name/cos:PutBucketReplication',
        // 'name/cos:GetBucketReplication',
        // 'name/cos:DeleteBucketReplication',
        // // 删除文件
        'name/cos:DeleteMultipleObject',
        'name/cos:DeleteObject',
        // 简单文件操作
        'name/cos:PutObject',
        'name/cos:PostObject',
        'name/cos:AppendObject',
        'name/cos:GetObject',
        'name/cos:HeadObject',
        'name/cos:OptionsObject',
        'name/cos:PutObjectCopy',
        'name/cos:PostObjectRestore',
        // 分片上传操作
        'name/cos:InitiateMultipartUpload',
        'name/cos:ListMultipartUploads',
        'name/cos:ListParts',
        'name/cos:UploadPart',
        'name/cos:CompleteMultipartUpload',
        'name/cos:AbortMultipartUpload',
      ],
      effect: 'allow',
      principal: { qcs: ['*'] },
      resource: [
        `qcs::cos:${config.Region}:uid/${AppId}:prefix//${AppId}/${ShortBucketName}/`,
        `qcs::cos:${config.Region}:uid/${AppId}:prefix//${AppId}/${ShortBucketName}/${config.AllowPrefix}`,
      ],
    }],
  };

  const policyStr = JSON.stringify(policy);

  // 有效时间小于 30 秒就重新获取临时密钥，否则使用缓存的临时密钥
  if (tempKeysCache.expiredTime - Date.now() / 1000 > 30 && tempKeysCache.policyStr === policyStr) {
    callback(null, tempKeysCache);
    return;
  }

  const Action = 'GetFederationToken';
  const Nonce = util.getRandom(10000, 20000);
  const Timestamp = parseInt(+new Date() / 1000);
  const Method = 'POST';

  const params = {
    Region: 'gz',
    SecretId: config.SecretId,
    Timestamp,
    Nonce,
    Action,
    durationSeconds: 7200,
    name: 'cos',
    policy: encodeURIComponent(policyStr).replace(/\*/g, '%2A'),
  };
  params.Signature = util.getSignature(params, config.SecretKey, Method);

  const opt = {
    method: Method,
    url: config.Url,
    rejectUnauthorized: false,
    json: true,
    form: params,
    headers: {
      Host: config.Domain,
    },
    proxy: config.Proxy || '',
  };
  request(opt, (err, response, body) => {
    if (body && body.data) body = body.data;
    tempKeysCache.credentials = body.credentials;
    tempKeysCache.expiredTime = body.expiredTime;
    tempKeysCache.policyStr = policyStr;
    callback(err, body);
  });
};

module.exports = {
  getTempKeys,
};
```

跟上面一样，你还是要改AllowPrefix，和policy['statement']['action']


然后做route:

```
router.all('*', (req, res, next) => {
  res.header('Content-Type', 'application/json');
  res.header('Access-Control-Allow-Origin', 'http://127.0.0.1');
  res.header('Access-Control-Allow-Headers', 'origin,accept,content-type');
  if (req.method.toUpperCase() === 'OPTIONS') {
    res.end();
  } else {
    next();
  }
});

router.all('/sts-auth', (req, res, next) => {
  // 获取临时密钥，计算签名
  getTempKeys((err, tempKeys) => {
    var err = null;
    let data;
    if (err) {
      data = err;
    } else {
      const pathname = req.body.pathname || req.query.pathname || '';
      const Key = pathname.substr(0, 1) === '/' ? pathname.substr(1) : pathname;
      const opt = {
        SecretId: tempKeys.credentials.tmpSecretId,
        SecretKey: tempKeys.credentials.tmpSecretKey,
        Method: req.body.method || req.query.method,
        Key,
        Query: req.body.query || req.query.query || {},
        Headers: req.body.headers || req.query.headers || {},
      };
      data = {
        Authorization: COS.getAuthorization(opt),
        XCosSecurityToken: tempKeys.credentials && tempKeys.credentials.sessionToken,
      };
    }
    res.send(err || data);
  });
});
```

现在你访问你 https://your-api.com/sts-auth 就会得到 "Authorization" 和 "XCosSecurityToken"了。


然后现在我们回到 WX COS SDK github的[demo](https://github.com/tencentyun/cos-wx-sdk-v5/blob/master/demo/demo-no-sdk.js)

复制粘贴进你的小程序就好了，当然你需要根据你的业务做一些refactor。在uploadFile这个function里有个key，如果你在后端规定了AllowPrefix，记得在前面加上路径