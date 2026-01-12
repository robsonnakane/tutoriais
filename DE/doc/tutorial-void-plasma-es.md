# 🐧 Void Linux + KDE Plasma + PipeWire — Tutorial

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

## 2. Instale el Plasma completo (metapaquete)
```
sudo xbps-install -y plasma noto-fonts-emoji
```

## 3. Instale SDDM (administrador de pantalla oficial de KDE)
```
sudo xbps-install -y sddm
```

## 4. Instalar audio con PipeWire (sonido completo)

### PipeWire + WirePlumber + ALSA + Compatible con pulsos
```
sudo xbps-install -y \
  pipewire \
  wireplumber \
  alsa-pipewire \
  libjack-pipewire \
  alsa-utils \
  pavucontrol
```

## 5. Controladores de vídeo (elige sólo uno)

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

### Nvidia (controlador abierto)
```
sudo xbps-install -y mesa-nouveau-dri
```

### Nvidia (propietario)
```
sudo xbps-install -y void-repo-nonfree
sudo xbps-install -y nvidia
```

## 6. Activar servicios obligatorios (runit)
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/sddm /var/service/
```

## 7. (Opcional) Cree .xinitrc para startx
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
exec startplasma-x11
EOF
```

## Refinamiento
- Usando SDDM → el sistema arranca directamente en KDE Plasma.
- Sin SDDM → use `startx` (si existe `.xinitrc`).

