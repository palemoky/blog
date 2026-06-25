---
title: 不同编程语言对比
date: 2026-06-25T19:33:18+08:00
draft: true
tags:
  - 编程
---
##

|            | 设计思想 | 适用场景 |
| ---------- | ---- | ---- |
| Java       |      |      |
| Python     |      |      |
| Golang     |      |      |
| Rust       |      |      |
| JavaScript |      |      |
| PHP        |      |      |

### 包管理工具

|         | 项目依赖                     | 全局安装                            | 临时运行                   | 备注                                                          |
| ------- | ------------------------ | ------------------------------- | ---------------------- | ----------------------------------------------------------- |
| Node.js | `npm install <package>`  | `npm install -g <package>`      | `npx <package>`        |                                                             |
| Go      | `go get <lib>`           | `go install <tool>@latest`      | `go run <tool>@latest` |                                                             |
| Rust    | `cargo add <pkg>`        | `cargo install <pkg>`           | 官方未内置（多用第三方工具 crgx）    |                                                             |
| Python  | `uv add <package>`       | `uv tool install <package>`     | `uvx <package>`        | uvx 就是 uv tool run 的快捷别名                                    |
| Python  | `pip install <package>`  | `pipx install <package>`        | `pipx run <package>`   | pip 会将工具装在当前激活的虚拟环境或全局；pipx 则会自动为每个工具创建隔离的虚拟环境，防止依赖冲突。      |
| PHP     | `composer require <pkg>` | `composer global require <pkg>` | `composer exec <tool>` | `composer exec` 仅用于执行本地已安装的工具，PHP 生态目前没有类似 npx 的临时远程下载执行工具。 |

Go 和 Rust是编译型语言，`go install` 和 `cargo install` 编译出来的是无外部依赖的静态二进制文件，它们运行不依赖系统中的其他库版本，因此天然不会发生依赖冲突。
