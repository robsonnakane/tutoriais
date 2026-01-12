# 🐧 Void Linux + XFCE4 — チュートリアル


> ⚠️ **重要 — 始める前にお読みください**
>
> このチュートリアルは、**明示的に示されている場合を除き、**「root」として実行しないでください**。
>
> すべてのコマンドは、必要に応じて `sudo` を使用して、**一般ユーザー** によって実行されるように設計されています。
>
> 「root」としてログインしてチュートリアル全体を実行します。
> - 権限ロジックを壊す
> - `sudo` 設定などの手順を無効にします
> - サイレントエラーまたは予期しない動作が発生する可能性があります
>
> 👉 **推奨事項**
> システムをインストールしたばかりで、「root」としてログインしている場合:
>
> 1. 共通ユーザーを作成する
> 2. このユーザーでログインします
> 3. 通常通りチュートリアルに従います
>
> Unix/Linux システムの古典的なルール:
>
> **`root` は例外です。一般ユーザーがルールです。**

---

## 0. sudo を設定します - (wheel グループ) - root パスワードを要求しないようにします
```
sudo usermod -aG wheel "$USER"

sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
EOF

#Permissões obrigatórias
sudo chmod 440 /etc/sudoers.d/g_wheel
```

## 1. システムをアップデートする
```
sudo xbps-install -Syu
```

```
#limpar cache de pacotes
sudo rm -fv /var/cache/xbps/*.xbps /var/cache/xbps/*.sig*
```

## 2. Xorg + Xinit + Xterm をインストールする
```
sudo xbps-install -y xorg xinit xterm
```

## 3.完全な XFCE4 をインストールする
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

## 4. ディスプレイマネージャー (LXDM または SDDM) をインストールします。

インストールして有効にするディスプレイ マネージャーを **1 つだけ**選択してください。

### 4.1 LXDM — 軽量で従来型 (単純なマシンに推奨)

```
sudo xbps-install -y lxdm
```

### 4.2 SDDM — 最新かつ完全な

```
sudo xbps-install -y sddm voidbr-sddm-themes
```

>注:
> LXDM はシンプルかつ高速で、リソースの消費がほとんどありません。
> SDDM は、より優れたビジュアル サポート、テーマ、Qt 環境との統合を提供します。

---

## 5. ディスプレイドライバー

### インテル
```
sudo xbps-install -y mesa-dri linux-firmware-intel
```

### 新しい AMD (amdgpu)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

### 古いAMD
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### Nvidia独自の
```
sudo xbps-install -y mesa-nouveau-dri
```

## 6. XFCE-Polkit を無効にする (バグがあるため使用しないでください)
⚠️ **重要:**
```
mkdir -p ~/.config/autostart
cp /etc/xdg/autostart/xfce-polkit.desktop ~/.config/autostart/
sed -i 's/^Hidden=.*/Hidden=true/' ~/.config/autostart/xfce-polkit.desktop || echo 'Hidden=true' >> ~/.config/autostart/xfce-polkit.desktop
```

## 7. PipeWire をインストールする (Modern Void Sound)
```
sudo xbps-install -y pipewire wireplumber alsa-pipewire pulseaudio-utils pavucontrol libspa-bluetooth libjack-pipewire alsa-plugins-pulseaudio xfce4-pulseaudio-plugin)
```



## 8. ALSA → PipeWire の統合
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 9. Pipewire-pulse サーバーを有効にする (PulseAudio compat)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 10. XFCE セッションで PipeWire 自動起動を有効にする
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 11.XFCE必須プラグイン（サウンドアイコン）
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
> 初回ログイン時にサウンドアイコンがまだ表示されない場合
ターミナルを開いてこのスクリプトを実行し、ログアウトして再度ログインします。

## 12. タイムゾーンの構成 - タイムゾーンを設定します
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 13. ロケールを設定する
```
sudo sed -i -e 's/^#\(en_US.UTF-8 UTF-8\)/\1/' -e 's/^#\(pt_BR.UTF-8 UTF-8\)/\1/' /etc/default/libc-locales
```

## 14. /etc/rc.conf をカスタマイズします。本体のデフォルトのタイムゾーン、キーボードレイアウト、フォントを設定します。
⚠️ **必要に応じて変更します**

```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="America/Sao_Paulo"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
EOF
```

## 15. /etc/locale.conf をカスタマイズします。言語を設定します。必要に応じて変更します。
⚠️ **重要:**
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
LANGUAGE=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
EOF
```

## 16. 再構成
```
sudo xbps-reconfigure -fa
```

## 17. .xinitrc を作成します (startx 用)
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
xsetroot -cursor_name left_ptr &
exec startxfce4
EOF
```

## 18. 必須サービス (runit) をアクティブ化します。
```
sudo ln -sf /etc/sv/dbus /var/service/
sudo ln -sf /etc/sv/elogind /var/service/
sudo ln -sf /etc/sv/polkitd /var/service/
sudo ln -sf /etc/sv/NetworkManager /var/service/
```

## 19. 選択したディスプレイマネージャー (runit) を有効にします。
⚠️ 注意: **いずれか 1 つだけを有効にしてください**。

**LXDM**
```
sudo ln -sf /etc/sv/lxdm /var/service/
```

**SDDM**
```
sudo ln -sf /etc/sv/sddm /var/service/
```

## 仕上げ
- LXDM がアクティブな場合: GUI を直接起動します。
- クラシックモードが必要な場合: startx
