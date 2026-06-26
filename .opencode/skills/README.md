# Skills Catalog

这个目录列出当前仓库中的所有 OpenCode/XCode 技能。技能越来越多时，先看这里判断应该用哪个技能。

## 快速索引

| Skill | 分类 | 什么时候用 | 典型触发语 | 交付结果 |
|---|---|---|---|---|
| [`dingding-report-writer`](dingding-report-writer/SKILL.md) | 办公自动化 | 写、生成、确认、发送或配置钉钉日报/周报 | 写日报、写周报、发送日报、确认发送周报、配置日报收件人 | 先生成清单式草稿，用户确认后调用钉钉日志发送 |
| [`frontend-design`](frontend-design/SKILL.md) | 前端设计 | 构建、美化或改造 Web 页面、组件、应用、仪表盘、落地页或 HTML/CSS 布局 | 做个页面、美化前端、设计 dashboard、改造 UI、做落地页 | 生成高辨识度、生产级、非模板化的前端界面，并在可预览时启动服务 |
| [`k0s-installer`](k0s-installer/SKILL.md) | 运维部署 | 在新服务器、裸机、虚拟机、内网或离线环境安装、卸载、重装、修复、验证 k0s/Kubernetes 单节点集群 | 安装 k0s、一键部署 k0s、卸载 k0s、重装 k0s、k0s token、k0s API 地址、airgap、CoreDNS CrashLoopBackOff、conntrack 缺失 | 完成 k0s 部署，处理依赖/DNS/CoreDNS/airgap 问题，GitHub 慢或失败时回退 CDN，输出 API URL 和完整 admin token；或在明确确认后清理 k0s 服务、数据、配置和二进制 |

## 按场景选择

### 办公自动化

使用 [`dingding-report-writer`](dingding-report-writer/SKILL.md)：

- 生成钉钉日报或周报草稿。
- 配置日报/周报收件人。
- 查询历史日报或周报并汇总。
- 用户确认后发送钉钉日志。

### 前端设计

使用 [`frontend-design`](frontend-design/SKILL.md)：

- 创建静态页面、Web 应用、组件或仪表盘。
- 优化页面视觉、布局、动效、响应式和可访问性。
- 需要高辨识度、非模板化的视觉设计。
- 创建静态前端产物后需要自动启动预览服务。

### 运维部署

使用 [`k0s-installer`](k0s-installer/SKILL.md)：

- 在新服务器上安装 k0s。
- 在用户明确确认后卸载或重装 k0s。
- 从官方源选择最新稳定版本，官方不可用、下载超时或速度过慢时回退 CDN。
- 处理内网、离线、airgap bundle、依赖缺失、apt 源不可达。
- 修复宿主机 DNS、CoreDNS CrashLoopBackOff、kube-router 相关网络问题。
- 最终输出 Kubernetes API 地址和 admin token。

## 新增技能时要更新这里

新增 `.opencode/skills/<skill-name>/SKILL.md` 后，请同步补充：

1. “快速索引”表格中的一行。
2. “按场景选择”里的对应分类说明。
3. 根目录 [`README.md`](../../README.md) 的技能表。

技能的 `description` 要覆盖触发词和使用场景，因为它是模型自动选择技能的主要依据。
