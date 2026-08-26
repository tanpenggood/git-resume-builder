---
name: ops-ticket-from-dingtalk
description: 钉钉运维票据整理器。从指定钉钉群收集指定时间范围内的聊天记录，智能分类整理为独立的运维问题票据，输出markdown格式的工作日报。Use when 用户说 运维票据/整理运维问题/运维日报/运维工单/收集运维问题/运维问题汇总/运维记录整理/系统运维群整理。命令前缀：dws chat。
metadata:
  cli_version: ">=0.2.14"
  category: workflow
  requires:
    bins:
      - dws
---

# 钉钉运维票据整理 Skill

## 快速开始

```bash
# 1. 创建本次任务的输出目录
mkdir -p ./ops-ticket-{群名}_{日期}

# 2. 获取消息（推荐使用Python脚本确保UTF-8编码）
python -c "
import subprocess, json, sys
sys.stdout.reconfigure(encoding='utf-8')
cmd = ['dws', 'chat', '+chat-messages', '--group', '<群名>', 
       '--start', '<开始时间>', '--end', '<结束时间>', 
       '--page-all', '--format', 'json']
result = subprocess.run(cmd, capture_output=True, text=True, encoding='utf-8')
with open('./ops-ticket-<群名>_<日期>/messages.json', 'w', encoding='utf-8') as f:
    f.write(result.stdout)
"

# 3. 下载图片
dws chat +messages-resource-download --message-id <id> --open-conversation-id <cid> \
  --resource-id <rid> --output ./ops-ticket-<群名>_<日期>/downloads/

# 4. 按自然周拆分生成票据
```

## 工作流程

### 步骤0：创建输出目录

每次执行都创建独立的输出目录，便于管理和分享：

```bash
# 目录命名规则：ops-ticket-{群名简称}_{日期}
mkdir -p ./ops-ticket-XX系统支持_20260826
```

**目录结构**：
```
ops-ticket-XX系统支持_20260826/
├── 运维票据-20260817~20260823.md    # 第1周票据
├── 运维票据-20260824~20260830.md    # 第2周票据
├── messages.json                    # 原始消息数据
└── downloads/                       # 下载的图片资源
    ├── img1.png
    └── img2.png
```

### 步骤1：获取群聊消息

使用 `dws chat +chat-messages` 或 `dws chat +search-msg` 获取指定群、指定时间范围的消息。

```bash
# 方式A：直接读取群消息（推荐）
dws chat +chat-messages \
  --group "<群名或openConversationId>" \
  --start "<开始时间ISO>" \
  --end "<结束时间ISO>" \
  --page-all \
  --format json

# 方式B：搜索方式（可按关键词筛选）
dws chat +search-msg \
  --chat-query "<群名>" \
  --query "<关键词，可选>" \
  --start-time "<开始时间ISO>" \
  --end-time "<结束时间ISO>" \
  --page-all \
  --format json
```

**时间格式说明：**
- ISO-8601格式：`2026-08-25T00:00:00+08:00`
- 时间范围为 `[start, end)`，包含开始时间，不包含结束时间

### 步骤2：下载所有图片资源

使用 `--download-resources` 参数自动下载消息中的所有图片：

```bash
cmd /c "dws chat +chat-messages \
  --group '{群名}' \
  --start {开始日期}T00:00:00+08:00 \
  --end {结束日期}T00:00:00+08:00 \
  --page-all \
  --download-resources \
  --output ops-ticket-{群名}_{日期}/messages.json \
  --overwrite"
```

**重要：图片下载目录必须在输出目录内**

```
ops-ticket-XX运维普通群_20260826/
├── messages.json
├── 运维票据-20260817~20260823.md
├── 运维票据-20260824~20260830.md
└── downloads/                       # 图片必须下载到这里
    ├── img1.png
    └── img2.png
```

**下载统计：**
- `discoveredCount`：发现的资源总数
- `downloadedCount`：成功下载的数量
- `failedCount`：失败数量
- `deduplicatedCount`：去重数量

**重要说明：**
- 合并转发消息中的图片也会被自动下载
- 每张图片会保存到 `./ops-ticket-{群名}_{日期}/downloads/` 目录
- 文件名使用 resourceId（去除特殊字符）
- 支持 PNG、JPG、JPEG 格式

### 步骤3：AI分析与分类

获取消息后，AI将自动执行以下分析：

1. **识别运维问题**
   - 从消息流中识别独立的运维问题/事件
   - **处理合并转发消息**：
     - 遍历 `forwarded` 数组中的所有原始消息
     - 提取每条原始消息的 `createTime`、`sender`、`text`、`resourceRefs`
     - 嵌套的合并消息也会被展开分析
   - 识别问题的开始、处理过程、结束
   - 关联相关消息（同一问题的多条消息）

2. **问题分类**
   - **故障类**：服务宕机、性能异常、错误报警
   - **需求类**：配置变更、权限申请、资源扩容
   - **咨询类**：技术问题、操作指导、流程咨询
   - **巡检类**：监控报告、健康检查、安全扫描

3. **信息提取**
   - 问题标题
   - 问题描述
   - 提出人/报告人
   - 处理人
   - **时间线**：使用原始消息时间（非转发时间）
   - 处理结果
   - 影响范围
   - 根因分析（如有）
   - **相关截图**：
      - 直接消息中的图片：`./downloads/{resourceId}.png`
      - 转发消息中的图片：同样保存到 `./downloads/` 目录
      - **在票据中引用时，每张图片必须在上方标注发送者和发送时间**
      - 格式：`**发送者** ({时间})：![描述](./downloads/{resourceId}.png)`
      - 示例：`**张三** (2026-08-20 09:11)：![报错截图](./downloads/xxx.png)`
      - **嵌套转发消息**：使用原始消息的发送者和时间（`forwarded[].sender`、`forwarded[].createTime`），而非转发者

### 步骤3：按自然周拆分并生成工作票据

**自然周定义**：周一至周日

**拆分规则**：
- **使用原始消息时间**（非转发时间）
  - 对于普通消息：使用 `createTime`
  - 对于合并转发消息：使用 `forwarded[].createTime`（原始消息时间）
- 根据消息时间戳，按自然周分组
- 每个自然周生成一个独立的票据文档
- 文件命名：`运维票据-{YYYYMMDD}~{YYYYMMDD}.md`
  - 起始日期：该周的周一
  - 结束日期：该周的周日

**示例**：
- 转发时间：2026-08-26，但原始消息时间：2026-08-18
- 该消息应归入：`运维票据-20260817~20260823.md`（2026-08-18所在的周）

**时间优先级**：
1. 合并转发消息：优先使用 `forwarded[].createTime`
2. 普通消息：使用 `createTime`
3. 如果 `forwarded` 中有多条消息，每条都独立判断所属周

将票据保存到输出目录：`./ops-ticket-{群名}_{日期}/`

按以下模板输出每个运维问题的票据：

```markdown
# 运维工作票据 - {日期}

## 📋 概览

| 统计项 | 数值 |
|--------|------|
| 运维问题总数 | X |
| 故障类 | X |
| 需求类 | X |
| 咨询类 | X |
| 巡检类 | X |
| 待处理 | X |
| 已解决 | X |

---

## 票据 #1：{问题标题}

**问题类型**：{故障/需求/咨询/巡检}
**优先级**：{P0/P1/P2/P3}
**状态**：{处理中/已解决/已关闭}

### 基本信息

- **报告人**：{姓名}
- **处理人**：{姓名}
- **发现时间**：{时间}
- **解决时间**：{时间}
- **耗时**：{X小时X分钟}

### 问题描述

{问题的详细描述}

### 处理过程

1. {时间} - {操作/事件}
2. {时间} - {操作/事件}
3. ...

### 解决方案

{最终的解决方案}

### 影响范围

{受影响的系统、用户、业务}

### 根因分析

{问题的根本原因，如有}

### 相关截图

<!-- 图片引用格式：每张图片上方标注发送者和时间 -->
**张三** (2026-08-20 09:11)：
![报错截图](./downloads/xxx.png)

**李四** (2026-08-20 09:15)：
![配置截图1](./downloads/yyy.png)

**王五** (2026-08-20 09:20)：
![监控图表](./downloads/zzz.png)

### 经验教训

{可改进的点、建议}

---

## 票据 #2：{问题标题}

...
```

## 高级功能

### 图片分析（重要）

聊天记录中的图片可以被下载和分析。消息中的图片以 `resourceRefs` 形式存在：

```json
{
  "sender": "张三",
  "text": "[图片消息](mediaId=@xxx)",
  "resourceRefs": [
    {
      "type": "mediaId",
      "resourceId": "@lQLPJx2_9Qs8v1vNAknNBI-wtqKGhonsik0KV8__0_8XAA",
      "download": {
        "shortcut": "+messages-resource-download"
      }
    }
  ]
}
```

**下载图片**：

需要下载两类图片：
1. **普通消息图片**：直接从 `resourceRefs` 下载
2. **转发消息图片**：从 `forwarded[].resourceRefs` 下载

```bash
# 下载单张图片
dws chat +messages-resource-download \
  --message-id <openMessageId> \
  --open-conversation-id <openConversationId> \
  --resource-id <resourceId> \
  --output ./downloads/

# 或使用原子命令
dws chat message download-media \
  --type mediaId \
  --resource-id <resourceId> \
  --message-id <messageId> \
  --open-conversation-id <conversationId> \
  --output ./downloads/
```

**分析图片内容**：

下载图片后，AI可以分析图片内容，识别：
- 错误截图（报错信息、异常堆栈）
- 监控图表（CPU、内存、网络）
- 配置界面截图
- 操作步骤截图
- 其他运维相关的可视化信息

**完整工作流**：

```bash
# 1. 获取消息（包含图片引用）
dws chat +chat-messages --group "群名" --page-all --format json --output ./messages.json

# 2. 下载所有图片
dws chat +chat-messages --group "群名" --page-all --format json --download-resources --output ./data/

# 3. 分析消息和图片，生成票据
```

告诉AI：
```
分析 ./data/ 目录下的消息和图片，生成运维票据
```

### 按关键词过滤

```bash
# 只收集包含特定关键词的消息
dws chat +search-msg --chat-query "系统运维群" --query "故障" --start-time "..." --end-time "..." --page-all --format json
```

### 按发送者过滤

```bash
# 只收集特定人员的消息
dws chat +chat-messages --group "系统运维群" --sender-query "张三" --start "..." --end "..." --page-all --format json
```

### 多群合并

```bash
# 收集多个群的消息
dws chat +search-msg --groups "群ID1,群ID2" --start-time "..." --end-time "..." --page-all --format json
```

## 输出目录结构

每次执行都会生成独立的输出目录，便于管理和分享：

```
ops-ticket-{群名}_{日期}/
├── 票据.md              # 运维票据（最终产出）
├── messages.json        # 原始消息数据
└── downloads/           # 下载的图片资源
    ├── 发送者_描述_001.png
    └── 发送者_描述_002.png
```

**目录命名规则**：`ops-ticket-{群名简称}_{YYYYMMDD}`

例如：`ops-ticket-XX系统支持_20260826`

## 输出文件

生成的票据保存在输出目录中：`./ops-ticket-{群名}_{日期}/票据.md`

如需保存到指定位置，使用 `--output` 参数或直接告诉AI保存路径。

## 注意事项

1. **时间范围**：建议单次不超过24小时，消息量大时可分批处理
2. **全量获取**：务必使用 `--page-all` 确保获取所有消息
3. **消息格式**：JSON格式包含完整的消息元数据，便于AI分析
4. **图片下载**：使用 `--download-resources` 自动下载图片到本地
5. **图片分析**：AI可分析截图中的错误信息、监控图表等内容
6. **报告嵌入**：下载的图片可在票据报告中作为 `![说明](路径)` 引用展示
7. **隐私保护**：输出的票据默认不包含敏感操作细节
8. **准确分类**：AI会尽力准确分类，复杂问题可能需要人工确认

## 错误处理

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| 群名不存在 | 群名有歧义或不存在 | 使用 `dws chat +chat-search --query <关键词>` 查找 |
| 时间范围无效 | 格式错误或范围不合理 | 使用ISO-8601格式，确保start < end |
| 消息为空 | 该时间范围无消息 | 检查时间范围和群名是否正确 |
| 权限不足 | 无权访问该群 | 确认已在群内且有读取权限 |
| 图片下载失败 | 资源不存在或无权限 | 检查resourceId是否正确 |
