+++
title = '第一篇：这套博客是怎么搭起来的'
date = '2026-11-08T21:30:00+08:00'
draft = false
summary = 'Hugo + Hugo Modules 管主题 + GitHub Actions 构建、GitHub Pages 托管，顺便记一下为什么这么选、以后怎么加文章。'
tags = ['hugo', 'github-pages', '博客']
categories = ['建站']
+++

## 技术栈

| 环节 | 用的什么 |
| --- | --- |
| 站点生成 | Hugo（Go，单二进制，无需运行时） |
| 主题 | PaperMod，通过 Hugo Modules 引入 |
| 构建与部署 | GitHub Actions |
| 托管 | GitHub Pages |
| 写作 | Markdown |

选 Hugo 的理由很简单：构建快到几乎没有感知，而且扩展版是单个 `hugo.exe`，不用在 Windows 上折腾 Ruby / Node 环境。主题走 Hugo Modules，本质上就是用 Go modules 管依赖，升级和回滚都很干脆。

## 仓库结构

```text
.
├── hugo.toml              # 全部站点配置
├── go.mod / go.sum        # 主题依赖（Hugo Modules）
├── archetypes/default.md  # hugo new 生成文章的模板
├── content/
│   ├── posts/             # 文章
│   ├── about.md
│   ├── archives.md
│   └── search.md
├── assets/                # 自定义 CSS/JS（覆盖主题用）
├── layouts/               # 覆盖主题模板
├── static/                # 原样拷贝的文件（favicon、图片）
└── .github/workflows/hugo.yaml
```

## 怎么加一篇文章

```bash
hugo new content posts/my-new-post.md
```

生成的草稿默认 `draft = true`，写完把 `draft` 改成 `false`，或者本地预览时用 `hugo server -D`。

本地预览：

```bash
hugo server -D          # http://localhost:1313
hugo --gc --minify      # 只构建，产物在 public/
```

## 发布流程

推到 `main` 分支就完事了。workflow 会：

1. 装 Go 和 Hugo（extended）
2. `hugo --minify` 构建
3. `upload-pages-artifact` 打包
4. `deploy-pages` 发布到 Pages

不需要手动构建、不需要 `gh-pages` 分支、也不需要提交 `public/`。

## 两个已经埋好的坑

**一是 baseURL。** 用户站的地址是 `https://<用户名>.github.io/`，末尾斜杠不能少。如果以后改成项目站（`https://<用户名>.github.io/<仓库名>/`），`baseURL` 必须带上仓库名，否则 CSS 会 404。

**二是主题下载。** 国内直连 `github.com` 经常不通，而 Hugo 的 `module.proxy` 默认是 `direct`，会绕过 `GOPROXY` 自己去 clone。所以 `hugo.toml` 里显式写死了 `proxy = "https://goproxy.cn,direct"`。

## 下一步想加的

- [ ] 评论：大概率用 Giscus（基于 GitHub Discussions，零后端）
- [ ] 统计：自建 Umami 或直接用 Cloudflare Web Analytics
- [ ] 自定义域名 + Cloudflare 代理
- [ ] 把旧文从别处搬过来
