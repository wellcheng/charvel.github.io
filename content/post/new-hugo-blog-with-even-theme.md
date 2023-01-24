---
title: "使用 Hugo 重新驱动博客系统"
date: 2022-10-07T19:06:01+08:00

---

Hugo 是开源的流行 GitHub Pages 博客搭建服务，节假日突然有空，想着把这一套东西收拾起来，顺便也体验一下 github 提供的 workflow 功能，让写 blog 更加舒服和快捷。

有关 hugo 的搭建可以直接看官方文档就好了，这里不再贴具体内容，因为随着时间的变化，内容可能会过期。

https://gohugo.io/getting-started/quick-start/


<!-- more -->

在部署 GitHub Actions 的过程中，发现了直接从 action market 中搜索的 hugo 并不能直接使用，在 deploy 阶段会报错。
于是降级使用了 hugo 官方的 Host on GitHub 教程：

https://gohugo.io/hosting-and-deployment/hosting-on-github/#build-hugo-with-github-action

大概看了一下，market 市场中的 hugo action 将整个部署过程分为了 build 和 deploy 两个阶段。
这样在出现错误时，可以单独 rebuild 每个 step，而不是整个重新来过。

使用 GitHub Actions 可以在线编辑 markdown 文件，提交后触发直接部署。

1、进入仓库

2、选择 post 目录 ： https://github.com/wellcheng/wellcheng.github.io/tree/main/content/post

3、新建 markdown 文件，格式为 2023-00-demo.md

4、编辑文本并提交 commit 即可。
