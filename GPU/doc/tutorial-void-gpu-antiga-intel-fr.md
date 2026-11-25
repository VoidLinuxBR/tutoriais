# Tutoriel complet : GPUs Intel sur Linux (Void Linux, XFCE et matériel ancien)

Ce guide explique comment identifier, configurer et stabiliser les GPU Intel sous Linux, en particulier le matériel ancien comme GMA 3100, GMA X3100, HD 2000/3000, etc.

> **Attention :** les anciennes GPU Intel se divisent en deux groupes, et la configuration change :
>
> - **Groupe A – GMA (950, 3000, 3100, X3100, X4500) :** suivent les étapes de stabilité (DRI2, compositeur et accélération de Firefox désactivés). Ils n'ont pas de décodage vidéo matériel utile.
> - **Groupe B – Intel HD (1ère génération Core/Ironlake, HD 2000/3000/Sandy Bridge, HD 2500/4000/Ivy Bridge, Haswell) :** en général, ils n'ont pas besoin des ajustements de stabilité et **peuvent et doivent utiliser l'accélération vidéo (VA-API)** avec le pilote `i965` (voir section 7).
>
> Utilisez la section 1 pour découvrir dans quel groupe se trouve votre GPU.

---

# 1. Identifier votre GPU Intel

Exécuter :

```bash
lspci -k | grep -A3 VGA
```

ou :

```bash
inxi -G
```

Exemple de sortie :

```
Device: Intel 82G33/G31 Express Integrated Graphics
Driver: i915
Display: X.Org modesetting
OpenGL: 2.1 Mesa i915
```

---

# 2. Pilotes Intel sous Linux

Toutes les cartes graphiques Intel utilisent :

```
kernel driver: i915
```

Vous n'avez PAS besoin d'installer de pilote propriétaire. Chez Intel, **tous les pilotes sont libres**.

## 2.1 i915, i965 et iHD : ne pas confondre

Ce sont des couches différentes, et il est courant de penser qu'elles sont des alternatives l'une à l'autre :

| Nom | Qu'est-ce que c'est | D'où ça vient |
|------|---------|----------|
| `i915` | Pilote du **noyau** (vidéo/modesetting) | Déjà inclus dans le noyau, n'installe rien |
| `i965` | Pilote **VA-API** (décodage vidéo) pour GPU jusqu'à ~Coffee Lake | Paquet `libva-intel-driver` |
| `iHD` | Pilote **VA-API** pour Broadwell (5e génération) et versions plus récentes | Paquet `intel-media-driver` |
| `crocus` / `iris` | Pilote **OpenGL** de Mesa (crocus : Gen4 à Haswell ; iris : Broadwell+) | Paquet `mesa-dri` |

C'est-à-dire : ceux qui ont un HD 2000/3000 utilisent `i915` (kernel) **et** `i965` (VA-API) en même temps.

## 2.2 Pilotes Xorg

Il existe deux pilotes Xorg possibles :

- modesetting → recommandé
- xf86-video-intel → obsolète dans la plupart des cas

Vérifiez lequel est en cours d'utilisation :

```bash
inxi -G
```

Si montrer :

```
driver: modesetting
```

C'est correct.

---

# 3. Pour les anciennes GPU Intel (GMA 950, 3000, 3100, X3100, X4500)

Ces GPU ont des limites :

- OpenGL 2.1 maximum
- DRI3 instable
- le compositeur provoque un blocage
- l'accélération moderne ne fonctionne pas correctement

---

# 4. Configuration stable recommandée

Crée :

```bash
sudo mkdir -p /etc/X11/xorg.conf.d
sudo nano /etc/X11/xorg.conf.d/20-intel.conf
```

Contenu :

```conf
Section "Device"
    Identifier "Intel Graphics"
    Driver "modesetting"
    Option "DRI" "2"
EndSection
```

---

# 5. Désactiver le compositeur XFCE (OBLIGATOIRE sur les GPU anciens)

Exécuter :

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

Vérifiez :

```bash
xfconf-query -c xfwm4 -p /general/use_compositing
```

Résultat attendu :

```
false
```

---

# 6. Configuration de Firefox pour les anciennes GPU Intel (seulement Groupe A – GMA)

> **Uniquement pour GMA 950/3000/3100/X3100/X4500.** Si votre GPU est Intel HD (HD 2000/3000 ou plus récent), **ne pas appliquer cette étape** : elle désactive l'accélération vidéo. Allez directement à la section 7.

Crée :

```bash
nano ~/.mozilla/firefox/*.default-release/user.js
```

Contenu :

```js
user_pref("layers.acceleration.disabled", true);
user_pref("gfx.webrender.force-disabled", true);
user_pref("gfx.xrender.enabled", true);
user_pref("media.ffmpeg.vaapi.enabled", false);
```

Cela évite les blocages.

---

# 7. Accélération vidéo (VA-API) pour Intel HD (Groupe B)

Avec VA-API, le GPU décode la vidéo et le CPU reste libre. Cela fait une grande différence sur les ordinateurs portables avec Core i3/i5 de 2ème/3ème génération regardant YouTube.

## 7.1 Ce que chaque GPU décode en matériel

| GPU | Codecs avec accélération | Pilote VA-API |
|-----|----------------------|---------------|
| GMA 950/3000/3100/X3100 | aucun | — |
| GMA X4500 | MPEG-2 | `i965` |
| HD Graphics (Ironlake, 1re génération Core) | MPEG-2, H.264, VC-1 | `i965` |
| HD 2000/3000 (Sandy Bridge) | MPEG-2, H.264, VC-1 | `i965` |
| HD 2500/4000 (Ivy Bridge) | MPEG-2, H.264, VC-1, JPEG | `i965` |
| HD 4200–5200 (Haswell) | MPEG-2, H.264, VC-1, JPEG | `i965` |
| Broadwell et plus récents | + VP8/HEVC/VP9/AV1 selon la génération | `iHD` |

**Aucune GPU jusqu'à Haswell ne décode VP9 ou AV1.** YouTube utilise VP9/AV1 par défaut, donc dans ces cas, installez dans le navigateur une extension qui force H.264 (ex. : *h264ify* ou *enhanced-h264ify*). Sans elle, la vidéo est à nouveau décodée par le CPU.

## 7.2 Installer les paquets

```bash
sudo xbps-install -S mesa-dri libva-intel-driver libva-utils intel-gpu-tools
```

Assurez-vous que l'utilisateur est dans le groupe `video` (quittez et reconnectez-vous ensuite) :

```bash
sudo usermod -aG video $USER
```

## 7.3 Forcer le pilote VA-API correct

Ajoutez dans `~/.profile` (ou dans `/etc/environment`, sans le `export`) :

```bash
export LIBVA_DRIVER_NAME=i965
```

## 7.4 Tester

```bash
vainfo
```

Résultat attendu (résumé) :

```
vainfo: Driver version: Intel i965 driver for Intel(R) Sandybridge Mobile
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointVLD
```

`VAEntrypointVLD` = décodage matériel disponible.

## 7,5 Firefox

Au `about:config` :

```
media.ffmpeg.vaapi.enabled = true
```

Dans les versions les plus récentes de Firefox, si le décodage ne se lance pas automatiquement, également :

```
media.hardware-video-decoding.force-enabled = true
```

Si vous avez appliqué le `user.js` de la section 6, supprimez-le. Redémarrez Firefox et vérifiez dans `about:support`, section **Média**, si le H264 apparaît avec décodage matériel.

## 7.6 Confirmer que le GPU fonctionne

Ouvrez une vidéo sur YouTube (avec l'extension H.264 activée) et lancez :

```bash
sudo intel_gpu_top
```

Si la ligne **Vidéo** est occupée pendant que la vidéo joue, le décodage se fait sur le GPU. L'utilisation du CPU diminue considérablement.

---

# 8. Vérifier le support OpenGL

Exécuter :

```bash
glxinfo | grep "OpenGL version"
```

Exemple :

```
OpenGL version string: 2.1 Mesa 25.3.3
```

C'est le maximum supporté par la GMA 3100.

---

# 9. Vérifier l'accélération active

Exécuter :

```bash
glxinfo | grep "direct rendering"
```

Résultat correct :

```
direct rendering: Yes
```

---

# 10. Vérifier le pilote chargé

Exécuter :

```bash
lsmod | grep i915
```

Résultat :

```
i915
```

---

# 11. Vérifier le statut de Xorg

Exécuter :

```bash
grep -i driver /var/log/Xorg.0.log
```

Résultat correct :

```
modesetting
```

---

# 12. N'utilisez pas xf86-video-intel sur du matériel très ancien

Supprimez si installé :

```bash
sudo xbps-remove xf86-video-intel
```

Le pilote modesetting est plus stable.

---

# 13. Vérifier le DRI

Exécuter :

```bash
glxinfo | grep DRI
```

Pour les anciennes GPU, utilisez :

```
DRI2
```

---

# 14. Vérifier la résolution

Exécuter :

```bash
xrandr
```

---

# 15. Problèmes courants et solution

## redémarrage lors de l'entrée dans XFCE

Désactiver le compositeur :

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## Firefox se bloque

Désactivez l'accélération via user.js (étape 6). **Seulement GMA** — sur Intel HD, il est préférable de conserver l'accélération.

---

## vidéo YouTube utilisant beaucoup de CPU (Intel HD)

- Vérifiez si l'extension qui force H.264 est activée (le GPU ne décode pas VP9/AV1).
- Exécutez `vainfo` et voyez si `VAProfileH264...: VAEntrypointVLD` apparaît.
- Confirmez `LIBVA_DRIVER_NAME=i965` avec `echo $LIBVA_DRIVER_NAME`.
- Confirmez que le `user.js` de la section 6 n'est pas appliqué (il désactive le VA-API).

---

## écran noir

Supprimez xf86-video-intel :

```bash
sudo xbps-remove xf86-video-intel
```

---

# 16. Vérification complète du système graphique

Exécuter :

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
vainfo          # somente Intel HD (Grupo B)
```

---

# 17. GPU Intel et support OpenGL

| GPU | OpenGL |
|----|--------|
| GMA 950 | 2.1 |
| GMA 3100 | 2.1 |
| X3100 | 2.1 |
| X4500 | 2.1 |
| HD 2000 | 3,3 |
| HD 3000 | 3.3 |
| HD 4000 | 4,0 |
| UHD 620 | 4,6 |

---

# 18. Conclusion

**Groupe A – GMA (950, 3000, 3100, X3100, X4500):**

- pilote noyau : i915
- pilote Xorg : modesetting
- DRI2
- compositeur désactivé
- accélération Firefox désactivée

**Groupe B – Intel HD (Ironlake, HD 2000/3000, HD 2500/4000, Haswell):**

- pilote noyau : i915
- pilote Xorg : modesetting
- pilote VA-API : i965 (`libva-intel-driver`) + `LIBVA_DRIVER_NAME=i965`
- VA-API activé dans Firefox
- extension qui force H.264 sur YouTube

Le système restera stable et fonctionnel.
