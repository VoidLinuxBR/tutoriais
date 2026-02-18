# Tutoriel complet : GPU Intel sous Linux (Void Linux, XFCE et ancien matériel)

Ce guide explique comment identifier, configurer et stabiliser les GPU Intel sous Linux, en particulier les matériels plus anciens comme GMA 3100, GMA X3100, HD 2000/3000, etc.

---

# 1. Identifiez votre GPU Intel

Exécuter:

```bash
lspci -k | grep -A3 VGA
```

ou:

```bash
inxi -G
```

Exemple de sortie :

```
Device: Intel 82G33/G31 Express Integrated Graphics
Driver: i915
Display: X.Org modesetting
OpenGL: 2.1 Mesa i915
```

---

# 2. Pilotes Intel pas Linux

Tous les GPU Intel utilisent :

```
kernel driver: i915
```

Vous n'avez PAS besoin d'installer un pilote propriétaire.

Il existe deux pilotes Xorg possibles :

- réglage du mode → recommandé
- xf86-video-intel → obsolète dans la plupart des cas

Vérifiez lequel est utilisé :

```bash
inxi -G
```

S'il affiche :

```
driver: modesetting
```

C'est exact.

---

# 3. Pour les anciens GPU Intel (GMA 950, 3000, 3100, X3100, X4500)

Ces GPU ont des limites :

- OpenGL 2.1 maximum
- DRI3 instable
- le compositeur provoque un crash
- l'accélération moderne ne fonctionne pas correctement

---

# 4. Configuration stable recommandée

Créer:

```bash
sudo mkdir -p /etc/X11/xorg.conf.d
sudo nano /etc/X11/xorg.conf.d/20-intel.conf
```

Contenu:

```conf
Section "Device"
    Identifier "Intel Graphics"
    Driver "modesetting"
    Option "DRI" "2"
EndSection
```

---

# 5. Désactivez le compositeur XFCE (REQUIS sur les anciens GPU)

Exécuter:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

Vérifier:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing
```

Résultat attendu :

```
false
```

---

# 6. Configuration de Firefox pour les anciens GPU Intel

Créer:

```bash
nano ~/.mozilla/firefox/*.default-release/user.js
```

Contenu:

```js
user_pref("layers.acceleration.disabled", true);
user_pref("gfx.webrender.force-disabled", true);
user_pref("gfx.xrender.enabled", true);
user_pref("media.ffmpeg.vaapi.enabled", false);
```

Cela évite les accidents.

---

# 7. Vérifiez la prise en charge d'OpenGL

Exécuter:

```bash
glxinfo | grep "OpenGL version"
```

Exemple:

```
OpenGL version string: 2.1 Mesa 25.3.3
```

C'est le maximum supporté par le GMA 3100.

---

# 8. Vérifiez l'accélération active

Exécuter:

```bash
glxinfo | grep "direct rendering"
```

Résultat correct :

```
direct rendering: Yes
```

---

# 9. Vérifiez le pilote chargé

Exécuter:

```bash
lsmod | grep i915
```

Résultat:

```
i915
```

---

# 10. Vérifiez le statut de Xorg

Exécuter:

```bash
grep -i driver /var/log/Xorg.0.log
```

Résultat correct :

```
modesetting
```

---

# 11. N'utilisez pas xf86-video-intel sur du matériel très ancien

Supprimer si installé :

```bash
sudo xbps-remove xf86-video-intel
```

Le pilote de réglage de mode est plus stable.

---

# 12. Vérifiez le DRI

Exécuter:

```bash
glxinfo | grep DRI
```

Pour les GPU plus anciens, utilisez :

```
DRI2
```

---

# 13. Vérifiez la résolution

Exécuter:

```bash
xrandr
```

---

# 14. Problèmes courants et solutions

## redémarrer en entrant dans XFCE

Désactiver le compositeur :

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## Firefox plante

Désactivez l'accélération via user.js (étape 6)

---

## écran noir

Supprimez xf86-video-intel :

```bash
sudo xbps-remove xf86-video-intel
```

---

# 15. Vérification complète du système graphique

Exécuter:

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
```

---

# 16. GPU Intel et prise en charge d'OpenGL

| GPU | OpenGL |
|----|--------|
| GMA950 | 2.1 |
| GMA3100 | 2.1 |
| X3100 | 2.1 |
| X4500 | 2.1 |
| HD2000 | 3.3 |
| HD3000 | 3.3 |
| HD4000 | 4.0 |
| UHD 620 | 4.6 |

---

# 17. Conclusion

Pour les anciens GPU Intel, utilisez :

- noyau du pilote : i915
- pilote Xorg : paramètre de mode
- DRI2
- compositeur désactivé
- Accélération Firefox désactivée

Le système sera stable et fonctionnel.
