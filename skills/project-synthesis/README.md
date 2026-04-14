# project-synthesis

按项目或创始人维度，汇总飞书日历日程、会议纪要和妙记，生成结构化的项目 Memo 文档并通过 IM 推送链接。

## 功能

- 以项目名/公司名/创始人名为关键词，聚合所有相关会议
- 默认查询过去 3 个月，按月切片查询，支持自定义时间范围
- 自动规范化关键词（中英文、别名、大小写变体），逐词分别查询后去重
- 交叉核对日历日程、会议纪要、妙记三个数据源
- 直连匹配 + 时间重叠匹配（overlap >= 10 分钟）
- 摘要提取优先级：会议纪要 > 妙记 AI 产物 > 逐字稿
- 权限不足时自动降级，无法降级则 hard fail
- 额外生成趋势分析：核心关注点、团队想法变化、当前最值得跟进的点
- 自动创建飞书文档并通过 IM 发送链接

## 输出结构

```
📝 <项目/创始人> 相关会议 memo
├── 时间顺序 Summary
├── 按日期展开的详细内容（摘要 + Follow-ups）
├── 过程中我们最关心的问题
├── 创始团队想法的变化，体现的优缺点
├── 当前最值得关注和跟进的点
├── 无日程匹配的妙记/会议纪要
└── 无妙记和会议纪要匹配的日程
```

## 使用示例

```
帮我整理一下和 ProjectX 相关的所有会议
汇总过去三个月和张三聊过的所有内容
把和 ABC 公司相关的会议做个综述
整理 XX 项目的会议 memo，放到研究知识库
```

## 所需飞书权限

首次运行时会自动检测并申请缺失权限。

**读取：**
`calendar:calendar:read`, `calendar:calendar.event:read`, `vc:meeting.search:read`, `vc:meeting.meetingevent:read`, `vc:note:read`, `vc:record:readonly`, `minutes:minutes.search:read`, `minutes:minutes:readonly`, `minutes:minutes.artifacts:read`, `minutes:minutes.transcript:export`, `docx:document:readonly`, `drive:drive.metadata:readonly`

**写入/发送：**
文档创建相关 scope, `im:message.send_as_user`, `im:message`
