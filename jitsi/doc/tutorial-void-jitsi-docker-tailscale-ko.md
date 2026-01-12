# Jitsi Meet + Docker + Tailscale no Void Linux
## 최종, 개정 및 올바른 안내서(루트)

이 튜토리얼에서는 다음을 포함하여 수행된 **모든 것**을 다룹니다.
- 올바른 패키지 설치
- Void(runit)에서 서비스 활성화
- 클론 다 스택 docker-jitsi-meet
- 포트 매핑 수정(127.0.0.1)
- .env 조정
- docker-compose.yml 조정
- 기본 nginx의 포트 80 문제
- Tailnet 제한사항(퍼널 없음)
- Tailscale Serve를 통한 솔루션(내부)
- `https://jitsi.tailf0138e.ts.net`를 통한 최종 액세스
- 고정 IP 없음, 포트 개방 없음, 외부 DNS 없음

---

## 1. 필수 패키지 설치

```bash
xbps-install -Sy docker docker-compose tailscale git
```

Void 서비스 활성화:

```bash
ln -s /etc/sv/docker /var/service/
ln -s /etc/sv/tailscaled /var/service/
sv status docker tailscaled
```

---

## 2. Tailscale 활성화 및 인증

```bash
tailscale up
```

브라우저에서 링크를 열고 로그인한 후 승인하세요.

Tailnet 내에서 장치 이름을 바꾸십시오.

```bash
tailscale set --hostname=jitsi
```

DNS 이름 확인:

```bash
tailscale status --json | grep DNSName
```

예상되는:

```
"DNSName": "jitsi.tailf0138e.ts.net.",
```

⚠ 중요:
- Tailnet 관리자가 아닌 경우
- 따라서 **퍼널이 차단되었습니다**
- 하지만 내부 서비스는 완벽하게 작동합니다.

---

## 3. Jitsi 스택 다운로드 및 준비

```bash
mkdir -p /opt/jitsi
cd /opt/jitsi
git clone https://github.com/jitsi/docker-jitsi-meet.git
cd docker-jitsi-meet
cp env.example .env
./gen-passwords.sh
```

---

## 4. Tailscale과 함께 사용할 수 있도록 `.env`를 조정합니다.
(먼저 Tailnet의 IP와 내부 DNS를 확인해야 합니다)

`.env`를 편집하기 전에 다음을 확인하세요.

**1 — 테일스케일 내부 IP**

```bash
tailscale ip -4
```

실제 사용된 예:
```
100.75.137.60
```

**2 — Tailnet에 있는 시스템의 내부 DNS 이름**

```bash
tailscale status --json | grep DNSName
```

실제 결과:

```
"DNSName": "jitsi.tailf0138e.ts.net.",
```

이는 구성된 호스트 이름 및 Tailnet ID를 기반으로 Tailscale에서 생성된 **실제 내부** 도메인입니다.

그 후에만 `.env`를 편집하십시오:

```bash
nano /opt/jitsi/docker-jitsi-meet/.env
```

다음과 같이 구성하십시오.

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

### 근거:
- **PUBLIC_URL은 Tailscale의 내부 이름을 나타냅니다**
이는 Tailnet 내에서 서버에 액세스하는 데 사용되는 실제 URL이기 때문입니다.

- **TLS가 Tailscale에서 제공되므로 컨테이너 내부의 HTTPS는 비활성화됩니다**
(Tailscale Serve는 내장형 HTTPS를 제공하며 Jitsi의 nginx TLS는 필요하지 않습니다.)

- **공개 도메인이나 퍼널이 공개되지 않았기 때문에 Let’s Encrypt를 사용하지 않습니다**
TailNet 관리자가 아직 이 기능을 활성화하지 않았으므로 공개 TLS는 존재하지 않으며 내부에만 존재합니다.

---

## 5. docker-compose.yml 조정
(매우 중요 - 여기서 가장 큰 문제를 해결했습니다)

이 단계는 우리가 수정한 부분이므로 절대적으로 중요합니다.

- Jitsi의 자리에 Void의 nginx가 나타남
- 80/8000 포트 충돌
- Tailscale Serve가 "localhost만 지원됩니다"라고 불평함
- Jitsi가 의도치 않게 외부에서 제공됨
- 백엔드가 Serve에서 작동하지 않습니다.
- 로컬 전용 노출 필요
- 자동 컨테이너 시작
- 이후 아무것도 바꾸지 않고 미래의 FUNNEL을 준비합니다.

`web` 서비스는 다음과 같은 이유로 localhost에만 노출되어야 합니다.

- **Tailscale Serve에는 127.0.0.1의 백엔드가 필요합니다**
(현재 버전의 Serve는 localhost만 허용합니다. 그렇지 않으면 프록시 오류가 발생합니다)
- **시스템 포트 80에서 실행되는 Void의 nginx와의 충돌을 방지합니다**
(그래서 "nginx에 오신 것을 환영합니다!"라고 적혀 있습니다)
- **호스트의 nginx가 아닌 Jitsi에 대한 경로 제공**을 보장합니다.
- **우발적인 인터넷 노출 방지**, localhost는 외부 연결을 허용하지 않습니다.
- **TailNet Admin이 출시되면 FUNNEL과의 향후 호환성 보장**
- **`restart: Always`를 사용하면 재부팅 후 컨테이너가 자동으로 시작됩니다**, 추가 실행 없이

작성을 편집합니다.

```bash
nano /opt/jitsi/docker-jitsi-meet/docker-compose.yml
```

그리고 정확히 다음과 같이 남겨두세요:

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

간략한 설명:

- **127.0.0.1:8000 → 80**
→ 컨테이너의 포트 80은 내부에만 존재하며 수신자는 127.0.0.1
→ 이것이 Tailscale Serve가 올바르게 리디렉션될 수 있는 이유입니다.

- **다시 시작: 항상**
→ Void가 다시 시작되면 Jitsi가 혼자 돌아옵니다.
→ Docker가 다시 시작되면 Jitsi가 자동으로 돌아옵니다.
→ 정전되면 혼자 복귀

- **이렇게 하면 Void nginx 문제가 100% 제거됩니다**
- **이렇게 하면 Jitsi가 공용 인터넷에서 보이지 않게 됩니다**(Tailnet 내에서 바람직함)
- **이것은 단 하나의 명령으로 향후 Funnel을 활성화하기 위한 모든 것을 설정합니다**

저장하고 종료합니다.

---

## 6. Docker 스택 업로드

```bash
docker-compose up -d
docker-compose ps
```

웹이 **127.0.0.1:8000 → 80**인지 확인합니다.

서버 내에서 프런트엔드를 테스트합니다.

```bash
curl -I http://127.0.0.1:8000
```

예상되는:

```
HTTP/1.1 200 OK
Server: nginx
```

⚠ "Welcome to nginx!"를 보시면 Void의 nginx였습니다.
이 테스트를 통해 Jitsi 백엔드가 올바른지 확인했습니다.

---

## 7. Tailscale Serve를 통해 노출(내부)

이전 규칙을 재설정합니다.

```bash
tailscale serve reset
```

내부 프록시를 만듭니다.

```bash
tailscale serve --bg http://127.0.0.1:8000
```

예상 출력:

```
Available within your tailnet:

https://jitsi.tailf0138e.ts.net/
|-- proxy http://127.0.0.1:8000
```

상태 확인:

```bash
tailscale serve status
```

Tailscale Serve가 이제 Jitsi를 올바르게 제공하고 있습니다.

---

## 8. Tailnet을 통한 액세스(모든 네트워크에서 작동)

노트북, 휴대폰, PC에서 — Tailscale에 로그인되어 있는 한:

```
https://jitsi.tailf0138e.ts.net/
```

예:

- HTTPS 작동
- Tailscale 인증서가 유효합니다.
- 경고 없음
- 공허 없음
- 아니오 : 8000
- 모든 것이 아름다운 도메인에 바로 있습니다.

⚠ 현재 Tailnet 회원만 **접속** 가능합니다.

---

## 9. 유용한 명령

버전 컨테이너:

```bash
docker-compose ps
```

로그:

```bash
docker-compose logs -f web
```

중지하려면:

```bash
docker-compose down
```

게재 상태:

```bash
tailscale serve status
```

서브 재설정:

```bash
tailscale serve reset
```

---

## 10. 사용자 추가

```bash
docker compose exec prosody prosodyctl --config /config/prosody.cfg.lua register admin meet.jitsi Jitsi1234
```

예상 출력:

```
usermanager         info	User account created: admin@meet.jitsi
```

---

## 11. TailNet Admin이 FUNNEL을 출시한 경우(선택, 공개 액세스)

**TailNet Admin**이 Funnel을 활성화하면 다음을 노출할 수 있습니다.
방화벽, 모뎀 또는 고정 IP에 의존하지 않고 유효한 HTTPS를 사용하여 전체 인터넷에 대한 Jitsi.

유입경로를 활성화하면 다음을 수행할 수 있습니다.

```bash
tailscale funnel --https=443 http://127.0.0.1:8000
```

그리고 액세스는 다음과 같습니다.

```
https://jitsi.tailf0138e.ts.net/
```

---

### 🔶 중요 사항: 퍼널 해제 방법

**TailNet 관리자**만 Funnel을 활성화할 수 있습니다.

관리자는 다음을 수행해야 합니다.

1. 입력하다:
https://login.tailscale.com/admin/acls

2. 사이드 메뉴에서 다음을 클릭합니다.
   **설정 → 유입경로**

3. 옵션을 활성화하십시오:
✔ **이 tailnet에 유입경로 허용**

4. 또한 다음을 활성화합니다.
✔ **jitsi** 장치 선택
(또는 `tailscale set --hostname`으로 설정한 이름)

5. 구하다.

그런 다음 다음을 테스트합니다.

```bash
tailscale funnel status
```

활성화되면 명령에서 오류 발생이 중지되고 Funnel을 정상적으로 활성화할 수 있습니다.

---

### ✔ 퍼널이 활성화되면 변경되는 사항

- Jitsi는 공개적으로 접근 가능합니다(TailNet 없이).
- 자동으로 유효한 HTTPS(Tailscale의 Let's Encrypt를 통해)
- URL은 그대로 유지됩니다.
  ```
  https://jitsi.tailf0138e.ts.net/
  ```
- 누구와도 공유 가능

---

### ✔ 변하지 않는 것

- 이전 튜토리얼의 어떤 내용도 중단되지 않습니다.
- 내부 서비스는 계속 작동합니다
- Docker를 수정할 필요가 없습니다.
- Jitsi는 다시 시작할 필요가 없습니다.

---

## 끝
수정된 구성, 깨끗함, 구멍 없음.
모든 것이 잘 정돈되어 있고 작동 중입니다.
