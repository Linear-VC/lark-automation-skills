# meeting-report

按时间范围汇总飞书日历日程、会议纪要、妙记和逐字稿，生成结构化的 Meeting Digest 文档并通过 IM 推送链接。

## 功能

- 默认汇总过去 24 小时内的所有会议，支持自定义时间范围
- 交叉核对四个数据源：日历日程、会议纪要、妙记、逐字稿
- 自动匹配日程与妙记（直连匹配 + 时间重叠匹配，overlap >= 10 分钟）
- 时间重叠仅产生候选，需通过主题一致性校验才合并
- 同一 minute_token 全局去重，避免重复出现
- 摘要提取优先级：会议纪要 > 妙记 AI 产物 > 逐字稿
- 权限不足时自动降级，无法降级则 hard fail
- 自动创建飞书文档并通过 IM 发送链接

## 输出结构

```
📅 Meeting report
├── 时间顺序 Meeting Digest
│   └── 每条包含：匹配类型 + 日程/纪要/妙记链接 + 摘要 + Follow-ups
└── 无妙记和会议纪要匹配的日程
```

## 使用示例

```
帮我汇总一下过去 24 小时的会议
整理一下昨天的所有会议纪要
生成本周的 meeting report
把会议报告放到 XXX 知识库
```

## 所需飞书权限

首次运行时会自动检测并申请缺失权限。

**读取：**
`calendar:calendar:read`, `calendar:calendar.event:read`, `vc:meeting.search:read`, `vc:meeting.meetingevent:read`, `vc:note:read`, `vc:record:readonly`, `minutes:minutes.search:read`, `minutes:minutes:readonly`, `minutes:minutes.artifacts:read`, `minutes:minutes.transcript:export`, `docx:document:readonly`, `drive:drive.metadata:readonly`

**写入/发送：**
文档创建相关 scope, `im:message.send_as_user`, `im:message`
