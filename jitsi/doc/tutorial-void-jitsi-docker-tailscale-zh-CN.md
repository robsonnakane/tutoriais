# Jitsi Meet + Docker + Tailscale no Void Linux
## 最终、修订和正确指南（根）

本教程涵盖了已完成的**所有内容**，包括：
- 安装正确的软件包
- 在 Void (runit) 中激活服务
- 克隆堆栈 docker-jitsi-meet
- 端口映射修复（127.0.0.1）
- .env 调整
- docker-compose.yml 调整
- 本机 nginx 的端口 80 问题
- Tailnet 限制（无漏斗）
- 通过 Tailscale Serve 解决方案（内部）
- 最终访问通过`https://jitsi.tailf0138e.ts.net`
- 无固定IP、无开放端口、无外部DNS

---

## 1.安装需要的包

```bash
xbps-install -Sy docker docker-compose tailscale git
```

启用无效服务：

```bash
ln -s /etc/sv/docker /var/service/
ln -s /etc/sv/tailscaled /var/service/
sv status docker tailscaled
```

---

## 2. 激活并验证 Tailscale

```bash
tailscale up
```

在浏览器中打开链接，登录，授权。

在 Tailnet 中重命名设备：

```bash
tailscale set --hostname=jitsi
```

确认 DNS 名称：

```bash
tailscale status --json | grep DNSName
```

预期的：

```
"DNSName": "jitsi.tailf0138e.ts.net.",
```

⚠ 重要：
- 如果您不是 Tailnet 管理员
- 因此**漏斗被堵塞**
- 但内部服务工作完美

---

## 3. 下载并准备 Jitsi 堆栈

```bash
mkdir -p /opt/jitsi
cd /opt/jitsi
git clone https://github.com/jitsi/docker-jitsi-meet.git
cd docker-jitsi-meet
cp env.example .env
./gen-passwords.sh
```

---

## 4. 调整 `.env` 以与 Tailscale 一起使用
（首先我们需要确认Tailnet的IP和内部DNS）

在编辑 `.env` 之前，请确认：

**1 — Tailscale 内部 IP**

```bash
tailscale ip -4
```

实际使用的例子：
```
100.75.137.60
```

**2 — Tailnet 上机器的内部 DNS 名称**

```bash
tailscale status --json | grep DNSName
```

实际结果：

```
"DNSName": "jitsi.tailf0138e.ts.net.",
```

这是 Tailscale 根据配置的主机名和 Tailnet ID 生成的**真正的内部**域。

仅在编辑`.env`之后：

```bash
nano /opt/jitsi/docker-jitsi-meet/.env
```

像这样配置它：

```ini
PUBLIC_URL=https://jitsi.tailf0138e.ts.net
ENABLE_LETSENCRYPT=0
DISABLE_HTTPS=1
ENABLE_AUTH=1
ENABLE_GUESTS=1
AUTH_TYPE=internal

XMPP_DOMAIN=meet.jitsi
#XMPP_AUTH_DOMAIN=auth.meet.jitsi
#XMPP_AUTH_DOMAIN_PREFIX=auth
XMPP_MUC_DOMAIN=muc.meet.jitsi
XMPP_INTERNAL_MUC_DOMAIN=internal-muc.meet.jitsi
XMPP_GUEST_DOMAIN=guest.meet.jitsi




```

### 理由：
- **PUBLIC_URL 指向 Tailscale 的内部名称**
因为它是用于访问 Tailnet 内服务器的实际 URL。

- **容器内的 HTTPS 被禁用，因为 TLS 来自 Tailscale**
（Tailscale Serve 提供内置 HTTPS，我们不需要 Jitsi 的 nginx TLS）。

- **我们不使用 Let’s Encrypt，因为没有发布公共域或漏斗**
并且 TailNet 管理员尚未启用该功能，因此公共 TLS 不存在——仅存在于内部。

---

## 5.调整docker-compose.yml
（非常重要——这是我们解决最头痛的问题的地方）

这一步绝对是必要的，因为这是我们更正的地方：

- Void 的 nginx 出现在 Jitsi 的位置
- 80/8000端口冲突
- Tailscale Serve 抱怨“仅支持 localhost”
- Jitsi 无意中被外部提供服务
- 后端无法在服务上运行
- 需要仅公开本地的
- 自动容器启动
- 为未来的 FUNNEL 做准备，事后不做任何改变

“web”服务必须仅在本地主机上公开，因为：

- **Tailscale 服务需要 127.0.0.1 上的后端**
（当前版本的Serve仅接受localhost，否则会给出代理错误）
- **避免与 Void 的 nginx 发生冲突**，后者运行在系统端口 80 上
（这就是为什么它说“欢迎来到 nginx！”）
- **确保 Serve 路由到 Jitsi**，而不是主机的 nginx
- **防止在互联网上意外暴露**，因为本地主机不接受外部连接
- **保证未来与 FUNNEL 的兼容性**（如果 TailNet Admin 发布）
- **使用“restart:always”，容器在重新启动后自动启动**，无需额外的运行

编辑撰写：

```bash
nano /opt/jitsi/docker-jitsi-meet/docker-compose.yml
```

并让它完全像这样：

```yaml
services:

  web:
    image: jitsi/web:unstable
    restart: always
    ports:
      - "127.0.0.1:8000:80"
      - "127.0.0.1:8443:443"

  prosody:
    image: jitsi/prosody:unstable
    restart: always

  jicofo:
    image: jitsi/jicofo:unstable
    restart: always

  jvb:
    image: jitsi/jvb:unstable
    restart: always
```

简要说明：

- **127.0.0.1:8000 → 80**
→ 容器的80端口仅在内部存在，接收者为127.0.0.1
→ 这就是为什么 Tailscale Serv 可以正确重定向

- **重新启动：始终**
→ 如果Void重新启动，Jitsi会独自回来
→ 如果 Docker 重新启动，Jitsi 会自行恢复
→ 停电时单独返回

- **这 100% 消除了 Void nginx 问题**
- **这使得 Jitsi 在公共互联网上不可见**（这是 Tailnet 中所需要的）
- **这将设置一切，以便将来只需一个命令即可激活漏斗**

保存并退出。

---

## 6.上传Docker堆栈

```bash
docker-compose up -d
docker-compose ps
```

确认网络为 **127.0.0.1:8000 → 80**。

测试服务器内的前端：

```bash
curl -I http://127.0.0.1:8000
```

预期的：

```
HTTP/1.1 200 OK
Server: nginx
```

⚠ 如果你看到“Welcome to nginx!”，那是 Void 的 nginx。
此测试确保 Jitsi 后端是正确的。

---

## 7. 通过 Tailscale Serve 暴露（内部）

重置之前的所有规则：

```bash
tailscale serve reset
```

创建内部代理：

```bash
tailscale serve --bg http://127.0.0.1:8000
```

预期输出：

```
Available within your tailnet:

https://jitsi.tailf0138e.ts.net/
|-- proxy http://127.0.0.1:8000
```

检查状态：

```bash
tailscale serve status
```

Tailscale Serve 现在可以正确服务 Jitsi。

---

## 8. 通过 Tailnet 访问（适用于任何网络）

在您的笔记本电脑、手机、PC 上 — 只要您登录 Tailscale：

```
https://jitsi.tailf0138e.ts.net/
```

是的：

- HTTPS 有效
- 尾鳞证书有效
- 无警告
- 没有无效的 nginx
- 编号：8000
- 一切都直接在美丽的领域

⚠ **仅限** Tailnet 会员访问（目前）。

---

## 9. 有用的命令

版本容器：

```bash
docker-compose ps
```

日志：

```bash
docker-compose logs -f web
```

停止：

```bash
docker-compose down
```

服务状态：

```bash
tailscale serve status
```

重置发球：

```bash
tailscale serve reset
```

---

## 10.添加用户

```bash
docker compose exec prosody prosodyctl --config /config/prosody.cfg.lua register admin meet.jitsi Jitsi1234
```

预期输出：

```
usermanager         info	User account created: admin@meet.jitsi
```

---

## 11.当TailNet Admin发布FUNNEL时（可选，公开访问）

如果**TailNet Admin**启用Funnel，您将能够公开
Jitsi 适用于整个互联网，具有有效的 HTTPS，无需依赖防火墙、调制解调器或固定 IP。

激活漏斗后，您可以执行：

```bash
tailscale funnel --https=443 http://127.0.0.1:8000
```

并且访问变为：

```
https://jitsi.tailf0138e.ts.net/
```

---

### 🔶 重要提示：如何释放漏斗

只有**TailNet 管理员**可以启用Funnel。

管理员需要做：

1. 进入：
辣椒_REF_0_辣椒

2. 在侧面菜单中，单击：
   **设置 → 漏斗**

3. 激活该选项：
✔ **允许此尾网漏斗**

4. 并且还激活：
✔ 选择 **jitsi** 设备
（或者您使用“tailscale set --hostname”设置的名称）

5. 节省。

之后，您测试：

```bash
tailscale funnel status
```

如果启用，该命令将停止给出错误，您可以正常激活漏斗。

---

### ✔ 漏斗激活时会发生什么变化

- Jitsi 可公开访问（无需 TailNet）
- 自动有效的 HTTPS（通过 Tailscale 的 Let's Encrypt）
- 网址仍然是：
  ```
  https://jitsi.tailf0138e.ts.net/
  ```
- 可以与任何人分享

---

### ✔ 什么不会改变

- 之前的教程没有任何中断
- 内部服务继续工作
- Docker不需要修改
- Jitsi 不需要重启

---

## 结尾
修改后的配置，干净，无孔洞。
一切都井井有条，运转良好。
