---
title: 常用工具
date: 2026-09-26
draft: false
tags: [工具]
---

## Ghostty

[Ghostty](https://ghostty.org/) 是 Mitchell Hashimoto（HashiCorp 创始人）用 Zig 编写的开源终端模拟器。它通过 GPU 加速渲染，在 macOS 和 Linux 上使用原生 UI，开箱即用，几乎不需要配置。

以下快捷键均为 macOS 下的默认值（基于 1.3），`⌘` 在 Linux 下通常对应 `Ctrl+Shift`。

### 分屏

不用 tmux，也能在一个窗口里并排开多个终端。

| 快捷键        | 作用                               |
| ------------- | ---------------------------------- |
| `⌘D`          | 向右分屏                           |
| `⌘⇧D`         | 向下分屏                           |
| `⌘[` / `⌘]`   | 切换到上一个 / 下一个分屏          |
| `⌘⌥ + 方向键` | 按方向切换分屏                     |
| `⌘⌃ + 方向键` | 调整分屏大小                       |
| `⌘⌃=`         | 均分所有分屏                       |
| `⌘⇧↩`         | 放大 / 还原当前分屏，临时专注一个  |
| `⌘W`          | 关闭当前分屏（或标签页）           |

### 标签页与窗口

| 快捷键          | 作用                                |
| --------------- | ----------------------------------- |
| `⌘T`            | 新建标签页                          |
| `⌘1` ~ `⌘8`     | 跳到第 N 个标签页                   |
| `⌘9`            | 跳到最后一个标签页                  |
| `⌘⇧[` / `⌘⇧]`   | 上一个 / 下一个标签页               |
| `⌘Z` / `⌘⇧T`    | 撤销关闭，误关的标签页和分屏能找回  |
| `⌘↩`            | 切换全屏                            |

### 输出与滚动

| 快捷键        | 作用                                                 |
| ------------- | ---------------------------------------------------- |
| `⌘↑` / `⌘↓`   | 跳到上一条 / 下一条命令的提示符，快速翻看长输出      |
| `⌘K`          | 清屏（连同滚动历史一起清掉）                         |
| `⌘F`          | 在滚动历史中搜索，`⌘G` / `⌘⇧G` 切换匹配项            |
| `⌘E`          | 用选中的文字直接搜索                                 |
| `⌘Home` / `⌘End` | 滚动到顶部 / 底部                                 |
| `⌘⇧J`         | 把当前屏幕内容存成文件，并把路径粘贴到命令行         |

跳转提示符依赖 shell integration。bash、zsh、fish 下 Ghostty 会自动注入，通常无需额外设置。

### 其他

| 快捷键   | 作用                                         |
| -------- | -------------------------------------------- |
| `⌘⇧P`    | 命令面板，忘了快捷键时直接搜索动作名         |
| `⌘,`     | 打开配置文件                                 |
| `⌘⇧,`    | 重新加载配置，改完立即生效                   |
| `⌘=` / `⌘-` / `⌘0` | 放大 / 缩小 / 重置字号             |

### 值得加的配置

配置文件位于 `~/.config/ghostty/config`：

```ini
# 全局快捷键呼出下拉式终端（Quick Terminal），类似 iTerm2 的 Hotkey Window
# 首次使用需在系统设置中授予辅助功能权限
keybind = global:cmd+grave_accent=toggle_quick_terminal

# 跟随系统深浅色自动切换主题
theme = light:Catppuccin Latte,dark:Catppuccin Mocha

# 让 Option 键作为 Alt 使用，方便 shell 和 Vim 里的 Alt 组合键
macos-option-as-alt = true
```

### 自带的命令行工具

| 命令                                  | 作用                                   |
| ------------------------------------- | -------------------------------------- |
| `ghostty +list-themes`                | 交互式预览并挑选内置主题               |
| `ghostty +list-keybinds --default`    | 列出所有默认快捷键                     |
| `ghostty +show-config --default --docs` | 列出所有配置项及说明                 |
| `ghostty +list-fonts`                 | 列出可用字体                           |

## chezmoi

[chezmoi](https://www.chezmoi.io/) 是一个用 Go 编写的 dotfiles 管理工具。它把 `~/.zshrc`、`~/.config/ghostty/config` 这类配置文件统一收进一个 Git 仓库，换新电脑时一条命令就能恢复整套环境；同时支持模板和密码管理器集成，能处理不同机器之间的差异和敏感信息。

### 工作方式

chezmoi 维护一个**源目录**（默认 `~/.local/share/chezmoi`，本身就是个 Git 仓库），里面存放配置文件的"标准版本"。`chezmoi apply` 会根据源目录计算出目标状态，再写入家目录。

源目录里的文件名通过前缀表达属性，例如：

| 源目录中的文件名                 | 对应的目标文件                    |
| -------------------------------- | --------------------------------- |
| `dot_zshrc`                      | `~/.zshrc`                        |
| `dot_config/ghostty/config`      | `~/.config/ghostty/config`        |
| `private_dot_ssh/config`         | `~/.ssh/config`（权限 600）       |
| `executable_dot_local/bin/foo`   | `~/.local/bin/foo`（可执行）      |
| `dot_gitconfig.tmpl`             | `~/.gitconfig`（作为模板渲染）    |
| `run_once_install-packages.sh`   | 只在首次 apply 时执行一次的脚本   |

### 常用命令

| 命令                          | 作用                                                   |
| ----------------------------- | ------------------------------------------------------ |
| `chezmoi init`                | 初始化源目录                                           |
| `chezmoi add ~/.zshrc`        | 把文件纳入管理（复制到源目录）                         |
| `chezmoi edit ~/.zshrc`       | 编辑源目录中的版本，加 `--apply` 保存后立即生效        |
| `chezmoi diff`                | 查看 apply 会对家目录做哪些改动                        |
| `chezmoi apply`               | 把源目录的状态应用到家目录，加 `-v` 显示详情           |
| `chezmoi status`              | 列出源目录与家目录不一致的文件                         |
| `chezmoi re-add`              | 直接改了家目录里的文件后，把改动同步回源目录           |
| `chezmoi managed`             | 列出所有已管理的文件                                   |
| `chezmoi cd`                  | 打开一个位于源目录的子 shell，方便执行 git 操作        |
| `chezmoi update`              | 从远程仓库拉取并 apply，相当于 `git pull` + `apply`    |

### 在新机器上恢复

把源目录推到 GitHub 上名为 `dotfiles` 的仓库后，新机器上只需：

```sh
# 安装后，从 github.com/<用户名>/dotfiles 克隆并直接应用
chezmoi init --apply <用户名>
```

### 模板：处理机器差异

以 `.tmpl` 结尾的文件会用 Go 模板渲染，可以根据系统、主机名等条件生成不同内容：

```sh
# dot_zshrc.tmpl
{{ if eq .chezmoi.os "darwin" -}}
export HOMEBREW_NO_ANALYTICS=1
{{ else if eq .chezmoi.os "linux" -}}
alias open=xdg-open
{{ end -}}
```

`chezmoi data` 可以查看所有可用的模板变量，`chezmoi execute-template` 可以用来调试模板。机器专属的变量（如邮箱）可以写在 `~/.config/chezmoi/chezmoi.toml` 的 `[data]` 中，这个文件不进仓库。
