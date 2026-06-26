# OpenCode Skills Workspace

本仓库用于维护 OpenCode/XCode 自定义技能。每个技能都放在 `.opencode/skills/<skill-name>/SKILL.md`，通过 frontmatter 里的 `name` 和 `description` 决定触发时机。

## 技能目录

完整目录见：

- [`.opencode/skills/README.md`](.opencode/skills/README.md)

当前已包含：

| Skill | 分类 | 什么时候用 | 典型触发语 | 结果 |
|---|---|---|---|---|
| `dingding-report-writer` | 办公自动化 | 写、生成、确认或发送钉钉日报/周报 | 写日报、写周报、确认发送日报 | 生成清单式草稿，用户确认后发送钉钉日志 |
| `frontend-design` | 前端设计 | 构建、美化或改造 Web 页面/组件/UI | 做个页面、美化前端、设计仪表盘 | 生成有辨识度、可预览的前端界面 |
| `k0s-installer` | 运维部署 | 新服务器安装、卸载、重装、修复或验证 k0s 单节点集群 | 安装 k0s、一键部署 k0s、卸载 k0s、重装 k0s、修复 CoreDNS | 完成 k0s 部署并输出 API URL 与完整 admin token；GitHub 慢或失败时回退 CDN；或清理 k0s 服务、数据、配置和二进制 |

## 仓库结构

```text
.opencode/skills/
├── README.md
├── dingding-report-writer/
│   └── SKILL.md
├── frontend-design/
│   └── SKILL.md
└── k0s-installer/
    └── SKILL.md
```

## 使用方式

在 OpenCode 会话中用自然语言描述任务即可触发对应技能，例如：

```text
写日报
确认发送日报
做一个登录页
美化这个仪表盘
安装 k0s
一键部署 k0s
卸载 k0s
修复 k0s CoreDNS
```

如果不确定该用哪个技能，先看 [技能目录](.opencode/skills/README.md) 中的“什么时候用”和“典型触发语”。

## 维护原则

- 每个技能只负责一个清晰领域，避免把多个无关流程塞进同一个 `SKILL.md`。
- `description` 要写清楚触发场景和典型关键词，这是自动触发的主要依据。
- 用户可变配置不要写进技能目录，例如收件人、token、消息通道账号、服务器密码等。
- 新增技能后，同步更新 `.opencode/skills/README.md` 和本文件的技能表。
- `.xcode/`、`.omo/` 等本地运行状态目录可能包含敏感信息，不要提交其中的私有值。

## 当前本地配置说明

本仓库原先包含钉钉日报/周报自动化配置说明。相关运行配置仍位于 `.xcode/`，属于本地环境状态，已经被 `.gitignore` 忽略。公开文档只描述配置用途，不记录具体 `userId`、渠道账号、访问令牌或其他敏感信息。
