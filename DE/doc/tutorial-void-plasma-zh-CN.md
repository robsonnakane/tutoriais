# 🐧 Void Linux + KDE Plasma + PipeWire — 教程

> ⚠️ **重要 — 在开始之前阅读**
>
> 本教程**不应以“root”身份运行**，除非**明确指出**。
>
> 所有命令均设计为由 **普通用户** 执行，必要时使用“sudo”。
>
> 以“root”身份登录运行整个教程：
> - 破坏权限逻辑
> - 使“sudo”配置等步骤无效
> - 可能会产生无提示错误或意外行为
>
> 👉 **推荐**
> 如果您刚刚安装系统并以“root”身份登录：
>
> 1.创建普通用户
> 2. 使用该用户登录
> 3.正常按照教程进行操作
>
> Unix/Linux 系统的经典规则：
>
> **`root` 是一个例外。普通用户是规则。**

---


## 0.配置sudo - (轮组) - 避免询问root密码
```
sudo usermod -aG wheel "$USER"

sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
EOF

#Permissões obrigatórias
sudo chmod 440 /etc/sudoers.d/g_wheel
```

## 1.更新系统
```
sudo xbps-install -Syu
```

## 2. 安装完整的 Plasma（元包）
```
sudo xbps-install -y plasma noto-fonts-emoji
```

## 3.安装SDDM（官方KDE显示管理器）
```
sudo xbps-install -y sddm
```

## 4.用PipeWire安装音频（全声音）

### PipeWire + WirePlumber + ALSA + 脉冲兼容
```
sudo xbps-install -y \
  pipewire \
  wireplumber \
  alsa-pipewire \
  libjack-pipewire \
  alsa-utils \
  pavucontrol
```

## 5. 视频驱动程序（仅选择一个）

### 英特尔
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

### Nvidia（开放驱动程序）
```
sudo xbps-install -y mesa-nouveau-dri
```

### 英伟达（所有者）
```
sudo xbps-install -y void-repo-nonfree
sudo xbps-install -y nvidia
```

## 6.激活强制服务（runit）
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/sddm /var/service/
```

## 7.（可选）为startx创建.xinitrc
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
exec startplasma-x11
EOF
```

## 精加工
- 使用 SDDM → 系统直接引导至 KDE Plasma。
- 没有 SDDM → 使用 `startx` （如果 `.xinitrc` 存在）。

