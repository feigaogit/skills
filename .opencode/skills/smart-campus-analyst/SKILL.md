---
name: smart-campus-analyst
description: 智慧校园数据分析报告生成。当用户要求分析学校数据、智慧校园指标、学校运行情况、导出校园数据报告时使用。自动通过 BDS MCP 获取数据，解析 JWT 确定学校，生成结构化分析报告。
---

# 智慧校园数据分析

## S - Scope
- 目标：基于 BDS MCP 平台数据，为学校生成智慧校园数据分析报告。
- 数据源：BDS MCP（`bds_list_metrics`、`bds_query_metric`、`bds_list_dimensions`、`bds_list_capabilities`）。
- 认证：从环境变量 `BDS_MCP_TOKEN` 读取 JWT，所有 BDS 工具调用传入该 token。
- 产出：结构化的 Markdown 分析报告 + **强制询问**可视化图表页面（HTML 单页，有数据时必须问）。

## P - Process

### 1) 认证与学校识别（静默完成，禁止输出）
- 从 `$BDS_MCP_TOKEN` 环境变量获取 JWT token。
- 静默解码 JWT payload（base64 解码中间段），提取 `filters` 中的学校名称、检查过期时间。**解码过程和结果一律不输出给用户。**
- 所有后续 BDS 调用统一传入该 token。
- **禁止输出的内容**：token 原文、JWT 解码结果、用户名（uname）、过期时间（exp/iat）、学校名称提取过程。学校名称仅用于报告标题，不在流程中展示。

### 2) 获取可用指标
- 调用 `bds_list_metrics`（传入 token）获取全部指标列表。
- 按 `category` 分组整理：
  - `基本情况`：教职工数、学生数、专业数、班级数、学期等
  - `学生服务`：缴费、请假、进出校、操行、实习等
  - `教务服务`：排课、调代课、考勤、成绩等
  - `行政办公`：考勤、工资、公告、公文等
  - `公共`：登录人数等

### 3) 确定分析范围
- 根据用户意图选择要查询的指标子集：
  - 用户说"整体情况" / "基本数据" → 查询所有 `基本情况` + `公共` 指标
  - 用户说"学生" → 查询 `学生服务` + 部分 `基本情况`
  - 用户说"教务" → 查询 `教务服务`
  - 用户说"行政" / "办公" → 查询 `行政办公`
  - 用户没有明确范围 → 默认查询全部类别的核心指标（每个类别 3-5 个）
- 时间范围：用户未指定时，默认近 30 天。使用 ISO 8601 / RFC3339 格式（带时区 `+08:00`）。
- 如用户需要按维度拆分（如按专业、班级），先调用 `bds_list_dimensions` 获取可用维度。
- **学校隔离**：JWT token 的 `filters` 中已限定学校范围，`bds_list_dimensions` 调用仅用于获取当前学校的维度（如专业、班级等），**禁止获取或展示其他学校名称**。报告中只出现当前 filters 中的学校信息。

### 4) 并行查询数据
- 所有指标查询并行发起（`bds_query_metric` 调用之间无依赖）。
- 每个查询指定 `metric_id`、`start`、`end`、`filters`（如需）、`groups`（如需）。
- 查询参数示例：
  ```json
  {
    "metric_id": "019b1124-a51c-7b12-97ba-0bd080a4ef23",
    "start": "2026-05-30T00:00:00+08:00",
    "end": "2026-06-29T23:59:59+08:00"
  }
  ```

### 5) 数据分析
对每个指标的数据点：
- **最新值**：最近一个数据点的值
- **趋势**：对比最早和最晚值，标注上升/下降/稳定
- **波动**：是否有突增突降（单日变化 >10% 视为异常）
- **峰值**：期间内的最大值和最小值

### 6) 生成报告
报告结构如下，用 Markdown 输出：

```
# {学校名称} 智慧校园数据分析报告
**分析时段**：{start} ~ {end}
**报告生成时间**：{now}

## 一、总体概览
一句话总结 + 核心数据摘要表（最新值 + 趋势方向）

## 二、分项分析
### 2.1 师生规模
### 2.2 教学运行
### 2.3 学生服务
### 2.4 行政办公

## 三、异常关注
列出波动异常的数据点，附上可能的原因推测

## 四、建议
基于数据趋势给出 2-3 条简短建议
```
报告最后一行必须加：`> 💡 正在为您准备可视化图表，请稍候…`

报告输出完毕 → **立即执行步骤 7（询问图表），不得在此结束流程。禁止写任何结束语或总结，下一个动作只能是在对话中直接询问用户是否需要图表。**

### 7) 🔴 生成可视化图表（有数据时必须执行，不可跳过）
- 文字报告输出完毕后，**若实际获取到了有效数据**，直接在对话中用文本询问用户："是否需要生成可视化分析图表？"
- 用户说"不需要" / "不用" / "算了" → 流程结束。
- 用户说"需要" / "生成图表" / "好" / "可以" / "行" → 执行 7.1。
- **若无有效数据（全部指标为空或失败），跳过此步骤，不询问。**

#### 7.1 生成 HTML 页面
- 基于已查询到的指标数据，生成一个 **HTML 单页统计分析页面**，保存至 `/workspace/.xcode/artifacts/smart-campus-report.html`。
- **技术栈**：纯 HTML + Chart.js（CDN 引入，不本地安装依赖）+ CSS Grid/Flexbox 布局。
- **图表类型选择**：
  - 时序数据（如日登录量、日考勤）→ 折线图 / 面积图
  - 分类汇总（如部门数、教学单位数）→ 柱状图
  - 占比/总量概览 → 数字卡片 + 环形图
- **页面风格**：
  - 深色科技主题（背景色 `#0a0e1a` 或类似深蓝黑）。
  - Canvas 粒子动画背景（粒子缓慢浮动、连线效果，纯 JS 实现，无外部依赖）。
  - 标题区域使用渐变文字或发光效果。
  - 卡片半透明毛玻璃效果（`backdrop-filter: blur`），圆角边框带微弱发光。
  - 简洁大气，不花哨，信息层级分明。
- **响应式**：H5 移动端适配（Grid 自动换行、图表自适应、字体缩放）。

#### 7.2 启动服务并返回链接
- HTML 页面保存到 `/workspace/.xcode/artifacts/smart-campus-report.html`。**多次生成直接覆盖原文件。**
- **使用 tmux 持久化启动静态服务**，固定端口 **8080**：
  ```bash
  # 先关闭已有服务（如果存在）
  tmux kill-session -t campus-report 2>/dev/null
  # 新建 tmux session 启动服务
  tmux new-session -d -s campus-report "python3 -m http.server 8080 --directory /workspace/.xcode/artifacts"
  ```
- 端口固定 8080，不自动分配新端口。多次生成复用同一端口。
- **外网链接**：从 `WORKSPACE_PORT_URL_TEMPLATE` 读取模板，端口固定替换为 `8080`，生成链接。
- **返回格式**：仅返回一条可点击链接，不描述图表内容：`[点击查看图表](https://{id}-8080.code.yzl.ltd:6443/smart-campus-report.html)`

### 7.3 验证图表页面（生成后必做，不可跳过）
- HTML 文件写入后，执行以下验证：
  1. 用 Node.js 做 HTML 结构校验：`node -e "const fs=require('fs');const h=fs.readFileSync('/workspace/.xcode/artifacts/smart-campus-report.html','utf8');const n=(h.match(/new Chart\(/g)||[]).length;console.log(n>=4?'PASS: '+n+' charts':'FAIL: only '+n+' charts',h.includes('</html>')?'HTML ok':'HTML BROKEN')"`
  2. 用 `curl -sI https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js | head -1` 确认 CDN 可达（返回 `HTTP/2 200`，非 404/502）。
  3. 启动静态服务后，用 `curl -s http://localhost:8080/smart-campus-report.html | head -5` 确认页面可访问（返回 HTTP 200 + HTML 内容）。
- 如任一步失败，分析原因、修复 HTML、重新验证，直到全部通过。
- 全部通过后 → 执行 7.4 返回链接。

### 7.4 最终输出格式（严格执行）
- 🔴 **生成图表后的 final 输出，只允许包含以下内容，禁止额外输出**：
  1. 一条图表链接（可点击 Markdown 格式）
  2. 可选：一行简短数据总结（不超过 30 字），如"教职工 481 人，近 30 天登录活跃，行政考勤正常"
- 🔴 **禁止输出以下内容**：
  - ❌ 流程检查清单（如 `[x] 步骤 N …`）
  - ❌ 图表内容描述（图表已有，无需文字描述）
  - ❌ "报告已生成" 之类的完成语
  - ❌ 任何 Markdown 标题、列表、分隔线等装饰性元素
- 正确示例：`[点击查看图表](https://…/smart-campus-report.html)` 或 `[点击查看图表](https://…/smart-campus-report.html)\n\n> 教职工 481 人，登录活跃，行政考勤正常。`

## 🔴 步骤 7 防遗漏机制

**常见失败模式**：输出完 Markdown 报告后，AI 容易产生"任务已完成"的错觉，直接跳过步骤 7。原因是长文本 Markdown 报告在视觉上构成了一个"完成的终点"。

**强制约束**：
- 在即将输出步骤 6 的 Markdown 报告之前，**先在心里默念三遍"报告结束后必须立即用 question 工具询问用户是否需要图表"**。
- 步骤 6 的 Markdown 报告中，最后一行必须是 `> 💡 正在为您准备可视化图表，请稍候…`（作为收尾，同时也是对自己的提醒）。
- 输出 Markdown 报告后，**禁止做任何其他事情**，下一句话必须是在对话中询问用户是否需要图表。
- 如果发现自己已经开始写"总结"、"报告完成"等结束语，**立刻停止，回头执行步骤 7**。

## 注意事项
- 所有数据来自 BDS MCP，如查询失败标注 `[数据缺失]`。
- token 过期时仅提示"认证已过期，请重新获取 token"，不输出 token 原文或解码信息。
- **静默原则**：认证环节全程不向用户展示 token 原文、JWT payload 解码结果、用户名、过期时间等任何 token 相关信息。获取学校名称后直接进入数据分析，不在输出中提及认证过程。
- **数据缺失处理**：若某指标查询返回空数据或查询失败，报告中对应位置标注 `[数据缺失]`，**不得编造数据或推测数值**。向用户如实说明哪些指标无数据。若全部指标均无数据，直接告知用户"当前时间段暂无数据"，不生成报告。
- 指标中存在重复口径（如 `教师数量`/`在职教职工总数` 数据一致、`学生数量`/`在读学生总数` 数据一致），报告中合并展示，避免冗余。
- 学期末数据可能出现下降，属于正常现象，报告中注明。
- **学校隔离**：JWT filters 已限定学校，禁止调用 `bds_list_dimensions` 获取或展示其他学校名称，报告中仅出现当前学校信息。

## JWT 自签常见陷阱

当用户要求用私钥自行生成 token（而非使用环境变量 `BDS_MCP_TOKEN`）时，以下三个坑必踩其一，务必逐条检查：

### 陷阱 1：密钥格式必须为 PKCS8 PEM

用户提供的私钥通常是裸 SEC1 格式的 base64 字符串（如 `MHcCAQEEI...`）。Node.js `crypto.createSign()` 要求密钥为 PKCS8 PEM 格式。

**正确做法**：先把裸 SEC1 私钥包装成临时 EC PRIVATE KEY PEM 文件，再通过 openssl 转换。临时文件只能写入 `/tmp`，不得写入仓库、日志或技能目录。

```bash
openssl pkcs8 -topk8 -nocrypt -in /tmp/ec-key.pem -out /tmp/pkcs8-key.pem
```

然后用 `fs.readFileSync('/tmp/pkcs8-key.pem', 'utf8')` 读取。

### 陷阱 2：ECDSA 签名必须用 IEEE P1363 格式，而非 DER

Node.js `crypto.createSign().sign()` 的 `dsaEncoding` 默认为 `'der'`，产出的是 ASN.1 DER 编码的签名（约 70-72 字节）。BDS 服务端期望的是 IEEE P1363 格式（r||s 原始拼接，P-256 曲线下固定 64 字节）。

**错误表现**：DER 签名在 JWT 第三段约为 95 个 base64url 字符；IEEE P1363 格式固定 86 字符。如果签名长度不是 86 字符，BDS 会返回 `token signature is invalid`。

**正确做法**：显式指定 `dsaEncoding: 'ieee-p1363'`。

```javascript
const sign = crypto.createSign('SHA256');
sign.update(unsignedToken);
sign.end();
const sigBuf = sign.sign({ key: pem, dsaEncoding: 'ieee-p1363' });
// sigBuf.length 必须等于 64（P-256）
```

### 陷阱 3：Base64 → Base64URL 转换

JWT 要求 base64url 编码（`-` 代替 `+`，`_` 代替 `/`，去除末尾 `=`）。不要直接使用 `buffer.toString('base64url')`（旧版 Node 不支持），手动替换更可靠：

```javascript
const b64url = (buf) =>
  buf.toString('base64')
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=/g, '');
```

### 陷阱 4：服务端公钥未注册

即使本地签名验证通过（`createVerify().verify()` 返回 true），BDS 服务端仍可能返回 `token signature is invalid`。原因是该私钥对应的公钥未在 BDS 后台与 `sub` 账号关联。这是服务端问题，无法在客户端修复——需确认私钥已在 BDS 注册。

### 完整签名示例

```javascript
const crypto = require('crypto');
const fs = require('fs');

const pem = fs.readFileSync('/tmp/pkcs8-key.pem', 'utf8');
const b64url = (s) => Buffer.from(s).toString('base64').replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');

const header = { alg: 'ES256', typ: 'JWT' };
const now = Math.floor(Date.now() / 1000);
const payload = {
  sub: '<bds-registered-subject>',                  // 必须与 BDS 后台注册的一致
  uname: '<bds-user-name>',                         // 与原始 token 保持一致
  tags: [],
  filters: ['school=目标学校名称'],
  exp: now + 86400,
  iat: now,
  jti: crypto.randomUUID()
};

const unsigned = b64url(JSON.stringify(header)) + '.' + b64url(JSON.stringify(payload));
const sign = crypto.createSign('SHA256');
sign.update(unsigned);
sign.end();

// ⚠️ 关键：必须指定 dsaEncoding: 'ieee-p1363'
const sigBuf = sign.sign({ key: pem, dsaEncoding: 'ieee-p1363' });
// 断言签名长度：P-256 曲线签名为 64 字节
console.assert(sigBuf.length === 64, 'Signature must be 64 bytes');

const token = unsigned + '.' + b64url(sigBuf);
// token 第三段（签名）应为 86 字符，若非 86 则格式有误
const sigB64Len = b64url(sigBuf).length;
console.assert(sigB64Len === 86, `Signature base64url must be 86 chars, got ${sigB64Len}`);
```
