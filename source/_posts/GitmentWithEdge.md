---
title: Edge浏览器下启用全功能Gitment
date: 2017-08-24 11:53:38
categories: 博客相关
tags:
    - Edge
    - Gitment
---
[Gitment](https://github.com/imsun/gitment)评论系统是一款基于 GitHub Issues 的评论系统，适合作为软件开发技术类博客作为评论系统使用，同时支持Hexo等博客框架。

因此在Hexo框架的博客中，使用Gitment搭建评论系统是一个不错的选择。但是其对浏览器兼容性不够强，根据[Issue #50](https://github.com/imsun/gitment/issues/50)中的描述：
> - Chrome通过
> - Firefox 能正常显示评论，但login时出现[object ProgressEvent]
> - Edge 只显示评论，无法进行评论
> - IE 11完全不显示

<!-- more -->

因此需要寻找提高Gitment兼容性的办法。作者[GeekaholicLin](https://github.com/GeekaholicLin) 在其发起的[Pull request #52](https://github.com/imsun/gitment/pull/52) 中提供了两个修改过的`gitment.browser.js`文件，地址为：
- http://blog.geekaholic.cn/js/thirdParty/gitment.browser.js
- http://blog.geekaholic.cn/js/thirdParty/gitment.browser.min.js

将这两个文件替换已经部署好的网站中的两个`gitment.browser`文件，即可实现在Edge浏览器中启用Gitment全功能。

效果图：
{% asset_img demo.jpg %}

> 经过测试，直接替换 node_modules/gitment 中的文件是没有效果的。目前只能在生成之后手动替换 gitment.browser 文件，或者等待作者更新。