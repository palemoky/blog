---
title: LazyVim
date: 2026-09-30
draft: true
tags: [工具, Neovim]
---

[LazyVim](https://www.lazyvim.org/) 是一套预先配好的 Neovim 配置，定位和 Emacs 里的 Doom 差不多：文件查找、代码补全、LSP、Git 这些都已配好，装完就能直接用，不用从零开始挑插件、写配置。

## 安装

先装好 Neovim（0.11.2 或以上）和几个依赖，再克隆官方的 starter 模板：

```bash
brew install neovim git lazygit ripgrep fd

mv ~/.config/nvim{,.bak}    # 备份已有配置，没有可跳过
git clone https://github.com/LazyVim/starter ~/.config/nvim
rm -rf ~/.config/nvim/.git
nvim
```

第一次启动会自动下载所有插件，等它装完重启一次即可。终端字体要换成 [Nerd Font](https://www.nerdfonts.com/)，不然图标会显示成方块。

## 按键写法

LazyVim 沿用 Vim 的按键写法：

| 写法         | 含义                                     |
| ------------ | ---------------------------------------- |
| `<leader>`   | 空格键                                   |
| `<C-s>`      | Ctrl+s                                   |
| `<S-h>`      | Shift+h，也就是大写的 H                  |
| `<leader>ff` | 依次按空格、`f`、`f`，每个键按完就松开   |

按下空格后稍等一下，会弹出 which-key 菜单，列出接下来可以按哪些键。键位大多按英文首字母组织，比如 `f` 是 file，`b` 是 buffer，`c` 是 code，`g` 是 git。

## 常用按键速查

`h j k l`、`dd`、`yy` 这些 Vim 的基础操作这里不再列出，和 Doom 里完全一样。

{{< keytable "lazyvim-keys.yaml" >}}

## 和 Doom Emacs 对照

两者的思路几乎一样：都以空格作为 leader 键，都按首字母组织键位。下表只列两边都有的通用操作，Org mode 这类 Doom 独有的功能不在其中。

{{< keytable "compare-keys.yaml" >}}

几处明显的差别：

- **切换缓冲区**：LazyVim 用 `<S-h>` / `<S-l>` 在缓冲区之间来回切，比 Doom 的 `[b` / `]b` 顺手，这是它最常用的按键之一。
- **分屏**：LazyVim 的 `|` 和 `-` 是按分割线的形状来记的，Doom 的 `v` / `s` 沿用了 Vim 的 `:vsplit` / `:split`。
- **Git**：Doom 用 Magit，LazyVim 用 Lazygit，两者思路相近，都是在一个界面里用单键完成暂存、提交、推送。
- **LSP**：LazyVim 默认就配好了，Doom 要先在 `init.el` 里开启 `lsp` 模块，再给对应语言加上 `+lsp`。

## 配置文件

个人配置都在 `~/.config/nvim/lua/` 下：

| 文件                  | 用途                                   |
| --------------------- | -------------------------------------- |
| `config/options.lua`  | 编辑器选项，比如行号、缩进             |
| `config/keymaps.lua`  | 自定义按键                             |
| `plugins/*.lua`       | 新增插件，或修改默认插件的配置         |

改完保存，重启 Neovim 就会生效。

## 常用命令

| 命令           | 作用                                                         |
| -------------- | ------------------------------------------------------------ |
| `:Lazy`        | 插件管理器，可以更新、清理插件，也可以按 `<leader>l` 打开    |
| `:LazyExtras`  | 按需开启预配好的功能包，比如某门语言的支持                   |
| `:Mason`       | 安装 LSP、格式化工具等外部程序                               |
| `:checkhealth` | 检查缺少的依赖，出问题时先跑它                               |
