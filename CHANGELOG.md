# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Changed
- 重构 `dingding-report-writer` (v0.2.0)：日报/周报生成流程新增工作连续性交叉对比 —— 自动查询昨天日报的"明天计划"、上周周报的"下周计划"，与今天/本周工作匹配，匹配不上的反问用户进展后归入对应章节，所有条目统一平铺不标注来源。

### Added
- 新增前端界面设计与美化技能 `.opencode/skills/frontend-design/SKILL.md`，定义了高辨识度、非模板化视觉设计原则及静态文件自动预览流程。
- 在 `AGENTS.md` 中注册并记录了新增的 `.opencode/skills/frontend-design/SKILL.md` 文件。
