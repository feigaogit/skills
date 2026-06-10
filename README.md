# 钉钉日报/周报自动化工作区

本项目是一个基于 OpenCode/XCode 的自动化工作区，用于生成、确认并发送钉钉日报和周报。核心能力由自定义技能 `dingding-report-writer` 提供，配合 XCode 定时任务、钉钉日志 MCP 工具和本地运行配置完成日报/周报草稿生成、收件人配置、模板解析和日志发送。

## 项目内容

- `.opencode/skills/dingding-report-writer/SKILL.md`：钉钉日报/周报写作与发送技能说明，定义生成草稿、确认发送、配置读取和钉钉日志调用规则。
- `.opencode/skills/frontend-design/SKILL.md`：前端页面和静态界面设计技能说明，用于需要构建或美化 Web UI 的场景。
- `.xcode/dingding-report-config.yaml`：日报和周报的模板名称、收件人 `userId`、来源标识配置。
- `.xcode/scheduled-tasks.yaml`：XCode 定时任务配置，包含日报草稿、周报草稿和本地总结类任务。
- `.opencode/package-lock.json`：OpenCode 插件依赖锁定文件。

## 核心流程

### 日报

1. 读取 `.xcode/dingding-report-config.yaml` 中的 `daily` 配置。
2. 按 `Asia/Shanghai` 时区查询当天创建和归档的 workspace sessions。
3. 按项目分组生成序号清单式日报草稿，不使用固定的“今日完成/正在进行/明日计划”章节。
4. 等待用户用自然语言确认发送。
5. 确认后调用钉钉日志 MCP 创建日志，且 `toChat` 固定为 `false`。

### 周报

1. 读取 `.xcode/dingding-report-config.yaml` 中的 `weekly` 配置。
2. 查询本周已发送的日报。
3. 基于日报内容按项目分组汇总生成序号清单式周报草稿，不使用固定的周报章节。
4. 等待用户确认后再发送。
5. 如果本周没有已发送日报，需要用户明确同意后才改用 workspace sessions 作为兜底数据源。

## 钉钉 MCP 服务配置

本工作区依赖钉钉 MCP 服务完成联系人解析、日志模板查询、已发送日报查询和日志创建。配置入口参考钉钉 AI 能力中心：[https://aihub.dingtalk.com/#/mcp](https://aihub.dingtalk.com/#/mcp)。

配置时需要确保当前 OpenCode/XCode 会话可用以下钉钉 MCP 能力：

- 联系人能力：按姓名或手机号搜索员工，并解析真实 `userId`。
- 日志能力：查询可用日志模板、按模板名称读取模板详情、查询已发送日志、读取日志详情、创建日志。
- 授权能力：当 MCP 工具提示需要授权时，按提示完成当前钉钉用户授权后再继续。

MCP 服务只负责访问钉钉侧数据和执行日志相关操作；日报/周报的模板名称、接收人 `userId` 和 `ddFrom` 仍保存在 `.xcode/dingding-report-config.yaml`。该配置属于本地运行状态，不应写入技能目录，也不要在公开文档中记录访问令牌、消息通道账号或其他敏感值。

## 当前配置

日报配置：

- 模板名称：`日报`
- 接收人：已配置 2 个 `userId`
- 来源标识：`opencode-daily-report`

周报配置：

- 模板名称：`周报`
- 接收人：已配置 2 个 `userId`
- 来源标识：`opencode-weekly-report`

## 定时任务

当前本地定时任务配置保存在 `.xcode/scheduled-tasks.yaml`，该目录已被 `.gitignore` 忽略。文档只描述任务用途，不记录消息通道账号等私有值。

### 钉钉日报草稿

- 名称：钉钉日报草稿
- 任务 ID：`dingding-daily-report-draft`
- 触发时间：`0 18 * * 1-6`，即每周一到周六 18:00
- 行为：生成当天钉钉日报草稿并请求用户确认，确认前不会发送钉钉日志
- 通知：已启用，成功后通过配置的消息通道通知

### 钉钉周报草稿

- 名称：钉钉周报草稿
- 任务 ID：`dingding-weekly-report-draft`
- 触发时间：`0 20 * * 0`，即每周日 20:00
- 行为：生成本周钉钉周报草稿并请求用户确认，确认前不会发送钉钉日志
- 通知：已启用，成功后通过配置的消息通道通知

### 昨日工单与日报总结

- 名称：昨日工单与日报总结
- 任务 ID：`daily-yesterday-worklog-ticket-summary`
- 触发时间：`0 8 * * *`，即每天 08:00
- 行为：查询昨日新增/完成工单和指定人员昨日提交的钉钉日报，输出简洁的项目分组总结
- 通知：已启用，成功后通过配置的消息通道通知

## 使用方式

在 OpenCode 会话中直接使用自然语言触发技能，例如：

```text
写日报
生成日报
确认发送日报
写周报
配置周报收件人
```

发送前必须先展示草稿，并由用户明确确认。没有确认时，不会调用 `dingding_worklog_create_report`。

## 注意事项

- 不要在技能目录中写入用户可变配置，配置应保存在 `.xcode/dingding-report-config.yaml`。
- 收件人必须通过钉钉联系人工具解析为真实 `userId`，不要手动猜测。
- 周报默认基于本周已发送日报汇总；如果没有日报，需要用户明确同意后才改用 workspace sessions 作为兜底数据源。
- `.xcode/` 已在 `.gitignore` 中忽略，如需共享配置请先确认其中不包含敏感信息。
