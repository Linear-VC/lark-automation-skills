# lark-automation-skills

基于 [larksuite/cli](https://github.com/larksuite/cli) 的自动化 Skill 扩展，通过 AI Agent 自动汇总飞书会议信息、生成结构化报告并推送到飞书文档。

## Skills

| Skill | 说明 |
|---|---|
| [meeting-report](meeting-report/) | 按时间范围汇总所有会议，生成 Meeting Digest |
| [project-synthesis](project-synthesis/) | 按项目/创始人维度汇总相关会议，生成项目 Memo |

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

## 安装

> **关键：** 安装时需将 `meeting-report/` 和 `project-synthesis/` 放到与 [larksuite/cli](https://github.com/larksuite/cli) 的 skills 同级目录下，以确保 SKILL.md 中的相对路径引用（如 `../lark-shared/`、`../lark-calendar/`）能正确解析。

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

# 其他 Agent 参考对应的自定义指令 / Skill 文档
```

## 前置依赖

需要 [larksuite/cli](https://github.com/larksuite/cli) 中的以下基础 Skill 已安装：

| Skill | 用途 |
|---|---|
| lark-shared | 认证、权限管理、身份切换 |
| lark-calendar | 读取日历日程 |
| lark-vc | 查询会议记录、获取纪要和录制 |
| lark-minutes | 搜索和读取妙记 |
| lark-doc | 创建飞书文档、读取文档内容 |
| lark-im | 发送消息链接 |

## 所需飞书权限

两个 Skill 首次运行时会自动检测并申请缺失权限。

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
└── project-synthesis/
    └── SKILL.md
```

## License

MIT
