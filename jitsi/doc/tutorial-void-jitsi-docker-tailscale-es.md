# Jitsi Meet + Docker + Tailscale no Void Linux
## Guía final, revisada y correcta (raíz)

Este tutorial cubre **todo** lo que se hizo, incluyendo:
- instalando los paquetes correctos
- activación de servicios en Void (runit)
- clonar la pila docker-jitsi-meet
- corrección de asignación de puertos (127.0.0.1)
- ajuste .env
- Ajuste docker-compose.yml
- Problemas del puerto 80 con nginx nativo
- Limitaciones de Tailnet (sin embudo)
- solución a través de Tailscale Serve (interno)
- acceso final a través de `https://jitsi.tailf0138e.ts.net`
- sin IP fija, sin puertos de apertura, sin DNS externo

---

## 1. Instale los paquetes necesarios

```bash
xbps-install -Sy docker docker-compose tailscale git
```

Habilitar servicios anulados:

```bash
ln -s /etc/sv/docker /var/service/
ln -s /etc/sv/tailscaled /var/service/
sv status docker tailscaled
```

---

## 2. Activar y autenticar Tailscale

```bash
tailscale up
```

Abra el enlace en el navegador, inicie sesión, autorice.

Cambie el nombre del dispositivo dentro de Tailnet:

```bash
tailscale set --hostname=jitsi
```

Confirmar nombre DNS:

```bash
tailscale status --json | grep DNSName
```

Esperado:

```
"DNSName": "jitsi.tailf0138e.ts.net.",
```

⚠ Importante:
- Si NO eres administrador de Tailnet
- Por lo tanto **El embudo está bloqueado**
- Pero el servicio interno funciona perfectamente.

---

## 3. Descarga y prepara la pila de Jitsi.

```bash
mkdir -p /opt/jitsi
cd /opt/jitsi
git clone https://github.com/jitsi/docker-jitsi-meet.git
cd docker-jitsi-meet
cp env.example .env
./gen-passwords.sh
```

---

## 4. Ajuste `.env` para usarlo con Tailscale
(primero necesitamos confirmar la IP y el DNS interno de Tailnet)

Antes de editar `.env`, confirme:

**1 — IP interna de escala trasera**

```bash
tailscale ip -4
```

Ejemplo real utilizado:
```
100.75.137.60
```

**2 — Nombre DNS interno de la máquina en Tailnet**

```bash
tailscale status --json | grep DNSName
```

Resultado real:

```
"DNSName": "jitsi.tailf0138e.ts.net.",
```

Este es el dominio **verdadero interno** generado por Tailscale, según el nombre de host configurado y el ID de Tailnet.

Solo después de eso edite el `.env`:

```bash
nano /opt/jitsi/docker-jitsi-meet/.env
```

Configúrelo así:

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

### Justificaciones:
- **PUBLIC_URL apunta al nombre interno de Tailscale**
ya que es la URL real utilizada para acceder al servidor dentro de Tailnet.

- **HTTPS dentro del contenedor está deshabilitado porque TLS proviene de Tailscale**
(Tailscale Serve proporciona HTTPS integrado y no necesitamos el nginx TLS de Jitsi).

- **No utilizamos Let's Encrypt porque no hay ningún dominio público ni embudo publicado**
y el administrador de TailNet aún no ha habilitado la función, por lo que no existe TLS público, solo interno.

---

## 5. Ajuste docker-compose.yml
(muy importante: aquí es donde solucionamos el mayor dolor de cabeza)

Este paso es absolutamente IMPRESCINDIBLE porque aquí es donde corregimos:

- El nginx de Void aparece en lugar de Jitsi
- Conflicto de puerto 80/8000
- Tailscale Serve se queja de que "solo se admite localhost"
- Jitsi recibe servicio externo sin querer
- El backend no funciona en Serve
- la necesidad de exponer solo local
- inicio automático del contenedor
- preparación para el futuro FUNNEL sin cambiar nada después

El servicio `web` SÓLO debe exponerse en localhost, porque:

- **Tailscale Serve REQUIERE backend en 127.0.0.1**
(La versión actual de Serve solo acepta localhost; de lo contrario, genera un error de proxy)
- **Evita conflictos con nginx de Void**, que se ejecuta en el puerto 80 del sistema
(por eso decía "¡Bienvenido a nginx!")
- **Garantiza que se sirvan rutas a Jitsi**, no al nginx del host
- **Evita la exposición accidental en Internet**, ya que localhost no acepta conexiones externas
- **Garantiza compatibilidad futura con FUNNEL**, si el administrador de TailNet lo lanza
- **Con `restart: always`, los contenedores se inician automáticamente después del reinicio**, sin runit adicional

Editar la redacción:

```bash
nano /opt/jitsi/docker-jitsi-meet/docker-compose.yml
```

Y déjalo EXACTAMENTE así:

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

Breve explicación:

- **127.0.0.1:8000 → 80**
→ El puerto 80 del contenedor solo existe internamente y el destinatario es 127.0.0.1
→ Por eso Tailscale Serve puede redirigir correctamente

- **reiniciar: siempre**
→ Si Void se reinicia, Jitsi vuelve sola
→ Si Docker se reinicia, Jitsi regresa solo
→ Si hay un corte de energía, regresa solo

- **Esto elimina al 100% el problema de Void nginx**
- **Esto hace que Jitsi sea invisible en la Internet pública** (lo cual es deseable dentro de Tailnet)
- **Esto configura todo para activar Funnel en el futuro con un solo comando**

Guardar y salir.

---

## 6. Cargue la pila de Docker

```bash
docker-compose up -d
docker-compose ps
```

Confirma que la web es **127.0.0.1:8000 → 80**.

Pruebe la interfaz dentro del servidor:

```bash
curl -I http://127.0.0.1:8000
```

Esperado:

```
HTTP/1.1 200 OK
Server: nginx
```

⚠ Si viste "¡Bienvenido a nginx!", era nginx de Void.
Esta prueba aseguró que el backend de Jitsi sea correcto.

---

## 7. Exponer mediante Tailscale Serve (interno)

Restablecer cualquier regla anterior:

```bash
tailscale serve reset
```

Cree el proxy interno:

```bash
tailscale serve --bg http://127.0.0.1:8000
```

Resultado esperado:

```
Available within your tailnet:

https://jitsi.tailf0138e.ts.net/
|-- proxy http://127.0.0.1:8000
```

Verificar estado:

```bash
tailscale serve status
```

Tailscale Serve ahora sirve correctamente a Jitsi.

---

## 8. Acceso a través de Tailnet (funciona en CUALQUIER red)

En su computadora portátil, teléfono celular, PC, siempre que haya iniciado sesión en Tailscale:

```
https://jitsi.tailf0138e.ts.net/
```

Sí:

- HTTPS funciona
- El certificado de escala trasera es válido
- Sin advertencia
- Sin nginx vacío
- No: 8000
- Todo directamente en el hermoso dominio.

⚠ Acceso **solo** para miembros de Tailnet (por ahora).

---

## 9. Comandos útiles

Ver contenedores:

```bash
docker-compose ps
```

Registros:

```bash
docker-compose logs -f web
```

Para parar:

```bash
docker-compose down
```

Estado de servicio:

```bash
tailscale serve status
```

Restablecer servicio:

```bash
tailscale serve reset
```

---

## 10. Agregar usuarios

```bash
docker compose exec prosody prosodyctl --config /config/prosody.cfg.lua register admin meet.jitsi Jitsi1234
```

Resultado esperado:

```
usermanager         info	User account created: admin@meet.jitsi
```

---

## 11. Cuando el administrador de TailNet lanza FUNNEL (opcional, acceso público)

Si el **Administrador de TailNet** habilita Funnel, podrá exponer
Jitsi para todo INTERNET, con HTTPS válido, sin depender de firewall, módem o IP fija.

Con Funnel activado, realizas:

```bash
tailscale funnel --https=443 http://127.0.0.1:8000
```

Y el acceso se convierte en:

```
https://jitsi.tailf0138e.ts.net/
```

---

### 🔶 NOTA IMPORTANTE: CÓMO LIBERAR EL EMBUDO

Solo el **administrador de TailNet** puede habilitar Funnel.

El administrador debe hacer:

1. Ingresar:
CHILE_REF_0_CHILI

2. En el menú lateral, haga clic en:
   **Configuración → Embudo**

3. Activa la opción:
✔ **Permitir embudo para esta red trasera**

4. Y también activa:
✔ seleccione el dispositivo **jitsi**
(o el nombre que configuró con `tailscale set --hostname`)

5. Ahorrar.

Después de eso, pruebas:

```bash
tailscale funnel status
```

Si está habilitado, el comando deja de dar error y puedes activar Funnel normalmente.

---

### ✔ ¿Qué cambia cuando el embudo está activo?

- Jitsi es accesible PÚBLICAMENTE (sin TailNet)
- HTTPS válido automáticamente (a través de Let's Encrypt de Tailscale)
- La URL permanece:
  ```
  https://jitsi.tailf0138e.ts.net/
  ```
- Se puede compartir con CUALQUIERA

---

### ✔ Lo que NO cambia

- Nada del tutorial anterior se rompe.
- El servicio interno sigue funcionando.
- No es necesario modificar Docker
- Jitsi no necesita reiniciar

---

## FIN
Configuración revisada, limpia, sin agujeros.
Todo en buen estado y funcionando.
