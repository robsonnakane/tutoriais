# Jitsi Meet + Docker + Tailscale no Void Linux
## Guida finale, rivista e corretta (root)

Questo tutorial copre **tutto** ciò che è stato fatto, incluso:
- installando i pacchetti giusti
- attivazione servizi in Void (runit)
- clona lo stack docker-jitsi-meet
- correzione della mappatura delle porte (127.0.0.1)
- modifica .env
- Modifica docker-compose.yml
- problemi con la porta 80 con nginx nativo
- Limitazioni Tailnet (senza Funnel)
- soluzione tramite Tailscale Serve (interno)
- accesso finale tramite `https://jitsi.tailf0138e.ts.net`
- nessun IP fisso, nessuna porta apribile, nessun DNS esterno

---

## 1. Installa i pacchetti richiesti

```bash
xbps-install -Sy docker docker-compose tailscale git
```

Abilita servizi Void:

```bash
ln -s /etc/sv/docker /var/service/
ln -s /etc/sv/tailscaled /var/service/
sv status docker tailscaled
```

---

## 2. Attiva e autentica Tailscale

```bash
tailscale up
```

Apri il collegamento nel browser, accedi, autorizza.

Rinominare il dispositivo all'interno di Tailnet:

```bash
tailscale set --hostname=jitsi
```

Conferma nome DNS:

```bash
tailscale status --json | grep DNSName
```

Previsto:

```
"DNSName": "jitsi.tailf0138e.ts.net.",
```

⚠ Importante:
- Se NON sei un amministratore di Tailnet
- Pertanto **Il funnel è bloccato**
- Ma il servizio interno funziona perfettamente

---

## 3. Scarica e prepara lo stack Jitsi

```bash
mkdir -p /opt/jitsi
cd /opt/jitsi
git clone https://github.com/jitsi/docker-jitsi-meet.git
cd docker-jitsi-meet
cp env.example .env
./gen-passwords.sh
```

---

## 4. Regola `.env` per l'uso con Tailscale
(prima dobbiamo confermare l'IP di Tailnet e il DNS interno)

Prima di modificare `.env`, conferma:

**1 — IP interno Tailscale**

```bash
tailscale ip -4
```

Esempio reale utilizzato:
```
100.75.137.60
```

**2 — Nome DNS interno della macchina su Tailnet**

```bash
tailscale status --json | grep DNSName
```

Risultato effettivo:

```
"DNSName": "jitsi.tailf0138e.ts.net.",
```

Questo è il **vero dominio interno** generato da Tailscale, in base al nome host configurato e all'ID Tailnet.

Solo dopo modifica il file `.env`:

```bash
nano /opt/jitsi/docker-jitsi-meet/.env
```

Configuralo in questo modo:

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

### Giustificazioni:
- **PUBLIC_URL punta al nome interno di Tailscale**
poiché è l'URL effettivo utilizzato per accedere al server all'interno di Tailnet.

- **HTTPS all'interno del contenitore è disabilitato perché TLS proviene da Tailscale**
(Tailscale Serve fornisce HTTPS integrato e non abbiamo bisogno di nginx TLS di Jitsi).

- **Non utilizziamo Let's Encrypt perché non viene rilasciato alcun dominio pubblico o Funnel**
e l'amministratore TailNet non ha ancora abilitato la funzione, quindi il TLS pubblico non esiste, ma solo interno.

---

## 5. Modifica docker-compose.yml
(molto importante: è qui che abbiamo risolto il problema più grande)

Questo passaggio è assolutamente ESSENZIALE perché è qui che abbiamo corretto:

- Nginx di Void appare al posto di Jitsi
- Conflitto porta 80/8000
- Tailscale Serve lamenta che "è supportato solo localhost"
- Jitsi viene servito esternamente involontariamente
- il backend non funziona su Serve
- la necessità di esporre solo a livello locale
- avvio automatico del contenitore
- preparazione per il futuro FUNNEL senza modificare nulla in seguito

Il servizio `web` deve essere esposto SOLO su localhost, perché:

- **Il servizio Tailscale RICHIEDE il backend su 127.0.0.1**
(la versione attuale di Serve accetta solo localhost, altrimenti restituisce un errore proxy)
- **Evita conflitti con nginx di Void**, che viene eseguito sulla porta di sistema 80
(ecco perché diceva "Benvenuto in nginx!")
- **Garantisce che Serve venga indirizzato a Jitsi**, non a nginx dell'host
- **Previene l'esposizione accidentale su Internet**, poiché localhost non accetta connessioni esterne
- **Garantisce la futura compatibilità con FUNNEL**, se viene rilasciato l'Admin TailNet
- **Con `restart: sempre`, i contenitori si avviano automaticamente dopo il riavvio**, senza runit aggiuntivo

Modifica la composizione:

```bash
nano /opt/jitsi/docker-jitsi-meet/docker-compose.yml
```

E lascialo ESATTAMENTE così:

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

Breve spiegazione:

- **127.0.0.1:8000 → 80**
→ La porta 80 del container esiste solo internamente e il destinatario è 127.0.0.1
→ Ecco perché Tailscale Serve può reindirizzare correttamente

- **riavvia: sempre**
→ Se Void si riavvia, Jitsi ritorna da solo
→ Se Docker si riavvia, Jitsi ritorna da solo
→ Se manca la corrente, ritorna da solo

- **Questo elimina al 100% il problema di Void nginx**
- **Ciò rende Jitsi invisibile sull'Internet pubblica** (cosa auspicata all'interno di Tailnet)
- **Questo configura tutto per attivare Funnel in futuro con un solo comando**

Salva ed esci.

---

## 6. Carica lo stack Docker

```bash
docker-compose up -d
docker-compose ps
```

Confermare che il Web sia **127.0.0.1:8000 → 80**.

Testare il frontend all'interno del server:

```bash
curl -I http://127.0.0.1:8000
```

Previsto:

```
HTTP/1.1 200 OK
Server: nginx
```

⚠ Se hai visto "Benvenuti in nginx!", era nginx di Void.
Questo test ha garantito che il backend Jitsi sia corretto.

---

## 7. Esposizione tramite Tailscale Serve (interno)

Reimposta eventuali regole precedenti:

```bash
tailscale serve reset
```

Crea il proxy interno:

```bash
tailscale serve --bg http://127.0.0.1:8000
```

Risultato previsto:

```
Available within your tailnet:

https://jitsi.tailf0138e.ts.net/
|-- proxy http://127.0.0.1:8000
```

Controlla lo stato:

```bash
tailscale serve status
```

Tailscale Serve ora serve correttamente Jitsi.

---

## 8. Accesso tramite Tailnet (funziona su QUALSIASI rete)

Sul tuo laptop, cellulare, PC, purché tu abbia effettuato l'accesso a Tailscale:

```
https://jitsi.tailf0138e.ts.net/
```

SÌ:

- HTTPS funziona
- Il certificato Tailscale è valido
- Nessun avvertimento
- Nessun nginx del Vuoto
- No: 8000
- Tutto direttamente nel bellissimo dominio

⚠ Accesso **solo** per i membri Tailnet (per ora).

---

## 9. Comandi utili

Contenitori Ver:

```bash
docker-compose ps
```

Registri:

```bash
docker-compose logs -f web
```

Per interrompere:

```bash
docker-compose down
```

Stato del servizio:

```bash
tailscale serve status
```

Reimposta servizio:

```bash
tailscale serve reset
```

---

## 10. Aggiunta di utenti

```bash
docker compose exec prosody prosodyctl --config /config/prosody.cfg.lua register admin meet.jitsi Jitsi1234
```

Risultato previsto:

```
usermanager         info	User account created: admin@meet.jitsi
```

---

## 11. Quando l'amministratore TailNet rilascia FUNNEL (facoltativo, accesso pubblico)

Se l'**amministratore TailNet** abilita la canalizzazione, sarai in grado di esporre
Jitsi per tutta INTERNET, con HTTPS valido, senza dipendere da firewall, modem o IP fisso.

Con Funnel attivato, esegui:

```bash
tailscale funnel --https=443 http://127.0.0.1:8000
```

E l'accesso diventa:

```
https://jitsi.tailf0138e.ts.net/
```

---

### 🔶 NOTA IMPORTANTE: COME RILASCIARE L'IMBUTO

Solo l'**amministratore TailNet** può abilitare la canalizzazione.

L'amministratore deve fare:

1. Inserisci:
https://login.tailscale.com/admin/acls

2. Nel menù laterale clicca su:
   **Impostazioni → Imbuto**

3. Attiva l'opzione:
✔ **Consenti canalizzazione per questa tailnet**

4. E attiva anche:
✔ seleziona il dispositivo **jitsi**
(o il nome impostato con `tailscale set --hostname`)

5. Salva.

Successivamente, prova:

```bash
tailscale funnel status
```

Se è abilitato, il comando smette di dare errori e puoi attivare Funnel normalmente.

---

### ✔ Cosa cambia quando Funnel è attivo

- Jitsi è accessibile PUBBLICAMENTE (senza TailNet)
- HTTPS automaticamente valido (tramite Let's Encrypt di Tailscale)
- L'URL rimane:
  ```
  https://jitsi.tailf0138e.ts.net/
  ```
- Può essere condiviso con CHIUNQUE

---

### ✔Cosa NON cambia

- Niente del tutorial precedente si interrompe
- Il servizio interno continua a funzionare
- Non è necessario modificare Docker
- Jitsi non ha bisogno di riavviarsi

---

## FINE
Configurazione rivista, pulita, senza buchi.
Tutto in ordine e funzionante.
