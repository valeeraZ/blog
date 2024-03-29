---
title: '2021年，我如何做网页'
date: '2021-11-04'
lastmod: '2021-11-04'
tags: ['Web']
draft: false
summary: 'How do I make websites in 2021'
---

之前和同学聊天，问我本科是不是学的做网站，本人好歹是全球top一万名大学的研究生，怎么本科就学了个做网站？

后来想想好像说的没错，我拿的出手的也就是“做网站”了。这里我用“做网站”来指代前端技术，当然本文中也会不可避免的提到部分后端技术。

# 起步

那肯定是先学习HTML和CSS。不过CSS我学的着实不行，导致经常还要csdn搜索“如何让div水平垂直居中“；经常padding和margin忘记都是什么，还得F12看图例。因为过早地接触了Bootstrap这类UI库产生了依赖，自己也就没有太注意学设计了。

# 动起来

除了简单的after，before effect，要想实现元素的交互那肯定得写JavaScript。但JavaScript我本人也从未花一天时间去找什么廖XX教程按部就班地学，只是大致了解了一些例如`document.getelementbyid` 这类的用法。不过19年却认真地学了jQuery，但它只是一个“简单的”JavaScript框架，帮助我们省去了许多写原生JS代码的时间。

一个网站的不同子页难免会有重复的组件，例如导航栏，页脚这些不变的组件，于是便需要将代码抽离出来重复利用。当时还没有接触到jQuery组件化这种开发模式，于是便写好每个组件的代码之后，在独立页面上将DOM渲染到对应`<div>`上。但许多问题还没有得到很好的解决，例如组件之间共享状态，网页跳转之间互相传递参数，有时便不得不写一些可读性很差的代码。

# 工程化

2020年开始接触React这类框架，高效灵活的设计极大的节约了开发时间，并使得测试变得简单。我们可以轻松制定事件发生后的render行为，更可以定义state的变化，以便更好地制作动态web应用。配合React，也开始使用Material UI和Ant UI这类较新的UI库。

自身的学习问题也暴露了出来：由于没有系统学习JS以及ES 6，在阅读一些网上的代码时往往需要大量地搜索学习才能明白；自己做的项目往往没有好的规划，导致前期写的问题在后期才发现并且难以解决，有时甚至需要重写。React自身并不是一个完整的框架，需要配合Redux和React Router这类需要花时间学习的内容。

# 基础

认识到自身的不足后，2021年我用了一个多月的时间去重新学习ES规范和Node.js。配合之前学过的Clojure，Haskell和OCaml这类声明式或函数式语言后，对JS的函数式开发有了更多的认识。在回顾基础的同时，发现了一些可用于生产环境的React前后端框架，以及使用Postgre SQL的supabase数据库。

# 更轻，更快

对于不同的需求，我们可以选择不同的渲染方案。对于静态站点选择使用SSG，能提高渲染速度并且更好地支持SEO；对动态站点使用SSR，解决长时间白屏问题。对于混合模式的开发，我开始学习并使用Next.js。动态路由，静态文件服务，后端API路由，中间件……这些功能都在Next.js上得到实现。

配合material UI，我同时也在使用Tailwind CSS。但我对UI的设计了解并不够深刻，目前仅处于使用Tailwind CSS来节省开发时间。

# 还需要学习……

客户的需求日新月异，功能的实现也在不断变化。比如我初学React时的一些生命周期函数，在不久后便被宣布过时不用；大多数新项目选择使用函数式组件，较多地使用hook以解决高阶组件和函数组件的嵌套过深。新的TypeScript已经得到广泛使用，减少代码错误。

前端开发人员的角色正日益向“全栈”转变，而工具和服务正变得足够复杂，以降低进入壁垒。而找到那个最适合需求的解决方案，是不变的目标。