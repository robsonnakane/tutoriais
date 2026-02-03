# 🌐 Subindo um Apache Server no Void Linux (glibc)
## 🔐 HTTPS + Redirect + Autenticação HTTP Basic

---

## 🧪 AMBIENTE

- **Sistema Operacional:** Void Linux Server (glibc)
- **Servidor Web:** Apache 2.4
- **Init System:** runit
- **Conteúdo:** Estático (MkDocs)
- **Autenticação:** HTTP Basic (htpasswd)

---

## 1️⃣ INSTALAÇÃO DOS PACOTES

Logar com o usuário root

```bash
su -
```

Atualizar repositórios, pacotes e sistema operacional:

```bash
xbps-install -Syu
```

Instalar Apache e utilitários:

```bash
xbps-install apache openssl
```

---

## 2️⃣ ATIVAR O SERVIÇO APACHE (RUNIT)

Ativar o serviço do Apache:

```bash
ln -s /etc/sv/apache /var/service/
```

Verificar status:

```bash
sv status apache
```

Resultado esperado:

```bash
run: apache: (pid 1045) 16s; run: log: (pid 1044) 16s
```

No Void Linux a configuração principal fica em:

```bash
/etc/apache/httpd.conf
```

As configurações auxiliares ficam em:

```bash
/etc/apache/extra/
```

Os VirtualHosts em:

```bash
/etc/apache/extra/httpd-vhosts.conf
```

E o arquivo de SSL em:

```bash
/etc/apache/extra/httpd-ssl.conf
```

---

## 3️⃣ TESTE O SERVIÇO DO APACHE

```bash
curl -I http://192.168.70.100
```

Resultado esperado:

```bash
HTTP/1.1 200 OK
Date: Mon, 02 Feb 2026 18:25:24 GMT
Server: Apache
Content-Type: text/html;charset=ISO-8859-1
```

---

## 4️⃣ CONFIRMAR OS INCLUDES QUE ATIVAM OS MÓDULOS

No Void, **não existe**:

- /etc/apache2
- sites-available
- a2enmod, a2ensite

Os módulos são carregados via `httpd.conf` através dos includes.

```bash
vim /etc/apache/httpd.conf
```

VirtualHosts e SSL:

```bash
Include /etc/apache/extra/httpd-vhosts.conf
Include /etc/apache/extra/httpd-ssl.conf
```

Módulos necessários:

```bash
LoadModule ssl_module libexec/apache/mod_ssl.so
LoadModule socache_shmcb_module libexec/apache/mod_socache_shmcb.so
LoadModule rewrite_module libexec/apache/mod_rewrite.so
LoadModule auth_basic_module libexec/apache/mod_auth_basic.so
LoadModule authn_file_module libexec/apache/mod_authn_file.so
```

---

## 5️⃣ DOCUMENT ROOT E PERMISSÕES

DocumentRoot padrão:

```bash
/srv/www/apache
```

Criar estrutura:

```bash
mkdir /srv/www/apache/publico
mkdir /srv/www/apache/aluno
```

Permissões:

```bash
chown -R _apache:_apache /srv/www/apache/
chmod -R 755 /srv/www/apache/
```

---

## 6️⃣ CERTIFICADO SSL AUTOASSINADO

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/certs/conectux.key \
  -out /etc/ssl/certs/conectux.crt \
  -subj "/C=BR/ST=SP/L=LOCAL/O=Conectux/CN=192.168.122.100" \
  -addext "subjectAltName = IP:192.168.122.100, DNS:conectux.edu"
```

---

## 7️⃣ CONFIGURAÇÃO HTTPS (VIRTUALHOST 443)

```bash
cp /etc/apache/extra/http-ssl.conf{,.bkp}
vim /etc/apache/extra/httpd-ssl.conf
```

*(conteúdo mantido conforme original)*

---

## 8️⃣ HTTP + REDIRECT PARA HTTPS

```bash
cp /etc/apache/extra/httpd-vhosts.conf{,.bkp}
vim /etc/apache/extra/httpd-vhosts.conf
```

---

## 9️⃣ CRIAÇÃO DE USUÁRIOS (HTPASSWD)

```bash
htpasswd -c /etc/apache/matriculados aluno01
```

---

## 🔟 VALIDAR CONFIGURAÇÃO

```bash
apachectl configtest
sv restart apache
```

---

## 🧪 TESTES E TROUBLESHOOTING

```bash
netstat -an | grep :80
netstat -an | grep :443
```

---

## 📄 CONTEÚDO HTML E TESTES

*(conteúdo mantido conforme original)*

---

## 🔐 IMPORTAR CERTIFICADO NO FIREFOX

about:preferences#privacy

---

## 📚 INSTALAÇÃO DO MKDOCS

https://github.com/VoidLinuxBR/tutoriais/blob/main/misc/tutorial-void-server-mkdocs.md

---

## ✅ CONCLUSÃO

- Apache com HTTPS funcional
- Redirect HTTP → HTTPS
- Autenticação HTTP Basic
- Estrutura limpa

🎯 **THAT'S ALL FOLKS**

