# Tutorial completo: GPUs Intel no Linux (Void Linux, XFCE e hardware antigo)

Este guia explica como identificar, configurar e estabilizar GPUs Intel no Linux, especialmente hardware antigo como GMA 3100, GMA X3100, HD 2000/3000, etc.

> **Atenção:** as GPUs Intel antigas se dividem em dois grupos, e a configuração muda:
>
> - **Grupo A – GMA (950, 3000, 3100, X3100, X4500):** seguem os passos de estabilidade (DRI2, compositor e aceleração do Firefox desativados). Não têm decodificação de vídeo útil por hardware.
> - **Grupo B – Intel HD (1ª geração Core/Ironlake, HD 2000/3000/Sandy Bridge, HD 2500/4000/Ivy Bridge, Haswell):** em geral não precisam dos ajustes de estabilidade e **podem e devem usar aceleração de vídeo (VA-API)** com o driver `i965` (ver seção 7).
>
> Use a seção 1 para descobrir em qual grupo está a sua GPU.

---

# 1. Identificar sua GPU Intel

Executar:

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

Você NÃO precisa instalar driver proprietário. Na Intel, **todos os drivers são livres**.

## 2.1 i915, i965 e iHD: não confunda

São camadas diferentes, e é comum achar que são alternativas entre si:

| Nome | O que é | Onde vem |
|------|---------|----------|
| `i915` | Driver do **kernel** (vídeo/modesetting) | Já vem no kernel, não instala nada |
| `i965` | Driver **VA-API** (decodificação de vídeo) para GPUs até ~Coffee Lake | Pacote `libva-intel-driver` |
| `iHD` | Driver **VA-API** para Broadwell (5ª geração) e posteriores | Pacote `intel-media-driver` |
| `crocus` / `iris` | Driver **OpenGL** do Mesa (crocus: Gen4 a Haswell; iris: Broadwell+) | Pacote `mesa-dri` |

Ou seja: quem tem HD 2000/3000 usa o `i915` (kernel) **e** o `i965` (VA-API) ao mesmo tempo.

## 2.2 Drivers Xorg

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

Executar:

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

# 6. Configuração do Firefox para GPUs Intel antigas (somente Grupo A – GMA)

> **Somente para GMA 950/3000/3100/X3100/X4500.** Se a sua GPU é Intel HD (HD 2000/3000 ou mais nova), **não aplique este passo**: ele desliga a aceleração de vídeo. Vá direto para a seção 7.

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

# 7. Aceleração de vídeo (VA-API) para Intel HD (Grupo B)

Com VA-API, a GPU decodifica o vídeo e a CPU fica livre. Isso faz muita diferença em notebooks com Core i3/i5 de 2ª/3ª geração assistindo YouTube.

## 7.1 O que cada GPU decodifica em hardware

| GPU | Codecs com aceleração | Driver VA-API |
|-----|----------------------|---------------|
| GMA 950/3000/3100/X3100 | nenhum | — |
| GMA X4500 | MPEG-2 | `i965` |
| HD Graphics (Ironlake, 1ª geração Core) | MPEG-2, H.264, VC-1 | `i965` |
| HD 2000/3000 (Sandy Bridge) | MPEG-2, H.264, VC-1 | `i965` |
| HD 2500/4000 (Ivy Bridge) | MPEG-2, H.264, VC-1, JPEG | `i965` |
| HD 4200–5200 (Haswell) | MPEG-2, H.264, VC-1, JPEG | `i965` |
| Broadwell e mais novas | + VP8/HEVC/VP9/AV1 conforme a geração | `iHD` |

**Nenhuma GPU até Haswell decodifica VP9 ou AV1.** O YouTube usa VP9/AV1 por padrão, então nesses casos instale no navegador uma extensão que force H.264 (ex.: *h264ify* ou *enhanced-h264ify*). Sem ela o vídeo volta a ser decodificado pela CPU.

## 7.2 Instalar os pacotes

```bash
sudo xbps-install -S mesa-dri libva-intel-driver libva-utils intel-gpu-tools
```

Garanta que o usuário está no grupo `video` (saia e entre na sessão depois):

```bash
sudo usermod -aG video $USER
```

## 7.3 Forçar o driver VA-API correto

Adicione no `~/.profile` (ou em `/etc/environment`, sem o `export`):

```bash
export LIBVA_DRIVER_NAME=i965
```

## 7.4 Testar

```bash
vainfo
```

Resultado esperado (resumido):

```
vainfo: Driver version: Intel i965 driver for Intel(R) Sandybridge Mobile
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointVLD
```

`VAEntrypointVLD` = decodificação por hardware disponível.

## 7.5 Firefox

Em `about:config`:

```
media.ffmpeg.vaapi.enabled = true
```

Nas versões mais recentes do Firefox, se a decodificação não iniciar sozinha, também:

```
media.hardware-video-decoding.force-enabled = true
```

Se você aplicou o `user.js` da seção 6, apague-o. Reinicie o Firefox e confira em `about:support`, seção **Mídia**, se o H264 aparece com decodificação por hardware.

## 7.6 Confirmar que a GPU está trabalhando

Abra um vídeo no YouTube (com a extensão H.264 ativa) e rode:

```bash
sudo intel_gpu_top
```

Se a linha **Vídeo** ficar ocupada enquanto o vídeo toca, a decodificação está na GPU. O uso de CPU cai bastante.

---

# 8. Verificar suporte OpenGL

Executar:

```bash
glxinfo | grep "OpenGL version"
```

Exemplo:

```
OpenGL version string: 2.1 Mesa 25.3.3
```

Isso é o máximo suportado pela GMA 3100.

---

# 9. Verificar aceleração ativa

Executar:

```bash
glxinfo | grep "direct rendering"
```

Resultado correto:

```
direct rendering: Yes
```

---

# 10. Verificar driver carregado

Executar:

```bash
lsmod | grep i915
```

Resultado:

```
i915
```

---

# 11. Verificar status do Xorg

Executar:

```bash
grep -i driver /var/log/Xorg.0.log
```

Resultado correto:

```
modesetting
```

---

# 12. Não use xf86-video-intel em hardware muito antigo

Remova se estiver instalado:

```bash
sudo xbps-remove xf86-video-intel
```

O driver modesetting é mais estável.

---

# 13. Verificar DRI

Executar:

```bash
glxinfo | grep DRI
```

Para GPUs antigas, use:

```
DRI2
```

---

# 14. Verificar resolução

Executar:

```bash
xrandr
```

---

# 15. Problemas comuns e solução

## reboot ao entrar no XFCE

Disable compositor:

```bash
xfconf-query -c xfwm4 -p /general/use_compositing -s false
```

---

## O Firefox trava

Desative aceleração via user.js (passo 6). **Somente GMA** — em Intel HD, prefira manter a aceleração.

---

## vídeo do YouTube usando muita CPU (Intel HD)

- Check if the extension that forces H.264 is active (the GPU does not decode VP9/AV1).
- Execute `vainfo` e veja se aparece `VAProfileH264...: VAEntrypointVLD`.
- Confirme `LIBVA_DRIVER_NAME=i965` com `echo $LIBVA_DRIVER_NAME`.
- Confirme que o `user.js` da seção 6 não está aplicado (ele desliga o VA-API).

---

## tela preta

Remova xf86-video-intel:

```bash
sudo xbps-remove xf86-video-intel
```

---

# 16. Verificação completa do sistema gráfico

Executar:

```bash
inxi -G
glxinfo | grep OpenGL
lsmod | grep i915
xfconf-query -c xfwm4 -p /general/use_compositing
vainfo          # somente Intel HD (Grupo B)
```

---

# 17. GPUs Intel e suporte a OpenGL

| GPU | OpenGL |
|----|--------|
| GMA 950 | 2.1 |
| GMA 3100 | 2.1 |
| X3100 | 2.1 |
| X4500 | 2.1 |
| HD 2000 | 3,3 |
| HD 3000 | 3,3 |
| HD 4000 | 4,0 |
| UHD 620 | 4,6 |

---

# 18. Conclusão

**Grupo A – GMA (950, 3000, 3100, X3100, X4500):**

- driver kernel: i915
- driver Xorg: configuração de modos
- DRI2
- compositor desativado
- aceleração Firefox desativada

**Grupo B – Intel HD (Ironlake, HD 2000/3000, HD 2500/4000, Haswell):**

- driver kernel: i915
- driver Xorg: configuração de modos
- driver VA-API: i965 (`libva-intel-driver`) + `LIBVA_DRIVER_NAME=i965`
- VA-API ativado no Firefox
- extensão que força H.264 no YouTube

Sistema ficará estável e funcional.
