# 🐧 Void Linux + XFCE4 — 튜토리얼


> ⚠️ **중요 — 시작하기 전에 읽어보세요**
>
> 이 튜토리얼은 **명시적으로 표시된** 경우를 제외하고 **`루트`로 실행해서는 안 됩니다**.
>
> 모든 명령은 **일반 사용자**가 필요할 때 `sudo`를 사용하여 실행하도록 설계되었습니다.
>
> `root`로 로그인하여 전체 튜토리얼을 실행합니다.
> - 권한 논리를 깨뜨림
> - `sudo` 구성과 같은 단계를 무효화합니다.
> - 소리 없는 오류나 예상치 못한 동작이 발생할 수 있습니다.
>
> 👉 **추천**
> 방금 시스템을 설치하고 `root`로 로그인한 경우:
>
> 1. 일반 사용자 생성
> 2. 이 사용자로 로그인
> 3. 정상적으로 튜토리얼을 따르세요.
>
> Unix/Linux 시스템의 기본 규칙:
>
> **`root`는 예외입니다. 일반 사용자가 원칙입니다.**

---

## 0. sudo 구성 - (휠 그룹) - 루트 비밀번호 묻지 않기
```
sudo usermod -aG wheel "$USER"

sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
EOF

#Permissões obrigatórias
sudo chmod 440 /etc/sudoers.d/g_wheel
```

## 1. 시스템 업데이트
```
sudo xbps-install -Syu
```

```
#limpar cache de pacotes
sudo rm -fv /var/cache/xbps/*.xbps /var/cache/xbps/*.sig*
```

## 2. Xorg + Xinit + Xterm 설치
```
sudo xbps-install -y xorg xinit xterm
```

## 3. 전체 XFCE4 설치
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

## 4. 디스플레이 관리자(LXDM 또는 SDDM) 설치

**단 하나의** 디스플레이 관리자를 선택하여 설치하고 활성화하세요.

### 4.1 LXDM — 가볍고 전통적(간단한 기계에 권장)

```
sudo xbps-install -y lxdm
```

### 4.2 SDDM — 현대적이고 완벽함

```
sudo xbps-install -y sddm voidbr-sddm-themes
```

>참고:
> LXDM은 간단하고 빠르며 리소스를 거의 소모하지 않습니다.
> SDDM은 더 나은 시각적 지원, 테마 및 Qt 환경과의 통합을 제공합니다.

---

## 5. 디스플레이 드라이버

### 인텔
```
sudo xbps-install -y mesa-dri linux-firmware-intel
```

### 새로운 AMD(amdgpu)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

### 오래된 AMD
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### 엔비디아 독점
```
sudo xbps-install -y mesa-nouveau-dri
```

## 6. XFCE-Polkit 비활성화(버그 발생 - 사용하지 않음)
⚠️ **중요:**
```
mkdir -p ~/.config/autostart
cp /etc/xdg/autostart/xfce-polkit.desktop ~/.config/autostart/
sed -i 's/^Hidden=.*/Hidden=true/' ~/.config/autostart/xfce-polkit.desktop || echo 'Hidden=true' >> ~/.config/autostart/xfce-polkit.desktop
```

## 7. PipeWire(Modern Void Sound) 설치
```
sudo xbps-install -y pipewire wireplumber alsa-pipewire pulseaudio-utils pavucontrol libspa-bluetooth libjack-pipewire alsa-plugins-pulseaudio xfce4-pulseaudio-plugin)
```



## 8. ALSA → PipeWire 통합
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 9. 파이프와이어 펄스 서버 활성화(PulseAudio 호환)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 10. XFCE 세션에서 PipeWire 자동 시작 활성화
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 11. XFCE 필수 플러그인(사운드 아이콘)
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
> 최초 로그인 시 소리 아이콘이 여전히 나타나지 않는 경우
터미널을 열고 이 스크립트를 실행한 다음 로그아웃하고 다시 로그인하세요.

## 12. 시간대 구성 - 시간대를 설정합니다.
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 13. 로케일 구성
```
sudo sed -i -e 's/^#\(en_US.UTF-8 UTF-8\)/\1/' -e 's/^#\(pt_BR.UTF-8 UTF-8\)/\1/' /etc/default/libc-locales
```

## 14. /etc/rc.conf를 사용자 정의합니다. 콘솔의 기본 시간대, 키보드 레이아웃 및 글꼴을 설정합니다.
⚠️ **필요에 따라 변경**

```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="America/Sao_Paulo"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
EOF
```

## 15. /etc/locale.conf를 사용자 정의합니다. 언어를 설정합니다. 필요에 따라 변경하세요.
⚠️ **중요:**
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
LANGUAGE=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
EOF
```

## 16. 재구성
```
sudo xbps-reconfigure -fa
```

## 17. .xinitrc 생성(startx용)
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
xsetroot -cursor_name left_ptr &
exec startxfce4
EOF
```

## 18. 필수 서비스 활성화(runit)
```
sudo ln -sf /etc/sv/dbus /var/service/
sudo ln -sf /etc/sv/elogind /var/service/
sudo ln -sf /etc/sv/polkitd /var/service/
sudo ln -sf /etc/sv/NetworkManager /var/service/
```

## 19. 선택한 디스플레이 관리자(runit)를 활성화합니다.
⚠️ 주의: **그 중 하나만 활성화하세요**.

**LXDM**
```
sudo ln -sf /etc/sv/lxdm /var/service/
```

**SDDM**
```
sudo ln -sf /etc/sv/sddm /var/service/
```

## 마무리 손질
- LXDM이 활성화된 경우: GUI로 직접 부팅합니다.
- 클래식 모드를 원하는 경우: startx
