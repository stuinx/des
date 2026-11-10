# des

`des` 是一个独立的轻量落地 / 中转脚本，适用于 **128MB 内存 / 1GB 硬盘 / NAT IPv4 的 Alpine LXC**。整个项目是一个 POSIX sh 脚本，服务由 OpenRC 管理，可以配合 de_GWD client 使用。

每台机器只承担一种角色：

| 角色 | 功能 | 常驻进程 | 内存 |
| --- | --- | --- | --- |
| **落地** | VLESS + XHTTP + TLS 主线节点（de_GWD client 可直连）；55 附加协议：Reality + Vision / SOCKS5 | Xray | 约 20–45 MB |
| **中转** | 44 端口转发（nftables 内核转发 / Xray 用户态，自动选择）；22 RproxyS | Xray（只有 nft 规则时不运行） | 0–40 MB |

- Xray 直接处理 TLS，不需要 nginx，也不提供 `/dq`。
- 本机不申请证书，使用其他 server 生成的证书，上传后导入。
- Xray 按内存上限自动设置 `GOMEMLIMIT`（128MB 约 44MiB），只解压 `xray` 一个文件，不下载 geodata。安装包的下载和解压放在磁盘上，避免 tmpfs 挤占内存。
- 端口转发参考 singbox-lite：能用 nftables 时走内核 DNAT，否则降级为 Xray 用户态（`dokodemo-door`）。域名目标由 crond 每 5 分钟重新解析一次。
- 所有修改先写入 `/etc/des/state.json`，再重新生成配置。配置要先通过 `xray -test` 和 `nft -c` 校验才会替换，失败时自动回滚。
- 依赖：`curl jq openssl ca-certificates`（中转可用时还会装 `nftables`）。

## 安装

```sh
apk add curl
curl -fsSL https://raw.githubusercontent.com/stuinx/des/main/des -o /usr/local/bin/des
chmod +x /usr/local/bin/des && des
```

- 也可以 scp 上传 `des` 后执行 `sh des`，安装时脚本会把自己复制到 `/usr/local/bin/des`。
- GitHub 连接不畅时，可以设置 `GH_PROXY=https://ghproxy.net/ des`，或把 `Xray-linux-64.zip` 先放到 `/root/` 离线安装。
- `des update` 默认从本仓库 main 分支更新，可用环境变量 `DES_URL` 覆盖。

## 证书

在已有证书的 server（例如 de_GWD server）上执行。`-P` 后填 NAT 映射的 SSH 端口：

```sh
scp -P 22022 /var/www/ssl/de_GWD.cer /var/www/ssl/de_GWD.key root@<LXC公网IP>:/root/
```

导入时会检查：

- 证书与私钥是否匹配
- 是否过期
- 是否自签（de_GWD client 设置了 `allowInsecure=false`，自签证书无法连接）
- 证书链是否完整
- 域名是否被覆盖

**自动同步续签后的证书**：在 server 的 crontab 中加入下面一行。证书没有变化时 des 不会重启服务。

```cron
0 4 * * * scp -P 22022 /var/www/ssl/de_GWD.cer /var/www/ssl/de_GWD.key root@<LXC>:/root/ && ssh -p 22022 root@<LXC> 'des cert /root/de_GWD.cer /root/de_GWD.key'
```

> 建议使用通配证书（`example.com` + `*.example.com`）。落地用 `land.example.com`，中转的 RproxyS 用 `r.example.com`，同一张证书两边都能用。

## 落地

安装时选 **1) 落地**，按提示填写：

- 节点域名：作为 SNI，必须被证书覆盖。
- client 连接地址：可以填解析到 NAT 公网 IP 的域名，也可以直接填 IP。
- 端口：分别填本机监听端口和 NAT 公网端口，也就是 NAT 面板里的「内网端口 → 外网端口」。
- UUID 和 path。

**de_GWD client 中填写**（`des node` 可以随时打印）：

```
domain : land.example.com:20001      ← 公网端口不是 443 时要带端口
tls    : land.example.com            ← 连接地址填 IP 时，在节点页的 tls 按钮中填写
UUID   : <uuid>
path   : /<uuid 前 8 位>
```

client 的 DoH 不再指向落地，请改用其他 DoH 服务。

**55 附加协议**：

- Reality + Vision 默认回落到本机主节点端口，借用自身证书。
- SOCKS5 带用户名和密码认证。
- 每个协议需要单独占用一个 NAT 端口。

## 中转

安装时选 **2) 中转**。

### 44 端口转发

```
本机监听 10001（公网 30001）  ->  land.example.com:20001   [tcp] [nft]
```

- de_GWD 主线节点是 XHTTP over TLS，L4 透传时选 **TCP** 即可。TLS 在 client 与落地之间端到端建立，中转机看不到明文。
- client 节点页：`domain` 填 `中转IP:30001`，**`tls` 填落地证书域名**，UUID 和 path 与落地相同。
- `nft` 引擎要求 LXC 具备 `NET_ADMIN` 并能开启 `ip_forward`，否则自动改用 `xray` 引擎。

### 22 RproxyS

- 需要证书，并且隧道域名（如 `r.example.com`）要解析到中转机的 NAT 公网 IP。
- de_GWD client 的 RproxyC 中，隧道地址填 `r.example.com:<公网端口>`，并填写隧道 UUID。
- 映射端口：RproxyC 的 `extPort` 填**本机监听端口**，外部访问者连接对应的 NAT 公网端口。

## 命令

```
des              菜单（1 安装 / 3 证书 / 5 主节点 / 55 附加协议 / 44 端口转发 / 22 RproxyS / 11 节点 / 7 状态）
des node         打印节点信息
des status       服务、内存、证书状态
des cert C K     导入证书
des apply        按状态文件重建配置
des update-xray  更新 Xray（新核心先用现有配置测试，不兼容则放弃）
des update       更新 des 脚本
des uninstall    卸载
```

## 文件位置

| 路径 | 内容 |
| --- | --- |
| `/usr/local/bin/des` | 脚本 |
| `/usr/local/bin/xray` | Xray 核心 |
| `/etc/des/state.json` | 唯一的状态源（包含 UUID 和密钥，权限 600） |
| `/etc/des/xray.json` | 自动生成的 Xray 配置 |
| `/etc/des/nft.conf` | 自动生成的 nft 规则（`table ip des`） |
| `/etc/des/ssl/` | 导入的证书 |
| `/etc/init.d/des-xray`、`/etc/init.d/des-nft` | OpenRC 服务 |
| `/var/log/des-xray.log` | Xray 日志（warning 级别，超过 1MB 自动清空） |

## 限制

- 仅支持 IPv4 目标，不包含 Web UI、WARP、内核优化和 DoH。
- 落地默认屏蔽访问内网 IP 字面量，防止被当作内网跳板。
- 如果 Alpine 自带的 `nftables` 服务启用了 `forward drop` 规则，会拦截 nft 转发，需要自行放行。
