---
name: project-synthesis
description: Use when a user wants a project- or founder-centric rollup across Lark calendar events, meeting notes, and minutes, ending in a Feishu memo document and a delivered message link.
---

# project-synthesis

**CRITICAL — 开始前 MUST 先用 Read 工具读取以下技能文档：**
- [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)
- [`../lark-calendar/references/lark-calendar-agenda.md`](../lark-calendar/references/lark-calendar-agenda.md)
- [`../lark-vc/references/lark-vc-search.md`](../lark-vc/references/lark-vc-search.md)
- [`../lark-vc/references/lark-vc-notes.md`](../lark-vc/references/lark-vc-notes.md)
- [`../lark-vc/references/lark-vc-recording.md`](../lark-vc/references/lark-vc-recording.md)
- [`../lark-minutes/references/lark-minutes-search.md`](../lark-minutes/references/lark-minutes-search.md)
- [`../lark-doc/references/lark-doc-create.md`](../lark-doc/references/lark-doc-create.md)
- [`../lark-im/references/lark-im-messages-send.md`](../lark-im/references/lark-im-messages-send.md)

## When to hard fail

- 用户没有提供任何 `项目` 或 `创始人` 线索时，**立即停止**，直接要求用户提供至少一个项目名、品牌名、公司名、创始人名或别名。
- 如果关键读取步骤在权限已经申请后仍返回 `missing_scope`、`permission deny`、`HTTP 403`，**通常立即停止**，给出具体失败命令和错误，不要继续产出部分报告。
- 例外：若某个直连 `minute_token` 读取报 `permission deny` / `HTTP 403`，但同一会议或同一日程的 `note_doc_token` 仍可正常 `docs +fetch`，则**允许继续**；此时必须明确标注“直连妙记存在但当前身份无权读取 AI 产物”，并改用可读的 `note_doc_token` 作为该条记录的摘要来源。
- 如果用户要求“发消息给我”，但当前身份对最终消息发送路径无权限，也要 **hard fail**，给出缺少的 scope 或失败命令。

## Input contract

至少要有以下之一：

- 项目名 / 品牌名 / 公司名
- 创始人名 / 英文名 / 别名

可选输入：

- 时间范围。若未给出，默认只查询过去 3 个月，并按月切片查询；只有用户明确指定其他时间范围（例如“全部历史”“过去 6 个月”“某个起止日期”）时，才按用户指定范围执行。
- 目标文档位置：
  - 若用户给出 `wiki-space` / `wiki-node` / 目标知识库，按其位置创建文档
  - 否则默认创建到 `my_library`
- 是否发送给当前用户本人。默认是。

## Required permissions

在开始大查询前，先确认当前 user/bot 身份已经覆盖以下能力。

### User-scoped reads

- `calendar:calendar:read`
- `calendar:calendar.event:read`
- `vc:meeting.search:read`
- `vc:meeting.meetingevent:read`
- `vc:note:read`
- `vc:record:readonly`
- `minutes:minutes.search:read`
- `minutes:minutes:readonly`
- `minutes:minutes.artifacts:read`
- `minutes:minutes.transcript:export`
- `docx:document:readonly`
- `drive:drive.metadata:readonly`

### User-scoped writes / delivery

- 文档创建所需 scope，以 `docs +create` 实际报错为准
- 若要用 user 身份发消息：`im:message.send_as_user`、`im:message`

### Bot fallback

- 若 user 身份不能发 IM，但 bot 可以向当前用户发 P2P 消息，可用 bot 发送最终链接
- 如果 bot 也不能发，hard fail

## Permission workflow

### 1. 先做窄验证，不要先跑全量

推荐验证顺序：

1. `calendar +agenda` 小时间窗读取
2. `vc +search` 用第一个项目/创始人关键词读取
3. `minutes +search --owner-ids me` 小时间窗读取
4. 找到第一个候选 `minute_token` 后，立即跑一次：
   ```bash
   lark-cli vc +notes --as user --minute-tokens <token> --format json
   ```
5. 找到第一个候选 `note_doc_token` 后，立即跑一次：
   ```bash
   lark-cli docs +fetch --doc <note_doc_token>
   ```
6. 最终发消息前，验证一次 `im +messages-send --dry-run` 或直接在最终发送阶段处理

### 2. 缺权限时一次性申请

如果任何一步报 `missing_scope`，把当前任务需要的所有缺失 scope 合并成**一次**授权请求，不要一条条补：

```bash
lark-cli auth login --scope "<space separated scopes>"
```

必须把设备授权链接发给用户，并等待用户确认。用户确认后，**必须重跑失败命令**；只有失败命令成功，才能继续。

### 3. 权限到账但资源仍 403

如果 scope 已到账，但读取具体 `note_doc_token` 仍然报 `permission deny` / `HTTP 403`，视为当前身份无法完成完整范围。直接 hard fail。

如果 scope 已到账，读取具体 `minute_token` 报 `permission deny` / `HTTP 403`：

- 若同一会议或同一日程存在可读的 `note_doc_token`，继续执行，不要 hard fail。
- 必须重跑并确认 `docs +fetch --doc <note_doc_token>` 成功后，才允许继续。
- 输出中必须标注该 `minute_token` 为“存在但 ACL blocked / 无权读取 AI 产物”。
- 只有在 `minute_token` 和对应 `note_doc_token` 都不可读时，才 hard fail。

## Search and matching workflow

### 0. 默认时间窗口

如果用户没有明确给出时间范围，所有 `calendar +agenda`、`vc +search`、`minutes +search` 查询都默认限制在**过去 3 个月**。

只有当用户明确要求更长、更短或“全部历史”时，才扩大或改写这个窗口。

### 1. 规范化关键词

把用户提供的信息整理成两组：

- `project_terms`
- `founder_terms`

同时保留中英文、空格变体、大小写变体和常见别名。搜索时逐个关键词分别查询，再去重。

### 2. 拉取相关日程

用 `calendar +agenda` 在目标时间范围内取主日历事件，然后按以下字段做关键词过滤：

- `summary`
- `description`
- `location.name`

命中任一关键词即可视为候选日程。

### 3. 拉取相关会议记录

对每个关键词分别跑 `vc +search`，去重后拿到 `meeting_id` 列表。

目的：

- 找出和项目/创始人直接相关的会议记录
- 补抓那些**不一定被日程文本命中**，但标题、录制、纪要文档文本中命中的会议

### 4. 为候选日程拉直连纪要和直连录制

对候选 `calendar_event_id` 批量跑：

- `vc +notes --calendar-event-ids`
- `vc +recording --calendar-event-ids`

从中拿：

- `note_doc_token`
- `verbatim_doc_token`
- `meeting_id`
- `minute_token`
- `recording_url`

### 5. 为关键词命中的会议记录补抓纪要和录制

对 `meeting_id` 批量跑：

- `vc +notes --meeting-ids`
- `vc +recording --meeting-ids`

这些记录里，如果最终没有对应到任何候选日程，就进入“没有日程匹配的会议纪要 / 妙记”分组。

### 6. 拉取时间范围内的全部个人妙记

不要只按关键词搜妙记。相关妙记经常不含项目或创始人字样。

按月切片，分别拉：

```bash
lark-cli minutes +search --as user --owner-ids me ...
lark-cli minutes +search --as user --participant-ids me ...
```

按 `token` 去重后，解析每条妙记的：

- 标题
- URL
- 开始时间
- 时长
- 结束时间

### 7. 时间匹配规则

把每个候选日程与全部个人妙记做时间重叠匹配。

**硬规则：排除 overlap < 10 分钟的匹配。**

即：

- `overlap_sec >= 600` 才算时间匹配
- 小于 10 分钟，一律视为不匹配

如果某个妙记已经是该日程的 `direct minute_token`，则不要再重复放入“时间匹配妙记”。

## Classification rules

### A. 日程主表

每条相关日程都要落入以下之一：

1. `有直连纪要`
2. `有直连妙记`
3. `有时间匹配妙记`
4. `以上都没有`

### B. 没有日程匹配的妙记 / 会议纪要

放入这一组的对象包括：

- 通过 `vc +search` 命中的会议，其 `meeting_id` 没有关联到任何候选日程
- 这些会议的 `note_doc_token`
- 这些会议的 `minute_token`
- 任何关键词直接命中的妙记，但最终没有关联到候选日程

### C. 没有妙记和会议纪要匹配的日程

候选日程若同时满足：

- 无 `note_doc_token`
- 无 `direct minute_token`
- 无 `time-matched minute (>=10m)`

则进入这一组。

## Summarization workflow

### 1. 先生成时间顺序 summary

放在文档最前面，用时间顺序列出每次相关接触，格式建议：

- 日期 / 时间
- 对象名
- 关联类型：`direct note` / `direct minute` / `time-matched minute(s)` / `no match`
- 一句话结论

### 2. 再按日期顺序展开正文

每个日期块包含：

- 日程标题
- 日程链接
- 直连纪要链接（如果有）
- 直连妙记链接（如果有）
- 时间匹配妙记链接（如果有）
- 摘要：
    - 第一行必须先写 **one-sentence summary**，用一句话概括会议主要内容
    - 随后再用 bullet points 展开会议讨论的主要内容、关键观点或章节
- Follow-ups：
    - 若有待办，统一放在 `Follow-ups:` 下面，用 bullet points 平铺列出
    - 不要把 follow-up 写成和 `Follow-ups:` 同级的一行行 `- Follow-ups: xxx`
- 关键待办 / follow-up（如果有）

### 3. 总结过程中的发现

总结以下内容，每一项下可以用 bullet points 展开

- 过程中我们最关心的问题
- 创始团队在过程中想法的变化，体现的优缺点
- 当前最值得关注的点

### 3. 摘要内容来源优先级

- `- 摘要：<one-sentence summary>`
- 其后紧跟若干 `  - ...` bullet，展开关键讨论点
- `- Follow-ups:`
- 其后紧跟若干 `  - ...` bullet，列出待办；若无待办则可省略整个 `Follow-ups:` 区块

#### 会议纪要

若有 `note_doc_token`：

1. `docs +fetch --doc <note_doc_token>`
2. 提取 `# 总结`
3. 提取 `# 待办`

#### 妙记

若有 `minute_token`：

1. `vc +notes --minute-tokens <token>`
2. 优先使用：
   - `artifacts.summary`
   - `artifacts.todos`
   - `artifacts.chapters`

若 `minute_token` 存在但 `vc +notes --minute-tokens <token>` 返回 `permission deny` / `HTTP 403`，则按以下顺序降级：

1. 若同一会议或同一日程有可读的 `note_doc_token`，改用 `docs +fetch --doc <note_doc_token>` 的 `# 总结` / `# 待办`
2. 在正文里保留直连妙记链接或 `minute_token` 记录，并明确标注“当前身份无权读取该妙记 AI 产物”
3. 不要把这个 ACL-blocked 的直连妙记重复放入“时间匹配妙记”
4. 只有在无可读 `note_doc_token` 可降级时，才视为 hard fail

### 4. 去重原则

- 同一日程内，直连妙记不重复出现在时间匹配列表
- 同一 `minute_token` 若匹配多个日程，可以分别出现，但要保留 overlap 分钟数
- 同一 `meeting_id` / `note_doc_token` / `minute_token` 在全局结果里只保留一个 canonical 记录

## Output template

最终输出必须同时包含：

1. **时间顺序 summary**
2. **按日期展开的详细 memo**
3. **过程中我们最关心的问题**
4. **创始团队在过程中想法的变化，体现的优缺点**
5. **当前最值得关注和跟进的点**
6. **没有日程匹配的妙记 / 会议纪要**
7. **没有妙记和会议纪要匹配的日程**

推荐文档结构：

```markdown
<callout emoji="📝" background-color="light-blue">
<项目> / <创始人> 相关会议 memo
</callout>

## 时间顺序 Summary
- ...

## 按日期展开
### YYYY-MM-DD | 标题
- 日程
- 纪要
- 妙记
- 摘要：一句话总结会议主要内容
  - 要点 1
  - 要点 2
- Follow-ups:
  - 待办 1
  - 待办 2

## 过程中我们最关心的问题
- ...

## 创始团队在过程中想法的变化，体现的优缺点
- ...

## 当前最值得关注和跟进的点
- ...

## 无日程匹配的妙记 / 会议纪要
- ...

## 无妙记和会议纪要匹配的日程
- ...
```

## Document creation and delivery

### 1. 创建飞书文档

优先级：

- 用户指定 `wiki-node`
- 用户指定 `wiki-space`
- 否则 `my_library`

使用：

```bash
lark-cli docs +create --as user --title "<title>" --markdown "<memo>"
```

### 2. 发送文档链接

默认把文档链接发送给当前授权用户本人。

优先顺序：

1. `im +messages-send --as user --user-id <current_user_open_id>`
2. 若缺 `im:message.send_as_user` / `im:message`，尝试 `--as bot`
3. 若 bot 也失败，hard fail

消息内容保持极简：

```text
<标题> 已创建
<doc_url>
```

## Final checks

完成前必须确认：

- 相关日程都已经分类
- 无日程匹配的会议纪要 / 妙记已列出
- 无妙记和会议纪要匹配的日程已列出
- 时间顺序 summary 在文档最前
- overlap < 10 分钟的时间匹配已全部排除
- 文档已创建成功
- 文档链接已成功通过消息发送给用户

## If you are unsure

- 不要猜测项目别名或创始人别名，直接问用户
- 不要在权限不完整时继续做部分结果
- 不要在没有重跑失败命令前认为授权已经成功
