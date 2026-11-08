# Dynamder's Blog

Hugo + PaperMod 的个人技术博客，GitHub Actions 构建，GitHub Pages 托管。

## 本地开发

仓库里没有提交 Hugo 二进制（`.tools/` 已 gitignore）。本地跑之前先把 Hugo 装到 `.tools/`，
或者自己 `winget install Hugo.Hugo.Extended` 装到系统里。

```bash
# 方式一：用仓库内的本地副本（没装到 PATH）
export HUGO_CACHEDIR="$PWD/.cache/hugo"   # 把模块缓存留在仓库内，免得写沙箱外的目录
./.tools/hugo/hugo.exe server -D          # http://localhost:1313

# 方式二：已装到 PATH
hugo server -D
```

只构建不预览：

```bash
./.tools/hugo/hugo.exe --gc --minify       # 产物在 public/
```

## 写一篇新文章

```bash
./.tools/hugo/hugo.exe new content posts/my-post.md
```

模板在 `archetypes/default.md`，生成的草稿是 `draft = true`，本地预览加 `-D` 能看到，
**发布前要改成 `draft = false`**（workflow 里没开 `--buildDrafts`）。

正文里的 `summary` 建议手写：列表页和搜索结果都用它，不写就自动截取开头。

## 发布

推到 `main` 即可。

```bash
git add -A && git commit -m "post: something" && git push
```

`.github/workflows/hugo.yaml` 会装 Go + Hugo(extended) → `hugo --minify` → 上传 artifact →
`deploy-pages`。不需要 `gh-pages` 分支，也不需要提交 `public/`。

Pages 的 Source 要在仓库 **Settings → Pages → Build and deployment** 里设为
**GitHub Actions**（不是 Deploy from a branch）。

## 目录

```text
hugo.toml                 # 所有配置：站点信息、菜单、主题参数、搜索
go.mod / go.sum           # 主题依赖（Hugo Modules）
archetypes/default.md     # 新文章模板
content/
  posts/                  # 文章
  about.md  archives.md  search.md
assets/                   # 自定义 CSS/JS，同名文件会覆盖主题的
layouts/                  # 覆盖主题模板
static/                   # 原样拷贝（favicon、图片等）
.github/workflows/hugo.yaml
```

## 评论（Giscus）

已经接好了，基于 GitHub Discussions，零后端。代码在
[layouts/_partials/comments.html](layouts/_partials/comments.html)（覆盖主题的空钩子），
配置在 `hugo.toml` 的 `[params.giscus]`。

**上线前还有一步：安装 giscus GitHub App**，否则 iframe 里会显示
"giscus is not installed on this repository"。

👉 https://github.com/apps/giscus/installations/new

选 `dynamder.github.io` 这个仓库即可。App 装在账号上时也可以只授权这一个仓库。

其他要点：

- 仓库必须是 **public**，且 **Discussions 已开启**（已通过 API 打开）
- `repoId` / `categoryId` 是仓库绑定的，换仓库要重新取：
  ```bash
  gh api repos/<owner>/<repo> --jq .node_id
  gh api graphql -f query='query{repository(owner:"<owner>",name:"<repo>"){discussionCategories(first:20){nodes{id name}}}}'
  ```
- 评论楼按 `mapping = "pathname"` 映射到讨论，**改标题不会断，改 URL 会断**
- 关闭某篇文章的评论：front matter 里加 `comments = false`
- 主题跟着 PaperMod 的明暗切换自动变（`html[data-theme]` + postMessage）

## 几个必须知道的点


**1. baseURL。** 现在是 `https://dynamder.github.io/`，对应仓库名 `dynamder.github.io`。
如果改成项目站（`https://<user>.github.io/<repo>/`），`baseURL` 得带上仓库名，否则 CSS 404。

**2. 主题走 Hugo Modules，不是 themes/ 目录。** 升级主题：

```bash
export HUGO_MODULE_PROXY=https://goproxy.cn
./.tools/hugo/hugo.exe mod get -u github.com/adityatelange/hugo-PaperMod
./.tools/hugo/hugo.exe mod tidy
```

**3. 国内网络。** `github.com` 常连不上，所以 `hugo.toml` 里写死了
`module.proxy = "https://goproxy.cn,direct"`。Hugo 的 `module.proxy` 默认是 `direct`，
它会绕过 `GOPROXY` 自己去 clone，只设环境变量没用。

**4. 中文 i18n。** `defaultContentLanguage = "zh"`（不是 `zh-cn`）。
主题只提供 `i18n/zh.yaml`，写成 `zh-cn` 会命中不了、回落到英文（"2 min" / "Table of Contents"）。

**5. draft 与日期。** `buildFuture = false`，所以 front matter 的 `date` 写成未来时间，
文章会直接不生成。

**6. workflow 里的上下文限制。** `${{ runner.temp }}` 这类 `runner` 上下文**只能在 step 级别**
用；写进作业级 `env` 不会报语法错，而是让整个 run 变成 `startup_failure`——0 个 job、
0 秒结束、日志里什么都没有。排查时先看 `gh run view <id> --json jobs`，`total_count=0` 就是这个毛病。

## 上线状态

- 站点：https://dynamder.github.io/
- Pages 源：GitHub Actions（`gh api -X PUT .../pages -f build_type=workflow` 改的，
  新建仓库默认是 `legacy`，按分支发布，会把 Actions 部署拒掉）

## 待办

- [x] 评论：Giscus
- [ ] 安装 giscus App（见上文，唯一还没做的一步）
- [ ] 统计：Cloudflare Web Analytics / 自建 Umami
- [ ] 自定义域名 + CNAME（放 `static/`，别放根目录，会被构建覆盖）
- [ ] favicon 和 OG 图
