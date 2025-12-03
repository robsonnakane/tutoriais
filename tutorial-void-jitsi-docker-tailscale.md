## TUTORIAL COMPLETO E LIMPO – JITSI + DOCKER + TAILSCALE NO VOID (ROOT)

===========================================
1. INSTALAR PACOTES
===========================================

```bash
xbps-install -Sy docker docker-compose tailscale git
ln -s /etc/sv/docker /var/service/
ln -s /etc/sv/tailscaled /var/service/
sv status docker tailscaled
```

===========================================
2. INICIAR TAILSCALE E NOMEAR O SERVIDOR
===========================================

```bash
tailscale up
tailscale set --hostname=jitsi
tailscale status --json | grep DNSName
```
esperado:
```
"DNSName": "jitsi.tailf0138e.ts.net.",
```

===========================================
3. BAIXAR O STACK DOCKER DO JITSI
===========================================

```bash
mkdir -p /opt/jitsi
cd /opt/jitsi
git clone https://github.com/jitsi/docker-jitsi-meet.git
cd docker-jitsi-meet
cp env.example .env
```

===========================================
4. EDITAR O .env PARA USAR TAILSCALE
===========================================

```ini
PUBLIC_URL=https://jitsi.tailf0138e.ts.net
ENABLE_LETSENCRYPT=0
DISABLE_HTTPS=1
```

===========================================
5. AJUSTAR O docker-compose.yml PARA LOCALHOST
===========================================

Alterar no serviço `web`:

**ANTES**
```yaml
ports:
  - "8000:80"
  - "8443:443"
```

**DEPOIS**
```yaml
ports:
  - "127.0.0.1:8000:80"
  - "127.0.0.1:8443:443"
```

===========================================
6. SUBIR O JITSI
===========================================

```bash
docker-compose up -d
docker-compose ps
curl -I http://127.0.0.1:8000
```

Esperado:
```
HTTP/1.1 200 OK
```

===========================================
7. CONFIGURAR TAILSCALE SERVE (INTERNO)
===========================================

```bash
tailscale serve reset
tailscale serve --bg http://127.0.0.1:8000
tailscale serve status
```

Deve aparecer:

```
https://jitsi.tailf0138e.ts.net/ → http://127.0.0.1:8000
```

===========================================
8. COMO ACESSAR O JITSI (QUALQUER DISPOSITIVO)
===========================================

1. Instale o Tailscale  
2. Logue na mesma tailnet  
3. Abra no navegador:

```
http://jitsi.tailf0138e.ts.net/
```

===========================================
9. COMANDOS ÚTEIS
===========================================

```bash
docker-compose ps
docker-compose logs -f web
docker-compose down
tailscale serve status
tailscale serve reset
```

===========================================
10. LIBERAR FUNNEL (OPCIONAL)
===========================================

```bash
tailscale funnel --https=443 http://127.0.0.1:8000
```

Acesso público:

```
https://jitsi.tailf0138e.ts.net/
```
