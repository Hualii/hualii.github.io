---
title: "我的第一篇 WPF 源码贡献：从 Fork 到 PR"
date: 2026-06-24
draft: false
tags: ["WPF", "开源贡献", "GitHub"]
categories: ["开源贡献"]
summary: "记录我从 Fork WPF 源码到提交第一个 PR 的完整过程，降低他人的入门门槛。"
---

## 为什么从 WPF 源码开始

WPF 是 .NET 生态中最成熟也最复杂的桌面框架之一。它的源码托管在 GitHub 上（[dotnet/wpf](https://github.com/dotnet/wpf)），任何人都可以阅读、Fork 和贡献。

我的目标是：通过深度阅读 WPF 源码，产出技术博客和视频，最终冲击微软 MVP。

## 第一步：Fork + 克隆

1. 访问 https://github.com/dotnet/wpf
2. 点击右上角 **Fork** 按钮
3. 克隆你 Fork 的仓库到本地：

```bash
git clone https://github.com/Hualii/wpf.git
cd wpf
```

## 第二步：编译通过

WPF 的构建依赖一些特殊工具链，官方文档有详细说明：

```bash
# 恢复依赖
.\restore.cmd
# 构建
.\build.cmd
```

> ⚠️ 首次编译可能需要 10~20 分钟，取决于机器配置。

## 第三步：找到 Good First Issue

在 WPF 仓库中搜索标记为 `good first issue` 的问题：

https://github.com/dotnet/wpf/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22

从最简单的开始——修复文档 typo、改进错误信息、替换废弃 API 调用。

## 第四步：提交 PR

1. 创建新分支：`git checkout -b fix/my-first-pr`
2. 修改代码
3. 提交：`git commit -m "fix: description of the change"`
4. 推送：`git push origin fix/my-first-pr`
5. 在 GitHub 上点击 **Compare & pull request**
6. 填写 PR 描述，说明改了什么、为什么改

## 总结

第一个 PR 不在于代码多重要，而在于**走通流程**。一旦流程熟悉了，后续的贡献会越来越顺畅。

接下来我会开始深入阅读 WPF 的 DependencyProperty 机制，敬请关注！
