---
title: Oauth2.0
date: 2020-04-30 16:48:37
tags: Oauth2.0
declare: true
---
Oauth（Open Authorization）是一种授权机制，核心是通过颁发令牌给第三方应用，使其可以携带令牌访问数据。

### 令牌与密码的差异

+ 令牌是短期的，**到期后会失效**，密码一般是长期的，用户不主动修改，不会改变。
+ 令牌可被颁发者主动撤销，而密码一般不可由他人控制。
+ 令牌的权限范围一般比密码小。

<!-- more -->

### 基本概念

+ Third-party application：第三方应用程序，例如微信网站应用。
+ HTTP service：HTTP服务提供商，例如微信。
+ Resource Owner：资源所有者，例如微信用户。
+ User Agent：用户代理，浏览器。
+ Authorization server：认证服务器，即服务提供商专门用来处理认证的服务器。
+ Resource server：资源服务器，即服务提供商存放用户生成的资源的服务器。它与认证服务器，可以是同一台服务器，也可以是不同的服务器。

### 4种授权方式

+ 授权码模式（authorization code）

  最安全的一种方式，适用于有后端的web应用

  1. 资源所有者访问第三方应用，后者将前者导向认证服务器。
  2. 资源所有者选择是否给予第三方应用授权。
  3. 假设资源所有者给予授权，认证服务器将资源所有者导向第三方应用事先指定的"重定向URI"，同时附上一个授权码。
  4. 第三方应用收到授权码，向认证服务器申请令牌。这一步是在第三方应用的后台的服务器上完成的，对资源所有者不可见。
  5. 认证服务器核对了授权码和重定向URI，确认无误后，向第三方应用发送访问令牌（access token）和更新令牌（refresh token）。

+ 隐藏（简化）模式（implicit）

  适用于纯前端应用
  
  1. 客户端将用户导向认证服务器。
  2. 用户决定是否给于客户端授权。
  3. 假设用户给予授权，认证服务器将用户导向客户端指定的"重定向URI"，并带上访问令牌，token的位置是url锚点

+ 密码模式（resource owner password credentials）

  适用于高度信任的应用

  1. 资源拥有者向第三方应用提供用户名和密码。
  2. 第三方应用将用户名和密码发给认证服务器，向后者请求令牌。
  3. 认证服务器确认无误后，向客户端提供访问令牌。

+ 客户端模式（client credentials）

  适用于没有前端的应用

  1. 第三方应用向认证服务器进行身份认证，并要求一个访问令牌。
  2. 认证服务器确认无误后，直接返回访问令牌。




本文由阮一峰3篇oauth2.0文章总结而来。

http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html

http://www.ruanyifeng.com/blog/2019/04/oauth-grant-types.html

http://www.ruanyifeng.com/blog/2019/04/oauth_design.html