# Jitsi Meet + Docker + Tailscale sans Void Linux
## Guide final, révisé et correct (racine)

Ce didacticiel couvre **tout** ce qui a été fait, notamment :
- installer les bons packages
- activation des services dans Void (runit)
- cloner la pile docker-jitsi-meet
- correctif de mappage de port (127.0.0.1)
- ajustement .env
- ajustement de docker-compose.yml
- problèmes de port 80 avec nginx natif
- Limitations du Tailnet (sans entonnoir)
- solution via Tailscale Serve (interne)
- accès final via `https://jitsi.tailf0138e.ts.net`
- pas d'IP fixe, pas d'ouverture de ports, pas de DNS externe

---

## 1. Installez les packages requis

```bash
xbps-install -Sy docker docker-compose tailscale git
```

Activer les services annulés :

```bash
ln -s /etc/sv/docker /var/service/
ln -s /etc/sv/tailscaled /var/service/
sv status docker tailscaled
```

---

## 2. Activez et authentifiez Tailscale

```bash
tailscale up
```

Ouvrez le lien dans le navigateur, connectez-vous, autorisez.

Renommez l'appareil dans Tailnet :

```bash
tailscale set --hostname=jitsi
```

Confirmez le nom DNS :

```bash
tailscale status --json | grep DNSName
```

Attendu:

```
"DNSName": "jitsi.tailf0138e.ts.net.",
```

⚠Important :
- Si vous n'êtes PAS un administrateur Tailnet
- Par conséquent, **l'entonnoir est bloqué**
- Mais le service interne fonctionne parfaitement

---

## 3. Téléchargez et préparez la pile Jitsi

```bash
mkdir -p /opt/jitsi
cd /opt/jitsi
git clone https://github.com/jitsi/docker-jitsi-meet.git
cd docker-jitsi-meet
cp env.example .env
./gen-passwords.sh
```

---

## 4. Ajustez `.env` pour une utilisation avec Tailscale
(nous devons d'abord confirmer l'adresse IP et le DNS interne de Tailnet)

Avant de modifier `.env`, confirmez :

**1 — IP interne de Tailscale**

```bash
tailscale ip -4
```

Exemple réel utilisé :
```
100.75.137.60
```

**2 — Nom DNS interne de la machine sur Tailnet**

```bash
tailscale status --json | grep DNSName
```

Résultat réel :

```
"DNSName": "jitsi.tailf0138e.ts.net.",
```

Il s'agit du **vrai domaine interne** généré par Tailscale, basé sur le nom d'hôte configuré et l'ID Tailnet.

Seulement après cela, modifiez le `.env` :

```bash
nano /opt/jitsi/docker-jitsi-meet/.env
```

Configurez-le comme ceci :

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

### Justifications :
- **PUBLIC_URL pointe vers le nom interne de Tailscale**
car il s'agit de l'URL réelle utilisée pour accéder au serveur dans Tailnet.

- **HTTPS à l'intérieur du conteneur est désactivé car TLS provient de Tailscale**
(Tailscale Serve fournit HTTPS intégré et nous n'avons pas besoin du TLS nginx de Jitsi).

- **Nous n'utilisons pas Let's Encrypt car aucun domaine public ni entonnoir n'est publié**
et l'administrateur TailNet n'a pas encore activé la fonctionnalité, donc le TLS public n'existe pas – uniquement interne.

---

## 5. Ajustez docker-compose.yml
(très important — c'est là que nous avons résolu le plus gros casse-tête)

Cette étape est absolument INDISPENSABLE car c'est là que nous avons corrigé :

- Le nginx de Void apparaît à la place de Jitsi
- Conflit de ports 80/8000
- Tailscale Serve se plaint que « seul localhost est pris en charge »
- Jitsi étant servi à l'extérieur involontairement
- le backend ne fonctionne pas sur Serve
- la nécessité d'exposer uniquement en local
- démarrage automatique du conteneur
- préparation du futur FUNNEL sans rien changer par la suite

Le service `web` doit s'exposer UNIQUEMENT sur localhost, car :

- **Tailscale Serve REQUIERT le backend sur 127.0.0.1**
(la version actuelle de Serve n'accepte que localhost, sinon elle donne une erreur de proxy)
- **Évite les conflits avec nginx de Void**, qui s'exécute sur le port système 80
(c'est pourquoi il est écrit « Bienvenue sur nginx ! »)
- **Garantit que les routes de service vers Jitsi**, et non vers le nginx de l'hôte
- **Empêche toute exposition accidentelle sur Internet**, car localhost n'accepte pas les connexions externes
- **Garantit la compatibilité future avec FUNNEL**, si l'administrateur TailNet publie
- **Avec `restart : always`, les conteneurs démarrent automatiquement après le redémarrage**, sans exécution supplémentaire

Modifiez la composition :

```bash
nano /opt/jitsi/docker-jitsi-meet/docker-compose.yml
```

Et laissez-le EXACTEMENT comme ceci :

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

Brève explication :

- **127.0.0.1:8000 → 80**
→ Le port 80 du conteneur n'existe qu'en interne, et le destinataire est 127.0.0.1
→ C'est pourquoi Tailscale Serve peut rediriger correctement

- **redémarrer : toujours**
→ Si Void redémarre, Jitsi revient seul
→ Si Docker redémarre, Jitsi revient tout seul
→ S'il y a une coupure de courant, il revient seul

- **Cela élimine à 100 % le problème de Void nginx**
- **Cela rend Jitsi invisible sur l'Internet public** (ce qui est souhaité dans Tailnet)
- **Cela configure tout pour activer Funnel à l'avenir avec une seule commande**

Enregistrez et quittez.

---

## 6. Téléchargez la pile Docker

```bash
docker-compose up -d
docker-compose ps
```

Confirmez que le Web est **127.0.0.1:8000 → 80**.

Testez le frontend au sein du serveur :

```bash
curl -I http://127.0.0.1:8000
```

Attendu:

```
HTTP/1.1 200 OK
Server: nginx
```

⚠ Si vous avez vu "Bienvenue sur nginx !", c'était le nginx de Void.
Ce test a permis de garantir que le backend Jitsi est correct.

---

## 7. Exposer via Tailscale Serve (interne)

Réinitialisez les règles précédentes :

```bash
tailscale serve reset
```

Créez le proxy interne :

```bash
tailscale serve --bg http://127.0.0.1:8000
```

Résultat attendu :

```
Available within your tailnet:

https://jitsi.tailf0138e.ts.net/
|-- proxy http://127.0.0.1:8000
```

Vérifier l'état :

```bash
tailscale serve status
```

Tailscale Serve sert désormais correctement Jitsi.

---

## 8. Accès via Tailnet (fonctionne sur N'IMPORTE QUEL réseau)

Sur votre ordinateur portable, téléphone portable, PC – tant que vous êtes connecté à Tailscale :

```
https://jitsi.tailf0138e.ts.net/
```

Oui:

- HTTPS fonctionne
- Le certificat Tailscale est valide
- Aucun avertissement
- Pas de vide nginx
- Non :8000
- Le tout directement dans le beau domaine

⚠ Accès **uniquement** pour les membres Tailnet (pour l'instant).

---

## 9. Commandes utiles

Ver conteneurs :

```bash
docker-compose ps
```

Journaux :

```bash
docker-compose logs -f web
```

Pour arrêter :

```bash
docker-compose down
```

Statut de service :

```bash
tailscale serve status
```

Réinitialiser le service :

```bash
tailscale serve reset
```

---

## 10. Ajout d'utilisateurs

```bash
docker compose exec prosody prosodyctl --config /config/prosody.cfg.lua register admin meet.jitsi Jitsi1234
```

Résultat attendu :

```
usermanager         info	User account created: admin@meet.jitsi
```

---

## 11. Lorsque l'administrateur TailNet publie FUNNEL (facultatif, accès public)

Si **TailNet Admin** active Funnel, vous pourrez exposer
Jitsi pour tout INTERNET, avec HTTPS valide, sans dépendre d'un pare-feu, d'un modem ou d'une IP fixe.

Avec Funnel activé, vous effectuez :

```bash
tailscale funnel --https=443 http://127.0.0.1:8000
```

Et l'accès devient :

```
https://jitsi.tailf0138e.ts.net/
```

---

### 🔶 REMARQUE IMPORTANTE : COMMENT LIBÉRER L'ENTONNOIR

Seul l'**administrateur TailNet** peut activer Funnel.

L'administrateur doit faire :

1. Entrer:
https://login.tailscale.com/admin/acls

2. Dans le menu latéral, cliquez sur :
   **Paramètres → Entonnoir**

3. Activez l'option :
✔ **Autoriser l'entonnoir pour ce filet arrière**

4. Et activez également :
✔ sélectionnez l'appareil **jitsi**
(ou le nom que vous avez défini avec `tailscale set --hostname`)

5. Sauvegarder.

Après cela, vous testez :

```bash
tailscale funnel status
```

Si elle est activée, la commande cesse de générer une erreur et vous pouvez activer Funnel normalement.

---

### ✔ Qu'est-ce qui change lorsque l'entonnoir est actif

- Jitsi est accessible PUBLIQUEMENT (sans TailNet)
- HTTPS automatiquement valide (via Let's Encrypt de Tailscale)
- L'URL reste :
  ```
  https://jitsi.tailf0138e.ts.net/
  ```
- Peut être partagé avec TOUT LE MONDE

---

### ✔ Ce qui NE change PAS

- Rien des pauses du tutoriel précédent
- Le service interne continue de fonctionner
- Docker n'a pas besoin d'être modifié
- Jitsi n'a pas besoin de redémarrer

---

## FIN
Configuration révisée, propre, sans trous.
Tout est en bon état et fonctionne.
