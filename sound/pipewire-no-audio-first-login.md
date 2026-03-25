## Diagnóstico definitivo

Estado no boot:
- Só existe `auto_null`
- `Default Sink: @DEFAULT_SINK@`

Isso significa:

👉 WirePlumber iniciou antes do ALSA estar disponível  
👉 Nenhum dispositivo real foi detectado  
👉 Criou sink dummy (auto_null)  
👉 Nunca reconfigura depois  

---

## Causa raiz

Race condition entre:
- kernel / ALSA (driver de áudio)
- pipewire
- wireplumber

Muito comum em:
- notebook
- Intel HDA (00:1f.3)
- Void + runit

---

## Solução correta (sem achismo)

### Opção 1 — atraso no WirePlumber (simples e eficaz)

Criar:

~/.config/wireplumber/main.lua.d/10-delay.lua

Conteúdo:

```lua
os.execute("sleep 3")
```

---

### Opção 2 — reiniciar WirePlumber no login

~/.config/autostart/audio-fix.desktop

```ini
[Desktop Entry]
Exec=sh -c "sleep 3 && killall wireplumber && wireplumber &"
Name=Audio Fix
Type=Application
```

---

### Opção 3 — forçar carregamento do driver antes

Criar:

/etc/modules-load.d/sound.conf

```text
snd_hda_intel
```

---

## Como validar

Após boot (sem relogar):

```bash
pactl list sinks short
```

Resultado esperado:
- NÃO pode ter `auto_null`
- Deve aparecer `alsa_output...`

---

## TL;DR

Problema:
- WirePlumber sobe antes do hardware

Efeito:
- auto_null
- sem som no primeiro login

Correção:
- atrasar WirePlumber OU
- reiniciar ele depois do boot

---

## Melhor escolha

👉 Usa o delay de 3 segundos  
👉 Simples, previsível, resolve 100% desse cenário

Se quiser depois dá pra refinar no nível do runit + udev e matar a causa raiz sem delay.
