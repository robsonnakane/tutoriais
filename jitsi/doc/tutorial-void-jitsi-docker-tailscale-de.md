# Jitsi Meet + Docker + Tailscale ohne Void Linux
## Endgültiger, überarbeiteter und korrekter Leitfaden (Root)

Dieses Tutorial deckt **alles** ab, was getan wurde, einschließlich:
- die richtigen Pakete installieren
- Aktivierung von Diensten in Void (runit)
- Klonen Sie den Stapel Docker-Jitsi-Meet
- Port-Mapping-Fix (127.0.0.1)
- .env-Optimierung
- docker-compose.yml-Optimierung
- Port 80-Probleme mit nativem Nginx
- Tailnet-Einschränkungen (ohne Funnel)
- Lösung über Tailscale Serve (intern)
- Endgültiger Zugriff über `https://jitsi.tailf0138e.ts.net`
- Keine feste IP, keine offenen Ports, kein externes DNS

---

## 1. Installieren Sie die erforderlichen Pakete

```bash
xbps-install -Sy docker docker-compose tailscale git
```

Stornieren Sie Dienste:

```bash
ln -s /etc/sv/docker /var/service/
ln -s /etc/sv/tailscaled /var/service/
sv status docker tailscaled
```

---

## 2. Tailscale aktivieren und authentifizieren

```bash
tailscale up
```

Link im Browser öffnen, anmelden, autorisieren.

Benennen Sie das Gerät in Tailnet um:

```bash
tailscale set --hostname=jitsi
```

Bestätigen Sie den DNS-Namen:

```bash
tailscale status --json | grep DNSName
```

Erwartet:

```
"DNSName": "jitsi.tailf0138e.ts.net.",
```

⚠ Wichtig:
- Wenn Sie KEIN Tailnet-Administrator sind
- Daher ist **Trichter blockiert**
- Aber Internal Serve funktioniert perfekt

---

## 3. Laden Sie den Jitsi-Stack herunter und bereiten Sie ihn vor

```bash
mkdir -p /opt/jitsi
cd /opt/jitsi
git clone https://github.com/jitsi/docker-jitsi-meet.git
cd docker-jitsi-meet
cp env.example .env
./gen-passwords.sh
```

---

## 4. Passen Sie „.env“ für die Verwendung mit Tailscale an
(Zuerst müssen wir Tailnets IP und internes DNS bestätigen)

Bestätigen Sie vor dem Bearbeiten von „.env“ Folgendes:

**1 – Tailscale interne IP**

```bash
tailscale ip -4
```

Tatsächlich verwendetes Beispiel:
```
100.75.137.60
```

**2 – Interner DNS-Name der Maschine auf Tailnet**

```bash
tailscale status --json | grep DNSName
```

Tatsächliches Ergebnis:

```
"DNSName": "jitsi.tailf0138e.ts.net.",
```

Dies ist die **echte interne** Domäne, die von Tailscale generiert wird, basierend auf dem konfigurierten Hostnamen und der Tailnet-ID.

Bearbeiten Sie erst danach die „.env“:

```bash
nano /opt/jitsi/docker-jitsi-meet/.env
```

Konfigurieren Sie es so:

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

### Begründungen:
- **PUBLIC_URL verweist auf den internen Namen von Tailscale**
da es sich um die tatsächliche URL handelt, die für den Zugriff auf den Server innerhalb von Tailnet verwendet wird.

- **HTTPS im Container ist deaktiviert, da TLS von Tailscale stammt**
(Tailscale Serve bietet integriertes HTTPS und wir benötigen Jitsis Nginx TLS nicht).

- **Wir verwenden Let’s Encrypt nicht, da keine Public Domain oder kein Funnel veröffentlicht ist**
und der TailNet-Administrator hat die Funktion noch nicht aktiviert, daher existiert kein öffentliches TLS – nur intern.

---

## 5. Passen Sie docker-compose.yml an
(sehr wichtig – hier haben wir die größten Kopfschmerzen behoben)

Dieser Schritt ist unbedingt WESENTLICH, da wir hier Folgendes korrigiert haben:

- Voids Nginx erscheint an Jitsis Stelle
- 80/8000-Port-Konflikt
- Tailscale Serve beschwert sich, dass „nur localhost unterstützt wird“
- Jitsi wird unbeabsichtigt von außen serviert
- Backend funktioniert bei Serve nicht
- die Notwendigkeit, nur lokal verfügbar zu machen
- Automatischer Containerstart
- Vorbereitung für den zukünftigen FUNNEL, ohne dass danach etwas geändert werden muss

Der „Web“-Dienst darf NUR auf localhost verfügbar gemacht werden, weil:

- **Tailscale Serve ERFORDERT Backend auf 127.0.0.1**
(Die aktuelle Version von Serve akzeptiert nur localhost, andernfalls wird ein Proxy-Fehler ausgegeben.)
- **Vermeidet Konflikte mit Voids Nginx**, das auf Systemport 80 läuft
(Deshalb hieß es „Willkommen bei Nginx!“)
- **Stellt sicher, dass Serve-Routen zu Jitsi** erfolgen, nicht zum Nginx des Hosts
- **Verhindert versehentliche Offenlegung im Internet**, da localhost keine externen Verbindungen akzeptiert
- **Garantiert zukünftige Kompatibilität mit FUNNEL**, wenn der TailNet Admin veröffentlicht wird
- **Mit „restart: Always“ starten Container nach dem Neustart automatisch**, ohne zusätzliche Runit

Bearbeiten Sie den Inhalt:

```bash
nano /opt/jitsi/docker-jitsi-meet/docker-compose.yml
```

Und lassen Sie es GENAU so:

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

Kurze Erklärung:

- **127.0.0.1:8000 → 80**
→ Port 80 des Containers existiert nur intern und der Empfänger ist 127.0.0.1
→ Deshalb kann Tailscale Serve korrekt umleiten

- **Neustart: immer**
→ Wenn Void neu startet, kommt Jitsi alleine zurück
→ Wenn Docker neu startet, kommt Jitsi von selbst zurück
→ Bei einem Stromausfall kehrt es von alleine zurück

- **Dadurch wird das Void-Nginx-Problem zu 100 % beseitigt**
- **Dadurch wird Jitsi im öffentlichen Internet unsichtbar** (was innerhalb von Tailnet erwünscht ist)
- **Damit ist alles vorbereitet, um Funnel in Zukunft mit nur einem Befehl zu aktivieren**

Speichern und beenden.

---

## 6. Laden Sie den Docker-Stack hoch

```bash
docker-compose up -d
docker-compose ps
```

Bestätigen Sie, dass das Web **127.0.0.1:8000 → 80** ist.

Testen Sie das Frontend innerhalb des Servers:

```bash
curl -I http://127.0.0.1:8000
```

Erwartet:

```
HTTP/1.1 200 OK
Server: nginx
```

⚠ Wenn Sie „Willkommen bei Nginx!“ gesehen haben, war es Voids Nginx.
Dieser Test stellte sicher, dass das Jitsi-Backend korrekt ist.

---

## 7. Offenlegung über Tailscale Serve (intern)

Setzen Sie alle vorherigen Regeln zurück:

```bash
tailscale serve reset
```

Erstellen Sie den internen Proxy:

```bash
tailscale serve --bg http://127.0.0.1:8000
```

Erwartete Ausgabe:

```
Available within your tailnet:

https://jitsi.tailf0138e.ts.net/
|-- proxy http://127.0.0.1:8000
```

Status prüfen:

```bash
tailscale serve status
```

„Tailscale Serve“ serviert Jitsi jetzt korrekt.

---

## 8. Zugriff über Tailnet (funktioniert in JEDEM Netzwerk)

Auf Ihrem Laptop, Mobiltelefon, PC – solange Sie bei Tailscale angemeldet sind:

```
https://jitsi.tailf0138e.ts.net/
```

Ja:

- HTTPS funktioniert
- Das Tailscale-Zertifikat ist gültig
- Keine Warnung
- Kein ungültiger Nginx
- Nein: 8000
- Alles direkt in der schönen Domäne

⚠ Zugriff **nur** für Tailnet-Mitglieder (vorerst).

---

## 9. Nützliche Befehle

Ver-Container:

```bash
docker-compose ps
```

Protokolle:

```bash
docker-compose logs -f web
```

Zum Stoppen:

```bash
docker-compose down
```

Servierstatus:

```bash
tailscale serve status
```

Aufschlag zurücksetzen:

```bash
tailscale serve reset
```

---

## 10. Benutzer hinzufügen

```bash
docker compose exec prosody prosodyctl --config /config/prosody.cfg.lua register admin meet.jitsi Jitsi1234
```

Erwartete Ausgabe:

```
usermanager         info	User account created: admin@meet.jitsi
```

---

## 11. Wenn der TailNet-Administrator FUNNEL freigibt (optional, öffentlicher Zugriff)

Wenn der **TailNet-Administrator** Funnel aktiviert, können Sie ihn freigeben
Jitsi für das gesamte INTERNET, mit gültigem HTTPS, ohne Abhängigkeit von Firewall, Modem oder fester IP.

Wenn der Funnel aktiviert ist, führen Sie Folgendes aus:

```bash
tailscale funnel --https=443 http://127.0.0.1:8000
```

Und der Zugriff wird:

```
https://jitsi.tailf0138e.ts.net/
```

---

### 🔶 WICHTIGER HINWEIS: SO LÖSEN SIE DEN TRICHTER

Nur der **TailNet-Administrator** kann Funnel aktivieren.

Der Administrator muss Folgendes tun:

1. Eingeben:
https://login.tailscale.com/admin/acls

2. Klicken Sie im Seitenmenü auf:
   **Einstellungen → Trichter**

3. Aktivieren Sie die Option:
✔ **Trichter für dieses Hecknetz zulassen**

4. Und aktivieren Sie außerdem:
✔ **Jitsi**-Gerät auswählen
(oder der Name, den Sie mit „tailscale set --hostname“ festgelegt haben)

5. Speichern.

Danach testen Sie:

```bash
tailscale funnel status
```

Wenn es aktiviert ist, gibt der Befehl keinen Fehler mehr aus und Sie können Funnel normal aktivieren.

---

### ✔ Was ändert sich, wenn Funnel aktiv ist?

- Jitsi ist ÖFFENTLICH zugänglich (ohne TailNet)
- Automatisch gültiges HTTPS (über Let's Encrypt von Tailscale)
- Die URL bleibt bestehen:
  ```
  https://jitsi.tailf0138e.ts.net/
  ```
- Kann mit JEDEM geteilt werden

---

### ✔ Was sich NICHT ändert

- Nichts von den vorherigen Tutorial-Unterbrechungen
- Der interne Service funktioniert weiterhin
- Docker muss nicht geändert werden
- Jitsi muss nicht neu gestartet werden

---

## ENDE
Überarbeitete Konfiguration, sauber, keine Löcher.
Alles in Ordnung und funktioniert.
