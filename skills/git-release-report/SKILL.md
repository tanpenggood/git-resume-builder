---
name: git-release-report
description: Git发版报告生成器。分析两个提交之间的变更，统计涉及的模块、功能、脚本等，生成文字版发版报告。Use when user says "发版报告"、"release report"、"changelog"、"版本报告"或需要生成发版变更记录。
---

# Git 发版报告生成器

## 功能

分析两个Git提交之间的变更记录，生成结构化的发版报告，包含：
- **涉及模块** - 变更涉及的代码模块/目录
- **功能列表** - 具体的功能变更
- **脚本检测** - 是否涉及执行脚本或数据库脚本
- **变更统计** - 文件数、行数等统计信息

## 输入参数

| 参数 | 是否必填 | 说明 |
|------|----------|------|
| 起始hash | ✅ 必填 | 上次发版的git提交记录hash值 |
| 结束hash | ❌ 可选 | 本次发版结束的hash值，默认为当前HEAD |
| 分支名 | ❌ 可选 | 目标分支，默认为当前分支 |

## 使用步骤

### 1. 收集参数

向用户询问以下信息（使用交互式对话）：

```
请提供以下信息来生成发版报告：

1. 上次发版的提交hash值（必填）：
2. 本次发版的结束hash值（可选，留空则使用当前HEAD）：
3. 目标分支（可选，留空则使用当前分支）：
```

### 2. 获取提交列表

使用以下命令获取两个提交之间的变更：

```bash
# 获取提交列表（如果指定了分支）
git log <start-hash>..<end-hash> --oneline --no-merges

# 获取提交列表（如果未指定分支，使用当前分支）
git log <start-hash>..<end-hash> --oneline --no-merges
```

### 3. 分析变更文件

获取变更的文件列表：

```bash
# 获取变更文件统计
git diff <start-hash>..<end-hash> --stat

# 获取详细文件列表
git diff <start-hash>..<end-hash> --name-status
```

### 4. 分析提交信息

分析每个提交的message，提取功能描述：

```bash
# 获取详细提交信息
git log <start-hash>..<end-hash> --pretty=format:"%h %s" --no-merges
```

### 5. 生成报告

根据收集的信息生成结构化报告。

## 报告模板

```markdown
# 发版报告

**报告生成时间**: {当前时间}
**版本范围**: {起始hash} → {结束hash}
**目标分支**: {分支名}

---

## 📊 变更统计

| 指标 | 数量 |
|------|------|
| 提交数量 | {commit_count} |
| 变更文件数 | {file_count} |
| 新增行数 | +{lines_added} |
| 删除行数 | -{lines_deleted} |

---

## 📁 涉及模块

{按目录分组列出变更的文件}

### {模块名1}
- `文件1`
- `文件2`

### {模块名2}
- `文件3`

---

## ✨ 功能变更

{根据提交信息整理的功能列表}

1. **{提交描述1}** (`{commit_hash}`)
2. **{提交描述2}** (`{commit_hash}`)

---

## 🔧 脚本检查

### 执行脚本
{检查是否有shell脚本、python脚本等}
- [ ] 涉及执行脚本: {是/否}
- 脚本文件: {列出具体脚本文件}

### 数据库脚本
{检查是否有SQL文件、数据库迁移脚本等}
- [ ] 涉及数据库变更: {是/否}
- 数据库脚本: {列出具体SQL文件}

---

## 📝 变更明细

{详细的文件变更列表}

| 状态 | 文件路径 |
|------|----------|
| 新增 | {file_path} |
| 修改 | {file_path} |
| 删除 | {file_path} |
```

## 分析逻辑

### 模块识别
通过文件路径的第一级目录识别模块，例如：
- `src/modules/user/` → 用户模块
- `src/api/` → API模块
- `src/components/` → 组件模块

### 脚本检测
检测以下类型的文件：
- **执行脚本**: `.sh`, `.py`, `.bat`, `.ps1`, `.js` (在scripts目录下)
- **数据库脚本**: `.sql`, 文件名包含 `migrate`, `schema`, `seed`

### 功能提取
从提交信息中提取功能描述，常见前缀：
- `feat:` / `feature:` → 新功能
- `fix:` / `bugfix:` → 问题修复
- `refactor:` → 重构
- `chore:` / `build:` → 构建/配置变更

## 完整交互流程

```
AI: 请提供上次发版的git提交hash值（必填）：
用户: a1b2c3d

AI: 请提供本次发版的结束hash值（可选，留空则使用当前HEAD）：
用户: （留空）

AI: 请提供目标分支（可选，留空则使用当前分支）：
用户: （留空）

AI: 正在生成发版报告...

[执行git命令收集信息]

AI: [输出完整的发版报告]
```

## 注意事项

1. 确保在Git仓库中执行命令
2. 起始hash必须存在且有效
3. 如果未指定结束hash，默认使用当前HEAD
4. 报告中的统计数据基于git diff结果
5. 中文提交信息会被保留用于功能描述

## 示例输出

```
# 发版报告

**报告生成时间**: 2026-07-23 18:00:00
**版本范围**: a1b2c3d → e5f6g7h
**目标分支**: main

---

## 📊 变更统计

| 指标 | 数量 |
|------|------|
| 提交数量 | 12 |
| 变更文件数 | 28 |
| 新增行数 | +456 |
| 删除行数 | -89 |

---

## 📁 涉及模块

### 用户模块 (src/modules/user/)
- `UserList.vue` (修改)
- `UserDetail.vue` (新增)
- `userService.js` (修改)

### 订单模块 (src/modules/order/)
- `OrderList.vue` (修改)
- `orderApi.js` (修改)

---

## ✨ 功能变更

1. **用户列表支持批量导出** (`e5f6g7h`)
2. **订单详情页新增物流跟踪** (`d4e5f6g`)
3. **修复用户登录超时问题** (`c3d4e5f`)
4. **优化订单查询性能** (`b2c3d4e`)

---

## 🔧 脚本检查

### 执行脚本
- [ ] 涉及执行脚本: 否

### 数据库脚本
- [ ] 涉及数据库变更: 是
- 数据库脚本: 
  - `migrations/20260723_add_user_export.sql`
  - `migrations/20260723_add_logistics_tracking.sql`

---

## 📝 变更明细

| 状态 | 文件路径 |
|------|----------|
| 新增 | src/modules/user/UserDetail.vue |
| 新增 | migrations/20260723_add_user_export.sql |
| 修改 | src/modules/user/UserList.vue |
| 修改 | src/modules/user/userService.js |
| 修改 | src/modules/order/OrderList.vue |
| 修改 | src/modules/order/orderApi.js |
| 删除 | src/utils/legacyHelper.js |
```

## 参考

- [Git Log 文档](https://git-scm.com/docs/git-log)
- [Git Diff 文档](https://git-scm.com/docs/git-diff)
