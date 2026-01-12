# 🐧 Void Linux + KDE Plasma + PipeWire — Tutorial

> ⚠️ **IMPORTANTE — LEGGERE PRIMA DI INIZIARE**
>
> Questo tutorial **NON deve essere eseguito come `root`**, tranne quando **esplicitamente indicato**.
>
> Tutti i comandi sono stati progettati per essere eseguiti da **un utente comune**, utilizzando `sudo` quando necessario.
>
> Esegui l'intero tutorial accedendo come `root`:
> - interrompe la logica delle autorizzazioni
> - invalida passaggi come la configurazione di `sudo`
> - potrebbe generare errori silenziosi o comportamenti imprevisti
>
> 👉 **Raccomandazione**
> Se hai appena installato il sistema e hai effettuato l'accesso come `root`:
>
> 1. Creare un utente comune
> 2. Accedi con questo utente
> 3. Segui normalmente il tutorial
>
> Regola classica per i sistemi Unix/Linux:
>
> **`root` è un'eccezione. L'utente comune è la regola.**

---


## 0. Configura sudo - (gruppo ruote) - evita di chiedere la password di root
```
sudo usermod -aG wheel "$USER"

sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
EOF

#Permissões obrigatórias
sudo chmod 440 /etc/sudoers.d/g_wheel
```

## 1. Aggiorna il sistema
```
sudo xbps-install -Syu
```

## 2. Installa il Plasma completo (metapacchetto)
```
sudo xbps-install -y plasma noto-fonts-emoji
```

## 3. Installa SDDM (display manager ufficiale di KDE)
```
sudo xbps-install -y sddm
```

## 4. Installa l'audio con PipeWire (audio completo)

### PipeWire + WirePlumber + ALSA + Pulse compat
```
sudo xbps-install -y \
  pipewire \
  wireplumber \
  alsa-pipewire \
  libjack-pipewire \
  alsa-utils \
  pavucontrol
```

## 5. Driver video (scegli solo uno)

### Intel
```
sudo xbps-install -y mesa-dri linux-firmware-intel
```

### nuova AMD (amdgpu)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

### vecchia AMD
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### Nvidia (driver aperto)
```
sudo xbps-install -y mesa-nouveau-dri
```

### Nvidia (proprietario)
```
sudo xbps-install -y void-repo-nonfree
sudo xbps-install -y nvidia
```

## 6. Attiva i servizi obbligatori (runit)
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/sddm /var/service/
```

## 7. (Facoltativo) Crea .xinitrc per startx
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
exec startplasma-x11
EOF
```

## Finitura
- Utilizzando SDDM → il sistema si avvia direttamente in KDE Plasma.
- Senza SDDM → usa `startx` (se esiste `.xinitrc`).

