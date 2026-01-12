# 🐧 Void Linux + KDE Plasma + PipeWire — Tutoriel

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

## 2. Installez le Plasma complet (méta-paquet)
```
sudo xbps-install -y plasma noto-fonts-emoji
```

## 3. Installez SDDM (gestionnaire d'affichage officiel de KDE)
```
sudo xbps-install -y sddm
```

## 4. Installez l'audio avec PipeWire (son complet)

### PipeWire + WirePlumber + ALSA + Compatibilité impulsion
```
sudo xbps-install -y \
  pipewire \
  wireplumber \
  alsa-pipewire \
  libjack-pipewire \
  alsa-utils \
  pavucontrol
```

## 5. Pilotes vidéo (choisissez-en un seul)

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

### Nvidia (pilote ouvert)
```
sudo xbps-install -y mesa-nouveau-dri
```

### Nvidia (propriétaire)
```
sudo xbps-install -y void-repo-nonfree
sudo xbps-install -y nvidia
```

## 6. Activer les services obligatoires (runit)
```
sudo ln -s /etc/sv/dbus /var/service/
sudo ln -s /etc/sv/elogind /var/service/
sudo ln -s /etc/sv/polkitd /var/service/
sudo ln -s /etc/sv/NetworkManager /var/service/
sudo ln -s /etc/sv/sddm /var/service/
```

## 7. (Facultatif) Créez .xinitrc pour startx
```
cat <<EOF > ~/.xinitrc
#!/bin/sh
setxkbmap -layout br -variant abnt2 &
exec startplasma-x11
EOF
```

## Finition
- En utilisant SDDM → le système démarre directement dans KDE Plasma.
- Sans SDDM → utilisez `startx` (si `.xinitrc` existe).

