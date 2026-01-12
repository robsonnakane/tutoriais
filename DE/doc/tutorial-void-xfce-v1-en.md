# 🐧 Void Linux + XFCE4 — Tutorial


> ⚠️ **IMPORTANT — READ BEFORE STARTING**
>
> This tutorial **should NOT be run as `root`**, except when **explicitly indicated**.
>
> All commands were designed to be executed by **a common user**, using `sudo` when necessary.
>
> Run the entire tutorial logged in as `root`:
> - breaks permissions logic
> - invalidates steps like `sudo` configuration
> - may generate silent errors or unexpected behavior
>
> 👉 **Recommendation**
> If you have just installed the system and are logged in as `root`:
>
> 1. Create a common user
> 2. Login with this user
> 3. Follow the tutorial normally
>
> Classic rule for Unix/Linux systems:
>
> **`root` is an exception. Common user is the rule.**

---

## 0. Configure sudo - (wheel group) - avoid asking for root password
```
sudo usermod -aG wheel "$USER"

sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
EOF

#Permissões obrigatórias
sudo chmod 440 /etc/sudoers.d/g_wheel
```

## 1. Update the system
```
sudo xbps-install -Syu
```

```
#limpar cache de pacotes
sudo rm -fv /var/cache/xbps/*.xbps /var/cache/xbps/*.sig*
```

## 2. Install Xorg + Xinit + Xterm
```
sudo xbps-install -y xorg xinit xterm
```

## 3. Install full XFCE4
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

## 4. Install a Display Manager (LXDM or SDDM)

Choose **just one** display manager to install and enable.

### 4.1 LXDM — light and traditional (recommended for simple machines)

```
sudo xbps-install -y lxdm
```

### 4.2 SDDM — modern and complete

```
sudo xbps-install -y sddm voidbr-sddm-themes
```

>Note:
> LXDM is simple, fast and consumes few resources.
> SDDM offers better visual support, themes and integration with Qt environments.

---

## 5. Display Drivers

### Intel
```
sudo xbps-install -y mesa-dri linux-firmware-intel
```

### new AMD (amdgpu)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

### old AMD
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### Nvidia proprietary
```
sudo xbps-install -y mesa-nouveau-dri
```

## 6. Disable XFCE-Polkit (bugged — don't use)
⚠️ **IMPORTANT:**
```
mkdir -p ~/.config/autostart
cp /etc/xdg/autostart/xfce-polkit.desktop ~/.config/autostart/
sed -i 's/^Hidden=.*/Hidden=true/' ~/.config/autostart/xfce-polkit.desktop || echo 'Hidden=true' >> ~/.config/autostart/xfce-polkit.desktop
```

## 7. Install PipeWire (Modern Void Sound)
```
sudo xbps-install -y pipewire wireplumber alsa-pipewire pulseaudio-utils pavucontrol libspa-bluetooth libjack-pipewire alsa-plugins-pulseaudio xfce4-pulseaudio-plugin)
```



## 8. Integrate ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 9. Enable pipewire-pulse server (PulseAudio compat)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 10. Enable PipeWire autostart in XFCE session
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 11. XFCE Essential Plugins (Sound Icon)
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
> If the sound icon still does not appear on the first login
open the terminal and run this script, then logout and login again

## 12. configure timezone - sets the time zone
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 13. configure locales
```
sudo sed -i -e 's/^#\(en_US.UTF-8 UTF-8\)/\1/' -e 's/^#\(pt_BR.UTF-8 UTF-8\)/\1/' /etc/default/libc-locales
```

## 14. Customize /etc/rc.conf. Sets the console's default time zone, keyboard layout, and font.
⚠️ **Change as needed**

```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="America/Sao_Paulo"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
EOF
```

## 15. Customize /etc/locale.conf. Sets the language. Change as needed.
⚠️ **IMPORTANT:**
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
LANGUAGE=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
EOF
```

## 16. Reconfigure
```
sudo xbps-reconfigure -fa
```

## 17. Create .xinitrc (for startx)
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
xsetroot -cursor_name left_ptr &
exec startxfce4
EOF
```

## 18. Activate mandatory services (runit)
```
sudo ln -sf /etc/sv/dbus /var/service/
sudo ln -sf /etc/sv/elogind /var/service/
sudo ln -sf /etc/sv/polkitd /var/service/
sudo ln -sf /etc/sv/NetworkManager /var/service/
```

## 19. Enable the chosen display manager (runit)
⚠️ Attention: **enable only one of them**.

**LXDM**
```
sudo ln -sf /etc/sv/lxdm /var/service/
```

**SDDM**
```
sudo ln -sf /etc/sv/sddm /var/service/
```

## Finishing
- If LXDM is active: boot directly into GUI.
- If you want classic mode: startx
