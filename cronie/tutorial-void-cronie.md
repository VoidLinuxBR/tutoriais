# Cronie no Void Linux — Execução diária de scripts como root

Este tutorial explica como configurar o **cronie (cron)** no **Void Linux** para executar scripts automaticamente, usando **runit**, com foco em jobs rodando como **root**.

Cenário real:
- Distro: Void Linux
- Init: runit
- Scheduler: cronie
- Script: /usr/local/bin/rsync-mirror.sh
- Execução: diária, como root

---

## 1. Instalar o cronie

Verificar se está instalado:

```sh
xbps-query -s cronie
```

Se não estiver:

```sh
xbps-install -S cronie
```

---

## 2. Ativar o serviço cronie (runit)

No Void, o cron **não inicia sozinho**.

```sh
ln -s /etc/sv/cronie /var/service/
```

Verificar status:

```sh
sv status cronie
```

Resultado esperado:

```text
run: cronie: (pid xxxx)
```

Se não estiver `run`, o cron **não executa nada**.

---

## 3. Preparar o script

O script deve:
- ter caminho absoluto
- ser executável
- pertencer ao root

```sh
chmod +x /usr/local/bin/rsync-mirror.sh
chown root:root /usr/local/bin/rsync-mirror.sh
```

Teste manual obrigatório:

```sh
sudo /usr/local/bin/rsync-mirror.sh
```

Se não rodar manualmente, **cron também não vai rodar**.

---

## 4. Editar o crontab do root usando nano

Por padrão, o `crontab -e` abre no `vi`.
Para usar `nano`:

```sh
sudo EDITOR=nano crontab -e
```

---

## 5. Corrigir erro comum do Void (/root/.cache)

Em instalações mínimas, pode aparecer:

```text
/root/.cache/crontab: mkdir: Arquivo ou diretório inexistente
```

Correção definitiva:

```sh
mkdir -p /root/.cache/crontab
chown -R root:root /root/.cache
chmod 700 /root/.cache
```

Depois disso, o aviso não aparece mais.

---

## 6. Configurar o crontab do root

Conteúdo recomendado:

```cron
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
0 3 * * * /usr/local/bin/rsync-mirror.sh
```

Isso significa:
- execução diária
- às 03:00
- como root
- com PATH completo
- usando caminho absoluto

Salvar e sair.

---

## 7. Conferir o crontab instalado

```sh
sudo crontab -l
```

Saída esperada:

```text
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
0 3 * * * /usr/local/bin/rsync-mirror.sh
```

---

## 8. (Opcional) Redirecionar saída para log

Cron não mostra saída em terminal.  
Para registrar execução:

```cron
0 3 * * * /usr/local/bin/rsync-mirror.sh >> /var/log/rsync-mirror.log 2>&1
```

Criar o log:

```sh
touch /var/log/rsync-mirror.log
chown root:root /var/log/rsync-mirror.log
chmod 640 /var/log/rsync-mirror.log
```

---

## 9. (Opcional) Testar sem esperar o horário

Trocar temporariamente para:

```cron
*/5 * * * * /usr/local/bin/rsync-mirror.sh
```

Esperar 5 minutos e verificar o resultado.  
Depois, **voltar para o horário definitivo**.

---

## 10. Boas práticas (recomendado)

### Lockfile (evitar execução dupla)

Dentro do script:

```sh
LOCK=/var/run/rsync-mirror.lock

exec 9>"$LOCK" || exit 1
flock -n 9 || exit 0
```

Evita rodar duas vezes em reboot ou atraso.

---

## Observações importantes

- Cron roda com ambiente mínimo
- Sempre definir PATH
- Sempre usar caminhos absolutos
- Void não cria diretórios automaticamente
- Sem systemd, sem timers
- Cronie + runit é previsível e estável

---

## Resultado final

- Script rodando automaticamente
- Execução diária
- Como root
- Sem intervenção manual
- Infra tradicional e confiável

Este método funciona há décadas e continua sendo a forma mais simples e robusta de automação no Unix.
