# PipeWire e WirePlumber no Void Linux
Guia objetivo, baseado diretamente no padrão adotado pelo Void Linux.

Documentação oficial:  
https://docs.voidlinux.org/config/media/pipewire.html

---

## 1. Instalação dos pacotes necessários

O Void Linux separa alguns componentes de áudio em pacotes distintos. Para uma configuração completa do PipeWire com substituição do PulseAudio, suporte a ALSA e JACK, instale:

```
sudo xbps-install -S \
  pipewire \
  wireplumber \
  pipewire-pulse \
  pipewire-alsa \
  pipewire-jack \
  pavucontrol \
  pulsemixer
```

Notas:
- `wireplumber` é o session manager recomendado.
- `pipewire-pulse` substitui totalmente o PulseAudio.
- Não execute `pipewire-media-session` junto do WirePlumber.

---

## 2. Estrutura de configuração

As configurações podem ser feitas por usuário ou por sistema.

Configuração por usuário:
```
~/.config/pipewire/
```

Exemplos fornecidos pelo Void ficam em:
```
/usr/share/examples/pipewire/
```

Crie os diretórios de drop-in:
```
mkdir -p ~/.config/pipewire/pipewire.conf.d
mkdir -p ~/.config/pipewire/pipewire-pulse.conf.d
```

---

## 3. Ativação da camada PulseAudio (pipewire-pulse)

O PipeWire utiliza arquivos `.example`. Para habilitar:

Arquivo principal:
```
ln -s /usr/share/examples/pipewire/pipewire-pulse.conf.example \
      ~/.config/pipewire/pipewire-pulse.conf
```

Drop-ins:
```
cp -a /usr/share/examples/pipewire/pipewire-pulse.conf.d/* \
      ~/.config/pipewire/pipewire-pulse.conf.d/ 2>/dev/null || true
```

---

## 4. Ativação do ALSA

```
ln -s /usr/share/examples/pipewire/pipewire.conf.d/10-alsa.conf.example \
      ~/.config/pipewire/pipewire.conf.d/10-alsa.conf
```

---

## 5. Ativação do JACK (opcional)

```
ln -s /usr/share/examples/pipewire/pipewire.conf.d/20-jack.conf.example \
      ~/.config/pipewire/pipewire.conf.d/20-jack.conf
```

---

## 6. Inicialização

### 6.1 Ambientes gráficos (XFCE, LXQt, GNOME, KDE, Cinnamon, MATE)

Nada precisa ser configurado.  
O PipeWire inicia automaticamente via DBus ao iniciar a sessão gráfica.

---

### 6.2 Window Managers (i3, bspwm, dwm, openbox, etc.)

O PipeWire depende de DBus. Em sessões via `.xinitrc`, utilize:

```
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

dbus-run-session -- sh -c '
  pipewire &
  wireplumber &
  pipewire-pulse &
  exec i3
'
```

Para Sway:

```
exec pipewire
exec wireplumber
exec pipewire-pulse
```

---

## 7. Grupos necessários (somente sem elogind)

Em sistemas somente com runit:

```
sudo usermod -aG audio,video $USER
```

Depois, faça logout e login.

> `loginctl` não funciona sem elogind.

---

## 8. Verificação

PipeWire:
```
ps aux | grep pipewire
```

WirePlumber:
```
ps aux | grep wireplumber
```

PulseAudio compatível:
```
pactl info | grep "Server Name"
```

Saída esperada:
```
Server Name: PulseAudio (on PipeWire 0.3.x)
```

---

## 9. Solução de problemas

Remover configurações antigas do PulseAudio:
```
rm -rf ~/.config/pulse ~/.pulse
```

Verificar DBus:
```
ps aux | grep dbus
```

Iniciar manualmente para depuração:
```
dbus-run-session -- pipewire
```

Caso aplicações reclamem de PulseAudio:
```
pipewire-pulse &
```

---

## 10. Ferramentas de diagnóstico

```
pw-cli ls Node
pw-dump
pw-top
```

---

## 11. Resumo

Instalação:
```
sudo xbps-install pipewire wireplumber pipewire-pulse pipewire-alsa pipewire-jack
```

Configuração:
```
~/.config/pipewire/pipewire.conf.d/
~/.config/pipewire/pipewire-pulse.conf.d/
```

Inicialização:
- DE: automático  
- WM: necessário configurar `.xinitrc` ou equivalente

PipeWire opera em nível de sessão, não utiliza serviços runit.

---
