---
name: k0s-installer
description: >-
  当用户要求在新服务器、裸机、虚拟机、内网机器或离线环境中安装、部署、卸载、重装、修复或验证 k0s/Kubernetes 单节点集群时，必须使用本技能。
  触发语包括：安装 k0s、一键部署 k0s、卸载 k0s、重装 k0s、删除 k0s、部署 Kubernetes、k0s token、k0s API 地址、airgap、离线安装 k0s、CoreDNS CrashLoopBackOff、conntrack 缺失、k0s 网络不通、kube-router DNS 问题。
  本技能指导代理从环境检查、版本选择、官方下载/CDN 回退、依赖处理、DNS/CoreDNS 修复、启动验证到输出 admin token 与 API URL；当用户明确要求卸载时，安全清理 k0s 服务、数据、配置、二进制和镜像包。
metadata:
  version: 0.1.0
---

# k0s 新服务器完整安装与修复

本技能用于把一台新服务器安装成可用的 k0s 单节点 Kubernetes 集群，也用于在用户明确要求时卸载或重装 k0s。安装最终交付必须包含：k0s API URL、admin token、`k0s status` 正常、所有系统 Pod Running/Completed，尤其是 CoreDNS 正常。

## 目标结果

完成后用下面格式回复用户：

```text
k0s 部署完成
API 地址: https://<server-ip>:<api-port>
Token: <完整 admin-user-token，禁止截断或用 ... 省略>

验证结果:
- k0s status: 正常
- kube-system pods: 全部 Running/Completed
- CoreDNS: Running
- 宿主机 DNS: 可解析外网或指定内网域名
```

Token 是集群管理员凭证。必须完整原样输出给用户，不能截断、脱敏、换成 `...` 或只展示前后片段。展示后提醒用户妥善保存，不要写入仓库、日志或公开文档。

## 首次判断

如果当前会话就在目标服务器上，直接开始检查。否则先向用户要一个必要信息：目标服务器连接方式，例如 SSH 地址、用户名、是否有 root/sudo 权限。不要一次问一长串问题。

优先使用用户已提供的服务器信息、CDN 地址、端口要求和网络环境。只有缺少会阻塞安装的信息时才问。

如果用户要求卸载、删除、清理或重装 k0s，先进入“卸载与重装”流程。卸载会删除集群数据和管理员 token，属于破坏性操作，必须先用普通文本确认用户明确同意。

## 版本与下载源策略

优先选择 k0s 官方最新稳定版本。执行前查询官方来源，而不是凭记忆猜版本：

- GitHub Releases API: `https://api.github.com/repos/k0sproject/k0s/releases`
- 官方下载域名: `https://github.com/k0sproject/k0s/releases/download/<version>/...`

选择规则：

1. 跳过 prerelease、draft、rc、alpha、beta。
2. 选择最新稳定 tag，例如 `v1.xx.x+k0s.x`。
3. 下载二进制：`k0s-<version>-amd64`。
4. 如果需要离线/airgap，再下载：`k0s-airgap-bundle-<version>-amd64`。

如果官方源无法访问、下载失败、目标服务器内网受限，或 GitHub 下载速度过慢/长时间无进度，回退到用户已验证 CDN。不要让安装流程卡在 GitHub 下载上。

```text
https://static.yzl.ltd/common/k0s/k0s-v1.33.1%2Bk0s.1-amd64
https://static.yzl.ltd/common/k0s/k0s-airgap-bundle-v1.33.1%2Bk0s.1-amd64
```

本地离线目录模式也必须支持。若脚本目录中存在：

```text
k0s-v1.33.1+k0s.1-amd64
k0s-airgap-bundle-v1.33.1+k0s.1-amd64
```

优先使用本地文件，不访问 CDN。离线模式必须要求二进制和 bundle 同时存在；缺一个就明确报错。

官方下载尝试必须设置超时和失败回退：

```bash
curl -fL --connect-timeout 10 --max-time 120 --retry 2 -o <tmp-file> <official-url>
```

如果 120 秒内下载不完、连接超时、HTTP 失败、速度长期为 0，立即切换 CDN。切换到 CDN 时要明确输出：

```text
官方源下载过慢或失败，切换到 CDN 包 v1.33.1+k0s.1
```

CDN 回退版本固定使用用户已验证版本：

```text
v1.33.1+k0s.1
```

切换 CDN 后，二进制和 airgap bundle 的版本必须保持一致，不要混用“官方新版二进制 + CDN 旧版 bundle”。

## 环境检查

安装前必须检查这些条件：

```bash
id -u
systemctl --version
command -v curl
command -v tar
command -v modprobe
command -v iptables
command -v conntrack
modprobe overlay
modprobe br_netfilter
free -h
ip route
cat /etc/resolv.conf
```

处理原则：

- 没有 root 权限：停止，要求用户切换 root 或提供 sudo 权限。
- 缺 `systemd`：停止。k0s controller 服务依赖 systemd，不要在不支持的环境强行安装。
- 缺 `overlay` 或 `br_netfilter`：先尝试 `modprobe`。失败则停止并说明需要内核支持。
- 缺 `conntrack`：不要默认静默改源。先判断包管理器是否可用；如果 apt 源不可达，建议用户提供内网源、换阿里云源，或预装 `conntrack`。
- 内存低于 2GB：警告用户，建议扩容后再装。

推荐持久化内核模块：

```bash
cat >/etc/modules-load.d/k8s.conf <<'EOF'
overlay
br_netfilter
EOF
```

## 网络与 DNS 判断

区分三种网络能力：

1. 目标服务器能访问官方 GitHub/k0s release。
2. 目标服务器能访问容器 registry，例如 `registry.k8s.io`。
3. 目标服务器只能访问用户 CDN 或只能离线本地文件。

测试命令：

```bash
curl -fsI --connect-timeout 5 https://github.com/ >/dev/null
curl -fsI --connect-timeout 5 https://registry.k8s.io/v2/ >/dev/null
curl -fsI --connect-timeout 5 https://static.yzl.ltd/common/k0s/ >/dev/null
```

如果 `registry.k8s.io` 可达，优先在线安装，不下载 airgap bundle，避免 CDN 流量。如果 registry 不可达，使用 airgap bundle。

如果域名解析报 `Could not resolve host`，先修宿主机 DNS。Ubuntu Server 默认可使用 `systemd-resolved`：

```bash
systemctl enable systemd-resolved --now
resolvectl dns <default-interface> 114.114.114.114 223.5.5.5
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

默认 DNS 可被用户或公司内网 DNS 覆盖。不要把 `chattr +i /etc/resolv.conf` 作为默认方案，它只是兜底手段。

## 文件安装原则

所有下载和复制都要原子化：先写临时文件，校验成功后再 `mv` 到正式路径，避免中断后留下坏文件。

二进制安装：

```bash
install -m 0755 <k0s-binary> /usr/local/bin/k0s
/usr/local/bin/k0s version
```

版本必须与选择的版本完全一致。机器已有 `/usr/local/bin/k0s` 时，不要盲目跳过，必须先执行 `k0s version` 校验。版本不一致时停止并询问用户是否替换。

airgap bundle 安装：

```bash
mkdir -p /var/lib/k0s/images
mv <bundle-file> /var/lib/k0s/images/image-bundle.tar
tar -tf /var/lib/k0s/images/image-bundle.tar >/dev/null
```

如果 `tar -tf` 失败，说明 bundle 损坏或下载不完整，必须重新下载/复制。

## k0s 配置与安装

生成配置：

```bash
k0s config create > k0s.yaml
```

如果用户要求管理端口，修改 `spec.api.port`。默认端口使用 `6443`。

安装启动逻辑：

1. 如果 `k0s status` 正常，说明已运行，跳过 install/start，继续验证 token 与 pods。
2. 如果 `k0scontroller.service` 已存在但未运行，只执行 `k0s start` 或 `systemctl start k0scontroller`。
3. 如果服务不存在，执行：

```bash
k0s install controller --single -c k0s.yaml
k0s start
```

不要默认执行 `k0s reset`、卸载、删除数据目录。只有用户明确要求重装时才做破坏性操作。

## 卸载与重装

只有当用户明确说“卸载 k0s”“删除 k0s”“清理 k0s”“重装 k0s”或类似含义时才执行本流程。不要因为安装失败就自动卸载；失败时优先诊断和修复。

执行前必须明确告知用户会删除：

- k0s controller 服务。
- Kubernetes 集群数据。
- `/etc/k0s` 配置。
- `/var/lib/k0s` 数据和 airgap 镜像包。
- `/usr/local/bin/k0s` 二进制。
- 当前目录中的 `k0s.yaml`（如果是本次安装生成的配置）。

如果用户只是说“重装”，先询问是否确认删除旧集群数据后重新安装。得到明确确认后再继续。

推荐卸载命令：

```bash
k0s stop || true
k0s reset --force || true
systemctl disable --now k0scontroller 2>/dev/null || true
rm -f /etc/systemd/system/k0scontroller.service
rm -f /etc/init.d/k0scontroller
systemctl daemon-reload 2>/dev/null || true
rm -rf /etc/k0s /var/lib/k0s /var/run/k0s /run/k0s
rm -f /usr/local/bin/k0s
```

如果本次安装脚本在工作目录生成了 `k0s.yaml`，用户确认清理配置后再删除：

```bash
rm -f k0s.yaml
```

卸载后验证：

```bash
command -v k0s || true
systemctl status k0scontroller 2>/dev/null || true
test ! -d /var/lib/k0s && test ! -d /etc/k0s
```

验证通过后回复：

```text
k0s 已卸载
已删除服务、数据目录、配置目录、二进制和镜像包
```

如果用户要求重装，卸载验证通过后回到“版本与下载源策略”和“k0s 配置与安装”流程重新部署，最终仍要输出 API URL 和 admin token。

## 等待与验证

启动后按顺序等待：

```bash
k0s status
k0s kubectl get namespace kube-system
k0s kubectl get pods -A
```

不要因为 pod 列表为空就判断成功。必须看到 kube-system 组件出现，并且所有 Pod 为 `Running` 或 `Completed`。

如果 3-6 分钟内还有 Pod 未就绪，读取具体原因：

```bash
k0s kubectl describe pod -n kube-system <pod-name>
k0s kubectl logs -n kube-system <pod-name> --tail=100
```

常见问题处理：

- `ImagePullBackOff`：registry 不可达，改用 airgap bundle 或确认镜像包已放入 `/var/lib/k0s/images/image-bundle.tar`。
- `CrashLoopBackOff` 且是 CoreDNS：按 CoreDNS 修复流程处理。
- kube-router 异常：检查 `br_netfilter`、`overlay`、`iptables`、`conntrack`。

## CoreDNS 修复

如果 CoreDNS 出现 CrashLoopBackOff，或 CoreDNS ConfigMap 里有以下配置：

```text
forward . /etc/resolv.conf
forward . 127.0.0.53
forward . localhost
```

将其改成明确上游 DNS：

```text
forward . 114.114.114.114 223.5.5.5
```

如果用户有公司内网 DNS，优先使用公司 DNS。

修改后重启 CoreDNS：

```bash
k0s kubectl -n kube-system delete pod -l k8s-app=kube-dns
k0s kubectl -n kube-system get pods -l k8s-app=kube-dns
```

验证 `READY` 为 `1/1`，`STATUS` 为 `Running`，`RESTARTS` 不再增长。

## Admin token

集群可用后创建管理员 ServiceAccount：

```bash
k0s kubectl -n kube-system create sa admin-user --dry-run=client -o yaml | k0s kubectl apply -f -
k0s kubectl -n kube-system create clusterrolebinding admin-user-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:admin-user \
  --dry-run=client -o yaml | k0s kubectl apply -f -
k0s kubectl -n kube-system create token admin-user --duration 168000h
```

如果 token 创建失败，通常是 API server 还没完全就绪。等待 10 秒后重试，最多重试 5 次。

token 输出规则：

- 必须完整输出 `create token` 命令返回的整段 token。
- 不要使用 `...`、`<redacted>`、`省略` 或前后截断展示。
- 如果 token 很长，单独放进 fenced code block，方便用户复制。
- 最终回答中可以提醒“请妥善保存，不要公开”，但不能因此省略 token。

## API URL 输出

API 地址优先从 `k0s.yaml` 的 `spec.api.address` 和 `spec.api.port` 提取。如果没有，则使用默认路由网卡 IP：

```bash
ip route get 1 | awk '{print $7; exit}'
```

输出格式必须是：

```text
https://<ip>:<port>
```

例如：

```text
https://172.16.2.43:6443
```

## 最小可用部署命令

当用户只想要一条命令，且 CDN 脚本已上传时，给出：

```bash
curl -fsSL "https://static.yzl.ltd/common/k0s/install-k0s.sh?v=<version-or-date>" | sudo bash
```

如果 CDN 缓存了旧脚本，使用新的 query string 绕过缓存，例如 `?v=20260626-1`。

## 完成前检查清单

交付前必须确认：

- `k0s status` 正常。
- `k0s kubectl get pods -A` 里所有 Pod 为 `Running` 或 `Completed`。
- CoreDNS `1/1 Running`。
- 宿主机能解析必要域名，或确认目标是纯离线环境。
- 已输出 API URL。
- 已完整输出 admin token，未截断、未脱敏、未使用 `...`。
- 没有把 token、私有 IP 以外的敏感信息写入仓库文件。
