# AGENTS.md

Palemoky 的个人中文技术博客，https://blog.palemoky.com 。基于 Hugo（extended）+ PaperMod 主题。

## 项目结构

- `hugo.toml`：站点唯一的配置文件。
- `content/posts/<slug>/index.md`：每篇文章是一个目录（page bundle），图片放在同目录的 `imgs/`，数据文件（如 keytable 的 yaml）也放在同目录。
- `content/` 下的 `about.md`、`resources.md`、`search.md`、`archives.md`：独立页面。
- 站点时区是 UTC（`timeZone = 'UTC'`），文章 `date` 里自带的 `+08:00` 时区照常生效。
- `archetypes/posts/`：新文章的目录模板。
- `layouts/`、`assets/css/extended/`：对主题的覆盖和扩展，见下文“修改主题”。
- `static/`：favicon 等静态文件。

## 新建文章

```bash
hugo new content posts/<slug>
```

- **路径只写到目录**，不要写成 `posts/<slug>/index.md`，否则不会使用 `archetypes/posts/` 这个目录模板，生成的文章缺少 `tags` 和 `imgs/`。
- `<slug>` 用小写英文加连字符，如 `distributed-order-number`。
- 新文章默认是 `draft: true`，发布前改为 `false`。

## Front matter

```yaml
---
title: 分布式系统中的订单号设计：唯一、可读、可路由
date: 2026-11-06T11:16:14+08:00
draft: false
tags: [Distributed Systems, Architecture, Database]
---
```

- **`date` 必须带完整时间和时区**（`YYYY-MM-DDTHH:MM:SS+08:00`）。只写日期时，同一天的文章会退回按标题排序，顺序不可控。
- **`date` 不能晚于当前时间**。Hugo 默认不构建未来日期的文章，文章会直接从站点消失。手动填写时间前先确认当前时间。
- 文章排序只靠 `date`，不要用 `weight` 调整顺序。系列文章按发布先后给 `date`，最新的排在最前面。
- 不要修改已发布文章的 `date`，除非明确要求。
- `tags` 优先复用已有标签，不要随意新建近义标签。现有标签：`工具`、`原理`、`学习`、`英语`、`随笔`、`Linux`、`Git`、`Emacs`、`Neovim`、`AI`、`Claude`、`Distributed Systems`、`Architecture`、`Database`、`Security`、`Docker`、`Redis`。查看最新列表：`grep -h '^tags:' content/posts/*/index.md`。
- 成系列的文章加 `series: [系列名]`，文末会自动列出同系列的所有文章（按 `date` 从早到晚排序），不需要手动维护链接。现有系列：`分布式系统设计`。标题里不加“（一）（二）”这类编号，每篇文章应能独立阅读。
- 新增分类法时，`hugo.toml` 的 `[taxonomies]` 要保留 `tag` 和 `category`。声明 `[taxonomies]` 会覆盖 Hugo 的默认值，漏写会导致标签页消失。

## 写作约定

- 正文用简体中文，引号用 `“”`。
- 文章之间互相链接用 `relref`，不要写死 URL：
  ```markdown
  [用户 UID 的设计]({{< relref "/posts/distributed-user-uid/index.md" >}})
  ```
- 图片用相对路径引用本目录下的 `imgs/`：`![说明](imgs/xxx.webp)`。不要写 HTML `<img>`，Hugo 默认会丢弃 Markdown 里的原始 HTML。
- **代码块必须标注语言**（` ```go `、` ```bash `、` ```text ` 等）。语言名会显示在代码块右上角，不标就不显示。
- 按键速查表用 `keytable` shortcode：`{{< keytable "xxx-keys.yaml" >}}`，yaml 放在文章目录里，数据格式见 `layouts/shortcodes/keytable.html` 顶部的注释。

### 数学公式

已启用 KaTeX，构建时在服务端渲染，页面不加载 JS，只在含公式的页面加载 CSS。

- 行内公式：`$...$` 或 `\(...\)`；行间公式：`$$...$$` 或 `\[...\]`。
- **正文里的普通美元符号要写成 `\$`**，否则会被当成公式。代码块和行内代码里的 `$` 不受影响。
- 只有真正的数学表达式才用 LaTeX（幂、科学计数法、概率估算、比例等）。普通数量（“1 亿单”“21 位”）保持纯文本。
- 公式语法错误会导致构建失败，报错里会给出位置。

## 修改主题

PaperMod 通过 Hugo Module 引入（见 `go.mod` 和 `hugo.toml` 的 `[module]`），源码在 Hugo 模块缓存里。**不要修改模块缓存中的主题文件**，所有定制都在项目里覆盖：

| 文件 | 作用 |
| --- | --- |
| `layouts/_markup/render-passthrough.html` | 用 `transform.ToMath` 渲染公式 |
| `layouts/_markup/render-codeblock.html` | 给代码块包一层带 `data-lang` 的容器 |
| `layouts/partials/extend_head.html` | 页面含公式时加载 KaTeX CSS |
| `layouts/partials/comments.html` | giscus 评论，并跟随站点明暗主题切换 |
| `layouts/partials/extend_post_content.html` | 文末的系列导航（样式见 `assets/css/extended/series.css`） |
| `layouts/shortcodes/keytable.html` | 按键速查表 |
| `assets/css/extended/*.css` | 自定义样式，PaperMod 会自动合并 |

- 想覆盖主题的某个模板或 partial，先在模块缓存里找到原文件，再复制到项目的同名路径修改。查看主题目录：`hugo config mounts | grep '"dir"'`（路径里含 `hugo-!paper!mod@...` 的那一行）。
- 代码高亮由 Hugo（Chroma）负责，`hugo.toml` 里已关闭主题自带的 highlight.js（`disableHLJS = true`），不要再引入其他高亮库。

## 验证

改完内容、配置或模板后，构建一遍确认没有报错：

```bash
hugo -d "$(mktemp -d)" --quiet
```

- 输出到临时目录，不要输出到 `public/`。
- 需要看效果时用 `hugo server`；看草稿加 `-D`。
- 涉及公式、代码块或排序时，到构建产物里检查实际渲染结果，例如首页的文章顺序、页面里是否有 `class="katex"`。

## Git

- 提交信息用 Conventional Commits 风格，英文小写开头，例如：
  - `docs: add post on Docker port bypassing ufw leading to Redis compromise`
  - `docs(redis-hacked): fix typo and drop redundant intro paragraph`
- 文章内容的改动用 `docs`，配置和模板的改动按性质用 `feat` / `fix` / `chore`。
- 只在明确要求时才提交或推送。
