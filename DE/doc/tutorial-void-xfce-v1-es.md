# 🐧 Void Linux + XFCE4 — Tutorial


> ⚠️ **IMPORTANTE — LEER ANTES DE COMENZAR**
>
> Este tutorial **NO debe ejecutarse como `root`**, excepto cuando **se indique explícitamente**.
>
> Todos los comandos fueron diseñados para ser ejecutados por **un usuario común**, usando `sudo` cuando sea necesario.
>
> Ejecute todo el tutorial iniciando sesión como `root`:
> - rompe la lógica de permisos
> - invalida pasos como la configuración `sudo`
> - puede generar errores silenciosos o comportamiento inesperado
>
> 👉 **Recomendación**
> Si acaba de instalar el sistema y ha iniciado sesión como `root`:
>
> 1. Crear un usuario común
> 2. Inicia sesión con este usuario
> 3. Sigue el tutorial normalmente.
>
> Regla clásica para sistemas Unix/Linux:
>
> **`root` es una excepción. El usuario común es la regla.**

---

## 0. Configure sudo - (grupo de ruedas): evite solicitar la contraseña de root
```
sudo usermod -aG wheel "$USER"

sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
EOF

#Permissões obrigatórias
sudo chmod 440 /etc/sudoers.d/g_wheel
```

## 1. Actualiza el sistema
```
sudo xbps-install -Syu
```

```
#limpar cache de pacotes
sudo rm -fv /var/cache/xbps/*.xbps /var/cache/xbps/*.sig*
```

## 2. Instale Xorg + Xinit + Xterm
```
sudo xbps-install -y xorg xinit xterm
```

## 3. Instale XFCE4 completo
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

## 4. Instale un Administrador de pantalla (LXDM o SDDM)

Elija **solo un** administrador de pantalla para instalar y habilitar.

### 4.1 LXDM: ligero y tradicional (recomendado para máquinas simples)

```
sudo xbps-install -y lxdm
```

### 4.2 SDDM: moderno y completo

```
sudo xbps-install -y sddm voidbr-sddm-themes
```

>Nota:
> LXDM es simple, rápido y consume pocos recursos.
> SDDM ofrece mejor soporte visual, temas e integración con entornos Qt.

---

## 5. Controladores de pantalla

### Intel
```
sudo xbps-install -y mesa-dri linux-firmware-intel
```

### nuevo AMD (amdgpu)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

### viejo AMD
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### propiedad de Nvidia
```
sudo xbps-install -y mesa-nouveau-dri
```

## 6. Deshabilite XFCE-Polkit (con errores, no lo use)
⚠️ **IMPORTANTE:**
```
mkdir -p ~/.config/autostart
cp /etc/xdg/autostart/xfce-polkit.desktop ~/.config/autostart/
sed -i 's/^Hidden=.*/Hidden=true/' ~/.config/autostart/xfce-polkit.desktop || echo 'Hidden=true' >> ~/.config/autostart/xfce-polkit.desktop
```

## 7. Instale PipeWire (sonido vacío moderno)
```
sudo xbps-install -y pipewire wireplumber alsa-pipewire pulseaudio-utils pavucontrol libspa-bluetooth libjack-pipewire alsa-plugins-pulseaudio xfce4-pulseaudio-plugin)
```



## 8. Integrar ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 9. Habilite el servidor pipewire-pulse (compatible con PulseAudio)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 10. Habilite el inicio automático de PipeWire en la sesión XFCE
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 11. Complementos esenciales de XFCE (icono de sonido)
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
> Si el icono de sonido aún no aparece en el primer inicio de sesión
abra la terminal y ejecute este script, luego cierre sesión y vuelva a iniciar sesión

## 12. configurar zona horaria: establece la zona horaria
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 13. configurar configuraciones regionales
```
sudo sed -i -e 's/^#\(en_US.UTF-8 UTF-8\)/\1/' -e 's/^#\(pt_BR.UTF-8 UTF-8\)/\1/' /etc/default/libc-locales
```

## 14. Personalice /etc/rc.conf. Establece la zona horaria, la distribución del teclado y la fuente predeterminadas de la consola.
⚠️ **Cambie según sea necesario**

```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="America/Sao_Paulo"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
EOF
```

## 15. Personalice /etc/locale.conf. Establece el idioma. Cambie según sea necesario.
⚠️ **IMPORTANTE:**
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
LANGUAGE=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
EOF
```

## 16. Reconfigurar
```
sudo xbps-reconfigure -fa
```

## 17. Cree .xinitrc (para startx)
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
xsetroot -cursor_name left_ptr &
exec startxfce4
EOF
```

## 18. Activar servicios obligatorios (runit)
```
sudo ln -sf /etc/sv/dbus /var/service/
sudo ln -sf /etc/sv/elogind /var/service/
sudo ln -sf /etc/sv/polkitd /var/service/
sudo ln -sf /etc/sv/NetworkManager /var/service/
```

## 19. Habilite el administrador de pantalla elegido (runit)
⚠️ Atención: **habilita solo uno de ellos**.

**LXDM**
```
sudo ln -sf /etc/sv/lxdm /var/service/
```

**SDDM**
```
sudo ln -sf /etc/sv/sddm /var/service/
```

## Refinamiento
- Si LXDM está activo: inicie directamente en la GUI.
- Si quieres el modo clásico: startx
