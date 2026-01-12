# 🐧 Void Linux + KDE Plasma + PipeWire – Tutorial

> ⚠️ **WICHTIG – LESEN SIE, BEVOR SIE BEGINNEN**
>
> Dieses Tutorial **sollte NICHT als „root“** ausgeführt werden, außer wenn **ausdrücklich angegeben**.
>
> Alle Befehle wurden so konzipiert, dass sie von **einem normalen Benutzer** ausgeführt werden können, wobei bei Bedarf „sudo“ verwendet wird.
>
> Führen Sie das gesamte Tutorial aus, angemeldet als „root“:
> – bricht die Berechtigungslogik
> – macht Schritte wie die „sudo“-Konfiguration ungültig
> – kann stille Fehler oder unerwartetes Verhalten erzeugen
>
> 👉 **Empfehlung**
> Wenn Sie das System gerade erst installiert haben und als „root“ angemeldet sind:
>
> 1. Erstellen Sie einen gemeinsamen Benutzer
> 2. Melden Sie sich mit diesem Benutzer an
> 3. Befolgen Sie die Anleitung wie gewohnt
>
> Klassische Regel für Unix/Linux-Systeme:
>
> **`root` ist eine Ausnahme. Normaler Benutzer ist die Regel.**

---


## 0. Konfigurieren Sie sudo – (Radgruppe) – vermeiden Sie die Nachfrage nach dem Root-Passwort
```
sudo usermod -aG wheel "$USER"

sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
EOF

#Permissões obrigatórias
sudo chmod 440 /etc/sudoers.d/g_wheel
```

## 1. Aktualisieren Sie das System
```
sudo xbps-install -Syu
```

## 2. Installieren Sie das komplette Plasma (Meta-Paket)
```
sudo xbps-install -y plasma noto-fonts-emoji
```

## 3. Installieren Sie SDDM (offizieller KDE-Display-Manager)
```
sudo xbps-install -y sddm
```

## 4. Audio mit PipeWire installieren (voller Sound)

### PipeWire + WirePlumber + ALSA + Pulse kompatibel
```
sudo xbps-install -y \
  pipewire \
  wireplumber \
  alsa-pipewire \
  libjack-pipewire \
  alsa-utils \
  pavucontrol
```

## 5. Videotreiber (wählen Sie nur einen aus)

### Intel
```
sudo xbps-install -y mesa-dri linux-firmware-intel
```

### neuer AMD (amdgpu)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

### alter AMD
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### Nvidia (offener Treiber)
```
sudo xbps-install -y mesa-nouveau-dri
```

### Nvidia (Eigentümer)
```
sudo xbps-install -y void-repo-nonfree
sudo xbps-install -y nvidia
```

## 6. Pflichtdienste aktivieren (runit)
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/sddm /var/service/
```

## 7. (Optional) Erstellen Sie .xinitrc für startx
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
exec startplasma-x11
EOF
```

## Abschluss
- Mit SDDM → bootet das System direkt in KDE Plasma.
- Ohne SDDM → „startx“ verwenden (falls „.xinitrc“ vorhanden ist).

