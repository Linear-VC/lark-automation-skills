# lark-automation-skills

基于 [larksuite/cli](https://github.com/larksuite/cli) 的自动化 Skill 扩展，覆盖会议汇总、项目综述、产品口碑分析、项目背景调研等场景，通过 AI Agent 自动检索、整理并推送到飞书文档或对话。

## Skills

| Skill | 说明 |
|---|---|
| [meeting-report](meeting-report/) | 按时间范围汇总所有会议，生成 Meeting Digest |
| [project-synthesis](project-synthesis/) | 按项目/创始人维度汇总相关会议，生成项目 Memo |
| [产品评价调研](产品评价调研/) | 全网检索产品用户评价，按六大维度生成飞书 docx 报告 |
| [项目信息查询](项目信息查询/) | 全网检索项目介绍、融资、团队和联系方式，5 段格式直接返回 |

---

## meeting-report — 会议报告

按**时间范围**汇总飞书日历日程、会议纪要、妙记和逐字稿，生成结构化的 Meeting Digest 文档并通过 IM 推送链接。

### 功能

- 默认汇总过去 24 小时内的所有会议，支持自定义时间范围
- 交叉核对四个数据源：日历日程、会议纪要、妙记、逐字稿
- 自动匹配日程与妙记（直连匹配 + 时间重叠匹配，overlap >= 10 分钟）
- 时间重叠仅产生候选，需通过主题一致性校验才合并
- 同一 minute_token 全局去重，避免重复出现
- 摘要提取优先级：会议纪要 > 妙记 AI 产物 > 逐字稿
- 权限不足时自动降级，无法降级则 hard fail
- 自动创建飞书文档并通过 IM 发送链接

### 输出结构

```
Meeting report
├── 时间顺序 Meeting Digest
│   └── 每条包含：匹配类型 + 日程/纪要/妙记链接 + 摘要 + Follow-ups
└── 无妙记和会议纪要匹配的日程
```

### 使用示例

```
帮我汇总一下过去 24 小时的会议
整理一下昨天的所有会议纪要
生成本周的 meeting report
把会议报告放到 XXX 知识库
```

---

## project-synthesis — 项目会议综述

按**项目/创始人**维度，汇总飞书日历日程、会议纪要和妙记，生成结构化的项目 Memo 文档并通过 IM 推送链接。

### 功能

- 以项目名/公司名/创始人名为关键词，聚合所有相关会议
- 默认查询过去 3 个月，按月切片查询，支持自定义时间范围
- 自动规范化关键词（中英文、别名、大小写变体），逐词分别查询后去重
- 交叉核对日历日程、会议纪要、妙记三个数据源
- 直连匹配 + 时间重叠匹配（overlap >= 10 分钟）
- 摘要提取优先级：会议纪要 > 妙记 AI 产物 > 逐字稿
- 权限不足时自动降级，无法降级则 hard fail
- 额外生成趋势分析：核心关注点、团队想法变化、当前最值得跟进的点
- 自动创建飞书文档并通过 IM 发送链接

### 输出结构

```
<项目/创始人> 相关会议 memo
├── 时间顺序 Summary
├── 按日期展开的详细内容（摘要 + Follow-ups）
├── 过程中我们最关心的问题
├── 创始团队想法的变化，体现的优缺点
├── 当前最值得关注和跟进的点
├── 无日程匹配的妙记/会议纪要
└── 无妙记和会议纪要匹配的日程
```

### 使用示例

```
帮我整理一下和 ProjectX 相关的所有会议
汇总过去三个月和张三聊过的所有内容
把和 ABC 公司相关的会议做个综述
整理 XX 项目的会议 memo，放到研究知识库
```

---

## 产品评价调研 — 全网产品口碑分析

输入产品名称或链接（Kickstarter / 官网 / Amazon 等），全网检索用户评价和评测者观点，按六大维度归类，生成飞书 docx 报告。

### 功能

- 自动补全产品全名、品牌、品类、价格区间和众筹情况
- 重点深挖 Kickstarter / YouTube / Reddit / Amazon 四个高密度评价渠道
- 按六大维度归类：产品好评、差评、未满足需求、付费点、不购买点、各平台整体风向
- 每条观点标注**极高 / 高 / 中 / 低**四档强度，依据独立信息源数量和跨平台共识度判定
- 平台风向按 Reddit / YouTube / Kickstarter / Amazon / 专业媒体 / 社交平台 拆分
- 关键句飞书高亮（`<text bgcolor="light-blue">…</text>`）
- 自动写入飞书 docx，写入失败兜底返回本地 markdown 路径

### 输出结构

```
用户评论分析｜<产品名>
├── 产品基本信息（200-400 字自然叙述简介）
└── 用户评论分析
    ├── 产品好评（极高/高/中/低 四档）
    ├── 产品差评
    ├── 未满足用户需求的地方或功能
    ├── 用户的付费点
    ├── 用户不购买点
    └── 全网各平台用户对这款产品的整体评价风向
```

### 使用示例

```
调研一下 RingConn Gen2 的用户评价
分析这款产品的口碑：https://www.kickstarter.com/projects/...
看看用户怎么评价 Flowtica
做个 XX 产品的竞品评价分析
```

---

## 项目信息查询 — 团队、融资、联系方式

输入产品名 / 项目名 / 公司名 / 文章 / 链接，全网检索公司定位、创始人和核心成员、融资历史、多渠道联系方式，**直接在对话中返回 5 段编号格式**（不写文件、不写飞书）。

### 功能

- 中英文项目通吃：中文走 36kr / 虎嗅 / 创业邦 / 量子位 / IT桔子，英文走 TechCrunch / Crunchbase / LinkedIn
- 团队信息深挖：教育背景 + 完整职业轨迹 + 创业经历 + 技术成就（每位 80-150 字）
- 融资历史串联：种子→A→B 全部按时间顺序展开，分清领投/跟投，金额单位标清
- 公司层面四项必查：官网 / LinkedIn / Twitter / 邮箱
- 个人层面联系方式：LinkedIn / Twitter / 邮箱 / GitHub / 微信公众号 / 脉脉 / ProductHunt / 个人站点
- 严格防编造：邮箱不从域名+姓名拼凑，低置信度内容不输出，未找到的整行省略
- 搜索预算控制：单项目 ≤12 次 web_search，并行执行避免串行等待

### 输出结构（5 段编号格式，对话直出）

```
1. 项目名称：{中文名}（{英文名}）｜ 官网：{URL}
2. 项目介绍：{2-3 句自然语言}
3. 融资情况：{按时间顺序串联所有轮次}
4. 团队介绍：{每位成员 80-150 字背景}
5. 联系方式：{个人在前，公司在末，未找到的整行省略}
```

### 使用示例

```
查一下言创万物这家公司的团队和融资
找一下这个项目的创始人联系方式：https://verdent.ai
谁做的 Flowtica，怎么联系到他们
帮我看看 RingConn 团队背景和投资方
```

---

## 安装

> **关键：** `meeting-report/`、`project-synthesis/`、`产品评价调研/` 这三个 Skill 在 SKILL.md 中引用了相对路径（如 `../lark-shared/`、`../lark-doc/`），安装时必须放到与 [larksuite/cli](https://github.com/larksuite/cli) 的 skills 同级目录下，否则相对路径无法解析。
>
> `项目信息查询/` 没有依赖飞书 Skill，单独安装在任意 skills 目录都能用。

### Agent 自动安装

将本仓库地址发给你使用的 AI Agent（Claude Code、Cursor、Windsurf、Cline 等均可）：

```
帮我安装这个仓库里的 Skill：https://github.com/Linear-VC/lark-automation-skills
```

Agent 会自动 clone 仓库，将 Skill 目录安装到与 larksuite/cli skills 同级的位置。

### 手动安装

```bash
git clone https://github.com/Linear-VC/lark-automation-skills.git

# 复制到与 larksuite/cli skills 同级目录
# 例如 Claude Code：
cp -r lark-automation-skills/meeting-report ~/.claude/skills/
cp -r lark-automation-skills/project-synthesis ~/.claude/skills/
cp -r lark-automation-skills/产品评价调研 ~/.claude/skills/
cp -r lark-automation-skills/项目信息查询 ~/.claude/skills/

# 其他 Agent 参考对应的自定义指令 / Skill 文档
```

## 前置依赖

`meeting-report` / `project-synthesis` / `产品评价调研` 需要 [larksuite/cli](https://github.com/larksuite/cli) 中的以下基础 Skill 已安装：

| Skill | 用途 | 被哪些 Skill 使用 |
|---|---|---|
| lark-shared | 认证、权限管理、身份切换 | meeting-report / project-synthesis / 产品评价调研 |
| lark-calendar | 读取日历日程 | meeting-report / project-synthesis |
| lark-vc | 查询会议记录、获取纪要和录制 | meeting-report / project-synthesis |
| lark-minutes | 搜索和读取妙记 | meeting-report / project-synthesis |
| lark-doc | 创建飞书文档、读取文档内容 | meeting-report / project-synthesis / 产品评价调研 |
| lark-im | 发送消息链接 | meeting-report / project-synthesis |

`项目信息查询` 仅依赖 Agent 内置的 web_search / web_fetch，无飞书依赖。

## 所需飞书权限

涉及飞书 API 的 Skill 首次运行时会自动检测并申请缺失权限。

**读取：**
`calendar:calendar:read`, `calendar:calendar.event:read`, `vc:meeting.search:read`, `vc:meeting.meetingevent:read`, `vc:note:read`, `vc:record:readonly`, `minutes:minutes.search:read`, `minutes:minutes:readonly`, `minutes:minutes.artifacts:read`, `minutes:minutes.transcript:export`, `docx:document:readonly`, `drive:drive.metadata:readonly`

**写入/发送：**
文档创建相关 scope, `im:message.send_as_user`, `im:message`

## 目录结构

```
lark-automation-skills/
├── README.md
├── meeting-report/
│   └── SKILL.md
├── project-synthesis/
│   └── SKILL.md
├── 产品评价调研/
│   └── SKILL.md
└── 项目信息查询/
    └── SKILL.md
```

## License

MIT
