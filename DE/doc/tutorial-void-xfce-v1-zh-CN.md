# 🐧 Void Linux + XFCE4 — 教程


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

```
#limpar cache de pacotes
sudo rm -fv /var/cache/xbps/*.xbps /var/cache/xbps/*.sig*
```

## 2.安装Xorg+Xinit+Xterm
```
sudo xbps-install -y xorg xinit xterm
```

## 3.安装完整的XFCE4
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

## 4. 安装显示管理器（LXDM 或 SDDM）

选择**仅一个**显示管理器来安装和启用。

### 4.1 LXDM — 轻型和传统（推荐用于简单机器）

```
sudo xbps-install -y lxdm
```

### 4.2 SDDM——现代且完整

```
sudo xbps-install -y sddm voidbr-sddm-themes
```

>注意：
> LXDM 简单、快速且消耗资源少。
> SDDM 提供更好的视觉支持、主题以及与 Qt 环境的集成。

---

## 5. 显示驱动程序

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

### Nvidia专有
```
sudo xbps-install -y mesa-nouveau-dri
```

## 6. 禁用 XFCE-Polkit（已被窃听 - 不要使用）
⚠️ **重要：**
```
mkdir -p ~/.config/autostart
cp /etc/xdg/autostart/xfce-polkit.desktop ~/.config/autostart/
sed -i 's/^Hidden=.*/Hidden=true/' ~/.config/autostart/xfce-polkit.desktop || echo 'Hidden=true' >> ~/.config/autostart/xfce-polkit.desktop
```

## 7.安装PipeWire（现代虚空声音）
```
sudo xbps-install -y pipewire wireplumber alsa-pipewire pulseaudio-utils pavucontrol libspa-bluetooth libjack-pipewire alsa-plugins-pulseaudio xfce4-pulseaudio-plugin)
```



## 8. 集成 ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 9.启用pipewire-pulse服务器（PulseAudio兼容）
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 10. 在 XFCE 会话中启用 PipeWire 自动启动
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 11. XFCE 基本插件（声音图标）
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
> 如果第一次登录时仍然没有出现声音图标
打开终端并运行此脚本，然后注销并再次登录

## 12.configure timezone——设置时区
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 13.配置区域设置
```
sudo sed -i -e 's/^#\(en_US.UTF-8 UTF-8\)/\1/' -e 's/^#\(pt_BR.UTF-8 UTF-8\)/\1/' /etc/default/libc-locales
```

## 14. 自定义/etc/rc.conf。设置控制台的默认时区、键盘布局和字体。
⚠️ **根据需要更改**

```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="America/Sao_Paulo"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
EOF
```

## 15. 自定义/etc/locale.conf。设置语言。根据需要进行更改。
⚠️ **重要：**
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
LANGUAGE=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
EOF
```

## 16. 重新配置
```
sudo xbps-reconfigure -fa
```

## 17.创建.xinitrc（用于startx）
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
xsetroot -cursor_name left_ptr &
exec startxfce4
EOF
```

## 18.激活强制服务（runit）
```
sudo ln -sf /etc/sv/dbus /var/service/
sudo ln -sf /etc/sv/elogind /var/service/
sudo ln -sf /etc/sv/polkitd /var/service/
sudo ln -sf /etc/sv/NetworkManager /var/service/
```

## 19. 启用所选的显示管理器（运行单元）
⚠️注意：**仅启用其中一项**。

**LXDM**
```
sudo ln -sf /etc/sv/lxdm /var/service/
```

**SDDM**
```
sudo ln -sf /etc/sv/sddm /var/service/
```

## 精加工
- 如果 LXDM 处于活动状态：直接引导至 GUI。
- 如果你想要经典模式：startx
