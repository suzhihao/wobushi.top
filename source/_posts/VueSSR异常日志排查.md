---
title: VueSSR异常日志排查
date: 2020-12-17 22:59:59
categories:
  - [后端, SSR]
tags:
  - Vue SSR
copyright:
  author: true
  link: true
  license: 未经允许请勿转载~
  published: true
  updated: true
---

### 场景复盘

近期在排查日志的时候发现服务端记录的请求日志中有大量的异常日志。具体表现为记录的 ip 为 127.0.0.1，一般情况下该字段记录的是用户访问的真实 ip，是从 header 中获取，继续查看 header 字段，发现了更奇怪的现象:

```json
{
  "accept": "application/json, text/plain, */*",
  "content-type": "application/json;charset=utf-8",
  "x-requested-with": "XMLHttpRequest",
  "396d5a0568f3d1b0bdbb": "02ffb05b42b76dd53c6d0cf576d189e05ba39a67304b4571bfb185d6ed4eb0600c5be0e7903742b08726197cd7c5cd8f896a3e6ac310974481288dcf81d0094b",
  "user-agent": "axios/0.17.1",
  "content-length": "65",
  "host": "127.0.0.1:9000",
  "connection": "keep-alive"
}
```

从 header 字段中进行分析：

【"accept":"application/json, text/plain, _/_"】

【"content-type":"application/json;charset=utf-8"】

【"x-requested-with":"XMLHttpRequest"】

【"396d5a0568f3d1b0bdbb":"02ffb05b42b76dd53c6d0cf576d189e05ba39a67304b4571bfb185d6ed4eb0600c5be0e7903742b08726197cd7c5cd8f896a3e6ac310974481288dcf81d0094b"】

以上 4 条记录显示这个请求是由 js 发起的 XHR 请求，正常情况下是由浏览器发起的，然而下面两个字段又推翻了这个猜想：

【"user-agent":"axios/0.17.1"】

【"host":"127.0.0.1:9000"】

如果是浏览器发起的请求，userAgent 应该显示为正常浏览器的 UA，且 host 应该为网站的域名。而现在我们看到的值更像是由服务端脚本请求记录的值。且如果从浏览器端发起的请求，经过代理转发后会带上特定的 header，然而这里却没有，说明请求时直接从服务器发起请求 127.0.0.1:9000 导致。

那又是什么情况会导致这种异常日志的出现呢？

#### 猜想一：服务端日志记录的逻辑存在 bug，将原本不需要记录的日志打到了 ELK

由于出现异常的项目是 SSR 同构的项目，当页面在服务端渲染时接口请求会由服务器请求到自己 127.0.0.1:9000 的情况，参考代码如下：

```js
if (!__BROWSER__) {
  let apiRoot = `http://127.0.0.1:9000`;
  createRequest = (req) => {
    axios.create({
      baseURL: apiRoot,
    });
  };
}
```

但正常情况下这类日志并不会记录到 ELK 中去，排查代码之后确认这里的逻辑并没有问题，可以排除这种情况。

#### 猜想二：服务存在 SSRF 漏洞

SSRF (Server-Side Request Forgery: 服务器端请求伪造) 是一种由攻击者构造形成由服务端发起请求的一个安全漏洞。一般情况下，SSRF 攻击的目标是从外网无法访问的内部系统。

有时候我们为了解决一些跨域问题会在服务端添加请求转发的接口，如果没有做好白名单限制就有可能被黑客利用从而产生 SSRF 攻击。日志中显示的情况很可能是由暴露在外网的转发接口转发异常请求导致的。

然而排查项目中的关键词查看是否存在这类接口后发现也没有问题。这样就可以基本排除 SSRF 漏洞导致的问题。但由于服务是发布在物理机上的，==机器上还有运行其他服务，不排除其他服务存在 SSRF 漏洞导致的情况。==

#### 猜想三：服务器被黑客入侵并植入执行脚本

有那么一瞬间怀疑服务器被入侵，但仔细观察 header 中的【"396d5a0568f3d1b0bdbb":"02ffb05b42b76dd53c6d0cf576d189e05ba39a67304b4571bfb185d6ed4eb0600c5be0e7903742b08726197cd7c5cd8f896a3e6ac310974481288dcf81d0094b"】请求头，这个请求头是由我们的程序生成的加密 token，黑客植入的脚本大概率不会去做这件事情。这种情况的可能性也几乎为 0。

#### 真相只有一个：

继续排查异常日志里面的其他字段，发现出现异常的接口基本上都是 /api/aaa/bbb 这一个接口，那我们只要分析这个接口的逻辑即可，搜索相关代码后终于发现出现问题的逻辑（代码经过简化处理）：

```js
// src\xxx\component.js
created() {
   service.getData({ xxx: xxx }, {})
}
// src\services\xxx.js
let getData = (params, req) => {
  return createRequest(req).post('/api/aaa/bbb', params)
}
```

问题就出现在第二个参数 req 上，这个参数原本设计的含义是让服务端请求自己的接口时能携带上用户请求的上下文信息。

当 created 钩子在服务端执行的时候，由于开发同学强制将 req 设置成了空对象，导致浏览器请求的上下文全部丢失，这个时候服务器请求自己的时候就无法判断这是一次来自自己的请求，丢失了相应的标记，导致日志被额外记录了下来，最终形成了我们看到的样子。

### 原因梳理

这个问题毫无疑问是因为开发者对框架不熟悉导致的，开发需要对 Vue 同构 SSR 的设计模式熟悉，需要遵循固定的写法达到最优的性能表现。

同构项目的请求流程如下：

{% asset_img 请求流程图.png [请求流程图] %}

[Vue SSR 官方文档参考](https://ssr.vuejs.org/zh/)
