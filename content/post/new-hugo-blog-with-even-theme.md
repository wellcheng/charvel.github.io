---
title: "使用 Hugo 重新驱动博客系统"
date: 2022-10-07T19:06:01+08:00

---

Hugo 是开源的流行 GitHub Pages 博客搭建服务 https://gohugo.io/getting-started/quick-start/

```Bash
brew install hugo

hugo new site charvel-blog

cd charvel-blog

git init



git submodule add https://github.com/olOwOlo/hugo-theme-even themes/even



echo theme = \"even\" >> config.toml



hugo new posts/new-hugo-blog-with-even-theme.md
```

