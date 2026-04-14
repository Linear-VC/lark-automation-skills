---
name: meeting-report
description: Use when a user wants a time-bounded rollup across Lark calendar events, meeting notes, minutes, and transcripts, ending in a Feishu doc and a delivered message link.
---

# meeting-report

**CRITICAL — 开始前 MUST 先用 Read 工具读取以下技能文档：**
- [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)
- [`../lark-calendar/references/lark-calendar-agenda.md`](../lark-calendar/references/lark-calendar-agenda.md)
- [`../lark-vc/references/lark-vc-search.md`](../lark-vc/references/lark-vc-search.md)
- [`../lark-vc/references/lark-vc-notes.md`](../lark-vc/references/lark-vc-notes.md)
- [`../lark-vc/references/lark-vc-recording.md`](../lark-vc/references/lark-vc-recording.md)
- [`../lark-minutes/references/lark-minutes-search.md`](../lark-minutes/references/lark-minutes-search.md)
- [`../lark-doc/references/lark-doc-create.md`](../lark-doc/references/lark-doc-create.md)
- [`../lark-doc/references/lark-doc-fetch.md`](../lark-doc/references/lark-doc-fetch.md)
- [`../lark-im/references/lark-im-messages-send.md`](../lark-im/references/lark-im-messages-send.md)

## Goal

在给定时间范围内，拉取并交叉核对以下数据源：

- 日历日程
- 会议纪要
- 妙记
- 逐字稿 / transcript

然后生成一份按时间顺序排列的 meeting digest 文档，并把文档链接发给用户。

## Input contract

必填输入：

- 无必填关键词。这个技能按时间范围汇总，不依赖项目名或创始人名。

可选输入：

- 时间范围。若未给出，默认查询**过去 24 小时**，即 `now - 24h` 到 `now`。
- 目标文档位置：
  - 若用户给出 `wiki-space` / `wiki-node` / 目标知识库，按其位置创建文档
  - 否则默认创建到 `my_library`
- 是否发送给当前用户本人。默认是。

## When to hard fail

- 如果关键读取步骤在权限已经申请后仍返回 `missing_scope`、`permission deny`、`HTTP 403`，**通常立即停止**，给出具体失败命令和错误，不要继续产出部分报告。
- 例外：若某个直连 `minute_token` 读取报 `permission deny` / `HTTP 403`，但同一会议或同一日程的 `note_doc_token` 或 `verbatim_doc_token` 仍可正常读取，则**允许继续**；此时必须明确标注“直连妙记存在但当前身份无权读取 AI 产物”，并改用可读的纪要或逐字稿作为该条记录的摘要来源。
- 如果用户要求“发消息给我”，但当前身份对最终消息发送路径无权限，也要 **hard fail**，给出缺少的 scope 或失败命令。

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
2. `vc +search` 小时间窗读取
3. `minutes +search --owner-ids me` 小时间窗读取
4. 找到第一个候选 `minute_token` 后，立即跑一次：
   ```bash
   lark-cli vc +notes --as user --minute-tokens <token> --format json
   ```
5. 找到第一个候选 `note_doc_token` 后，立即跑一次：
   ```bash
   lark-cli docs +fetch --doc <note_doc_token>
   ```
6. 找到第一个候选 `verbatim_doc_token` 后，立即跑一次：
   ```bash
   lark-cli docs +fetch --doc <verbatim_doc_token>
   ```
7. 最终发消息前，验证一次 `im +messages-send --dry-run` 或直接在最终发送阶段处理

### 2. 缺权限时一次性申请

如果任何一步报 `missing_scope`，把当前任务需要的所有缺失 scope 合并成**一次**授权请求，不要一条条补：

```bash
lark-cli auth login --scope "<space separated scopes>"
```

必须把设备授权链接发给用户，并等待用户确认。用户确认后，**必须重跑失败命令**；只有失败命令成功，才能继续。

### 3. 权限到账但资源仍 403

如果 scope 已到账，但读取具体 `note_doc_token` 或 `verbatim_doc_token` 仍然报 `permission deny` / `HTTP 403`，视为当前身份无法完成完整范围。直接 hard fail。

如果 scope 已到账，读取具体 `minute_token` 报 `permission deny` / `HTTP 403`：

- 若同一会议或同一日程存在可读的 `note_doc_token` 或 `verbatim_doc_token`，继续执行，不要 hard fail。
- 必须重跑并确认至少一个替代读取命令成功后，才允许继续。
- 输出中必须标注该 `minute_token` 为“存在但 ACL blocked / 无权读取 AI 产物”。
- 只有在 `minute_token`、`note_doc_token`、`verbatim_doc_token` 都不可读时，才 hard fail。

## Search and matching workflow

### 0. 默认时间窗口

如果用户没有明确给出时间范围，所有 `calendar +agenda`、`vc +search`、`minutes +search` 查询都默认限制在**过去 24 小时**。

只有当用户明确要求其他时间范围（例如“今天”“昨天”“过去 3 天”“某个起止日期”）时，才按用户指定范围执行。

### 1. 拉取时间范围内的全部日程

用 `calendar +agenda` 在目标时间范围内取主日历事件。

不要先按关键词过滤；这个技能默认汇总时间范围内的全部候选会议日程。

### 2. 拉取时间范围内的全部会议记录

直接按时间范围跑 `vc +search`，收集：

- `meeting_id`
- 会议标题
- 开始时间 / 结束时间

如果时间范围超过单次接口上限，必须按月切片查询并去重。

### 3. 为日程批量拉直连纪要和直连录制

对候选 `calendar_event_id` 批量跑：

- `vc +notes --calendar-event-ids`
- `vc +recording --calendar-event-ids`

从中拿：

- `note_doc_token`
- `verbatim_doc_token`
- `meeting_id`
- `minute_token`
- `recording_url`

### 4. 为会议记录补抓纪要和录制

对 `meeting_id` 批量跑：

- `vc +notes --meeting-ids`
- `vc +recording --meeting-ids`

目的：

- 补抓那些**没有被日程接口直接关联到**的会议纪要 / 妙记 / transcript
- 让“无日程匹配的妙记 / 会议纪要”分组完整

### 5. 拉取时间范围内的全部个人妙记

不要只依赖会议搜索结果。需要分别拉：

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

如果时间范围较长，按月切片查询并去重。

### 6. 交叉匹配规则

优先按以下顺序匹配：

1. `calendar_event_id` 直连的 `note_doc_token` / `verbatim_doc_token` / `minute_token`
2. `meeting_id` 对应的纪要、录制、妙记
3. 日程与妙记 / transcript 的时间重叠候选，再做内容一致性校验

### 7. 时间匹配规则

把每个候选日程与全部个人妙记做时间重叠匹配。

**硬规则：排除 overlap < 10 分钟的匹配。**

即：

- `overlap_sec >= 600` 才算时间匹配
- 小于 10 分钟，一律视为不匹配

如果某个妙记已经是该日程的 `direct minute_token`，则不要再重复放入“时间匹配妙记”。

### 8. 时间重叠后的内容校验

时间重叠**只产生候选关系，不足以直接合并**。

对于每个 time-overlap 候选，必须额外检查以下内容是否明显指向同一主题：

- 日程标题
- 纪要标题 / 妙记标题
- `# 总结`
- `artifacts.summary`
- `artifacts.chapters`
- transcript 中的核心议题、产品名、人物名、待办

只有在你对“它们在讨论同一主题 / 同一项目 / 同一场会议”有**明确置信度**时，才允许把 time-overlapped minute / transcript 合并进同一个 digest item。

若只是时间接近，但主题不一致，必须分开列为不同 item。

这个语义校验**只用于 time-overlap 候选**，不用于直连关系。

若某条 time-overlapped minute / transcript 同时匹配多个 calendar item：

- 优先合并进已经存在直连证据的 canonical digest item（例如该 calendar item 已有 `direct note` / `direct transcript` / `direct minute`）
- 若多个候选都只是 overlap 候选、没有更强直连证据，则不要强行选择；保留为 standalone item

### 9. 多段录制 / 多段妙记规则

同一个 calendar entry 可能对应**多段** minutes / transcripts，例如会议中途多次开始 / 停止录制，或会后分段整理。

只要同时满足以下条件，这些多段材料都应该合并进**同一个** calendar digest item：

- 每一段都与该 calendar entry 存在 `overlap >= 10m`
- 每一段都通过了上面的主题一致性校验
- 这些 minutes / transcripts **彼此之间不重叠**

也就是说：

- `一个 calendar item : 多个非重叠的、同主题的 minute/transcript segments` 是合法且预期的
- 这些 segments 不应被拆成多个独立 digest item
- digest 中应把它们作为同一个会议的多段材料一起展示

若多个 time-overlap materials 彼此也重叠，则视为**竞争性候选**而不是天然的多段切片：

- 若有更强直连证据，优先附着到已有 canonical digest item
- 若没有更强直连证据，避免把多个互相重叠的候选都并入同一个 calendar item；应进一步判断哪一个才是正确归属，必要时保留为 standalone item

## Classification rules

### A. Digest 主表

每条记录都应尽量归并成一个 canonical meeting item，并按开始时间排序。

**不要把仅有 calendar entry、但没有任何纪要 / 妙记 / transcript / recording 的日程放进 digest 主表。**

这类记录只允许出现在文末的 `无妙记和会议纪要匹配的日程` 分组中。

每个 item 至少包含以下之一：

- 日程
- 会议纪要
- 妙记
- 逐字稿

并标注匹配类型：

- `direct note`
- `direct minute`
- `direct transcript`
- `time-matched minute`

### B. 去重与唯一归属

- 同一个 `minute_token` 在最终文档中**只能出现一次**
- 若某条妙记已经以 `direct minute` 作为 canonical item 出现在 digest 中，不能再作为任何日程的 `time-matched minute`
- 若某条妙记与多个 calendar entry 都满足 `overlap >= 10m`，但没有任何直连证据（`calendar_event_id` / `meeting_id` / `minute_token` 直连），则视为**歧义匹配**
- 若某条妙记与某个 calendar entry 只有时间重叠，但主题校验不通过，则视为**不匹配**
- 若多个 `minute_token` / transcript segments 指向同一个 calendar item，且它们彼此不重叠并且主题一致，则允许它们共同归属于同一个 digest item
- 对歧义匹配：
  - 不要把这条妙记映射给任何 calendar entry
  - 保留该妙记自身作为 standalone digest item
  - 相关 calendar entry 若无其他直连纪要 / transcript / recording，放到 `无妙记和会议纪要匹配的日程`

### C. 无妙记和会议纪要匹配的日程

候选日程若同时满足：

- 无 `note_doc_token`
- 无 `verbatim_doc_token`
- 无 `direct minute_token`
- 无**唯一、无歧义且主题校验通过**的 `time-matched minute (>=10m)`

则进入这一组。

## Summarization workflow

### 1. 时间顺序 Meeting Digest

每个日期块包含：

- 日程标题
- 日程链接
- 直连纪要链接（如果有）
- 直连妙记链接（如果有）
- 时间匹配妙记 / transcript 链接（如果有，可以有多条）
- 摘要：
  - 第一行必须先写 **one-sentence summary**，用一句话概括会议主要内容
  - 随后再用 bullet points 展开会议讨论的主要内容、关键观点或章节
- Follow-ups：
  - 若有待办，统一放在 `Follow-ups:` 下面，用 bullet points 平铺列出
  - 不要把 follow-up 写成和 `Follow-ups:` 同级的一行行 `- Follow-ups: xxx`

digest 中只保留：

- 有 `direct note`
- 有 `direct minute`
- 有 `direct transcript`
- 有**唯一、无歧义且主题校验通过**的 `time-matched minute`

不要把 `no match` 的 calendar-only 日程写入 digest。

### 2. 每条 digest 的摘要内容来源优先级

无论摘要来自纪要、妙记还是逐字稿，最终都必须整理成以下结构：

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

#### 逐字稿 / transcript

若有 `verbatim_doc_token`：

1. `docs +fetch --doc <verbatim_doc_token>`
2. 用于：
   - 交叉验证纪要或妙记摘要是否覆盖关键结论
   - 在无纪要摘要、无妙记 AI 产物时，回退提取 transcript 中最核心的议题和结论

若 `minute_token` 存在但 `vc +notes --minute-tokens <token>` 返回 `permission deny` / `HTTP 403`，则按以下顺序降级：

1. 若同一会议或同一日程有可读的 `note_doc_token`，改用 `docs +fetch --doc <note_doc_token>` 的 `# 总结` / `# 待办`
2. 若无可读纪要但有可读 `verbatim_doc_token`，改用逐字稿提炼摘要
3. 在正文里保留直连妙记链接或 `minute_token` 记录，并明确标注“当前身份无权读取该妙记 AI 产物”
4. 不要把这个 ACL-blocked 的直连妙记重复放入“时间匹配妙记”
5. 只有在无可读 `note_doc_token` 和 `verbatim_doc_token` 可降级时，才视为 hard fail

### 3. 去重原则

- 同一日程内，直连妙记不重复出现在时间匹配列表
- 同一 `minute_token` 不得在 digest 或文末分组中重复出现
- 同一 `minute_token` 若匹配多个日程，且无法确定唯一归属，则保留为 standalone minute item，不映射到任何 calendar entry
- 同一 `minute_token` 若和某个日程时间重叠，但内容不是同一主题，也必须保留为 standalone item
- 同一 `meeting_id` / `note_doc_token` / `verbatim_doc_token` / `minute_token` 在全局结果里只保留一个 canonical 记录

## Output template

最终输出必须同时包含：

1. **时间顺序 Meeting Digest**
2. **无妙记和会议纪要匹配的日程**

推荐文档结构：

```markdown
<callout emoji="🗓️" background-color="light-blue">
Meeting report
</callout>

## 时间顺序 Meeting Digest
### YYYY-MM-DD HH:mm | 标题
- 匹配类型: direct note / direct minute / direct transcript / time-matched minute
- 日程: ...
- 纪要: ...
- 妙记: ... 
- 摘要：一句话总结会议主要内容
  - 要点 1
  - 要点 2
- Follow-ups:
  - 待办 1
  - 待办 2

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

标题建议包含时间范围，例如：

- `Meeting report | 过去 24 小时`
- `Meeting report | <start> - <end>`

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

- digest 列表已按时间顺序排序
- 每条 digest 已完成直连信息和时间匹配信息的交叉核对
- 同一 `minute_token` 未在 digest 或文末分组中重复出现
- 歧义时间匹配没有被强行映射到 calendar entry
- 所有 time-overlap 合并都经过了主题一致性检查
- 无妙记和会议纪要匹配的日程已列出
- overlap < 10 分钟的时间匹配已全部排除
- 文档已创建成功
- 文档链接已成功通过消息发送给用户

## If you are unsure

- 不要猜测时间范围；若用户描述模糊，先显式说明你采用的默认窗口是过去 24 小时
- 不要在权限不完整时继续做部分结果
- 不要在没有重跑失败命令前认为授权已经成功
