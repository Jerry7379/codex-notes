# 西海岸 VPS：3x-ui + VLESS-REALITY + Clash 订阅 复刻手册

更新日期：2026-04-21  
适用场景：RackNerd 新购 VPS，Ubuntu 24.04，目标是给 Clash Verge / 手机客户端提供可直接导入的节点与在线订阅。

> 安全说明：本文为可公开归档的脱敏版，不包含当前线上可直接利用的密码、订阅 ID、UUID、Reality 密钥、公网 IP 或面板路径。实际值请从安全存储、服务器本机，或当前运维会话里读取，严禁再次提交到 Git。

---

## 1. 当前已成功跑通的线上配置

### 服务器信息
- 服务商：RackNerd
- 地区：西海岸（当前公网 IP）
- 公网 IPv4：`<服务器公网 IP，已脱敏>`
- 系统：`Ubuntu 24.04 LTS`
- 虚拟化：`KVM`
- root 密码：`<已脱敏；请勿入库>`

### 3x-ui 面板信息
- 版本：`v2.9.0`
- 面板地址：`https://<服务器公网 IP>:2053/<PANEL_BASE_PATH>/`
- 用户名：`<PANEL_USERNAME，已脱敏>`
- 密码：`<PANEL_PASSWORD，已脱敏>`
- 证书：IP 证书
  - 公钥：`/root/cert/ip/fullchain.pem`
  - 私钥：`/root/cert/ip/privkey.pem`

### 当前节点信息
- 节点显示名：`美国🇺🇸-西海岸-01`
- 协议：`VLESS + REALITY`
- 端口：`443`
- UUID：`<UUID，已脱敏>`
- Public Key：`<REALITY_PUBLIC_KEY，已脱敏>`
- Short ID：`<SHORT_ID，已脱敏>`
- SNI：`www.tesla.com`
- Client Email：`01`
- subId：`<SUB_ID，已脱敏>`

### 在线订阅
- Clash / Mihomo YAML：
  - `https://<服务器公网 IP>:2096/clash/<SUB_ID>`
- 普通订阅：
  - `https://<服务器公网 IP>:2096/sub/<SUB_ID>`

### 备用直连 VLESS 链接
```text
vless://<UUID>@<服务器公网 IP>:443?type=tcp&encryption=none&security=reality&pbk=<REALITY_PUBLIC_KEY>&fp=chrome&sni=www.tesla.com&sid=<SHORT_ID>&spx=%2F&flow=xtls-rprx-vision#美国🇺🇸-西海岸-01
```

---


### 读取当前线上实际值（不入 Git）
如需恢复当前线上具体值，请在服务器本机或受信会话中读取：
```bash
/usr/local/x-ui/x-ui setting -show true
cat /root/xui-add-meta.json
```

## 2. 这次和旧纽约机最大的差异

**重点：这次不需要再编译 master。**

原因：
- 新机安装到的是 `3x-ui v2.9.0`
- 已确认 `panel/setting/all` 返回里直接包含：
  - `subClashEnable`
  - `subClashPath`
  - `subClashURI`
- 也已实测 `https://IP:2096/clash/<subid>` 直接返回标准 Clash/Mihomo YAML

所以：
- 旧机那套“先装 release，再编译 master，替换二进制”的步骤，这次**不用做**。

---

## 3. 从零复刻到新服务器的标准流程

> 下面这套就是以后换新 VPS 时，另一个会话也能直接照着做的流程。

### 第 1 步：登录服务器
```bash
ssh root@<新服务器IP>
```

如果 SSH 总断：
```bash
ssh -o ServerAliveInterval=30 -o ServerAliveCountMax=6 -o TCPKeepAlive=yes root@<新服务器IP>
```

如果需要非交互直连：
```bash
sshpass -p '<ROOT_PASSWORD>' ssh -o StrictHostKeyChecking=no root@<新服务器IP>
```

---

### 第 2 步：安装 3x-ui
这次实际成功使用的安装方式是：

```bash
printf 'y\n2053\n2\n\n\n' | bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
```

含义：
- `y`：自定义 panel port
- `2053`：面板端口
- `2`：选择 IP 证书
- 后两个空行：沿用默认项

安装完成后，服务会自动启动。

---

### 第 3 步：把面板参数改回固定值
为了和旧机保持一致，实际执行的是：

```bash
/usr/local/x-ui/x-ui setting \
  -port 2053 \
  -username '<PANEL_USERNAME>' \
  -password '<PANEL_PASSWORD>' \
  -webBasePath '/<PANEL_BASE_PATH>/'

systemctl restart x-ui
```

检查：
```bash
/usr/local/x-ui/x-ui setting -show true
systemctl status x-ui --no-pager -n 20
ss -ltnup | egrep ':(2053|2096|443|80)\b'
```

期望看到：
- 面板 SSL 已启用
- `2053` 在监听
- `2096` 在监听

---

### 第 4 步：登录 panel API
```bash
curl -sk -c /tmp/xui.cookie \
  -X POST 'https://127.0.0.1:2053/<PANEL_BASE_PATH>/login' \
  -H 'Content-Type: application/x-www-form-urlencoded; charset=UTF-8' \
  --data-urlencode 'username=<PANEL_USERNAME>' \
  --data-urlencode 'password=<PANEL_PASSWORD>'
```

检查订阅能力：
```bash
curl -sk -b /tmp/xui.cookie \
  -X POST 'https://127.0.0.1:2053/<PANEL_BASE_PATH>/panel/setting/all'
```

如果返回里已有：
- `subClashEnable: true`
- `subClashPath: /clash/`

就说明这台机已经**原生支持 Clash/Mihomo 在线订阅**。

---

### 第 5 步：生成 REALITY X25519 密钥
```bash
curl -sk -b /tmp/xui.cookie \
  'https://127.0.0.1:2053/<PANEL_BASE_PATH>/panel/api/server/getNewX25519Cert'
```

本次会返回一组新的密钥，请保存到安全位置；公开文档中只保留字段名，不保留实际值：
- privateKey：`<REALITY_PRIVATE_KEY，已脱敏>`
- publicKey：`<REALITY_PUBLIC_KEY，已脱敏>`

---

### 第 6 步：通过 API 创建 VLESS + REALITY 入站

#### 关键命名规则
3x-ui 的 Clash 订阅名默认是：

`入站 remark + 分隔符 + client email`

因此，如果想让最终订阅里显示：

`美国🇺🇸-西海岸-01`

则要这样设置：
- inbound `remark`：`美国🇺🇸-西海岸`
- client `email`：`01`

#### 实际创建参数
- protocol：`vless`
- port：`443`
- flow：`xtls-rprx-vision`
- network：`tcp`
- security：`reality`
- target：`www.tesla.com:443`
- serverNames：`www.tesla.com`
- fingerprint：`chrome`
- spiderX：`/`
- shortId：`<SHORT_ID，已脱敏>`

#### 创建用 payload 示例
```json
{
  "up": 0,
  "down": 0,
  "total": 0,
  "remark": "美国🇺🇸-西海岸",
  "enable": true,
  "expiryTime": 0,
  "trafficReset": "never",
  "lastTrafficResetTime": 0,
  "listen": "",
  "port": 443,
  "protocol": "vless",
  "settings": "{...}",
  "streamSettings": "{...}",
  "sniffing": "{...}"
}
```

实际 API：
```bash
curl -sk -b /tmp/xui.cookie \
  -H 'Content-Type: application/json' \
  --data @/root/xui-add.json \
  'https://127.0.0.1:2053/<PANEL_BASE_PATH>/panel/api/inbounds/add'
```

如果后续只是改名字，可直接更新：
```bash
curl -sk -b /tmp/xui.cookie \
  -H 'Content-Type: application/json' \
  --data @/root/xui-update.json \
  'https://127.0.0.1:2053/<PANEL_BASE_PATH>/panel/api/inbounds/update/1'
```

---

## 4. 当前新机上已经落地的本地文件

这些文件现在在服务器上：
- `/root/xui-add.json`
- `/root/xui-update.json`
- `/root/xui-add-meta.json`

其中 `/root/xui-add-meta.json` 保存了本次节点核心信息（敏感，禁止入库）：
- UUID
- subId
- shortId
- publicKey
- privateKey
- remark
- email
- server
- sni

如果另一个会话需要快速接力，可在受信环境中先看：
```bash
cat /root/xui-add-meta.json
```

---

## 5. 验证命令

### 面板与端口
```bash
/usr/local/x-ui/x-ui setting -show true
systemctl status x-ui --no-pager -n 20
ss -ltnup | egrep ':(2053|2096|443)\b'
```

### 查看入站列表
```bash
curl -sk -b /tmp/xui.cookie \
  'https://127.0.0.1:2053/<PANEL_BASE_PATH>/panel/api/inbounds/list'
```

### 验证 Clash 在线订阅
服务器本机验证：
```bash
curl -sk --resolve '<服务器公网 IP>:2096:127.0.0.1' \
  'https://<服务器公网 IP>:2096/clash/<SUB_ID>' | sed -n '1,40p'
```

期望返回里有：
```yaml
name: 美国🇺🇸-西海岸-01
server: <服务器公网 IP>
type: vless
udp: true
```

### 验证普通订阅
```bash
curl -sk 'https://<服务器公网 IP>:2096/sub/<SUB_ID>'
```

### 验证 443 入站
```bash
ss -ltnup | grep ':443 '
```

---

## 6. 服务器侧优化（已完成）

### 当前优化项
已启用：
- `BBR`
- `fq`
- `tcp_fastopen = 3`
- `tcp_slow_start_after_idle = 0`

### 当前生效值
```bash
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_slow_start_after_idle = 0
```

### 持久化文件
- `/etc/modules-load.d/tcp_bbr.conf`
- `/etc/sysctl.d/99-codex-netopt.conf`

### 复刻命令
```bash
modprobe tcp_bbr
printf 'tcp_bbr\n' > /etc/modules-load.d/tcp_bbr.conf

cat > /etc/sysctl.d/99-codex-netopt.conf <<'EOF2'
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_slow_start_after_idle = 0
EOF2

sysctl --system
```

### 验证命令
```bash
lsmod | grep '^tcp_bbr'
sysctl net.core.default_qdisc net.ipv4.tcp_congestion_control net.ipv4.tcp_fastopen net.ipv4.tcp_slow_start_after_idle
sysctl net.ipv4.tcp_available_congestion_control
```

---

## 7. 常见坑

### 坑 1：SSH 容易断
原因通常有两个：
1. 本机 Clash/TUN 让 SSH 走了这台机器自己，形成自咬尾巴
2. SSH 没开 keepalive

建议：
- 本机把 `<服务器公网 IP>/32` 设为 `DIRECT`
- SSH 使用：
```bash
ssh -o ServerAliveInterval=30 -o ServerAliveCountMax=6 -o TCPKeepAlive=yes root@<服务器公网 IP>
```

---

### 坑 2：手机端报 `unsupport proxy type:vless`
这通常不是订阅坏了，而是客户端内核不支持 `vless`。

处理方式：
- 换支持 Mihomo/VLESS 的手机端
- 或者别导 Clash 订阅，直接导入备用 `vless://...` 链接

---

### 坑 3：以为还要编译 master
这台新机上的 `3x-ui v2.9.0` 已经自带 Clash/Mihomo 在线订阅。

判断标准：
```bash
curl -sk -b /tmp/xui.cookie \
  -X POST 'https://127.0.0.1:2053/<PANEL_BASE_PATH>/panel/setting/all'
```

如果返回里有：
- `subClashEnable`
- `subClashPath`
- `subClashURI`

就不要再重复走“编译替换二进制”。

---

## 8. 最短接力指令（给未来会话）

如果未来开一个新会话，只要把下面这段给它看，基本就能无缝接上：

```text
请按 /Users/nrick/Documents/Codex/2026-04-20-computer-use-plugin-computer-use-openai/西海岸VPS-3x-ui-VLESS-REALITY-复刻手册.md 继续。当前线上新机参数已脱敏，3x-ui v2.9.0，已确认原生支持 Clash 在线订阅，不需要编译 master。若要复刻到另一台新 VPS，请直接复用文档里的安装、API 建站、验证和 BBR 优化流程。
```

---

## 9. 当前状态一句话总结

这台西海岸 VPS 已完成：
- 3x-ui 安装
- 面板固定化
- VLESS + REALITY 入站
- Clash/Mihomo 在线订阅
- 服务器侧 BBR 优化

可以直接给 Clash Verge 和支持 Mihomo 的手机客户端使用。
