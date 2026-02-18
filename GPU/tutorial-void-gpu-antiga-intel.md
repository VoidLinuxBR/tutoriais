# Tutorial completo: GPUs Intel no Linux (Void Linux, XFCE e hardware antigo)

Este guia explica como identificar, configurar e estabilizar GPUs Intel no Linux, especialmente hardware antigo como GMA 3100, GMA X3100, HD 2000/3000, etc.

---

# 1. Identificar sua GPU Intel

Execute:

```bash
lspci -k | grep -A3 VGA
```

ou:

```bash
inxi -G
```

Exemplo de saída:

```
Device: Intel 82G33/G31 Express Integrated Graphics
Driver: i915
Display: X.Org modesetting
OpenGL: 2.1 Mesa i915
```

---

# 2. Drivers Intel no Linux

Todas as GPUs Intel usam:

```
kernel driver: i915
```

Você NÃO precisa instalar driver proprietário.

Existem dois drivers Xorg possíveis:

- modesetting → recomendado
- xf86-video-intel → obsoleto na maioria dos casos

Verifique qual está em uso:

```bash
inxi -G
```

Se mostrar:

```
driver: modesetting
```

Está correto.

---

# 3. Para GPUs Intel antigas (GMA 950, 3000, 3100, X3100, X4500)

Essas GPUs têm limitações:

- OpenGL 2.1 máximo
- DRI3 instável
- compositor causa travamento
- aceleração moderna não funciona corretamente

---

# 4. Configuração estável recomendada

Crie:

```bash
sudo mkdir -p /etc/X11/xorg.conf.d
sudo nano /etc/X11/xorg.conf.d/20-intel.conf
```

Conteúdo:

```conf
Section "Device"
    Identifier "Intel Graphics"
    Driver "modesetting"
    Option "DRI" "2"
EndSection
```

---

# 5. Desativar compositor XFCE (OBRIGATÓRIO em GPUs antigas)

Execute:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

Verifique:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing
```

Resultado esperado:

```
false
```

---

# 6. Configuração do Firefox para GPUs Intel antigas

Crie:

```bash
nano ~/.mozilla/firefox/*.default-release/user.js
```

Conteúdo:

```js
user_pref("layers.acceleration.disabled", true);
user_pref("gfx.webrender.force-disabled", true);
user_pref("gfx.xrender.enabled", true);
user_pref("media.ffmpeg.vaapi.enabled", false);
```

Isso evita travamentos.

---

# 7. Verificar suporte OpenGL

Execute:

```bash
glxinfo | grep "OpenGL version"
```

Exemplo:

```
OpenGL version string: 2.1 Mesa 25.3.3
```

Isso é o máximo suportado pela GMA 3100.

---

# 8. Verificar aceleração ativa

Execute:

```bash
glxinfo | grep "direct rendering"
```

Resultado correto:

```
direct rendering: Yes
```

---

# 9. Verificar driver carregado

Execute:

```bash
lsmod | grep i915
```

Resultado:

```
i915
```

---

# 10. Verificar status do Xorg

Execute:

```bash
grep -i driver /var/log/Xorg.0.log
```

Resultado correto:

```
modesetting
```

---

# 11. Não use xf86-video-intel em hardware muito antigo

Remova se estiver instalado:

```bash
sudo xbps-remove xf86-video-intel
```

O driver modesetting é mais estável.

---

# 12. Verificar DRI

Execute:

```bash
glxinfo | grep DRI
```

Para GPUs antigas, use:

```
DRI2
```

---

# 13. Verificar resolução

Execute:

```bash
xrandr
```

---

# 14. Problemas comuns e solução

## reboot ao entrar no XFCE

Desative compositor:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## Firefox trava

Desative aceleração via user.js (passo 6)

---

## tela preta

Remova xf86-video-intel:

```bash
sudo xbps-remove xf86-video-intel
```

---

# 15. Verificação completa do sistema gráfico

Execute:

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
```

---

# 16. GPUs Intel e suporte OpenGL

| GPU | OpenGL |
|----|--------|
| GMA 950 | 2.1 |
| GMA 3100 | 2.1 |
| X3100 | 2.1 |
| X4500 | 2.1 |
| HD 2000 | 3.3 |
| HD 3000 | 3.3 |
| HD 4000 | 4.0 |
| UHD 620 | 4.6 |

---

# 17. Conclusão

Para GPUs Intel antigas, use:

- driver kernel: i915
- driver Xorg: modesetting
- DRI2
- compositor desativado
- aceleração Firefox desativada

Sistema ficará estável e funcional.
