# 🐧 Void Linux + XFCE4 — 教程


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

```
#limpar cache de pacotes
sudo rm -fv /var/cache/xbps/*.xbps /var/cache/xbps/*.sig*
```

## 2.安裝Xorg+Xinit+Xterm
```
sudo xbps-install -y xorg xinit xterm
```

## 3.安裝完整的XFCE4
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

## 4. 安裝顯示管理器（LXDM 或 SDDM）

選擇**僅一個**顯示管理器來安裝和啟用。

### 4.1 LXDM — 輕型和傳統（推薦用於簡單機器）

```
sudo xbps-install -y lxdm
```

### 4.2 SDDM——現代且完整

```
sudo xbps-install -y sddm voidbr-sddm-themes
```

>注意：
> LXDM 簡單、快速且消耗資源少。
> SDDM 提供更好的視覺支持、主題以及與 Qt 環境的集成。

---

## 5. 顯示驅動程序

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

### Nvidia專有
```
sudo xbps-install -y mesa-nouveau-dri
```

## 6. 禁用 XFCE-Polkit（已被竊聽 - 不要使用）
⚠️ **重要：**
```
mkdir -p ~/.config/autostart
cp /etc/xdg/autostart/xfce-polkit.desktop ~/.config/autostart/
sed -i 's/^Hidden=.*/Hidden=true/' ~/.config/autostart/xfce-polkit.desktop || echo 'Hidden=true' >> ~/.config/autostart/xfce-polkit.desktop
```

## 7.安裝PipeWire（現代虛空聲音）
```
sudo xbps-install -y pipewire wireplumber alsa-pipewire pulseaudio-utils pavucontrol libspa-bluetooth libjack-pipewire alsa-plugins-pulseaudio xfce4-pulseaudio-plugin)
```



## 8. 集成 ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 9.啟用pipewire-pulse服務器（PulseAudio兼容）
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 10. 在 XFCE 會話中啟用 PipeWire 自動啟動
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 11. XFCE 基本插件（聲音圖標）
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
> 如果第一次登錄時仍然沒有出現聲音圖標
打開終端並運行此腳本，然後註銷並再次登錄

## 12.configure timezone——設置時區
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 13.配置區域設置
```
sudo sed -i -e 's/^#\(en_US.UTF-8 UTF-8\)/\1/' -e 's/^#\(pt_BR.UTF-8 UTF-8\)/\1/' /etc/default/libc-locales
```

## 14. 自定義/etc/rc.conf。設置控制台的默認時區、鍵盤佈局和字體。
⚠️ **根據需要更改**

```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="America/Sao_Paulo"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
EOF
```

## 15. 自定義/etc/locale.conf。設置語言。根據需要進行更改。
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

## 17.創建.xinitrc（用於startx）
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
xsetroot -cursor_name left_ptr &
exec startxfce4
EOF
```

## 18.激活強制服務（runit）
```
sudo ln -sf /etc/sv/dbus /var/service/
sudo ln -sf /etc/sv/elogind /var/service/
sudo ln -sf /etc/sv/polkitd /var/service/
sudo ln -sf /etc/sv/NetworkManager /var/service/
```

## 19. 啟用所選的顯示管理器（運行單元）
⚠️注意：**僅啟用其中一項**。

**LXDM**
```
sudo ln -sf /etc/sv/lxdm /var/service/
```

**SDDM**
```
sudo ln -sf /etc/sv/sddm /var/service/
```

## 精加工
- 如果 LXDM 處於活動狀態：直接引導至 GUI。
- 如果你想要經典模式：startx
