# lark-automation-skills

基于 [larksuite/cli](https://github.com/larksuite/cli) 的自动化 Skill 扩展，帮助你通过 AI Agent 自动汇总飞书会议信息、生成结构化报告并推送到飞书文档。

## Skills

| Skill | 说明 |
|---|---|
| [meeting-report](skills/meeting-report/) | 按时间范围汇总所有会议，生成 Meeting Digest |
| [project-synthesis](skills/project-synthesis/) | 按项目/创始人维度汇总相关会议，生成项目 Memo |

## 安装

### Agent 自动安装

将本仓库地址发给你使用的 AI Agent（Claude Code、Cursor、Windsurf、Cline 等均可）：

```
帮我安装这个仓库里的 Skill：https://github.com/Linear-VC/lark-automation-skills
```

Agent 会自动 clone 仓库，将 `skills/` 下的 Skill 安装到对应目录中。

### 手动安装

```bash
git clone https://github.com/Linear-VC/lark-automation-skills.git
```

将 `skills/` 下的目录复制到你所使用的 Agent 的 skills 目录。例如：

```bash
# Claude Code
cp -r lark-automation-skills/skills/meeting-report ~/.claude/skills/
cp -r lark-automation-skills/skills/project-synthesis ~/.claude/skills/

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

## 目录结构

```
lark-automation-skills/
├── README.md
└── skills/
    ├── meeting-report/
    │   ├── README.md
    │   └── SKILL.md
    └── project-synthesis/
        ├── README.md
        └── SKILL.md
```

## License

MIT
