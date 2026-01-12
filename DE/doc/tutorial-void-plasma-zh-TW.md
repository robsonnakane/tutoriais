# 🐧 Void Linux + KDE Plasma + PipeWire — 教程

> ⚠️ **重要 — 在開始之前閱讀**
>
> 本教程**不應以“root”身份運行**，除非**明確指出**。
>
> 所有命令均設計為由 **普通用戶** 執行，必要時使用“sudo”。
>
> 以“root”身份登錄運行整個教程：
> - 破壞權限邏輯
> - 使“sudo”配置等步驟無效
> - 可能會產生無提示錯誤或意外行為
>
> 👉 **推薦**
> 如果您剛剛安裝系統並以“root”身份登錄：
>
> 1.創建普通用戶
> 2. 使用該用戶登錄
> 3.正常按照教程進行操作
>
> Unix/Linux 系統的經典規則：
>
> **`root` 是一個例外。普通用戶是規則。 **

---


## 0.配置sudo - (輪組) - 避免詢問root密碼
```
sudo usermod -aG wheel "$USER"

sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
EOF

#Permissões obrigatórias
sudo chmod 440 /etc/sudoers.d/g_wheel
```

## 1.更新系統
```
sudo xbps-install -Syu
```

## 2. 安裝完整的 Plasma（元包）
```
sudo xbps-install -y plasma noto-fonts-emoji
```

## 3.安裝SDDM（官方KDE顯示管理器）
```
sudo xbps-install -y sddm
```

## 4.用PipeWire安裝音頻（全聲音）

### PipeWire + WirePlumber + ALSA + 脈衝兼容
```
sudo xbps-install -y \
  pipewire \
  wireplumber \
  alsa-pipewire \
  libjack-pipewire \
  alsa-utils \
  pavucontrol
```

## 5. 視頻驅動程序（僅選擇一個）

### 英特爾
```
sudo xbps-install -y mesa-dri linux-firmware-intel
```

### 新AMD (AMDGPU)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

### 老AMD
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### Nvidia（開放驅動程序）
```
sudo xbps-install -y mesa-nouveau-dri
```

### 英偉達（所有者）
```
sudo xbps-install -y void-repo-nonfree
sudo xbps-install -y nvidia
```

## 6.激活強制服務（runit）
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/sddm /var/service/
```

## 7.（可選）為startx創建.xinitrc
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
exec startplasma-x11
EOF
```

## 精加工
- 使用 SDDM → 系統直接引導至 KDE Plasma。
- 沒有 SDDM → 使用 `startx` （如果 `.xinitrc` 存在）。

