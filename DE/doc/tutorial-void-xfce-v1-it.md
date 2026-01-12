# 🐧 Void Linux + XFCE4 — Tutorial


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

```
#limpar cache de pacotes
sudo rm -fv /var/cache/xbps/*.xbps /var/cache/xbps/*.sig*
```

## 2. Installa Xorg + Xinit + Xterm
```
sudo xbps-install -y xorg xinit xterm
```

## 3. Installa XFCE4 completo
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

## 4. Installa un Display Manager (LXDM o SDDM)

Scegli **solo un** display manager da installare e abilitare.

### 4.1 LXDM — leggero e tradizionale (consigliato per macchine semplici)

```
sudo xbps-install -y lxdm
```

### 4.2 SDDM: moderno e completo

```
sudo xbps-install -y sddm voidbr-sddm-themes
```

>Nota:
> LXDM è semplice, veloce e consuma poche risorse.
> SDDM offre un migliore supporto visivo, temi e integrazione con gli ambienti Qt.

---

## 5. Driver video

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

### Di proprietà di Nvidia
```
sudo xbps-install -y mesa-nouveau-dri
```

## 6. Disabilita XFCE-Polkit (buggato: non utilizzare)
⚠️ **IMPORTANTE:**
```
mkdir -p ~/.config/autostart
cp /etc/xdg/autostart/xfce-polkit.desktop ~/.config/autostart/
sed -i 's/^Hidden=.*/Hidden=true/' ~/.config/autostart/xfce-polkit.desktop || echo 'Hidden=true' >> ~/.config/autostart/xfce-polkit.desktop
```

## 7. Installa PipeWire (Modern Void Sound)
```
sudo xbps-install -y pipewire wireplumber alsa-pipewire pulseaudio-utils pavucontrol libspa-bluetooth libjack-pipewire alsa-plugins-pulseaudio xfce4-pulseaudio-plugin)
```



## 8. Integra ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 9. Abilita il server pipewire-pulse (compatibile con PulseAudio)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 10. Abilita l'avvio automatico di PipeWire nella sessione XFCE
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 11. Plugin essenziali XFCE (icona audio)
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
> Se l'icona dell'audio continua a non essere visualizzata al primo accesso
apri il terminale ed esegui questo script, quindi disconnettiti e accedi nuovamente

## 12. configura fuso orario: imposta il fuso orario
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 13. configurare le impostazioni locali
```
sudo sed -i -e 's/^#\(en_US.UTF-8 UTF-8\)/\1/' -e 's/^#\(pt_BR.UTF-8 UTF-8\)/\1/' /etc/default/libc-locales
```

## 14. Personalizza /etc/rc.conf. Imposta il fuso orario, il layout della tastiera e il carattere predefiniti della console.
⚠️ **Cambia secondo necessità**

```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="America/Sao_Paulo"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
EOF
```

## 15. Personalizza /etc/locale.conf. Imposta la lingua. Cambia secondo necessità.
⚠️ **IMPORTANTE:**
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
LANGUAGE=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
EOF
```

## 16. Riconfigurare
```
sudo xbps-reconfigure -fa
```

## 17. Crea .xinitrc (per startx)
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
xsetroot -cursor_name left_ptr &
exec startxfce4
EOF
```

## 18. Attivare i servizi obbligatori (runit)
```
sudo ln -sf /etc/sv/dbus /var/service/
sudo ln -sf /etc/sv/elogind /var/service/
sudo ln -sf /etc/sv/polkitd /var/service/
sudo ln -sf /etc/sv/NetworkManager /var/service/
```

## 19. Abilita il display manager scelto (runit)
⚠️ Attenzione: **abilitane solo uno**.

**LXDM**
```
sudo ln -sf /etc/sv/lxdm /var/service/
```

**SDDM**
```
sudo ln -sf /etc/sv/sddm /var/service/
```

## Finitura
- Se LXDM è attivo: avvia direttamente nella GUI.
- Se vuoi la modalità classica: startx
