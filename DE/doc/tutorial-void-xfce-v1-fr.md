# 🐧 Void Linux + XFCE4 — Tutoriel


> ⚠️ **IMPORTANT — À LIRE AVANT DE COMMENCER**
>
> Ce tutoriel **ne doit PAS être exécuté en tant que `root`**, sauf lorsque **explicitement indiqué**.
>
> Toutes les commandes ont été conçues pour être exécutées par **un utilisateur commun**, en utilisant `sudo` si nécessaire.
>
> Exécutez l'intégralité du tutoriel connecté en tant que `root` :
> - brise la logique des autorisations
> - invalide les étapes comme la configuration `sudo`
> - peut générer des erreurs silencieuses ou un comportement inattendu
>
> 👉 **Recommandation**
> Si vous venez d'installer le système et que vous êtes connecté en tant que « root » :
>
> 1. Créer un utilisateur commun
> 2. Connectez-vous avec cet utilisateur
> 3. Suivez le tutoriel normalement
>
> Règle classique pour les systèmes Unix/Linux :
>
> **`root` est une exception. L'utilisateur commun est la règle.**

---

## 0. Configurez sudo - (wheel group) - évitez de demander le mot de passe root
```
sudo usermod -aG wheel "$USER"

sudo tee -a /etc/sudoers.d/g_wheel >/dev/null << EOF
%wheel ALL=(ALL:ALL) NOPASSWD: ALL
EOF

#Permissões obrigatórias
sudo chmod 440 /etc/sudoers.d/g_wheel
```

## 1. Mettre à jour le système
```
sudo xbps-install -Syu
```

```
#limpar cache de pacotes
sudo rm -fv /var/cache/xbps/*.xbps /var/cache/xbps/*.sig*
```

## 2. Installez Xorg + Xinit + Xterm
```
sudo xbps-install -y xorg xinit xterm
```

## 3. Installez XFCE4 complet
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

## 4. Installez un gestionnaire d'affichage (LXDM ou SDDM)

Choisissez **un seul** gestionnaire d'affichage à installer et à activer.

### 4.1 LXDM — léger et traditionnel (recommandé pour les machines simples)

```
sudo xbps-install -y lxdm
```

### 4.2 SDDM — moderne et complet

```
sudo xbps-install -y sddm voidbr-sddm-themes
```

>Remarque :
> LXDM est simple, rapide et consomme peu de ressources.
> SDDM offre un meilleur support visuel, des thèmes et une meilleure intégration avec les environnements Qt.

---

## 5. Pilotes d'affichage

### Intel
```
sudo xbps-install -y mesa-dri linux-firmware-intel
```

### nouveau AMD (amdgpu)
```
sudo xbps-install -y mesa-dri xf86-video-amdgpu
```

### vieille DMLA
```
sudo xbps-install -y mesa-dri xf86-video-ati
```

### Propriétaire Nvidia
```
sudo xbps-install -y mesa-nouveau-dri
```

## 6. Désactivez XFCE-Polkit (buggé – ne pas utiliser)
⚠️ **IMPORTANT :**
```
mkdir -p ~/.config/autostart
cp /etc/xdg/autostart/xfce-polkit.desktop ~/.config/autostart/
sed -i 's/^Hidden=.*/Hidden=true/' ~/.config/autostart/xfce-polkit.desktop || echo 'Hidden=true' >> ~/.config/autostart/xfce-polkit.desktop
```

## 7. Installez PipeWire (Modern Void Sound)
```
sudo xbps-install -y pipewire wireplumber alsa-pipewire pulseaudio-utils pavucontrol libspa-bluetooth libjack-pipewire alsa-plugins-pulseaudio xfce4-pulseaudio-plugin)
```



## 8. Intégrer ALSA → PipeWire
```
sudo mkdir -p /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/50-pipewire.conf /etc/alsa/conf.d
sudo ln -sf /usr/share/alsa/alsa.conf.d/99-pipewire-default.conf /etc/alsa/conf.d
```

## 9. Activer le serveur pipewire-pulse (compatibilité PulseAudio)
```
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo ln -sf /usr/share/examples/pipewire/20-pipewire-pulse.conf /etc/pipewire/pipewire.conf.d/
```

## 10. Activer le démarrage automatique de PipeWire dans la session XFCE
```
mkdir -p ~/.config/autostart
ln -sf /usr/share/applications/pipewire.desktop ~/.config/autostart/
ln -sf /usr/share/applications/pipewire-pulse.desktop ~/.config/autostart/
ln -sf /usr/share/applications/wireplumber.desktop ~/.config/autostart/
```

## 11. Plugins essentiels XFCE (icône sonore)
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
> Si l'icône son n'apparaît toujours pas lors de la première connexion
ouvrez le terminal et exécutez ce script, puis déconnectez-vous et reconnectez-vous

## 12. configurer le fuseau horaire - définit le fuseau horaire
```
sudo ln -sfv /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

## 13. configurer les paramètres régionaux
```
sudo sed -i -e 's/^#\(en_US.UTF-8 UTF-8\)/\1/' -e 's/^#\(pt_BR.UTF-8 UTF-8\)/\1/' /etc/default/libc-locales
```

## 14. Personnalisez /etc/rc.conf. Définit le fuseau horaire, la disposition du clavier et la police par défaut de la console.
⚠️ **Modifier si nécessaire**

```
sudo tee -a /etc/rc.conf >/dev/null << EOF
TIMEZONE="America/Sao_Paulo"
KEYMAP="br-abnt2"
FONT=Lat2-Terminus16
EOF
```

## 15. Personnalisez /etc/locale.conf. Définit la langue. Changez si nécessaire.
⚠️ **IMPORTANT :**
```
sudo tee /etc/locale.conf >/dev/null << EOF
LANG=pt_BR.UTF-8
LANGUAGE=pt_BR.UTF-8
LC_COLLATE=pt_BR.UTF-8
EOF
```

## 16. Reconfigurer
```
sudo xbps-reconfigure -fa
```

## 17. Créez .xinitrc (pour startx)
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
xsetroot -cursor_name left_ptr &
exec startxfce4
EOF
```

## 18. Activer les services obligatoires (runit)
```
sudo ln -sf /etc/sv/dbus /var/service/
sudo ln -sf /etc/sv/elogind /var/service/
sudo ln -sf /etc/sv/polkitd /var/service/
sudo ln -sf /etc/sv/NetworkManager /var/service/
```

## 19. Activez le gestionnaire d'affichage choisi (runit)
⚠️ Attention : **n'activez qu'un seul d'entre eux**.

**LXDM**
```
sudo ln -sf /etc/sv/lxdm /var/service/
```

**SDDM**
```
sudo ln -sf /etc/sv/sddm /var/service/
```

## Finition
- Si LXDM est actif : démarrez directement dans l’interface graphique.
- Si vous voulez le mode classique : startx
