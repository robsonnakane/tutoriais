# 🐧 Void Linux + XFCE4 – Tutorial


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

```
#limpar cache de pacotes
sudo rm -fv /var/cache/xbps/*.xbps /var/cache/xbps/*.sig*
```

## 2. Installieren Sie Xorg + Xinit + Xterm
```
sudo xbps-install -y xorg xinit xterm
```

## 3. Installieren Sie das vollständige XFCE4
```
sudo xbps-install -y xfce4 \
   network-manager-applet \
   xfce4-plugins \
   arc-theme \
   xfce4-screenshooter \
   xfce4-whiskermenu-plugin \
   firefox \
   firefox-i18n-pt-BR \
   xarchiver \
   thunar-archive-plugin \
   gnome-disk-utility \
   gparted \
   gvfs \
   p7zip \
   unzip \
   noto-fonts-emoji \
   font-hack-ttf \
   nerd-fonts-symbols-ttf \
   nerd-fonts-ttf \
   htop
```

## 4. Installieren Sie einen Display Manager (LXDM oder SDDM)

Wählen Sie **nur einen** Display Manager zum Installieren und Aktivieren aus.

### 4.1 LXDM – leicht und traditionell (empfohlen für einfache Maschinen)

```
sudo xbps-install -y lxdm
```

### 4.2 SDDM – modern und vollständig

```
sudo xbps-install -y sddm voidbr-sddm-themes
```

>Hinweis:
> LXDM ist einfach, schnell und verbraucht wenig Ressourcen.
> SDDM bietet bessere visuelle Unterstützung, Themen und Integration mit Qt-Umgebungen.

---

## 5. Treiber anzeigen

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

### Nvidia-proprietär
```
sudo xbps-install -y mesa-nouveau-dri
```

## 6. Deaktivieren Sie XFCE-Polkit (fehlerhaft – nicht verwenden)
⚠️ **WICHTIG:**
```
mkdir -p ~/.config/autostart
cp /etc/xdg/autostart/xfce-polkit.desktop ~/.config/autostart/
sed -i 's/^Hidden=.*/Hidden=true/' ~/.config/autostart/xfce-polkit.desktop || echo 'Hidden=true' >> ~/.config/autostart/xfce-polkit.desktop
```

## 7. Installieren Sie PipeWire (Modern Void Sound)
```
sudo xbps-install -y pipewire wireplumber alsa-pipewire pulseaudio-utils pavucontrol libspa-bluetooth libjack-pipewire alsa-plugins-pulseaudio xfce4-pulseaudio-plugin)
```



## 8. Integrieren Sie ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 9. Pipewire-Pulse-Server aktivieren (PulseAudio-kompatibel)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 10. Aktivieren Sie den PipeWire-Autostart in der XFCE-Sitzung
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 11. XFCE Essential Plugins (Sound-Symbol)
```
sudo xbps-install -y xfce4-pulseaudio-plugin xfce4-notifyd

XML="$HOME/.config/xfce4/xfconf/xfce-perchannel-xml/xfce4-panel.xml"
if [[ -f "$XML" ]] && ! grep -q 'pulseaudio' "$XML"; then
   next=$(grep -o 'plugin-[0-9]*' "$XML" 2>/dev/null | sed 's/.*-//' | sort -n | tail -1)
   next=$((next + 1))

   sed -i '$d' "$XML"   # remove o ũltimo </channel>
   sed -i '$d' "$XML"   # remove o último </property>

   cat << EOF >> "$XML"
    <property name="plugin-$next" type="string" value="pulseaudio">
      <property name="enable-keyboard-shortcuts" type="bool" value="true"/>
    </property>
  </property>
</channel>
EOF

   sed -i '/<property name="panel-1" type="empty">/,/<\/property>/ {
      /<property name="plugin-ids" type="array">/,/<\/property>/ {
      /<\/property>/i\
         <value type="int" value="'"$next"'"/>
      }
   }' "$XML"
fi
```
> Wenn das Soundsymbol beim ersten Login immer noch nicht erscheint
Öffnen Sie das Terminal und führen Sie dieses Skript aus. Melden Sie sich dann ab und erneut an

## 12. Zeitzone konfigurieren – legt die Zeitzone fest
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 13. Konfigurieren Sie Gebietsschemas
```
sudo sed -i -e 's/^#\(en_US.UTF-8 UTF-8\)/\1/' -e 's/^#\(pt_BR.UTF-8 UTF-8\)/\1/' /etc/default/libc-locales
```

## 14. Passen Sie /etc/rc.conf an. Legt die Standardzeitzone, das Tastaturlayout und die Schriftart der Konsole fest.
⚠️ **Bei Bedarf ändern**

```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="America/Sao_Paulo"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
EOF
```

## 15. Passen Sie /etc/locale.conf an. Legt die Sprache fest. Ändern Sie es nach Bedarf.
⚠️ **WICHTIG:**
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
LANGUAGE=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
EOF
```

## 16. Neu konfigurieren
```
sudo xbps-reconfigure -fa
```

## 17. Erstellen Sie .xinitrc (für startx)
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
xsetroot -cursor_name left_ptr &
exec startxfce4
EOF
```

## 18. Pflichtdienste aktivieren (runit)
```
sudo ln -sf /etc/sv/dbus /var/service/
sudo ln -sf /etc/sv/elogind /var/service/
sudo ln -sf /etc/sv/polkitd /var/service/
sudo ln -sf /etc/sv/NetworkManager /var/service/
```

## 19. Aktivieren Sie den ausgewählten Display-Manager (Runit).
⚠️ Achtung: **Aktivieren Sie nur eine davon**.

**LXDM**
```
sudo ln -sf /etc/sv/lxdm /var/service/
```

**SDDM**
```
sudo ln -sf /etc/sv/sddm /var/service/
```

## Abschluss
- Wenn LXDM aktiv ist: Booten Sie direkt in die GUI.
- Wenn Sie den klassischen Modus wünschen: startx
