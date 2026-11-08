# Dynamder's Blog

个人技术博客的站点源码，线上地址 <https://dynamder.github.io/>。

内容围绕后端、Go 和基础设施，写作与发布都只依赖 Markdown 和一次 `git push`。

## 技术栈

| 环节 | 选型 | 说明 |
| --- | --- | --- |
| 生成器 | Hugo 0.169 extended | Go 编写的单二进制，无运行时依赖 |
| 主题 | PaperMod | 通过 Hugo Modules 引入，不在 `themes/` 下 |
| 构建 / 部署 | GitHub Actions | `main` 分支 push 触发 |
| 托管 | GitHub Pages | 用户站，挂在根域名下 |
| 评论 | Giscus | 基于 GitHub Discussions，零后端 |
| 搜索 | Fuse.js | 构建时输出 `index.json`，客户端检索 |

## 目录结构

```text
hugo.toml                  所有配置：站点信息、菜单、主题参数、评论、搜索
go.mod / go.sum            主题依赖（Hugo Modules 的 go module）
archetypes/default.md      hugo new 使用的文章模板
content/
  posts/                   文章；英文版同目录同名加 .en.md
  about.md                 关于页
  archives.md              归档页（layout: archives）
  search.md                搜索页（layout: search）
  *.en.md                  以上各页的英文版
layouts/_partials/
  comments.html            评论区，覆盖主题的空钩子
assets/                    自定义 CSS/JS，同名文件会覆盖主题资源
static/                    原样拷贝到站点根目录（favicon、CNAME、图片）
.github/workflows/hugo.yaml
```

`themes/` 目录没有被使用；主题由 `go.mod` 声明、Hugo 在构建时拉取。

## 环境准备

- Hugo **extended** ≥ 0.158（主题要求）
- Go ≥ 1.21（Hugo Modules 拉取主题需要）

Windows 上装 Hugo 用 `winget install Hugo.Hugo.Extended`，或把二进制解压到任意目录。

## 本地预览

```bash
hugo server -D            # http://localhost:1313，包含草稿
```

只构建不预览，产物在 `public/`：

```bash
hugo --gc --minify
```

模块缓存默认写在用户目录下；想让它留在仓库内可以设 `HUGO_CACHEDIR=$PWD/.cache/hugo`。

仓库内可能放有一份便携版 Hugo（`.tools/`，已 gitignore）。若 `hugo` 不在 PATH 里，
把上面的命令换成 `./.tools/hugo/hugo.exe`。

## 写作

```bash
hugo new content posts/my-post.md
```

生成的文件带 `draft = true`，本地预览加 `-D` 可见，**发布前改为 `draft = false`**
（构建流程没有开 `--buildDrafts`）。

front matter 可用字段：

| 字段 | 作用 |
| --- | --- |
| `title` | 标题 |
| `date` | 发布时间，同时决定归档排序；不要填未来时间（见「约束」） |
| `draft` | `true` 时只在 `-D` 预览中可见 |
| `summary` | 列表页摘要与搜索结果显示的内容，不写则自动截取正文开头 |
| `tags` / `categories` | 分类索引 |
| `comments` | 设为 `false` 可关掉本文评论 |
| `cover.image` | 封面图，放在文章同目录或 `static/` |

## 发布

提交并推送到 `main`：

```bash
git add -A && git commit -m "post: 标题" && git push
```

工作流依次执行：检出代码 → 装 Go 与 Hugo(extended) → `hugo --minify` →
上传 artifact → `deploy-pages`。不需要 `gh-pages` 分支，也不需要提交 `public/`。

仓库的 **Settings → Pages → Build and deployment → Source** 必须是 **GitHub Actions**。
新建仓库默认是 `Deploy from a branch`，用 Actions 部署会被拒绝，可以用下面的命令切换：

```bash
gh api -X PUT repos/<owner>/<repo>/pages -f build_type=workflow
```

## 配置

全部集中在 `hugo.toml`，常用的几项：

| 配置 | 说明 |
| --- | --- |
| `baseURL` | 站点绝对地址，**必须与仓库形态匹配** |
| `title` / `params.author` / `params.description` | 站点元信息，影响 SEO 与 OG 卡片 |
| `params.homeInfoParams` | 首页自我介绍 |
| `params.mainSections` | 首页列表展示哪些 section |
| `params.ShowToc` / `params.comments` 等 | 主题功能开关，见 PaperMod 文档 |
| `[[menu.main]]` | 顶部导航 |
| `[params.giscus]` | 评论区凭据 |
| `[params.fuseOpts]` | 搜索匹配策略 |

## 评论

Giscus 把每篇文章映射到一个 GitHub Discussion，`mapping = "pathname"` 表示按 URL 路径
建立映射，因此改标题不会断楼、改 URL 会断。

启用条件：

1. 仓库 **public**
2. Discussions **已开启**
3. 仓库上安装了 [giscus App](https://github.com/apps/giscus/installations/new)

缺第 3 条时，评论框会显示 "giscus is not installed on this repository"。

`repoId` 和 `categoryId` 与仓库绑定，换仓库需要重新获取后更新 `hugo.toml`：

```bash
gh api repos/<owner>/<repo> --jq .node_id
gh api graphql -f query='query{repository(owner:"<owner>",name:"<repo>"){discussionCategories(first:20){nodes{id name}}}}'
```

## 多语言

中文是默认语言，留在根路径；英文在 `/en/` 下。

| | 中文 | 英文 |
| --- | --- | --- |
| 内容文件 | `content/posts/foo.md` | `content/posts/foo.en.md` |
| URL | `/posts/foo/` | `/en/posts/foo/` |
| 语言码 | `zh` | `en` |

同一路径加 `.en.md` 后缀即自动配对成翻译，主题会据此生成 `hreflang`、文章 meta 下的
「Translations」互链，以及头部的语言切换器，不需要额外配置。

几点约定：

- **菜单按语言各写一份。** 根级 `[[menu.main]]` 只服务中文，英文用
  `[[languages.en.menu.main]]`；某语言一旦定义自己的菜单，根级菜单不会合并进去。
- **面向读者的 `params` 要在 `[languages.en.params]` 下重写。** 根 `[params]` 的值不会自动
  翻译，`homeInfoParams` 不重写就会出现中文首页文案配英文菜单。映射类型的 params
  （如 `[params.giscus]`）是按语言合并的，只需覆盖要改的字段。
- **标签是分语言的。** `/tags/` 和 `/en/tags/` 各自成页，所以中文用 `安全`、英文用
  `security` 不会互相污染。
- **`defaultContentLanguageInSubdir` 必须保持 `false`。** 改成 `true` 会让中文搬到
  `/zh/` 下，现有 Giscus 讨论楼按 pathname 建立，URL 一变就全部失联。
- 没有对应翻译的文章不会出现在另一种语言的列表、归档和搜索索引里。

英文 slug 想和中文不同时，文件名配对失效，需要在两份 front matter 里写相同的
`translationKey`：

```toml
translationKey = 'memory-social-engineer'
```

## 约束与易错点

**`baseURL` 必须与仓库名匹配。** 仓库叫 `<user>.github.io` 时站点在根路径，`baseURL` 是
`https://<user>.github.io/`；换成项目站（`https://<user>.github.io/<repo>/`）时 `baseURL`
必须带上仓库名，否则样式与脚本全部 404。

**文章 URL 由文件名决定，一旦发布就不要改名。** 默认 permalink 是 `/posts/<文件名>/`，
评论楼按路径映射，重命名会让旧讨论楼失效，外部链接也会断。要改结构应在发布前定好，
或改用 front matter 的 `url` 并做跳转。

**`module.proxy` 默认是 `direct`。** 它不会读 `GOPROXY` 环境变量，而是自己 clone 仓库，
所以在 `github.com` 不可达的网络下必须显式配置 `module.proxy = "https://goproxy.cn,direct"`。
升级主题：

```bash
hugo mod get -u github.com/adityatelange/hugo-PaperMod
hugo mod tidy
```

**语言码用 `zh` 而不是 `zh-cn`。** 主题只提供 `i18n/zh.yaml`，写成 `zh-cn` 命中不到翻译，
界面会回落到英文（`2 min`、`Table of Contents`）。

**`buildFuture = false`。** front matter 的 `date` 晚于构建时间时文章不会生成，且构建过程
不报错。

**工作流里的上下文作用域。** `runner`、`steps` 等上下文只能在 step 级别使用；写进作业级
`env` 不会产生语法错误，而是让整个 run 以 `startup_failure` 结束——0 个 job、0 秒、
没有日志可看。

**自定义域名。** `CNAME` 文件要放 `static/`，放仓库根目录会被构建产物覆盖。
两者不要同时配，Pages 设置里的 custom domain 会和 `CNAME` 互相影响。

## 主题定制

覆盖主题文件只需在仓库里放同路径文件，Hugo 会优先使用仓库内的版本：

- 改样式 → `assets/css/extended/custom.css`（PaperMod 约定，自动追加）
- 改模板 → `layouts/` 下同名文件
- 改文案 → `i18n/<lang>.yaml`

主题升级不会影响这些覆盖文件，但主题改动了被覆盖的模板时需要手动同步。
