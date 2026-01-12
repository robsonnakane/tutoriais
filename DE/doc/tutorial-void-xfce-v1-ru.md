# 🐧 Void Linux + XFCE4 — Учебное пособие


> ⚠️ **ВАЖНО — ПРОЧТИТЕ ПЕРЕД НАЧАЛОМ**
>
> Это руководство **НЕ следует запускать от имени пользователя root**, за исключением случаев, когда **явно указано**.
>
> Все команды были разработаны для выполнения **обычным пользователем** с использованием `sudo` при необходимости.
>
> Запустите все руководство, войдя в систему как root:
> - нарушает логику разрешений
> - делает недействительными такие шаги, как конфигурация `sudo`
> - может генерировать тихие ошибки или неожиданное поведение
>
> 👉 **Рекомендация**
> Если вы только что установили систему и вошли в систему как root:
>
> 1. Создать обычного пользователя
> 2. Войти под этим пользователем
> 3. Следуйте инструкциям в обычном режиме.
>
> Классическое правило для систем Unix/Linux:
>
> **`root` — исключение. Обычный пользователь – это правило.**

---

## 0. Настройте sudo — (группа колес) — не запрашивайте пароль root.
```
sudo usermod -aG wheel "$USER"

sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
EOF

#Permissões obrigatórias
sudo chmod 440 /etc/sudoers.d/g_wheel
```

## 1. Обновите систему
```
sudo xbps-install -Syu
```

```
#limpar cache de pacotes
sudo rm -fv /var/cache/xbps/*.xbps /var/cache/xbps/*.sig*
```

## 2. Установите Xorg + Xinit + Xterm.
```
sudo xbps-install -y xorg xinit xterm
```

## 3. Установите полную версию XFCE4.
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

## 4. Установите диспетчер дисплея (LXDM или SDDM).

Выберите **только один** диспетчер дисплея для установки и включения.

### 4.1 LXDM — легкий и традиционный (рекомендуется для простых машин)

```
sudo xbps-install -y lxdm
```

### 4.2 SDDM — современный и полный

```
sudo xbps-install -y sddm voidbr-sddm-themes
```

>Примечание:
> LXDM прост, быстр и потребляет мало ресурсов.
> SDDM предлагает улучшенную визуальную поддержку, темы и интеграцию со средами Qt.

---

## 5. Драйверы дисплея

### Интел
```
sudo xbps-install -y mesa-dri linux-firmware-intel
```

### новый AMD (амдгпу)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

### старый AMD
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### собственность NVIDIA
```
sudo xbps-install -y mesa-nouveau-dri
```

## 6. Отключите XFCE-Polkit (с ошибками — не используйте)
⚠️ **ВАЖНО:**
```
mkdir -p ~/.config/autostart
cp /etc/xdg/autostart/xfce-polkit.desktop ~/.config/autostart/
sed -i 's/^Hidden=.*/Hidden=true/' ~/.config/autostart/xfce-polkit.desktop || echo 'Hidden=true' >> ~/.config/autostart/xfce-polkit.desktop
```

## 7. Установите PipeWire (Modern Void Sound)
```
sudo xbps-install -y pipewire wireplumber alsa-pipewire pulseaudio-utils pavucontrol libspa-bluetooth libjack-pipewire alsa-plugins-pulseaudio xfce4-pulseaudio-plugin)
```



## 8. Интегрируйте ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 9. Включите сервер Pipewire-Pulse (совместим с PulseAudio).
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 10. Включите автозапуск PipeWire в сеансе XFCE.
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 11. Основные плагины XFCE (значок звука)
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
> Если значок звука по-прежнему не появляется при первом входе в систему
откройте терминал и запустите этот скрипт, затем выйдите из системы и войдите снова

## 12. настроить часовой пояс - устанавливает часовой пояс
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 13. настроить локали
```
sudo sed -i -e 's/^#\(en_US.UTF-8 UTF-8\)/\1/' -e 's/^#\(pt_BR.UTF-8 UTF-8\)/\1/' /etc/default/libc-locales
```

## 14. Настройте /etc/rc.conf. Устанавливает часовой пояс консоли по умолчанию, раскладку клавиатуры и шрифт.
⚠️ **Изменяйте по мере необходимости**

```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="America/Sao_Paulo"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
EOF
```

## 15. Настройте /etc/locale.conf. Устанавливает язык. Меняйте по мере необходимости.
⚠️ **ВАЖНО:**
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
LANGUAGE=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
EOF
```

## 16. Перенастроить
```
sudo xbps-reconfigure -fa
```

## 17. Создайте .xinitrc (для startx)
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
xsetroot -cursor_name left_ptr &
exec startxfce4
EOF
```

## 18. Активировать обязательные службы (runit)
```
sudo ln -sf /etc/sv/dbus /var/service/
sudo ln -sf /etc/sv/elogind /var/service/
sudo ln -sf /etc/sv/polkitd /var/service/
sudo ln -sf /etc/sv/NetworkManager /var/service/
```

## 19. Включите выбранный диспетчер дисплея (запустите)
⚠️ Внимание: **включите только один из них**.

**LXDM**
```
sudo ln -sf /etc/sv/lxdm /var/service/
```

**СДДМ**
```
sudo ln -sf /etc/sv/sddm /var/service/
```

## Отделка
- Если LXDM активен: загрузитесь непосредственно в графический интерфейс.
- Если вам нужен классический режим: startx
