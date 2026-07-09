# Oracle Cloud + sing-box 自用部署教程

> 适合场景：个人自用的代理、网站、网盘、小服务。  
> 不要把真实密码、UUID、Reality 公钥、订阅链接提交到公开仓库。

## 1. 推荐服务器配置

Oracle Cloud A1 ARM 免费实例建议：

| 用途 | 推荐配置 |
|---|---|
| 只跑代理 + 小网站 | 1 OCPU + 2GB/4GB |
| 代理 + 网站 + Alist/Cloudreve | 2 OCPU + 4GB/6GB |
| 代理 + 网站 + Nextcloud/数据库/Docker | 2 OCPU + 8GB |
| 不想折腾、资源拉满 | 4 OCPU + 24GB |

个人推荐：

```text
2 OCPU
8GB 内存
80GB~150GB 硬盘
Ubuntu 24.04
预留公网 IP
```

如果已经创建了 150GB 启动盘，可以先不改。Oracle 启动盘通常只能扩容，不能直接缩小；刚创建还没部署东西时，想改小硬盘可以重建实例。

---

## 2. 公网 IP 选择

Oracle 实例页面里可以给 VNIC 添加公网 IP：

```text
实例详情
→ 附加的 VNIC
→ 查看详细信息
→ IP 管理
→ 编辑私有 IP
→ 公共 IP 类型
```

选择建议：

| 类型 | 说明 | 建议 |
|---|---|---|
| 临时公共 IP | 停止/启动一般不变，删除实例/VNIC 后会释放 | 临时测试可以用 |
| 预留公共 IP | 独立存在，可解绑后再绑定到其他实例 | 长期使用推荐 |

长期跑网站、网盘、代理，建议用 **预留公共 IP**。

---

## 3. SSH 登录

Ubuntu 镜像默认用户名：

```bash
ubuntu
```

Oracle Linux 默认用户名：

```bash
opc
```

Windows PowerShell 登录示例：

```powershell
ssh -i "$env:USERPROFILE\.ssh\oracle.key" ubuntu@YOUR_SERVER_IP
```

如果出现：

```text
WARNING: UNPROTECTED PRIVATE KEY FILE!
Permissions are too open
```

说明 Windows 私钥权限太开放。可以执行：

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.ssh"
Copy-Item "C:\path\to\ssh-key.key" "$env:USERPROFILE\.ssh\oracle.key"
icacls "$env:USERPROFILE\.ssh\oracle.key" /inheritance:r
icacls "$env:USERPROFILE\.ssh\oracle.key" /grant:r "$($env:USERNAME):R"
icacls "$env:USERPROFILE\.ssh\oracle.key" /remove:g "Everyone" "Users" "Authenticated Users" "BUILTIN\Users"
```

然后重新登录：

```powershell
ssh -i "$env:USERPROFILE\.ssh\oracle.key" ubuntu@YOUR_SERVER_IP
```

### Xshell 注意点

Xshell 登录失败但 PowerShell 能成功，一般不是服务器问题，而是 Xshell 密钥导入问题。

检查：

```text
主机：YOUR_SERVER_IP
端口：22
用户名：ubuntu
认证方式：Public Key
私钥：选择本地 private key，不是 .pub 公钥
密码：如果私钥没有 passphrase，就留空
```

---

## 4. 安装 sing-box-yg

登录服务器后先切 root：

```bash
sudo -i
```

更新系统并安装基础工具：

```bash
apt update -y
apt install -y curl wget sudo socat cron qrencode
```

运行甬哥 sing-box-yg 脚本：

```bash
bash <(curl -Ls https://raw.githubusercontent.com/yonggekkk/sing-box-yg/main/sb.sh)
```

如果 curl 失败，用 wget：

```bash
bash <(wget -qO- https://raw.githubusercontent.com/yonggekkk/sing-box-yg/main/sb.sh)
```

后续管理命令：

```bash
sb
```

脚本安装完成后，保存好输出的节点信息，但不要公开提交：

```text
端口
UUID
密码
Reality public key
Reality short id
订阅链接
二维码
```

---

## 5. Oracle 控制台开放端口

需要开两层：

```text
1. Oracle 控制台安全列表 / 网络安全组
2. Ubuntu 系统防火墙
```

Oracle 控制台路径：

```text
实例
→ 附加的 VNIC
→ 查看详细信息
→ 子网
→ 安全列表 / Security List
→ 添加入站规则
```

基础端口：

| 端口 | 协议 | 用途 |
|---|---|---|
| 22 | TCP | SSH 登录 |
| 80 | TCP | 申请证书 / 网站 |
| 443 | TCP | HTTPS / 常用节点端口 |
| 443 | UDP | QUIC / Hysteria2 / TUIC 可能用到 |

如果脚本生成的是这些示例端口，可以按需开放：

| 协议 | 示例端口 | Oracle 入站规则 |
|---|---:|---|
| VLESS Reality | 15555 | TCP 15555 |
| VMess WebSocket | 25555 | TCP 25555 |
| Hysteria2 | 35555 | UDP 35555 |
| TUIC | 45555 | UDP 45555 |
| AnyTLS | 55555 | TCP 55555 |

规则示例：

```text
源类型：CIDR
源 CIDR：0.0.0.0/0
IP 协议：TCP 或 UDP
源端口范围：全部 / 留空
目标端口范围：你的端口
```

不建议开放：

```text
全部端口
0-65535
全部协议
```

---

## 6. Ubuntu 防火墙检查

查看防火墙状态：

```bash
sudo ufw status
```

如果显示 inactive，说明系统防火墙没启用，主要看 Oracle 控制台即可。

如果 UFW 是 active，按需放行：

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 443/udp
sudo ufw reload
```

其他端口示例：

```bash
sudo ufw allow 15555/tcp
sudo ufw allow 25555/tcp
sudo ufw allow 35555/udp
sudo ufw allow 45555/udp
sudo ufw allow 55555/tcp
sudo ufw reload
```

检查服务监听：

```bash
ss -tulnp
```

只看 sing-box：

```bash
ss -tulnp | grep -E "sing|sb|xray"
```

---

## 7. Clash / Mihomo 配置思路

推荐使用 Mihomo / Clash.Meta 内核，因为老版 Clash 不一定支持：

```text
AnyTLS
Hysteria2
TUIC
VLESS Reality
```

节点信息不要直接提交真实值。可以用占位符：

```yaml
proxies:
  - name: "OCI AnyTLS"
    type: anytls
    server: YOUR_SERVER_IP
    port: 55555
    password: "YOUR_PASSWORD"
    tls: true
    sni: www.bing.com
    skip-cert-verify: true
```

常见需要替换的字段：

| 协议 | 需要替换 |
|---|---|
| AnyTLS | server、port、password、sni |
| Hysteria2 | server、port、password、sni、fingerprint |
| VLESS Reality | server、port、uuid、serverName、public-key、short-id |
| TUIC | server、port、uuid、password、sni |
| VMess | server、port、uuid、ws path、host |

---

## 8. TUN / 虚拟网卡模式说明

开了虚拟网卡 TUN 模式，不代表所有流量都一定走代理，还要看客户端模式：

| 模式 | 行为 |
|---|---|
| Rule / 规则模式 | 国内直连，国外走代理，测速网可能显示本地 IP |
| Global / 全局模式 | 大多数流量都走代理 |
| Direct / 直连模式 | 不走代理 |

测试代理是否生效，不建议用国内测速网。可以用：

```text
https://ip.sb
https://ipinfo.io
https://ifconfig.me
https://browserleaks.com/ip
```

如果国内测速网显示本地 IP，大概率是规则命中了：

```text
geosite-cn / geoip-cn → DIRECT
```

### WebSocket、邮箱会不会走代理？

在 TUN + Global 下，大多数都会走代理，包括：

```text
浏览器 HTTPS
WebSocket / WSS
curl / npm / git / pip
邮箱客户端 IMAP / SMTP / POP3
```

在 Rule 模式下，是否走代理取决于规则。可以在 Clash/Mihomo 客户端的：

```text
Connections / 连接
```

查看每条连接是走节点还是 DIRECT。

---

## 9. 自用安全建议

建议：

```text
只自己使用
不要公开订阅链接
不要把密码/UUID提交到仓库
不要开放全部端口
保留 SSH 22 端口，或者后续改成更安全的方式
定期更新系统
```

不要做：

```text
公开卖节点
开放代理给陌生人
发垃圾邮件
扫描公网
爆破、撞库、爬虫代理池
挖矿
批量注册云账号薅资源
```

---

## 10. 常用命令汇总

```bash
# 进入 root
sudo -i

# 更新系统
apt update -y && apt upgrade -y

# 安装基础工具
apt install -y curl wget sudo socat cron qrencode

# 安装 / 管理 sing-box-yg
bash <(curl -Ls https://raw.githubusercontent.com/yonggekkk/sing-box-yg/main/sb.sh)
sb

# 查看端口监听
ss -tulnp

# 查看防火墙
sudo ufw status

# 查看机器配置
nproc
free -h
df -h
```
