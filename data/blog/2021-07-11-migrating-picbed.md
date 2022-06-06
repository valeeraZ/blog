---
title: 图床迁移记
summary: 将markdown文章中的图床迁移到Azure DevOps Git
date: 2021-07-11
tags: ['Installation']
---

在用MarkDown写文章的时候，如果需要上传图片往往需要一些在线图床服务。但如果某天在线图床停止服务，那自己markdown文章里的图片也就会无法查看。考虑到这一点，我开始搜索自建图床或使用某些Git服务的办法，以及如何将散落在文章各处的图片链接全部迁移过去的方法。

# 过去使用的服务

我通常[用PicGo Core搭配Typora写文章传图片](https://sowevo.com/PicGo-Core%E6%90%AD%E9%85%8DTypora%E8%87%AA%E5%8A%A8%E6%8F%92%E5%85%A5%E5%9B%BE%E7%89%87/)。如果你没用过，可以看一下[PicGo GUI的使用说明](https://picgo.github.io/PicGo-Doc/)以及[PicGo Core CLI版](https://picgo.github.io/PicGo-Core-Doc/)的。

一开始写文章的时候，用的是[sm.ms](https://sm.ms/)这个免费的图床网站。免费版的虽好，但是却有很多限制，例如存储空间有限，上传图片的大小受限制，无法保证CDN质量等等。最重要的一点是，你无法保证一个在线服务永久的有效性。

后来开始使用GitHub的服务，。但是，每个GitHub仓库的建议大小仅为1G，无法保证在国内时访问的CDN，而且存放图片的仓库需要设置为public，很不美观……

但除了GitHub还有其他Git服务商，在一番搜寻之后找到了[Azure DevOps](https://dev.azure.com/)。它允许单个仓库有10G的容量，并且我相信我一时半会还不会需要使用这个Git服务用来工作……

那么问题来了，如何快速地把图片上传到Azure Git，以及如何批量将博客中文章的图片转移到Azure？

#  使用picgo-plugin-azure

尽管PicGo官方还未支持Azure图床，但已经有了一款个人开发的[picgo-plugin-azure](https://www.kikt.top/posts/other/upload-image-to-azure/)插件允许你像使用GitHub那样用picgo上传图片。对于Azure API的分析，可以阅读这位开发者所写的[文章](https://www.kikt.top/posts/other/upload-image-to-azure/)。

在注册Azure账号并且成功新建项目后，在Repo中请先提交一次commit（可以直接点击右下角的Initializa Repository with README按钮）（详情见这个我提的[issue](https://github.com/CaiJingLong/picgo-plugin-azure/issues/7)）。阅读这个插件的[README](https://github.com/CaiJingLong/picgo-plugin-azure/blob/master/README.md)，得到token并且通过picgo设置后，就可以用CLI或GUI上传本地图片了。

# 迁移图床

 在上一步将azure设置为默认图床后，就可以使用[picgo-plugin-pic-migrater](picgo-plugin-pic-migrater)来配合迁移图床。在使用的过程中，遇到了一些bug……

阅读[PicGo Core插件开发指南](https://picgo.github.io/PicGo-Core-Doc/zh/dev-guide/cli.html#%E7%AE%80%E4%BB%8B)，发现他们在[transformer的生命周期](https://picgo.github.io/PicGo-Core-Doc/zh/dev-guide/cli.html#transformer)中使用了两种格式：base64和path。但是：

- picgo-plugin-azure使用的是path，也就是图片在本地的路径
- picgo-plugin-pic-migrater使用的是base64，也就是图片的buffer

或许picgo-plugin-azure的开发者为了方便起见没有考虑到两种格式的兼容性。为了解决这个“bug”，我fork了一份并且提交了[PR](https://github.com/CaiJingLong/picgo-plugin-azure/pull/8)。如果你想使用我修改后的版本，以便配合picgo-plugin-pic-migrater，那么可以clone[我的代码](https://github.com/valeeraZ/picgo-plugin-azure)然后[文档](https://picgo.github.io/PicGo-Core-Doc/zh/dev-guide/deploy.html#%E6%8F%92%E4%BB%B6%E6%B5%8B%E8%AF%95)使用我修改后的插件。



祝各位写博客愉快！