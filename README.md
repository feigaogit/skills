# 钉钉日报/周报自动化工作区

本项目是一个基于 OpenCode/XCode 的自动化工作区，用于生成、确认并发送钉钉日报和周报。核心能力由自定义技能 `dingding-report-writer` 提供，配合 XCode 定时任务与钉钉日志 MCP 工具完成日报草稿生成、收件人配置、模板解析和日志发送。

## 项目内容

- `.opencode/skills/dingding-report-writer/SKILL.md`：钉钉日报/周报写作与发送技能说明，定义生成草稿、确认发送、配置读取和钉钉日志调用规则。
- `.xcode/dingding-report-config.yaml`：日报和周报的模板名称、收件人 `userId`、来源标识配置。
- `.xcode/scheduled-tasks.yaml`：XCode 定时任务配置，目前包含每日 18:00 生成钉钉日报草稿的任务。
- `.opencode/package-lock.json`：OpenCode 插件依赖锁定文件。

## 核心流程

### 日报

1. 读取 `.xcode/dingding-report-config.yaml` 中的 `daily` 配置。
2. 按 `Asia/Shanghai` 时区查询当天创建和归档的 workspace sessions。
3. 生成清单式日报草稿。
4. 等待用户用自然语言确认发送。
5. 确认后调用钉钉日志 MCP 创建日志，且 `toChat` 固定为 `false`。

### 周报

1. 读取 `.xcode/dingding-report-config.yaml` 中的 `weekly` 配置。
2. 查询本周已发送的日报。
3. 基于日报内容汇总生成清单式周报草稿。
4. 等待用户确认后再发送。

## 当前配置

日报配置：

- 模板名称：`日报`
- 接收人：已配置 1 个 `userId`
- 来源标识：`opencode-daily-report`

周报配置：

- 模板名称：`周报`
- 接收人：暂未配置
- 来源标识：`opencode-weekly-report`

## 定时任务

当前已配置任务：`dingding-daily-report-draft`。

- 名称：钉钉日报草稿
- 触发时间：`0 18 * * 1-6`，即每周一到周六 18:00
- 行为：生成当天钉钉日报草稿并请求用户确认，确认前不会发送钉钉日志
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
