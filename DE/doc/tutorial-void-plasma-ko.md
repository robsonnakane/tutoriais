# 🐧 Void Linux + KDE 플라즈마 + PipeWire — 튜토리얼

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

## 2. 전체 플라즈마(메타 패키지) 설치
```
sudo xbps-install -y plasma noto-fonts-emoji
```

## 3. SDDM(공식 KDE 디스플레이 관리자)을 설치합니다.
```
sudo xbps-install -y sddm
```

## 4. PipeWire로 오디오 설치(풀 사운드)

### PipeWire + WirePlumber + ALSA + 펄스 호환
```
sudo xbps-install -y \
  pipewire \
  wireplumber \
  alsa-pipewire \
  libjack-pipewire \
  alsa-utils \
  pavucontrol
```

## 5. 비디오 드라이버(하나만 선택)

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

### 엔비디아(오픈 드라이버)
```
sudo xbps-install -y mesa-nouveau-dri
```

### 엔비디아 (소유자)
```
sudo xbps-install -y void-repo-nonfree
sudo xbps-install -y nvidia
```

## 6. 필수 서비스 활성화(runit)
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/sddm /var/service/
```

## 7. (선택 사항) startx에 대한 .xinitrc 생성
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
exec startplasma-x11
EOF
```

## 마무리 손질
- SDDM 사용 → 시스템이 KDE Plasma로 직접 부팅됩니다.
- SDDM이 없으면 → `startx`를 사용하십시오(`.xinitrc`가 존재하는 경우).

