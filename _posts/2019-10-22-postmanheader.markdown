---
layout: post
category: "问题排查"
title:  "Postman 请求跨域问题"
tags: [Postman]
---

* content
{:toc}

![90-800x300.jpg](https://i.loli.net/2019/10/22/FnbuQgsiy9exDXo.jpg)

在调试调用公共网关服务时出现了很奇怪的现象，用 Postman 请求接口会提示跨域。
```
Invalid CORS request
```
而将请求 cURL code 原样复制出来使用 curl 命令请求则会正常返回。





在网关服务方开发人员协助下找到了问题，用 Postman 请求时会添加 Header。
```json
{
    "Origin": "chrome-extension //aicmkgpgakddgnaphhhpliifpcfhicfo"
}
```
这个其实并非 Postman 的问题，我所使用的 Postman 是 Chrome 的插件版本，在发送请求时会添加`chrome-extension`的 origin。
解决方式有两种：
- 将 Postman 更换为桌面应用版本。
- 在请求 Header 中将 Origin 设置为自己的 host。
