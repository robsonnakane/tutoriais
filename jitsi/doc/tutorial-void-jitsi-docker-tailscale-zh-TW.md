# Jitsi Meet + Docker + Tailscale no Void Linux
## 最終、修訂和正確指南（根）

本教程涵蓋了已完成的**所有內容**，包括：
- 安裝正確的軟件包
- 在 Void (runit) 中激活服務
- 克隆堆棧 docker-jitsi-meet
- 端口映射修復（127.0.0.1）
- .env 調整
- docker-compose.yml 調整
- 本機 nginx 的端口 80 問題
- Tailnet 限制（無漏斗）
- 通過 Tailscale Serve 解決方案（內部）
- 最終訪問通過`https://jitsi.tailf0138e.ts.net`
- 無固定IP、無開放端口、無外部DNS

---

## 1.安裝需要的包

```bash
xbps-install -Sy docker docker-compose tailscale git
```

啟用無效服務：

```bash
ln -s /etc/sv/docker /var/service/
ln -s /etc/sv/tailscaled /var/service/
sv status docker tailscaled
```

---

## 2. 激活並驗證 Tailscale

```bash
tailscale up
```

在瀏覽器中打開鏈接，登錄，授權。

在 Tailnet 中重命名設備：

```bash
tailscale set --hostname=jitsi
```

確認 DNS 名稱：

```bash
tailscale status --json | grep DNSName
```

預期的：

```
"DNSName": "jitsi.tailf0138e.ts.net.",
```

⚠ 重要：
- 如果您不是 Tailnet 管理員
- 因此**漏斗被堵塞**
- 但內部服務工作完美

---

## 3. 下載並準備 Jitsi 堆棧

```bash
mkdir -p /opt/jitsi
cd /opt/jitsi
git clone https://github.com/jitsi/docker-jitsi-meet.git
cd docker-jitsi-meet
cp env.example .env
./gen-passwords.sh
```

---

## 4. 調整 `.env` 以與 Tailscale 一起使用
（首先我們需要確認Tailnet的IP和內部DNS）

在編輯 `.env` 之前，請確認：

**1 — Tailscale 內部 IP**

```bash
tailscale ip -4
```

實際使用的例子：
```
100.75.137.60
```

**2 — Tailnet 上機器的內部 DNS 名稱**

```bash
tailscale status --json | grep DNSName
```

實際結果：

```
"DNSName": "jitsi.tailf0138e.ts.net.",
```

這是 Tailscale 根據配置的主機名和 Tailnet ID 生成的**真正的內部**域。

僅在編輯`.env`之後：

```bash
nano /opt/jitsi/docker-jitsi-meet/.env
```

像這樣配置它：

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
- **PUBLIC_URL 指向 Tailscale 的內部名稱**
因為它是用於訪問 Tailnet 內服務器的實際 URL。

- **容器內的 HTTPS 被禁用，因為 TLS 來自 Tailscale**
（Tailscale Serve 提供內置 HTTPS，我們不需要 Jitsi 的 nginx TLS）。

- **我們不使用 Let’s Encrypt，因為沒有發佈公共域或漏斗**
並且 TailNet 管理員尚未啟用該功能，因此公共 TLS 不存在——僅存在於內部。

---

## 5.調整docker-compose.yml
（非常重要——這是我們解決最頭痛的問題的地方）

這一步絕對是必要的，因為這是我們更正的地方：

- Void 的 nginx 出現在 Jitsi 的位置
- 80/8000端口衝突
- Tailscale Serve 抱怨“僅支持 localhost”
- Jitsi 無意中被外部提供服務
- 後端無法在服務上運行
- 需要僅公開本地的
- 自動容器啟動
- 為未來的 FUNNEL 做準備，事後不做任何改變

“web”服務必須僅在本地主機上公開，因為：

- **Tailscale 服務需要 127.0.0.1 上的後端**
（當前版本的Serve僅接受localhost，否則會給出代理錯誤）
- **避免與 Void 的 nginx 發生衝突**，後者運行在系統端口 80 上
（這就是為什麼它說“歡迎來到 nginx！”）
- **確保 Serve 路由到 Jitsi**，而不是主機的 nginx
- **防止在互聯網上意外暴露**，因為本地主機不接受外部連接
- **保證未來與 FUNNEL 的兼容性**（如果 TailNet Admin 發布）
- **使用“restart:always”，容器在重新啟動後自動啟動**，無需額外的運行

編輯撰寫：

```bash
nano /opt/jitsi/docker-jitsi-meet/docker-compose.yml
```

並讓它完全像這樣：

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

簡要說明：

- **127.0.0.1:8000 → 80**
→ 容器的80端口僅在內部存在，接收者為127.0.0.1
→ 這就是為什麼 Tailscale Serv 可以正確重定向

- **重新啟動：始終**
→ 如果Void重新啟動，Jitsi會獨自回來
→ 如果 Docker 重新啟動，Jitsi 會自行恢復
→ 停電時單獨返回

- **這 100% 消除了 Void nginx 問題**
- **這使得 Jitsi 在公共互聯網上不可見**（這是 Tailnet 中所需要的）
- **這將設置一切，以便將來只需一個命令即可激活漏斗**

保存並退出。

---

## 6.上傳Docker堆棧

```bash
docker-compose up -d
docker-compose ps
```

確認網絡為 **127.0.0.1:8000 → 80**。

測試服務器內的前端：

```bash
curl -I http://127.0.0.1:8000
```

預期的：

```
HTTP/1.1 200 OK
Server: nginx
```

⚠ 如果你看到“Welcome to nginx!”，那是 Void 的 nginx。
此測試確保 Jitsi 後端是正確的。

---

## 7. 通過 Tailscale Serve 暴露（內部）

重置之前的所有規則：

```bash
tailscale serve reset
```

創建內部代理：

```bash
tailscale serve --bg http://127.0.0.1:8000
```

預期輸出：

```
Available within your tailnet:

https://jitsi.tailf0138e.ts.net/
|-- proxy http://127.0.0.1:8000
```

檢查狀態：

```bash
tailscale serve status
```

Tailscale Serve 現在可以正確服務 Jitsi。

---

## 8. 通過 Tailnet 訪問（適用於任何網絡）

在您的筆記本電腦、手機、PC 上 — 只要您登錄 Tailscale：

```
https://jitsi.tailf0138e.ts.net/
```

是的：

- HTTPS 有效
- 尾鱗證書有效
- 無警告
- 沒有無效的 nginx
- 編號：8000
- 一切都直接在美麗的領域

⚠ **僅限** Tailnet 會員訪問（目前）。

---

## 9. 有用的命令

版本容器：

```bash
docker-compose ps
```

紀錄:

```bash
docker-compose logs -f web
```

停止：

```bash
docker-compose down
```

服務狀態：

```bash
tailscale serve status
```

重置發球：

```bash
tailscale serve reset
```

---

## 10.添加用戶

```bash
docker compose exec prosody prosodyctl --config /config/prosody.cfg.lua register admin meet.jitsi Jitsi1234
```

預期輸出：

```
usermanager         info	User account created: admin@meet.jitsi
```

---

## 11.當TailNet Admin發布FUNNEL時（可選，公開訪問）

如果**TailNet Admin**啟用Funnel，您將能夠公開
Jitsi 適用於整個互聯網，具有有效的 HTTPS，無需依賴防火牆、調製解調器或固定 IP。

激活漏斗後，您可以執行：

```bash
tailscale funnel --https=443 http://127.0.0.1:8000
```

並且訪問變為：

```
https://jitsi.tailf0138e.ts.net/
```

---

### 🔶 重要提示：如何釋放漏斗

只有**TailNet 管理員**可以啟用Funnel。

管理員需要做：

1. 進入：
辣椒_REF_0_辣椒

2. 在側面菜單中，單擊：
   **設置 → 漏斗**

3. 激活該選項：
✔ **允許此尾網漏斗**

4. 並且還激活：
✔ 選擇 **jitsi** 設備
（或者您使用“tailscale set --hostname”設置的名稱）

5. 節省。

之後，您測試：

```bash
tailscale funnel status
```

如果啟用，該命令將停止給出錯誤，您可以正常激活漏斗。

---

### ✔ 漏斗激活時會發生什麼變化

- Jitsi 可公開訪問（無需 TailNet）
- 自動有效的 HTTPS（通過 Tailscale 的 Let's Encrypt）
- 網址仍然是：
  ```
  https://jitsi.tailf0138e.ts.net/
  ```
- 可以與任何人分享

---

### ✔ 什麼不會改變

- 之前的教程沒有任何中斷
- 內部服務繼續工作
- Docker不需要修改
- Jitsi 不需要重啟

---

## 結尾
修改後的配置，乾淨，無孔洞。
一切都井井有條，運轉良好。
