# Jitsi Meet + Docker + Tailscale なし Void Linux
## 最終改訂版の正しいガイド (ルート)

このチュートリアルでは、以下を含む**すべての内容**を説明します。
- 適切なパッケージをインストールする
- Void (runit) でのサービスのアクティブ化
- スタック docker-jitsi-meet のクローンを作成します
- ポートマッピング修正 (127.0.0.1)
- .env の微調整
- docker-compose.yml の微調整
- ネイティブ nginx でのポート 80 の問題
- テールネットの制限 (ファンネルなし)
- Tailscale Serve によるソリューション (内部)
- `https://jitsi.tailf0138e.ts.net` 経由の最終アクセス
- 固定IPなし、ポート開放なし、外部DNSなし

---

## 1. 必要なパッケージをインストールする

```bash
xbps-install -Sy docker docker-compose tailscale git
```

無効なサービスを有効にします。

```bash
ln -s /etc/sv/docker /var/service/
ln -s /etc/sv/tailscaled /var/service/
sv status docker tailscaled
```

---

## 2. Tailscale をアクティブ化して認証する

```bash
tailscale up
```

ブラウザでリンクを開き、ログインし、認証します。

Tailnet 内のデバイスの名前を変更します。

```bash
tailscale set --hostname=jitsi
```

DNS名を確認します:

```bash
tailscale status --json | grep DNSName
```

期待される：

```
"DNSName": "jitsi.tailf0138e.ts.net.",
```

⚠ 重要:
- Tailnet 管理者ではない場合
- したがって **ファネルがブロックされています**
- しかし、インターナルサーブは完璧に機能します

---

## 3. Jitsi スタックをダウンロードして準備します

```bash
mkdir -p /opt/jitsi
cd /opt/jitsi
git clone https://github.com/jitsi/docker-jitsi-meet.git
cd docker-jitsi-meet
cp env.example .env
./gen-passwords.sh
```

---

## 4. Tailscale で使用するために `.env` を調整します
(最初に Tailnet の IP と内部 DNS を確認する必要があります)

`.env` を編集する前に、次のことを確認してください。

**1 — テールスケール内部 IP**

```bash
tailscale ip -4
```

実際に使用した例:
```
100.75.137.60
```

**2 — Tailnet 上のマシンの内部 DNS 名**

```bash
tailscale status --json | grep DNSName
```

実際の結果:

```
"DNSName": "jitsi.tailf0138e.ts.net.",
```

これは、構成されたホスト名とテールネット ID に基づいて、Tailscale によって生成された **真の内部** ドメインです。

その後でのみ `.env` を編集します。

```bash
nano /opt/jitsi/docker-jitsi-meet/.env
```

次のように設定します。

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

### 正当な理由:
- **PUBLIC_URL は Tailscale の内部名を指します**
これは、Tailnet 内のサーバーにアクセスするために使用される実際の URL であるためです。

- **TLS は Tailscale からのものであるため、コンテナ内の HTTPS は無効になっています**
(Tailscale Serve は組み込みの HTTPS を提供するため、Jitsi の nginx TLS は必要ありません)。

- **パブリック ドメインやファネルがリリースされていないため、Let’s Encrypt は使用しません**
また、TailNet 管理者はまだこの機能を有効にしていないため、パブリック TLS は存在せず、内部のみに存在します。

---

## 5. docker-compose.ymlを調整する
(非常に重要です。これが最大の頭痛の種を解決した場所です)

ここが修正箇所であるため、このステップは絶対に不可欠です。

- Jitsi の代わりに Void の nginx が登場
- 80/8000 ポートの競合
- Tailscale Serve が「localhost のみがサポートされている」と不満を漏らす
- Jitsi が意図せずに外部から提供される
- バックエンドがServeで動作しない
- ローカルのみを公開する必要性
- コンテナの自動起動
- 後は何も変えずに将来のFUNNELに備える

次の理由により、「web」サービスはローカルホスト上でのみ公開する必要があります。

- **Tailscale Serve には 127.0.0.1 のバックエンドが必要です**
(現在のバージョンの Serve はローカルホストのみを受け入れます。それ以外の場合はプロキシ エラーが発生します)
- **システム ポート 80 で実行される Void の nginx との競合を回避します**
(それが「nginx へようこそ!」と表示された理由です)
- **Serve がホストの nginx ではなく Jitsi にルーティングされるようにします**
- **ローカルホストは外部接続を受け付けないため、**インターネット上での偶発的な公開を防ぎます**
- **TailNet Admin がリリースした場合、FUNNEL との将来の互換性を保証します**
- **`restart: always` を使用すると、追加の runit を必要とせずに、再起動後にコンテナが自動的に起動します**

構成を編集します。

```bash
nano /opt/jitsi/docker-jitsi-meet/docker-compose.yml
```

そして、正確に次のように残します。

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

簡単な説明:

- **127.0.0.1:8000 → 80**
→ コンテナのポート 80 は内部のみに存在し、受信者は 127.0.0.1 です。
→ これが、Tailscale Serve が正しくリダイレクトできる理由です

- **再起動: 常に**
→ヴォイドが再起動するとジツィは一人で戻ってくる
→ Docker を再起動すると Jitsi が勝手に戻ってくる
→停電しても単独復帰

- **これにより、Void nginx の問題が 100% 解消されます**
- **これにより、公共のインターネット上で Jitsi が見えなくなります** (これは、Tailnet 内で望まれます)
- **これにより、今後は 1 つのコマンドだけで Funnel をアクティブ化できるようにすべてが設定されます**

保存して終了します。

---

## 6. Docker スタックをアップロードする

```bash
docker-compose up -d
docker-compose ps
```

Web が **127.0.0.1:8000 → 80** であることを確認します。

サーバー内のフロントエンドをテストします。

```bash
curl -I http://127.0.0.1:8000
```

期待される：

```
HTTP/1.1 200 OK
Server: nginx
```

⚠「Welcome to nginx!」と表示された場合は、Void の nginx でした。
このテストにより、Jitsi バックエンドが正しいことが確認されました。

---

## 7. Tailscale Serve 経由で公開する (内部)

以前のルールをリセットします。

```bash
tailscale serve reset
```

内部プロキシを作成します。

```bash
tailscale serve --bg http://127.0.0.1:8000
```

期待される出力:

```
Available within your tailnet:

https://jitsi.tailf0138e.ts.net/
|-- proxy http://127.0.0.1:8000
```

ステータスを確認します:

```bash
tailscale serve status
```

Tailscale Serve は Jitsi を正しく提供するようになりました。

---

## 8. テールネット経由でアクセス (どのネットワークでも動作)

Tailscale にログインしている限り、ラップトップ、携帯電話、PC で次の操作を実行します。

```
https://jitsi.tailf0138e.ts.net/
```

はい：

- HTTPS は機能します
- テールスケール証明書は有効です
- 警告なし
- Void nginx はありません
- いいえ：8000
- すべてが美しいドメインに直接含まれています

⚠ (現時点では) Tailnet メンバーのみ**にアクセスできます。

---

## 9. 便利なコマンド

コンテナのバージョン:

```bash
docker-compose ps
```

ログ:

```bash
docker-compose logs -f web
```

停止するには:

```bash
docker-compose down
```

サービスステータス:

```bash
tailscale serve status
```

サーブをリセットします:

```bash
tailscale serve reset
```

---

## 10. ユーザーの追加

```bash
docker compose exec prosody prosodyctl --config /config/prosody.cfg.lua register admin meet.jitsi Jitsi1234
```

期待される出力:

```
usermanager         info	User account created: admin@meet.jitsi
```

---

## 11. TailNet 管理者が FUNNEL をリリースするとき (オプション、パブリック アクセス)

**TailNet 管理者** がファネルを有効にすると、次の情報を公開できるようになります。
ファイアウォール、モデム、固定 IP に依存せず、有効な HTTPS を使用してインターネット全体を対象とした Jitsi。

ファネルをアクティブ化すると、次のことを実行できます。

```bash
tailscale funnel --https=443 http://127.0.0.1:8000
```

そして、アクセスは次のようになります。

```
https://jitsi.tailf0138e.ts.net/
```

---

### 🔶 重要な注意: ファネルのリリース方法

Funnel を有効にできるのは **TailNet 管理者** だけです。

管理者は次のことを行う必要があります。

1. 入力：
https://login.tailscale.com/admin/acls

2. サイド メニューで、次をクリックします。
   **設定 → ファネル**

3. オプションを有効にします。
✔ **このテールネットのファネルを許可します**

4. また、以下を有効にします。
✔ **jitsi** デバイスを選択します
(または「tailscale set --hostname」で設定した名前)

5. 保存。

その後、以下をテストします。

```bash
tailscale funnel status
```

有効にすると、コマンドでエラーが表示されなくなり、Funnel を正常にアクティブ化できるようになります。

---

### ✔ ファネルがアクティブになると何が変わるか

- Jitsi はパブリックにアクセス可能です (TailNet なし)
- 自動的に有効な HTTPS (Tailscale の Let's Encrypt 経由)
- URL は残ります:
  ```
  https://jitsi.tailf0138e.ts.net/
  ```
- 誰とでも共有できる

---

### ✔ 変わらないもの

- 前のチュートリアルの内容は何も変わりません
- 内部サービスは引き続き機能します
- Dockerを変更する必要はありません
- Jitsi を再起動する必要はありません

---

## 終わり
構成が修正され、きれいで、穴はありません。
すべてが順調で、機能しています。
