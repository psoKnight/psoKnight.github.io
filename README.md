# psoKnight

> Royal War — the wild sheep Knight.

[psoKnight.github.io](https://psoKnight.github.io) 的源码 —— 一个基于 Jekyll 的个人技术博客,记录后端、数据与 AI 工程实践,以深入源码的中文技术长文为主。

## 本地运行

需要 Ruby 与 Jekyll。在仓库根目录:

```bash
bundle exec jekyll serve   # 或 jekyll serve
```

浏览器打开 <http://localhost:4000> 预览。分页依赖 `jekyll-paginate`,如未安装先 `gem install jekyll-paginate`。

## 写文章

在 `_posts/` 下新建 Markdown,文件名格式 `YYYY-MM-DD-标题.md`,头部写 front matter:

```yaml
---
layout: post
title: '文章标题'
subtitle: '副标题'
date: 2026-07-09 10:00:00 +0800
author: Gonzo Sun
tags: [标签一, 标签二]
---
```

`date` 决定文章在首页、标签页、时间线中的排序;`tags` 决定它出现在哪些标签分组下。

## 目录结构

- `_posts/` —— 博客文章(Markdown)
- `_layouts/`、`_includes/` —— 页面骨架与可复用片段
- `assets/` —— 样式、脚本、图片
- `tags.html`、`timeline.html`、`about.html` —— 标签、时间线、关于页
- `_config.yml` —— 站点配置(标题 / 导航 / 分页 / 评论 / 统计等)

## 技术栈

Jekyll · kramdown(GFM)· Rouge 代码高亮(深空黑玻璃配色)· 来必力评论 · GitHub Pages 托管。

## 致谢与许可

页面框架改自 [HardCandy-Jekyll](https://github.com/xukimseven/HardCandy-Jekyll)(MIT),在其之上定制了深空黑玻璃主题与部分页面逻辑。本仓库沿用 [MIT License](https://github.com/xukimseven/HardCandy-Jekyll/blob/master/LICENSE);文章内容版权归作者所有。
