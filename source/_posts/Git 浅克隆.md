---
title: Git 浅克隆 (Shallow Clone)
date: 2025-09-25 15:25:00
excerpt: 浅克隆，顾名思义，就是“不完整”的克隆
categories:
  - Git
---

> **一句话总结**：浅克隆就是只获取最近若干个提交的仓库副本，换取更小的下载体积和更快的速度，但会丢失完整历史，部分 Git 操作会受限。

## 什么是浅克隆

- **定义**：通过 `--depth` 限制，只克隆最近 N 个提交的历史。
- **作用**：显著加快 clone/fetch 速度，节省带宽和磁盘。
- **原理**：Git 会在 `.git/shallow` 文件里记录“历史被截断”的状态。

## 常见用法

```bash
# 只克隆最近 1 个提交
git clone --depth 1 https://example.com/repo.git

# 克隆 develop 分支，最近 50 个提交
git clone --depth 50 --branch develop --single-branch https://example.com/repo.git

# 子模块浅克隆
git submodule update --init --recursive --depth 1
```

## depth 的语义

- `--depth=N`：最近 **N 个 commit**。
- 不加 `--depth`：默认完整克隆，包含所有历史。
- 默认行为：`--depth` 会隐式包含 `--single-branch`（除非加 `--no-single-branch`）。

## 加深或恢复历史

```bash
# 把历史加深到 100 个 commit
git fetch --depth=100 origin main

# 彻底恢复完整历史
git fetch --unshallow
```

## 浅克隆的限制与坑点

- **历史相关命令失效**：`git bisect`、`git blame` 可能失真。
- **push 限制**：有些服务拒绝浅克隆直接 push，需要先 `--unshallow`。
- **子模块问题**：可能遇到主仓库记录的 commit 拉不到。
- **不是“按需”下载**：浅克隆只是截断历史，后续想要更多必须手动 fetch。

## 浅克隆 vs 部分克隆

| 特性     | 浅克隆（`--depth`）     | 部分克隆（`--filter=blob:none`） |
| -------- | --------------------- | ------------------------------ |
| 提交历史 | 截断，只保留最近 N 个 | 完整历史                       |
| 文件对象 | 全部下载              | 按需下载                       |
| 场景     | CI/CD、快速构建       | 大仓库、本地开发节省空间       |

## CI/CD 实战建议

- Jenkins/GitHub Actions 等常见配置：

  ```bash
  git clone --depth 1 --branch main --single-branch --no-tags https://example.com/repo.git
  ```

- 适合场景：只需要最新代码，构建后仓库可丢弃。
- 如果需要生成 changelog、回溯历史：在构建中先 `git fetch --unshallow`。

## 常见 Q&A

**Q1: depth 相当于是 commit 个数吗？**
✅ 是的，就是最近 N 个提交。

**Q2: 不加 depth 会怎样？**
✅ 默认拉完整历史。

**Q3: 哪些命令支持 depth？**
✅ `clone`、`fetch`、`pull`、`submodule update`。

**Q4: 浅克隆能恢复成完整吗？**
✅ 可以，`git fetch --unshallow`。

**Q5: 和 partial clone 有啥区别？**
✅ 浅克隆截断历史，partial clone 保留完整历史但懒下载文件。

## 命令 vs depth 对比表

| 命令                                    | 是否支持 `--depth` | 主要用途                      | 备注                 |
| --------------------------------------- | ------------------ | ----------------------------- | -------------------- |
| `git clone --depth <n>`                 | ✅                  | 初次克隆，只获取最近 n 个提交 | CI/CD 常用           |
| `git fetch --depth <n>`                 | ✅                  | 给浅克隆加深历史              | 逐步加深             |
| `git fetch --unshallow`                 | ✅                  | 恢复完整历史                  | 从浅仓库变成完整仓库 |
| `git pull --depth <n>`                  | ✅                  | 更新时只拉最近 n 个提交       | 底层 fetch + merge   |
| `git submodule update --depth <n>`      | ✅                  | 子模块浅克隆                  | CI/CD 常用           |
| `git rev-parse --is-shallow-repository` | ⚠️                  | 检查是否为浅克隆              | 输出 true/false      |

## 总结

- **浅克隆**：适合 CI/CD，快速拉最新代码，省时省力。
- **完整克隆**：适合长期开发，功能全面。
- **部分克隆**：适合大仓库，兼顾历史完整与按需下载。

👉 记住一句话：**线上构建用浅克隆，本地开发别偷懒。**
