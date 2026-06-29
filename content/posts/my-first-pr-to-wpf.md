---
title: "我的第一个 WPF PR：修复 CA1036 代码分析警告"
date: 2026-06-26
draft: false
tags: ["WPF", "开源贡献", "GitHub", "CA1036", "代码分析"]
categories: ["开源贡献"]
summary: "记录我提交的第一个 WPF 源码 PR 的完整过程：从 Fork 仓库到代码审查，修复 CA1036 规则警告的实战经验。"
---

## 前言

2026 年 6 月，我完成了人生中第一个向微软官方仓库的代码贡献。这篇博客记录从 Fork 到 PR 合并的完整流程，希望能帮助更多想参与 .NET 开源的朋友跨过第一道门槛。

## 什么是 CA1036？

**CA1036** 是 .NET 代码分析器的一条规则，全称是：

> **"Override methods on comparable types"**（在可比较类型上重写方法）

当一个类型实现了 `IComparable<T>` 接口时，这条规则要求你同时实现以下成员：

- `Equals(object)` 和 `GetHashCode()`
- `Equals(T)`（`IEquatable<T>` 接口）
- 比较操作符：`<`, `>`, `<=`, `>=`

**为什么重要？** 如果只实现 `CompareTo` 而不提供操作符，开发者在使用时会出现直觉上的不一致：

```csharp
// 可以这样比较
if (a.CompareTo(b) < 0) { ... }

// 但不能这样——编译错误！
if (a < b) { ... }  // ❌ 操作符未定义
```

## 找到切入点

在 [dotnet/wpf](https://github.com/dotnet/wpf) 仓库的 Issues 中，我发现了 [#10271](https://github.com/dotnet/wpf/issues/10271)：

> **"Fix CA1036: Override methods on comparable types"**

这是一个标记为 `good first issue` 的任务，目标是修复 WPF 源码中违反 CA1036 规则的类型。

## 准备工作

### 1. Fork 仓库

访问 https://github.com/dotnet/wpf，点击右上角 **Fork** 按钮，将仓库复制到自己的账号下。

### 2. 克隆到本地

```bash
git clone https://github.com/Hualii/wpf.git
cd wpf
```

### 3. 配置编译环境

WPF 是一个大型项目，编译需要特定的 SDK 版本。我遇到了一些环境配置问题：

```bash
# 恢复依赖（会自动下载所需 SDK）
.\restore.cmd

# 启动配置了正确环境的 VS
.\start-vs.cmd
```

## 分析需要修复的类型

通过代码搜索，我找到了 4 个需要修复的类型：

| 类型 | 所在项目 | 修复内容 |
|------|---------|---------|
| `GlyphLookupRecord` | `PresentationCore` | 补充 4 个比较操作符 |
| `TextEffectBoundary` | `PresentationCore` | 添加 `Equals`、`GetHashCode` 和操作符 |
| `ValidatedPartUri` | `WindowsBase` | 添加 `Equals`、`GetHashCode` 和操作符 |
| `MemoryStreamBlock` | `WindowsBase` | 提取 `Compare` 方法，完整实现所有成员 |

其中 `GlyphLookupRecord` 和 `TextEffectBoundary` 在 `PresentationCore` 项目中，`ValidatedPartUri` 和 `MemoryStreamBlock` 在 `WindowsBase` 项目中。

## 代码实现示例

以 `TextEffectBoundary` 为例，这是一个 struct，需要完整的可比较实现：

```csharp
// 原始：只实现了 IComparable<TextEffectBoundary>
internal readonly struct TextEffectBoundary : IComparable<TextEffectBoundary>
{
    public int CompareTo(TextEffectBoundary other) { ... }
}

// 修复后：添加 Equals、GetHashCode 和操作符
internal readonly struct TextEffectBoundary : IComparable<TextEffectBoundary>
{
    private readonly int _position;
    private readonly bool _isStart;
    
    public int CompareTo(TextEffectBoundary other) { ... }
    
    // 新增：Equals 方法（满足 IEquatable<T> 模式）
    public bool Equals(TextEffectBoundary other)
    {
        return _position == other._position && _isStart == other._isStart;
    }
    
    // 新增：Equals + GetHashCode
    public override bool Equals(object obj)
    {
        return obj is TextEffectBoundary other && Equals(other);
    }
    
    public override int GetHashCode()
    {
        return HashCode.Combine(_position, _isStart);
    }
    
    // 新增：相等性操作符
    public static bool operator ==(TextEffectBoundary left, TextEffectBoundary right)
    {
        return left.Equals(right);
    }
    
    public static bool operator !=(TextEffectBoundary left, TextEffectBoundary right)
    {
        return !left.Equals(right);
    }
    
    // 新增：比较操作符
    public static bool operator <(TextEffectBoundary left, TextEffectBoundary right)
    {
        return left.CompareTo(right) < 0;
    }
    
    public static bool operator >(TextEffectBoundary left, TextEffectBoundary right)
    {
        return left.CompareTo(right) > 0;
    }
    
    public static bool operator <=(TextEffectBoundary left, TextEffectBoundary right)
    {
        return left.CompareTo(right) <= 0;
    }
    
    public static bool operator >=(TextEffectBoundary left, TextEffectBoundary right)
    {
        return left.CompareTo(right) >= 0;
    }
}
```

> **注意**：虽然添加了 `Equals(T)` 方法满足 `IEquatable<T>` 的模式，但并没有显式声明接口。这在 C# 中是允许的，编译器会通过方法签名识别出实现了该模式。

### MemoryStreamBlock 的特殊处理

`MemoryStreamBlock` 是一个 class（引用类型），且原来的 `CompareTo` 是显式接口实现。为了保持代码整洁，我提取了一个私有的 `Compare` 方法：

```csharp
// 原始：显式接口实现
int IComparable<MemoryStreamBlock>.CompareTo(MemoryStreamBlock other)
{
    // 比较逻辑...
}

// 修复后：提取私有方法 + 完整实现
private int Compare(MemoryStreamBlock other)
{
    // 原比较逻辑...
}

int IComparable<MemoryStreamBlock>.CompareTo(MemoryStreamBlock other)
{
    return Compare(other);
}

public bool Equals(MemoryStreamBlock other)
{
    if (other == null)
        return false;
    // 基于 Offset 和 EndOffset 的相等性判断...
}

// 其他操作符实现...
```

这种重构方式让代码更清晰，也符合 CA1036 的要求。

## 提交 PR

### 1. 创建分支

```bash
git checkout -b fix-ca1036-comparable-types
```

### 2. 提交更改

```bash
git add .
git commit -m "Implement CA1036: Add missing comparison operators to IComparable<T> types

- Add IEquatable<T> implementation to TextEffectBoundary
- Add comparison operators to GlyphLookupRecord, TextEffectBoundary, 
  ValidatedPartUri, and MemoryStreamBlock
- Ensure Equals and GetHashCode are consistent with CompareTo

Fixes #10271"
```

### 3. 推送并创建 PR

```bash
git push origin fix-ca1036-comparable-types
```

然后在 GitHub 上点击 **Compare & pull request**，填写 PR 描述：

- **标题**：清晰说明修复内容
- **描述**：关联 Issue #10271，列出修改的文件和类型
- **检查清单**：确认代码风格、测试通过

## PR 状态

我的 PR [#11733](https://github.com/dotnet/wpf/pull/11733) 已成功创建：

- ✅ 4 个文件改动
- ✅ 可自动合并（无冲突）
- ✅ 引用了 Issue #10271
- ⏳ 等待代码审查中

## 收获与反思

### 技术层面

1. **深入理解了 CA1036 规则** —— 不只是记住规则，而是理解了设计意图
2. **熟悉了 WPF 的代码结构** —— PresentationCore、PresentationFramework 的模块划分
3. **学习了大型项目的编译流程** —— `restore.cmd`、`build.cmd`、`start-vs.cmd` 的配合

### 流程层面

1. **Good First Issue 是真的** —— 这个 Issue 被标记为适合新手，确实难度适中
2. **环境配置是最大的门槛** —— 代码修改本身不难，让项目跑起来花了更多时间
3. **Git 操作要熟练** —— Fork、Branch、Commit、Push、PR 的流程要形成肌肉记忆

### 下一步

- 等待 PR 审查反馈，根据意见修改
- 寻找下一个贡献机会（可能是 #10270 列表中的其他代码分析规则）

## 给新手的建议

如果你也想开始开源贡献：

1. **从文档/小修复开始** —— 不要一上来就改核心逻辑
2. **先让项目跑起来** —— 能编译、能运行，再谈贡献
3. **仔细阅读 Issue 描述** —— 往往包含重要线索
4. **不要怕问问题** —— 在 Issue 下留言，维护者通常很友好
5. **记录你的过程** —— 写成博客，帮助后来者

---

**相关链接：**
- [我的 PR #11733](https://github.com/dotnet/wpf/pull/11733)
- [原始 Issue #10271](https://github.com/dotnet/wpf/issues/10271)
- [CA1036 规则文档](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca1036)
- [WPF 源码仓库](https://github.com/dotnet/wpf)

---

*这篇博客也是我自己学习过程的记录。如果你有任何问题，欢迎在评论区留言交流！*
