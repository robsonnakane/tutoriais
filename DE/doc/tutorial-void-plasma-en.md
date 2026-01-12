# 🐧 Void Linux + KDE Plasma + PipeWire — Tutorial

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

## 2. Install the complete Plasma (meta-package)
```
sudo xbps-install -y plasma noto-fonts-emoji
```

## 3. Install SDDM (official KDE display manager)
```
sudo xbps-install -y sddm
```

## 4. Install audio with PipeWire (full sound)

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

## 5. Video drivers (choose just one)

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

### Nvidia (open driver)
```
sudo xbps-install -y mesa-nouveau-dri
```

### Nvidia (owner)
```
sudo xbps-install -y void-repo-nonfree
sudo xbps-install -y nvidia
```

## 6. Activate mandatory services (runit)
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/sddm /var/service/
```

## 7. (Optional) Create .xinitrc for startx
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
exec startplasma-x11
EOF
```

## Finishing
- Using SDDM → the system boots directly into KDE Plasma.
- Without SDDM → use `startx` (if `.xinitrc` exists).

