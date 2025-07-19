---
title: 'Istio Ingress Gateway & Virtual Service & Authentication 相关blog'
date: '2025-07-13'
lastmod: '2025-07-13'
tags: ['Infra']
draft: false
images: ['/static/images/istio_k8s.png']
layout: PostLayout
summary: 'Some useful blog posts about Istio Ingress Gateway & Virtual Service & Authenticaiton.'
---

## Ingress Gateway and Ingress Controler

在 Kubernetes 中，Ingress 是一种 API 对象，用于管理外部访问集群服务的方式。它允许用户定义路由规则，将外部 HTTP/S 流量路由到集群内的服务。Kubernetes Ingress Controller 是实现这些规则的组件。
Istio Ingress Gateway 是 Istio 服务网格中的一个组件，专门用于处理进出服务网格的流量。它提供了更高级的流量管理功能，如负载均衡、路由、故障恢复等。

### 相关文档和博客

- [配置 Ingress Gateway，和 Ingress Controller 的区别, Istio 基本能力](https://www.cnblogs.com/ZhuChangwu/p/16464316.html#82%E4%BD%BF%E7%94%A8%E5%9F%9F%E5%90%8D%E7%9A%84%E6%96%B9%E5%BC%8F%E5%8F%91%E5%B8%83https://www.jianshu.com/p/0f8c6b1d2a3e)

- [Ingress Gateway HTTP 重定向到 HTTPS](https://www.treesir.pub/post/istio-in-action-study/#http-%E9%87%8D%E5%AE%9A%E5%90%91%E5%88%B0-https)

- [Mergeable Ingress Gateway: 同域名多 Ingress 资源](https://medium.com/@kedarnath93/ingress-in-kubernetes-with-nginx-ed31607fa339)

- [Mergeable Ingress Types Support](https://github.com/wallarm/ingress-plus/blob/wallarm/examples/mergeable-ingress-types/README.mdl)

## Virtual Service

Virtual Service 是 Istio 中的一个重要概念，用于定义服务之间的路由规则。它允许用户指定如何将流量路由到不同版本的服务，支持基于请求头、路径等条件进行路由。

### 相关文档和博客

- [Istio Virtual Service](https://www.cnblogs.com/zhangmingcheng/p/15717351.html)

- [Istio 模式：基于头部/cookie 的 Kubernetes 流量拆分](https://www.itwonderlab.com/istio-patterns-traffic-splitting-in-kubernetes-header-based/)

## Request Authentication & Authorization Policy

Istio 提供了强大的身份验证和授权功能，可以对进入服务网格的请求进行验证和授权。Request Authentication 用于验证请求的身份，而 Authorization Policy 则用于定义访问控制规则。

### 相关文档和博客

- [Request Authentication & Authorization Policy](https://www.cnblogs.com/whuanle/p/17538694.html)

- [Request Authentication 返回 Jwks doesn't have key to match kid or alg from Jwt](https://github.com/istio/istio/issues/37710)
