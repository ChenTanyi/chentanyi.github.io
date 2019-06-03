---
title: 使用 Hexo + Next 构建 Blog
date: 2019-06-02 23:08:50
tags: sample
categories: nothing
---
# 这是一个测试 Blog，介绍如何构建此 Blog

## Hexo + Next

### 介绍

* [Hexo](https://hexo.io) 是一个 blog 框架
* [Next](https://github.com/theme-next/hexo-theme-next) 是 hexo 的一个主题
> [使用文档](https://theme-next.iissnan.com/)

### 安装 hexo

执行以下命令（把 &lt;repo&gt; 换成自己的 repo）

```bash
npm install -g hexo-cli
hexo init <repo>
cd <repo>
```

### 修改 theme

```bash
git clone https://github.com/theme-next/hexo-theme-next theme/next
```

* 修改 _config.yml 中的 theme 为 next（在 _config.yml 和 theme/next/_config.yml 有大部分的配置可以修改）
* 可以使用 `hexo s` 进行测试，在 http://localhost:4000/ 查看效果

## Push to Github.io

* 在 github 创建 &lt;username&gt;.github.io，在 settings 中配好 github page，并配好 [ssh](https://help.github.com/en/articles/connecting-to-github-with-ssh)

* 修改 _config.yml (repository 可以在 github repo 的 clone 中获得)
```yaml
deploy:
  type: git
  repository: git@github.com:<username>/<uesrname>.github.io.git
  branch: master
```

* Push 生成的页面到 github
```bash
hexo g
hexo d
```

* 在 &lt;username&gt;.github.io 中查看

> 可以把上述过程放到 ci 中生成 travis-ci 或 appveyor，只需要把 ssh 换成 github token 即可，不过由于每次都要进行测试，因为我不采用 ci 方式部署