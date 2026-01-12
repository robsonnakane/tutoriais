# 🐧 Void Linux + KDE Plasma + PipeWire — チュートリアル

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

## 2. 完全な Plasma (メタパッケージ) をインストールします。
```
sudo xbps-install -y plasma noto-fonts-emoji
```

## 3. SDDM (公式 KDE ディスプレイマネージャー) をインストールします。
```
sudo xbps-install -y sddm
```

## 4. PipeWire を使用してオーディオをインストールする (フルサウンド)

### PipeWire + WirePlumber + ALSA + パルス互換
```
sudo xbps-install -y \
  pipewire \
  wireplumber \
  alsa-pipewire \
  libjack-pipewire \
  alsa-utils \
  pavucontrol
```

## 5. ビデオドライバー (1 つだけ選択)

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

### Nvidia (オープンドライバー)
```
sudo xbps-install -y mesa-nouveau-dri
```

### エヌビディア（オーナー）
```
sudo xbps-install -y void-repo-nonfree
sudo xbps-install -y nvidia
```

## 6. 必須サービス (runit) をアクティブ化します。
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/sddm /var/service/
```

## 7. (オプション) startx の .xinitrc を作成します。
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
exec startplasma-x11
EOF
```

## 仕上げ
- SDDM を使用すると、システムが KDE Plasma で直接起動します。
- SDDM を使用しない場合 → `startx` を使用します (`.xinitrc` が存在する場合)。

