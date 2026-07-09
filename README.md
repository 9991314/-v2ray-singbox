# sing-box-yg 一键部署教程

> 本教程只记录如何在一台 Linux VPS 上部署 sing-box-yg 脚本。  
> 仅建议个人自用，不要公开分享节点、订阅链接、密码、UUID、私钥等敏感信息。

---

## 1. 准备环境

需要一台可以 SSH 登录的 Linux VPS。

推荐系统：

```text
Ubuntu 22.04 / Ubuntu 24.04
Debian 11 / Debian 12
```

最低建议配置：

```text
1 核 CPU
1GB~2GB 内存
10GB 以上硬盘
公网 IPv4 或 IPv6
```

如果还要同时跑网站、网盘、Docker，可以适当提高配置。

---

## 2. SSH 登录服务器

普通登录示例：

```bash
ssh root@你的服务器IP
```

如果是密钥登录：

```bash
ssh -i ./your_private_key root@你的服务器IP
```

如果不是 root 用户，登录后切换到 root：

```bash
sudo -i
```

---

## 3. 更新系统并安装基础工具

Ubuntu / Debian：

```bash
apt update -y
apt install -y curl wget sudo socat cron qrencode
```

如果系统比较干净，也可以顺手升级：

```bash
apt upgrade -y
```

---

## 4. 安装 sing-box-yg

运行一键脚本：

```bash
bash <(curl -Ls https://raw.githubusercontent.com/yonggekkk/sing-box-yg/main/sb.sh)
```

如果 curl 失败，可以用 wget：

```bash
bash <(wget -qO- https://raw.githubusercontent.com/yonggekkk/sing-box-yg/main/sb.sh)
```

安装过程中按脚本提示选择即可。新手可以优先选择简单模式。

安装完成后，脚本通常会输出：

```text
节点链接
订阅链接
二维码
端口
UUID
密码
Reality public key
Reality short id
```

这些信息要保存好，不要提交到公开仓库。

---

## 5. 后续管理命令

安装完成后，可以直接使用：

```bash
sb
```

常见用途：

```text
查看节点信息
重新生成节点
更换协议
更新内核
重启服务
卸载脚本
```

---

## 6. 开放端口

脚本安装完成后，看它最终生成了哪些端口，然后在服务器防火墙和云平台安全规则里放行。

常见协议和端口类型：

| 协议 | 常见传输 | 需要开放 |
|---|---|---|
| AnyTLS | TCP | 对应 TCP 端口 |
| VLESS Reality | TCP | 对应 TCP 端口 |
| VMess WebSocket | TCP | 对应 TCP 端口 |
| Hysteria2 | UDP | 对应 UDP 端口 |
| TUIC | UDP | 对应 UDP 端口 |

示例：如果脚本生成的端口是：

```text
15555 TCP
25555 TCP
35555 UDP
45555 UDP
55555 TCP
```

那么需要放行：

```text
TCP 15555
TCP 25555
UDP 35555
UDP 45555
TCP 55555
```

不要直接开放全部端口，按需开放即可。

---

## 7. Ubuntu 防火墙设置

查看 UFW 状态：

```bash
sudo ufw status
```

如果显示：

```text
Status: inactive
```

说明系统防火墙未启用，主要检查云平台安全规则即可。

如果是 active，需要放行对应端口，例如：

```bash
sudo ufw allow 22/tcp
sudo ufw allow 15555/tcp
sudo ufw allow 25555/tcp
sudo ufw allow 35555/udp
sudo ufw allow 45555/udp
sudo ufw allow 55555/tcp
sudo ufw reload
```

查看端口监听：

```bash
ss -tulnp
```

只看相关进程：

```bash
ss -tulnp | grep -E "sing|box|xray|hysteria|tuic"
```

---

## 8. 导入客户端

脚本一般会输出多种格式：

```text
sing-box 链接
Clash / Mihomo 订阅
二维码
v2rayN 链接
```

Windows 推荐：

```text
v2rayN
Clash Verge Rev
FlClash
Mihomo Party
```

Android 推荐：

```text
v2rayNG
NekoBox
sing-box
Clash Meta for Android
```

iOS 可使用支持 sing-box / Clash / Hysteria2 / TUIC 的客户端。

如果使用 Clash 类客户端，建议使用 Mihomo / Clash.Meta 内核，因为老版 Clash 可能不支持：

```text
AnyTLS
Hysteria2
TUIC
VLESS Reality
```

---

## 9. TUN / 虚拟网卡模式说明

开启 TUN / 虚拟网卡模式后，大多数软件流量都会被客户端接管，但是否走代理还取决于模式：

| 模式 | 效果 |
|---|---|
| Rule / 规则模式 | 按规则分流，国内可能直连，国外走代理 |
| Global / 全局模式 | 大多数流量都走代理 |
| Direct / 直连模式 | 不走代理 |

如果测速网站显示的不是服务器 IP，可能是规则模式下该网站被直连了。可以临时切换到 Global 全局模式测试。

测试 IP 可以用：

```text
https://ip.sb
https://ipinfo.io
https://ifconfig.me
https://browserleaks.com/ip
```

---

## 10. WebSocket、邮箱等请求是否走代理

如果开启 TUN + Global，大多数请求都会走代理，包括：

```text
浏览器 HTTPS
WebSocket / WSS
curl / npm / git / pip
邮箱客户端 IMAP / SMTP / POP3
```

如果是 Rule 模式，就看规则命中结果。

最准确的判断方式是在客户端里打开：

```text
Connections / 连接
```

查看目标连接是走节点还是 DIRECT。

---

## 11. 安全建议

建议：

```text
只自己使用
不要公开订阅链接
不要提交真实密码、UUID、私钥
不要开放全部端口
保留 SSH 登录方式
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
```

---

## 12. 常用命令

```bash
# 进入 root
sudo -i

# 更新系统
apt update -y && apt upgrade -y

# 安装基础工具
apt install -y curl wget sudo socat cron qrencode

# 安装 sing-box-yg
bash <(curl -Ls https://raw.githubusercontent.com/yonggekkk/sing-box-yg/main/sb.sh)

# 管理脚本
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
