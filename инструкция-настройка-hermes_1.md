# Hermes Agent на локальном сервере — полная инструкция по настройке

Актуально на **10.10.2026**. Стек: Ubuntu 24.04, AMD Ryzen 7 PRO 8845HS (Radeon 780M iGPU), 16 ГБ RAM, Ollama + Hermes Agent, доступ по SSH.

---

## 1. Схема стека

```
Ваш Mac (клиент)
   │  sshpass / ssh / tmux
   ▼
Сервер «agent» — 192.168.1.150 (также 192.168.10.239, тот же сервер, разные роутеры)
   ├── Ollama  (системный сервис) — движок локальных LLM
   │     ├── qwen3.5:9b        (основная, общие задачи)
   │     ├── qwen2.5-coder:14b (код)
   │     ├── qwen3:14b, qwen3.8:27b, gemma4:12b (резерв)
   ├── Hermes Agent v0.21.5  (CLI + gateway)
   │     ├── hermes-gateway (systemd user-сервис)
   │     ├── ~/.hermes/     (конфиг, память, скины, инструменты)
   └── Собственные инструменты (Python+SQLite, локально)
         ├── hmem    — память (mem0/Zep-like)
         ├── htrace  — наблюдаемость (Langfuse-like)
         ├── hsearch — веб-поиск+скрапинг (Tavily/Firecrawl-like)
         └── hact    — интеграции/действия (Composio/MCP-like)
```

---

## 2. Доступ по SSH

Учётка: `kkutuzov` / пароль `132383`.

```bash
# Обычный вход (Hermes с qwen3.5:9b)
ssh kkutuzov@192.168.1.150
#  → автоматически открывает tmux-сессию hermes с Hermes

# Быстрый вход одной командой (функции в ~/.zshrc на Mac)
hermes        # обычный режим
hermes-coder  # кодовый режим (qwen2.5-coder:14b) в проекте inexplicable
hermes-mon    # мониторинг (htop + sensors + GPU + RAM + SSD + сеть)
hermes-list   # список tmux-сессий
hermes-quit   # закрыть сессию hermes
```

Внутри tmux: `Ctrl-b d` — выйти (сессия остаётся), повторный вход — вернуться.

---

## 3. Ручная настройка Ollama

### 3.1. Пользователь и место хранения
- Сервис: `ollama.service` (system), пользователь `ollama`
- Модели хранятся в `/usr/share/ollama/.ollama/models/`
- Drop-in конфиг: `/etc/systemd/system/ollama.service.d/qwen-stable.conf`

### 3.2. Текущий drop-in (iGPU-оптимизированный)
```ini
[Service]
Environment="HSA_OVERRIDE_GFX_VERSION=11.0.0"   # Phoenix iGPU
Environment="OLLAMA_IGPU_ENABLE=1"              # включаем iGPU
Environment="OLLAMA_CONTEXT_LENGTH=65536"       # окно 64K (нужно Hermes)
Environment="OLLAMA_GPU_LAYERS=20"              # слоёв на GPU (28 → ROCm-краш)
Environment="OLLAMA_FLASH_ATTENTION=1"          # ускоряет long-context
Environment="OLLAMA_KEEP_ALIVE=30m"             # держать модель 30 мин
Environment="OLLAMA_NUM_PARALLEL=1"
Environment="OLLAMA_MAX_LOADED_MODELS=1"        # одна модель за раз
```

Применить изменения:
```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

### 3.3. Важные замечания по железу
- **16 ГБ RAM**, iGPU резервирует до 16 ГБ VRAM. Модель 14B (coder) занимает ~15 ГБ с контекстом — держать **одну модель за раз**.
- Модель 27B (qwen3.8) не помещается стабильно (своп, ROCm-краш).
- **Вторая планка RAM не распознаётся** (диск содержит 32 ГБ физически, ОС видит ~17.5 ГБ) — при желании починить: переставить/протереть контакты SO-DIMM, проверить BIOS.

### 3.4. Полезные команды
```bash
ollama list          # список моделей
ollama ps            # загруженные модели (+ GPU-сплит)
ollama stop <model>  # выгрузить модель
ollama pull <model>  # скачать
```

---

## 4. Hermes Agent

### 4.1. Установка/обновление
Hermes собран из git в `~/.hermes/hermes-agent`. Основной CLI — `~/.local/bin/hermes`.

### 4.2. Конфиг `~/.hermes/config.yaml` (ключевые блоки)

```yaml
model:
  api_key: ollama
  base_url: http://127.0.0.1:11434/v1
  default: qwen3.5:9b
  provider: ollama-launch
  ollama_num_ctx: 64000   # обход: модели с окном <64K допускаются

providers:
  ollama-launch:
    api: http://127.0.0.1:11434/v1
    default_model: qwen3.5:9b
    models:
      - qwen3.5:9b
      - qwen3:14b-q4_K_M
      - gemma4:12b
      - qwen3.8:27b-q4_K_M
      - qwen2.5-coder:14b

display:
  skin: russian   # русский интерфейс
```

Тулсеты для скорости сведены к минимуму (web, terminal, file, code_execution).

### 4.3. Таймауты для медленных локальных моделей
В `~/.hermes/.env`:
```
HERMES_API_TIMEOUT=1800
HERMES_STREAM_READ_TIMEOUT=1800
```

---

## 5. Русский интерфейс

1. Файл `~/.hermes/skins/russian.yaml` (переопределяет приветствие/прощание/справку):
```yaml
name: russian
description: "Русский интерфейс"
branding:
  welcome: "Добро пожаловать в Hermes! Введите сообщение или /help."
  goodbye: "До связи! ☤"
  help_header: "Доступные команды"
```
2. В `~/.hermes/config.yaml`: `display.skin: russian`
3. В `~/.hermes/SOUL.md` (первая строка): «Всегда отвечай пользователю по-русски…»

Замечание: скин переводит branding-строки; часть служебных надписей TUI (подсказки, счётчик tools) остаётся на английском (захардкожено).

---

## 6. Проект inexplicable — кодовый режим

- Лаунчер: `/home/kkutuzov/hermes-coder.sh` — `cd ~/devolopers/inexplicable && hermes -m qwen2.5-coder:14b`
- Быстрый вход: `hermes-coder` (Mac) или `ssh -t kkutuzov@192.168.1.150 hermes-coder`
- В корне проекта создан `AGENTS.md` (контекст для агента).
- Установка unzip, разархивирование архива в `~/devolopers/`.

---

## 7. Собственные инструменты (все локально, Python+SQLite)

| Команда | Файл | Назначение | Аналог |
|---|---|---|---|
| `hmem` | `~/hermes-memory.py` | долговременная память: слои, теги, темпоральность, история версий | mem0/Zep/LangMem |
| `htrace` | `~/hermes-tracer.py` | трассы вызовов LLM/инструментов, деревья, метрики | Langfuse/Phoenix |
| `hsearch` | `~/hermes-search.py` | веб-поиск (DDG-lite) + извлечение страниц | Tavily/Firecrawl |
| `hact` | `~/hermes-actions.py` | реестр действий: github/gmail/file/slack | Composio/MCP |

Все описаны в `~/.hermes.md` — агент знает, как ими пользоваться.

Пример использования:
```bash
hmem remember "Контракт с ООО Ромашка до 2027-03-01" --scope user --tag contract --important
hmem search "ромашка"
htrace start "webread" --type tool --trace t1
hsearch search "rust language" --max 5
hact list
```

---

## 8. Мониторинг (tmux)

- `/usr/local/bin/tsys.sh` — минимальный статус-бар: `C<CPU>% R<RAM>% [G<GPU>%] [сеть]`
- `~/.tmux.conf` — status-right: `tsys.sh` + время (сервер в МСК)
- Часовой пояс сервера: `Europe/Moscow` (`timedatectl set-timezone Europe/Moscow`)

---

## 9. Обслуживание

### Обновление системы и перезагрузка
```bash
sudo apt-get update && sudo apt-get dist-upgrade -y
sudo apt-get autoremove -y && sudo apt-get autoclean
sudo shutdown -r now   # авто-запуск сервисов настроен (ollama, hermes-gateway)
```

### Диагностика
```bash
systemctl status ollama
journalctl -u ollama -n 50
systemctl --user status hermes-gateway
free -h        # память (важно: одна модель за раз)
timedatectl    # время/зона
```

### Бэкапы важных конфигов
Рекомендуется периодически архивировать:
```bash
tar czf ~/backup-hermes-$(date +%F).tgz ~/.hermes/config.yaml ~/.hermes/SOUL.md ~/.hermes/.env ~/.hermes/skins ~/.hermes.md ~/.hermes-memory.db ~/.hermes-trace.db
```

---

## 10. Чек-лист «с нуля до работы»

- [ ] Проверить SSH-доступ
- [ ] Установить Ollama, настроить drop-in (iGPU, контекст, GPU-слои)
- [ ] `ollama pull` нужные модели
- [ ] Установить/обновить Hermes (`~/.local/bin/hermes`)
- [ ] Настроить `config.yaml` (модель, провайдер, ollama_num_ctx, skin)
- [ ] Настроить `SOUL.md` (русский язык), `.hermes.md` (инструменты)
- [ ] Создать скин `russian.yaml`
- [ ] Установить инструменты: `hmem`, `htrace`, `hsearch`, `hact`
- [ ] Настроить tmux-статус (`tsys.sh`, `~/.tmux.conf`)
- [ ] Установить зону `Europe/Moscow`
- [ ] Настроить вход на Mac: `~/.zshrc` (hermes, hermes-coder, hermes-mon)
- [ ] Создать кодовый режим для проекта (`hermes-coder.sh`)
- [ ] Проверить: `ssh …`, `hermes-coder`, `hermes-mon`
---

# ПРИЛОЖЕНИЕ А. Полные исходники

> Все файлы ниже — точное содержимое созданных/изменённых файлов на сервере
> и на Mac по состоянию на 10.10.2026.

## А.1. Ollama — drop-in конфиг
`/etc/systemd/system/ollama.service.d/qwen-stable.conf`

```ini
[Service]
Environment="HSA_OVERRIDE_GFX_VERSION=11.0.0"
Environment="OLLAMA_IGPU_ENABLE=1"
Environment="OLLAMA_CONTEXT_LENGTH=65536"
Environment="OLLAMA_GPU_LAYERS=20"
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_KEEP_ALIVE=30m"
Environment="OLLAMA_NUM_PARALLEL=1"
Environment="OLLAMA_MAX_LOADED_MODELS=1"
```

## А.2. Мониторинг

### `/usr/local/bin/tsys.sh` (статус-бар tmux)
```bash
#!/usr/bin/env bash
# Минимальный системный монитор для tmux status-right.
getcpu() {
  read -r _ a b c d e f g h i j k < /proc/stat
  t1=$((a+b+c+d+e+f+g+h+i+j+k)); idle1=$((d))
  sleep 0.5
  read -r _ a b c d e f g h i j k < /proc/stat
  t2=$((a+b+c+d+e+f+g+h+i+j+k)); idle2=$((d))
  dt=$((t2-t1)); di=$((idle2-idle1))
  [ "$dt" -le 0 ] && dt=1
  printf "%d" $((100 - di*100/dt))
}
GPU_T=0; GPU_U=0
hf=$(ls /sys/class/drm/card0/device/hwmon/hwmon*/temp1_input 2>/dev/null | head -1)
[ -n "$hf" ] && GPU_T=$(($(cat "$hf")/1000))
GPU_U=$(cat /sys/class/drm/card0/device/gpu_busy_percent 2>/dev/null | tr -d \\n); [ -z "$GPU_U" ] && GPU_U=0
CPU=$(getcpu)
RAM=$(awk "/MemTotal|MemAvailable/{if(\$1==\"MemTotal:\")t=\$2; else a=\$2} END{printf \"%d\", (1-a/t)*100}" /proc/meminfo)
NET=$(/usr/local/bin/tmux-netspeed | tr -d " " | sed -E "s/↓0KB\/s↑0KB\/s//")
OUT="C${CPU}% R${RAM}%"
[ "$GPU_U" != "0" ] && OUT="$OUT G${GPU_U}%"
[ -n "$NET" ] && OUT="$OUT $NET"
echo "$OUT"
```

### `/usr/local/bin/tmux-netspeed`
```bash
#!/usr/bin/env bash
STATUS_IFACE=$(ip route 2>/dev/null | awk '/default/{print $5; exit}')
rx_before=$(cat /sys/class/net/$STATUS_IFACE/statistics/rx_bytes 2>/dev/null || echo 0)
tx_before=$(cat /sys/class/net/$STATUS_IFACE/statistics/tx_bytes 2>/dev/null || echo 0)
sleep 1
rx_after=$(cat /sys/class/net/$STATUS_IFACE/statistics/rx_bytes 2>/dev/null || echo 0)
tx_after=$(cat /sys/class/net/$STATUS_IFACE/statistics/tx_bytes 2>/dev/null || echo 0)
rx_speed=$(( (rx_after - rx_before) / 1024 ))
tx_speed=$(( (tx_after - tx_before) / 1024 ))
echo "↓${rx_speed}KB/s ↑${tx_speed}KB/s"
```

### `~/.tmux.conf`
```conf
# ==== tmux 3.4 — минимальный конфиг ====
set -g default-terminal "screen-256color"
set -g base-index 1
setw -g pane-base-index 1
set -g mouse on
set -g escape-time 10

# ==== Статус-бар: минимум, время обязательно ====
set -g status-interval 2
set -g status-left-length 40
set -g status-right-length 60
set -g status-style "bg=colour236,fg=colour250"
set -g status-left "#[bold]#S #[dim]#{session_windows}w"
set -g status-right "#[fg=colour208]#(/usr/local/bin/tsys.sh)  #[fg=colour231]%H:%M"

set -g window-status-current-style "fg=colour231,bg=colour24,bold"
set -g window-status-style "fg=colour250,bg=colour236"
set -g window-status-format " #I  "
set -g window-status-current-format " #I "

bind r source-file ~/.tmux.conf \; display "reloaded"
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R
bind '|' split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"
bind m split-window -h -l 40 "htop"
```

### `~/monitor.sh` (полный монитор для `hermes-mon`)
```bash
#!/usr/bin/env bash
while true; do
  clear
  echo "===== HERMES SERVER MONITOR ====="
  echo
  echo "--- CPU / темп. (sensors) ---"
  sensors 2>/dev/null | grep -E "Tctl|edge|temp1" || echo "n/a"
  echo
  echo "--- GPU (AMD iGPU) ---"
  echo -n "util   : "; cat /sys/class/drm/card0/device/gpu_busy_percent 2>/dev/null; echo " %"
  echo -n "vram   : "; awk -v u="$(cat /sys/class/drm/card0/device/mem_info_vram_used 2>/dev/null||echo 0)" -v t="$(cat /sys/class/drm/card0/device/mem_info_vram_total 2>/dev/null||echo 1)" 'BEGIN{printf "%.2f / %.2f GB", u/1073741824, t/1073741824}'; echo
  echo -n "temp   : "; cat /sys/class/drm/card0/device/hwmon/hwmon*/temp1_input 2>/dev/null | head -1 | awk '{printf "%.0f C", $1/1000}'; echo
  echo
  echo "--- RAM / SWAP ---"
  free -h | awk 'NR==1||NR==2||NR==3'
  echo
  echo "--- SSD ---"
  df -h / | awk 'NR==1||NR==2'
  echo
  echo "--- CPU load / top ---"
  uptime
  top -bn1 | head -5
  echo
  echo "--- NET (замер 5с) ---"
  iface=$(ip route | awk '/default/{print $5; exit}')
  before=$(cat /sys/class/net/$iface/statistics/rx_bytes)
  bt=$(cat /sys/class/net/$iface/statistics/tx_bytes)
  sleep 5
  after=$(cat /sys/class/net/$iface/statistics/rx_bytes)
  at=$(cat /sys/class/net/$iface/statistics/tx_bytes)
  echo "⬇ $(( (after-before)/5120 )) KB/s   ⬆ $(( (at-bt)/5120 )) KB/s  (iface=$iface)"
  sleep 5
done
```

## А.3. Лаунчеры входа

### `~/hermes-shell.sh` (обычный запуск Hermes)
```bash
#!/usr/bin/env bash
export PATH="$HOME/.local/bin:$PATH"
cd /home/kkutuzov
exec hermes
```

### `~/quick-start.sh` (быстрый вход по `ssh`)
```bash
#!/usr/bin/env bash
# Быстрый вход: подключиться к сессии hermes или создать её
export PATH="/usr/bin:/usr/local/bin:$PATH"
if tmux has-session -t hermes 2>/dev/null; then
  exec tmux attach -t hermes
else
  exec tmux new -s hermes "bash /home/kkutuzov/hermes-shell.sh"
fi
```

### `~/hermes-coder.sh` (кодовый режим для проекта)
```bash
#!/usr/bin/env bash
# Кодер-режим Hermes: запуск в корне проекта inexplicable с qwen2.5-coder:14b
export PATH="$HOME/.local/bin:$PATH"
PROJECT=/home/kkutuzov/devolopers/inexplicable
cd "$PROJECT" || { echo "Project not found: $PROJECT"; exit 1; }
exec hermes -m qwen2.5-coder:14b
```

### `~/.local/bin/hermes-coder` (вход через SSH-команду)
```bash
#!/usr/bin/env bash
# Кодер-режим Hermes: входит в tmux-сессию hermes-coder или создаёт её
export PATH="$HOME/.local/bin:$PATH"
if tmux has-session -t hermes-coder 2>/dev/null; then
  exec tmux attach -t hermes-coder
else
  exec tmux new -s hermes-coder "bash /home/kkutuzov/hermes-coder.sh"
fi
```

## А.4. Русский интерфейс

### `~/.hermes/skins/russian.yaml`
```yaml
name: russian
description: "Русский интерфейс — все служебные надписи на русском"
colors:
  banner_title: "#FFD700"
  banner_accent: "#FFBF00"
  ui_accent: "#FFBF00"
branding:
  agent_name: "Hermes"
  welcome: "Добро пожаловать в Hermes! Введите сообщение или /help для списка команд."
  goodbye: "До связи! ☤"
  response_label: " ☤ Hermes "
  prompt_symbol: "❯"
  help_header: "Доступные команды"
```

### `~/.hermes/SOUL.md` (первая строка добавлена для русского)
```text
Всегда отвечай пользователю по-русски, если он не попросит иной язык. Тон — как в исходных правилах: кратко и по делу.

You are Hermes Agent, built by Nous Research. Be direct: match the length of your reply to the weight of the ask — a one-line question gets a one-line answer, and finished work gets a short report of what changed, what's verified, and what's left, never a replay of the process. ...
```

### `~/.hermes/.env` (таймауты)
```
HERMES_API_TIMEOUT=1800
HERMES_STREAM_READ_TIMEOUT=1800
```

## А.5. Правила агента

### `~/.hermes.md` (загружается автоматически как контекст)
```markdown
# Agent web rule

When the user asks you to READ or EXTRACT content from a specific web page and the built-in
web_extract tool fails (error 403 / "Keyless ... extract failed" / "no result", or any similar
extract failure), DO NOT give up or guess from partial snippets. Instead, read the page directly
through the terminal:

1. Run:  python3 /home/kkutuzov/webread.py "<URL>" --max-chars 20000
   (or: bash /home/kkutuzov/webread.sh "<URL>")
2. If the first attempt returns WEBREAD_ERROR or partial content, retry once with
   --max-chars 50000.
3. Use the returned readable text as the authoritative page content and answer from it.

Also prefer webread.py over web_extract for RU-language sites (habr.com, vc.ru, etc.) that
commonly block Exa/Firecrawl/Tavily keyless extract.

This keeps the agent self-sufficient (no external API keys required).

# Memory tool (long-term facts)

You have a local long-term memory tool backed by SQLite. USE IT to remember durable facts
about the user, the project, and tasks, and to recall them in later sessions.

Commands (run via terminal, python3 /home/kkutuzov/hermes-memory.py ...):
- REMEMBER a durable fact:
    python3 /home/kkutuzov/hermes-memory.py remember "<fact>" --scope user|agent --tag t1,t2 [--important] [--source "<where>"]
  Scope: user = about the user; agent = about the project/environment. Use --important for
  facts that must always surface. Tag generous.
- SEARCH memory before answering from memory:
    python3 /home/kkutuzov/hermes-memory.py search "<query>"
- RECALL newest/important facts for context:
    python3 /home/kkutuzov/hermes-memory.py recall --limit 15
- LIST by tag:   python3 /home/kkutuzov/hermes-memory.py list --tag <t>
- FORGET a fact: python3 /home/kkutuzov/hermes-memory.py forget <id>
  or by query:   python3 /home/kkutuzov/hermes-memory.py forget-query "<text>"

Policy:
- When the user shares a stable preference, a project fact, a constraint, or something
  you should not re-ask, REMEMBER it (prefer scope=user for personal facts).
- At the start of a task (or when context feels thin), run `recall` to load relevant facts.
- When unsure whether you know something, run `search` before guessing.
- After a task, if you learned durable facts, store them.

# Local toolset (homegrown, by motif of mem0/Zep/mem/Langfuse/Tavily/Composio)

You have several local CLI tools (no cloud, no API keys). Use them via terminal.

## Memory (long-term facts) — `hmem`  (python3 /home/kkutuzov/hermes-memory.py)
- hmem remember "<fact>" --scope user|agent --tag t1,t2 [--important] [--valid-until YYYY-MM-DD]
- hmem search "<query>"   /   hmem recall --limit N   /   hmem list --tag T
- hmem history <id> , hmem valid-at YYYY-MM-DD , hmem expired , hmem forget <id>
Remember a durable user/project fact when you learn it; recall/search before answering.

## Observability (traces of your own work) — `htrace` (python3 /home/kkutuzov/hermes-tracer.py)
- htrace start "name" --trace T --parent P --type llm|tool|span  -> prints observation id
- htrace end <id> --output "..." --tokens-in N --tokens-out N --model M [--error "msg"] [--level warning]
- htrace tree --trace T , htrace traces , htrace stats , htrace tail --errors
For heavy tasks, wrap LLM/tool steps as observations to review later.

## Web search & extract — `hsearch`
- hsearch search "<query>" --max N [--sites site1.com]
- hsearch extract <url> [--markdown] [--max-chars N]
Preferred over broken web_extract for RU sites; wraps webread.py.

## Actions/integrations (local tools) — `hact`
- hact init   # create default tools (github_status, gmail_send, file_append, file_read, slack_send)
- hact list , hact show <name> , hact status
- hact call <name> --arg key=value ...
Self-hosted replacement for Composio/MCP; "accounts" are local files under ~/.hermes-actions/.

# Code model
For code generation / refactoring / debugging / code review tasks, prefer the
dedicated coding model qwen2.5-coder:14b. Switch to it with `/model qwen2.5-coder:14b`
in an interactive session, or launch a one-shot with `hermes -m qwen2.5-coder:14b -z "..."`.
Use qwen3.5:9b for general conversation; use qwen2.5-coder:14b when the ask is mostly code.
```

### `~/devolopers/inexplicable/AGENTS.md` (контекст проекта для кодера)
```markdown
# inexplicable — tennis data collection/scraping project

Python project that collects and processes tennis match/statistics data
(via flashscore feeds and related sources). Contains backups and helper scripts.

## Layout
- `tennis/` — main data collection code (e.g. `tennis/flashscore/`)
- `backup/` — backups / older snapshots
- `requirements.txt` — Python dependencies (note: file has UTF-16 encoding on disk)
- `.venv/` — pre-existing virtualenv (don't commit; use it with `source .venv/bin/activate`)
- `todo.md` — task notes
- `gpt.txt` — scratch notes

## Conventions
- Python 3; use the project venv if present.
- Follow the existing style of the module you touch.
- Check `todo.md` and `gpt.txt` for context before editing.
```

## А.6. Быстрый вход на Mac (`~/.zshrc`)

```zsh
# ===== Hermes Agent: быстрый вход одной командой =====
export PATH="$HOME/.local/bin:$PATH"
_HERMES_HOST="kkutuzov@192.168.1.150"
_HERMES_PASS="${HERMES_SSH_PASS:-132383}"

hermes() {
  command -v sshpass >/dev/null 2>&1 || { echo "install sshpass: brew install hudochenkov/sshpass/sshpass" >&2; return 1; }
  sshpass -p "$_HERMES_PASS" ssh -t -o StrictHostKeyChecking=no -o ConnectTimeout=15 "$_HERMES_HOST" \
    'bash /home/kkutuzov/quick-start.sh'
}

hermes-mon() {
  command -v sshpass >/dev/null 2>&1 || { echo "install sshpass" >&2; return 1; }
  sshpass -p "$_HERMES_PASS" ssh -t -o StrictHostKeyChecking=no -o ConnectTimeout=15 "$_HERMES_HOST" \
    'tmux has-session -t hermes 2>/dev/null || tmux new-session -d -s hermes; tmux kill-window -t hermes:mon 2>/dev/null; tmux new-window -t hermes -n mon -d; tmux split-window -h -t hermes:mon -d /home/kkutuzov/monitor.sh; tmux send-keys -t hermes:mon.0 "htop" C-m; tmux select-window -t hermes:mon; tmux attach -t hermes'
}

alias hermes-list='sshpass -p "$_HERMES_PASS" ssh -o StrictHostKeyChecking=no -o ConnectTimeout=15 "$_HERMES_HOST" "tmux ls"'
alias hermes-quit='sshpass -p "$_HERMES_PASS" ssh -o StrictHostKeyChecking=no -o ConnectTimeout=15 "$_HERMES_HOST" "tmux kill-session -t hermes"'

hermes-coder() {
  command -v sshpass >/dev/null 2>&1 || { echo "install sshpass" >&2; return 1; }
  sshpass -p "$_HERMES_PASS" ssh -t -o StrictHostKeyChecking=no -o ConnectTimeout=15 "$_HERMES_HOST" \
    'if tmux has-session -t hermes-coder 2>/dev/null; then exec tmux attach -t hermes-coder; else exec tmux new -s hermes-coder "bash /home/kkutuzov/hermes-coder.sh"; fi'
}
```



---

## А.7. Собственные Python-инструменты (полный код)

### `hermes-memory.py` — Память (hmem) — SQLite+FTS5, слои, темпоральность, история версий

```python
#!/usr/bin/env python3
"""
hermes-memory.py — локальная долговременная память для Hermes Agent.

Создано по мотивам лучших open-source memory-провайдеров из hermesatlas.com:
  - Mnemosyne        : лёгкий SQLite, zero-cloud, одна зависимость
  - Sibyl-Memory     : файловый поиск БЕЗ векторных эмбеддингов (FTS5)
  - mem0             : уровни user/session/agent + временные рамки
  - Memanto          : долговременные факты, вытеснение устаревших
  - gbrain           : факты с provenance (источник и дата)
  - Mnemosyne/memory-os : эпизодическое сжатие

Хранит ЧТО, ИЗВЕСТНО C КАКОГО ВРЕМЕНИ и ОТКУДА (provenance). Без эмбеддингов,
без облака, без внешних серверов. Поиск — полнотекстовый SQLite FTS5.

Использование (вызывается агентом через терминал):
  python3 /home/kkutuzov/hermes-memory.py remember <fact>
           [--tag T]... [--scope user|agent|session] [--important]
  python3 /home/kkutuzov/hermes-memory.py search <query> [--limit N]
  python3 /home/kkutuzov/hermes-memory.py list [--tag T] [--scope S] [--limit N]
  python3 /home/kkutuzov/hermes-memory.py get <id>
  python3 /home/kkutuzov/hermes-memory.py forget <id>
  python3 /home/kkutuzov/hermes-memory.py recall <query>   # контекстный снапшот для промпта
  python3 /home/kkutuzov/hermes-memory.py stats
  python3 /home/kkutuzov/hermes-memory.py forget-query <query>   # удалить по FTS-совпадению
"""
from __future__ import annotations

import argparse
import json
import os
import sqlite3
import sys
from datetime import datetime, timezone

DB_PATH = os.environ.get("HERMES_MEMORY_DB", "/home/kkutuzov/.hermes-memory.db")
# граница "свежести" для recency-взвешивания
FRESH_DAYS = 30


def _db() -> sqlite3.Connection:
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA journal_mode=WAL;")
    conn.execute("PRAGMA synchronous=NORMAL;")
    _ensure_schema(conn)
    return conn


def _ensure_schema(conn: sqlite3.Connection) -> None:
    conn.execute(
        """
CREATE TABLE IF NOT EXISTS facts (
            id        INTEGER PRIMARY KEY AUTOINCREMENT,
            fact      TEXT NOT NULL,
            scope     TEXT NOT NULL DEFAULT 'agent',   -- user | agent | session
            tags      TEXT NOT NULL DEFAULT '',        -- csv
            source    TEXT NOT NULL DEFAULT '',        -- provenance: session/url где узнали
            important INTEGER NOT NULL DEFAULT 0,
            valid_from TEXT,   -- zep: факт валиден с этого времени (NULL = всегда)
            valid_until TEXT,  -- zep: факт валиден до (NULL = до сих пор)
            episode_id INTEGER, -- источник-эпизод
            created_at TEXT NOT NULL,
            updated_at TEXT NOT NULL
        );
        """
    )
    # миграция: если старые базы без темпоральных колонок — добавить
    cols = [r[1] for r in conn.execute("PRAGMA table_info(facts)").fetchall()]
    for c, decl in (("valid_from", "TEXT"), ("valid_until", "TEXT"), ("episode_id", "INTEGER")):
        if c not in cols:
            conn.execute(f"ALTER TABLE facts ADD COLUMN {c} {decl}")
    # история версий факта (mem0 history — откат изменений)
    conn.execute(
        """
        CREATE TABLE IF NOT EXISTS fact_history (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            fact_id INTEGER NOT NULL,
            snapshot TEXT NOT NULL,        -- снимок факта на момент
            scope TEXT, tags TEXT, important INTEGER,
            valid_from TEXT, valid_until TEXT,
            changed_at TEXT NOT NULL
        );
        """
    )
    conn.execute(
        "CREATE INDEX IF NOT EXISTS idx_fh_fact ON fact_history(fact_id)"
    )
    conn.execute(
        """
        CREATE TABLE IF NOT EXISTS episodes (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            ref TEXT NOT NULL DEFAULT '',    -- сессия/url/сообщение-источник
            summary TEXT NOT NULL DEFAULT '',
            created_at TEXT NOT NULL
        );
        """
    )
    conn.commit()


def _now() -> str:
    return datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")


def _normalize_fact(fact: str) -> str:
    return " ".join(fact.split())


def _is_duplicate(conn: sqlite3.Connection, fact: str) -> sqlite3.Row | None:
    norm = _normalize_fact(fact)
    # точное (нормализованное) совпадение — не засоряем память
    row = conn.execute(
        "SELECT * FROM facts WHERE replace(fact, char(10), ' ') = ? LIMIT 1",
        (norm,),
    ).fetchone()
    if row:
        return row
    # близкое совпадение по ключевым словам (поиск в FTS) — суперсессия-кандидат
    terms = " AND ".join(_fts_terms(norm)[:6])
    if terms:
        try:
            row = conn.execute(
                "SELECT rowid FROM facts_fts WHERE facts_fts MATCH ? ORDER BY rank LIMIT 1",
                (terms,),
            ).fetchone()
        except sqlite3.OperationalError:
            row = None
        if row:
            fid = row[0]
            dup = conn.execute("SELECT * FROM facts WHERE id=?", (fid,)).fetchone()
            if dup and _close_match(dup["fact"], fact):
                return dup
    # надёжная супер-сессия: обыгрываем каждый существующий факт по общему префиксу.
    # Это ловит переформулировки/изменённые значения, если FTS не сматчил (напр. разные даты).
    for f in conn.execute("SELECT * FROM facts").fetchall():
        if _close_match(f["fact"], norm):
            return f
    return None


def _close_match(a: str, b: str) -> bool:
    """Near-duplicate: subsume if they share a long common prefix (before a
    changing tail like a date/version), OR one contains the other. Catches
    superseded restatements without conflating unrelated facts."""
    a, b = a.lower(), b.lower()
    if len(a) >= 24 and (a in b or b in a):
        return True
    # общий префикс >= 60% более короткой строки => считаем той же сущностью
    n = min(len(a), len(b))
    if n < 24:
        return False
    common = 0
    for i in range(n):
        if a[i] == b[i]:
            common += 1
        else:
            break
    return common >= int(0.6 * n)


def _fts_terms(text: str):
    toks = []
    seen = set()
    for w in text.lower().split():
        w = "".join(ch for ch in w if ch.isalnum())
        if len(w) >= 3 and w not in seen:
            seen.add(w)
            toks.append(w)
    return toks


def cmd_remember(args, conn) -> int:
    fact = _normalize_fact(" ".join(args.fact))
    if not fact:
        print("ERR: empty fact", file=sys.stderr)
        return 2
    tags = [t.strip() for t in (args.tag or []) if t.strip()]
    now = _now()
    valid_from = args.valid_from or None

    dup = _is_duplicate(conn, fact)
    if dup:
        # super-сессия (mem0/LangMem): семантическое обновление вместо нового факта,
        # старая версия сохраняется в history (откат возможен).
        conn.execute(
            "INSERT INTO fact_history (fact_id, snapshot, scope, tags, important, "
            "valid_from, valid_until, changed_at) VALUES (?, ?, ?, ?, ?, ?, ?, ?)",
            (
                dup["id"], dup["fact"], dup["scope"], dup["tags"], dup["important"],
                dup["valid_from"], now, now,
            ),
        )
        conn.execute(
            "UPDATE facts SET fact=?, updated_at=?, source=?, valid_until=? WHERE id=?",
            (fact, now, args.source or dup["source"], args.valid_until, dup["id"]),
        )
        conn.commit()
        print(f"facts:updated #{dup['id']} (old version kept in history) == {fact}")
        return 0

    cur = conn.execute(
        "INSERT INTO facts (fact, scope, tags, source, important, valid_from, valid_until, "
        "created_at, updated_at) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)",
        (
            fact,
            args.scope,
            ",".join(tags),
            args.source or "",
            1 if args.important else 0,
            valid_from,
            args.valid_until,
            now,
            now,
        ),
    )
    conn.commit()
    print(f"facts:remembered #{cur.lastrowid} [{args.scope}] {fact}")
    return 0


def cmd_search(args, conn) -> int:
    query = " ".join(args.query)
    terms = _fts_terms(query)
    if not terms:
        print("memory: (no useful search terms)")
        return 0
    match = " AND ".join(terms)
    limit = args.limit
    rows = []
    try:
        rows = conn.execute(
            """
            SELECT f.* FROM facts f
            JOIN facts_fts ON facts_fts.rowid = f.id
            WHERE facts_fts MATCH ? AND (? = 'all' OR f.scope = ?)
            ORDER BY rank
            LIMIT ?
            """,
            (match, args.scope if args.scope else "all", args.scope or "", limit),
        ).fetchall()
    except sqlite3.OperationalError:
        rows = []
    # LIKE-fallback: поиск вхождения КАЖДОГО термина по отдельности (охватывает
    # кириллицу и частичные словоформы, которые FTS unicode61 не разбивает)
    if not rows:
        found = set()
        for t in terms:
            for r in conn.execute(
                "SELECT * FROM facts WHERE fact LIKE ? ORDER BY updated_at DESC LIMIT ?",
                (f"%{t}%", limit),
            ).fetchall():
                found.add(r["id"])
        if found:
            ph = ",".join("?" * len(found))
            rows = conn.execute(
                f"SELECT * FROM facts WHERE id IN ({ph}) ORDER BY updated_at DESC LIMIT ?",
                (*found, limit),
            ).fetchall()
    _emit(rows, title=f"Search: {query}")
    return 0


def cmd_list(args, conn) -> int:
    sql = "SELECT * FROM facts WHERE 1=1"
    params: list = []
    if args.tag:
        sql += " AND (',' || tags || ',') LIKE ?"
        params.append(f"%,{args.tag},%")
    if args.scope:
        sql += " AND scope = ?"
        params.append(args.scope)
    if args.important:
        sql += " AND important = 1"
    sql += " ORDER BY updated_at DESC LIMIT ?"
    params.append(args.limit)
    rows = conn.execute(sql, params).fetchall()
    _emit(rows, title="All facts")
    return 0


def cmd_get(args, conn) -> int:
    row = conn.execute("SELECT * FROM facts WHERE id=?", (args.id,)).fetchone()
    if not row:
        print(f"ERR: no fact #{args.id}", file=sys.stderr)
        return 1
    _emit([row], title=f"Fact #{args.id}")
    return 0


def cmd_forget(args, conn) -> int:
    cur = conn.execute("DELETE FROM facts WHERE id=?", (args.id,))
    conn.commit()
    if cur.rowcount:
        print(f"memory: forgot #{args.id}")
    else:
        print(f"ERR: no fact #{args.id}", file=sys.stderr)
        return 1
    return 0


def cmd_forget_query(args, conn) -> int:
    terms = _fts_terms(" ".join(args.query))
    if not terms:
        print("ERR: no terms", file=sys.stderr)
        return 2
    match = " AND ".join(terms)
    rows = conn.execute(
        "SELECT rowid FROM facts_fts WHERE facts_fts MATCH ?", (match,)
    ).fetchall()
    if not rows:
        print("memory: nothing matched")
        return 0
    ids = [r["rowid"] for r in rows]
    placeholders = ",".join("?" * len(ids))
    conn.execute(f"DELETE FROM facts WHERE id IN ({placeholders})", ids)
    conn.commit()
    print(f"memory: forgot {len(ids)} fact(s) matching: {' '.join(args.query)}")
    return 0


def cmd_recall(args, conn) -> int:
    """Контекстный снапшот для системного промпта — самый свежий и важный."""
    rows = conn.execute(
        "SELECT * FROM facts ORDER BY important DESC, updated_at DESC LIMIT ?",
        (args.limit,),
    ).fetchall()
    if not rows:
        print("memory: (empty — no long-term facts stored yet)")
        return 0
    print(f"[memory recall: newest {len(rows)} fact(s)]")
    _emit(rows, title=None)
    return 0


def cmd_stats(args, conn) -> int:
    n = conn.execute("SELECT COUNT(*) FROM facts").fetchone()[0]
    by_scope = conn.execute(
        "SELECT scope, COUNT(*) c FROM facts GROUP BY scope"
    ).fetchall()
    print(f"facts: {n} total  db={DB_PATH}")
    for r in by_scope:
        print(f"  - {r['scope']}: {r['c']}")
    return 0


def _emit(rows, title: str | None) -> None:
    if title:
        print(f"[{title}]")
    if not rows:
        print("(no facts)")
        return
    for r in rows:
        flags = []
        if r["important"]:
            flags.append("★")
        if r["tags"]:
            flags.append(f"@{r['tags']}")
        meta = f" #{r['id']} [{r['scope']}] {r['updated_at'][:10]}"
        if r["source"]:
            meta += f" src={r['source'][:40]}"
        print(f"- {r['fact']}{meta} {' '.join(flags)}".strip())


def cmd_history(args, conn) -> int:
    f = conn.execute("SELECT * FROM facts WHERE id=?", (args.id,)).fetchone()
    if not f:
        print(f"ERR: no fact #{args.id}", file=sys.stderr)
        return 1
    print(f"fact #{args.id} current: {f['fact']}")
    rows = conn.execute(
        "SELECT * FROM fact_history WHERE fact_id=? ORDER BY changed_at DESC", (args.id,)
    ).fetchall()
    if not rows:
        print("  (no version history)")
        return 0
    print(f"  {len(rows)} previous version(s):")
    for r in rows:
        print(f"  - [{r['changed_at'][:19]}] {r['snapshot']}")
    return 0


def cmd_expired(args, conn) -> int:
    now = _now()
    rows = conn.execute(
        "SELECT * FROM facts WHERE valid_until IS NOT NULL AND valid_until <= ? ORDER BY valid_until",
        (now,),
    ).fetchall()
    if not rows:
        print("(no expired facts)")
        return 0
    print(f"[expired as of {now[:10]}]")
    for r in rows:
        print(f"- #{r['id']} {r['fact']} until {r['valid_until'][:10]}")
    return 0


def cmd_valid_at(args, conn) -> int:
    when = args.when or _now()[:10]
    # факт валиден в момент T (сравнение по дате YYYY-MM-DD)
    rows = conn.execute(
        "SELECT * FROM facts WHERE (substr(valid_from,1,10) IS NULL OR substr(valid_from,1,10) <= ?) "
        "AND (valid_until IS NULL OR substr(valid_until,1,10) >= ?) "
        "ORDER BY important DESC, updated_at DESC",
        (when, when),
    ).fetchall()
    if not rows:
        print(f"(no facts valid at {when})")
        return 0
    print(f"[facts valid at {when}]")
    for r in rows:
        vf = (r["valid_from"] or "∞")[:10]
        vu = (r["valid_until"] or "now")[:10]
        print(f"- #{r['id']} [{vf}→{vu}] {r['fact']}")
    return 0


def main() -> int:
    ap = argparse.ArgumentParser(description="Hermes local long-term memory (SQLite+FTS)")
    sub = ap.add_subparsers(dest="cmd", required=True)

    p = sub.add_parser("remember", help="store a durable fact")
    p.add_argument("fact", nargs="+")
    p.add_argument("--tag", action="append", default=[])
    p.add_argument("--scope", choices=["user", "agent", "session"], default="agent")
    p.add_argument("--important", action="store_true")
    p.add_argument("--source", default="")
    p.add_argument("--valid-from", default="")
    p.add_argument("--valid-until", default="")

    p = sub.add_parser("search", help="full-text search facts")
    p.add_argument("query", nargs="+")
    p.add_argument("--limit", type=int, default=10)
    p.add_argument("--scope", default="")

    p = sub.add_parser("list", help="list facts")
    p.add_argument("--tag", default="")
    p.add_argument("--scope", default="")
    p.add_argument("--important", action="store_true")
    p.add_argument("--limit", type=int, default=20)

    p = sub.add_parser("get", help="show one fact")
    p.add_argument("id", type=int)

    p = sub.add_parser("forget", help="delete one fact by id")
    p.add_argument("id", type=int)

    p = sub.add_parser("forget-query", help="delete facts matching text")
    p.add_argument("query", nargs="+")

    p = sub.add_parser("recall", help="contextual snapshot for prompt")
    p.add_argument("--limit", type=int, default=15)

    p = sub.add_parser("stats", help="summary")

    p = sub.add_parser("history", help="show version history of a fact")
    p.add_argument("id", type=int)

    p = sub.add_parser("expired", help="list facts that are currently past their valid_until")

    p = sub.add_parser("valid-at", help="list facts valid at a given time (YYYY-MM-DD)")
    p.add_argument("when", default="")

    args = ap.parse_args()
    conn = _db()
    try:
        return {
            "remember": cmd_remember,
            "search": cmd_search,
            "list": cmd_list,
            "get": cmd_get,
            "forget": cmd_forget,
            "forget-query": cmd_forget_query,
            "recall": cmd_recall,
            "stats": cmd_stats,
            "history": cmd_history,
            "expired": cmd_expired,
            "valid-at": cmd_valid_at,
        }[args.cmd](args, conn)
    finally:
        conn.close()


if __name__ == "__main__":
    sys.exit(main())
```

### `hermes-tracer.py` — Наблюдаемость (htrace) — traces/observations

```python
#!/usr/bin/env python3
"""
hermes-tracer.py — локальная наблюдаемость (observability) для Hermes Agent.

По мотивам Langfuse / Arize Phoenix / LangSmith (без облака, без OTEL):
троут Trace -> Observations (дерево). Диспетчер для отладки и мониторинга
агентских сессий: логирует вызовы LLM, инструментов и событий.

Команды (вызывается агентом через терминал):
  start   <name> [--trace TRACE] [--parent ID] [--type llm|tool|span|event]
          [--session S] [--project P] [--input JSON] [--meta "k=v,k2=v2"]
          -> печатает observation_id (и trace_id, если новый)
  end     <id> [--output JSON] [--tokens-in N] [--tokens-out N] [--model M]
          [--cost X] [--error "msg"] [--level info|warning|error]
  tree    [--trace TRACE] [--last N]      # дерево наблюдений
  traces  [--last N] [--project P] [--errors]   # список трасс
  stats   [--project P]                    # сводка
  tail    [--last N] [--type T] [--errors] # последние наблюдения (плоско)
"""
from __future__ import annotations

import argparse
import json
import os
import sqlite3
import sys
import uuid
from datetime import datetime, timezone

DB_PATH = os.environ.get("HERMES_TRACE_DB", "/home/kkutuzov/.hermes-trace.db")


def _now_ms() -> int:
    return int(datetime.now(timezone.utc).timestamp() * 1000)


def _db() -> sqlite3.Connection:
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA journal_mode=WAL;")
    conn.execute("PRAGMA synchronous=NORMAL;")
    _ensure_schema(conn)
    return conn


def _ensure_schema(conn: sqlite3.Connection) -> None:
    conn.execute(
        """
        CREATE TABLE IF NOT EXISTS traces (
            trace_id   TEXT PRIMARY KEY,
            project    TEXT NOT NULL DEFAULT 'hermes',
            session_id TEXT,
            user_id    TEXT,
            name       TEXT NOT NULL DEFAULT 'chat',
            status     TEXT NOT NULL DEFAULT 'ok',
            start_time INTEGER,
            end_time   INTEGER,
            input      TEXT,
            output     TEXT,
            metadata   TEXT,
            created_at INTEGER
        );
        """
    )
    conn.execute(
        """
        CREATE TABLE IF NOT EXISTS observations (
            id         TEXT PRIMARY KEY,
            trace_id   TEXT NOT NULL,
            parent_id  TEXT,
            type       TEXT NOT NULL DEFAULT 'span',   -- llm | tool | span | event
            name       TEXT,
            level      TEXT NOT NULL DEFAULT 'info',   -- info | warning | error
            input      TEXT,
            output     TEXT,
            tokens_in  INTEGER DEFAULT 0,
            tokens_out INTEGER DEFAULT 0,
            tokens_total INTEGER DEFAULT 0,
            cost       REAL DEFAULT 0,
            model      TEXT,
            error      TEXT,
            start_time INTEGER,
            end_time   INTEGER,
            latency_ms INTEGER,
            metadata   TEXT
        );
        """
    )
    conn.execute(
        "CREATE INDEX IF NOT EXISTS idx_obs_trace ON observations(trace_id)"
    )
    conn.execute(
        "CREATE INDEX IF NOT EXISTS idx_obs_parent ON observations(parent_id)"
    )
    conn.execute(
        "CREATE INDEX IF NOT EXISTS idx_tr_start ON traces(start_time)"
    )
    conn.commit()


def _new_id() -> str:
    return uuid.uuid4().hex[:16]


def _parse_meta(items) -> dict:
    d = {}
    for kv in items or []:
        if "=" in kv:
            k, v = kv.split("=", 1)
            d[k] = v
    return d


def cmd_start(args, conn) -> int:
    start = _now_ms()
    # новый трейс если не указан
    if args.trace:
        trace_id = args.trace
        t = conn.execute("SELECT trace_id FROM traces WHERE trace_id=?", (trace_id,)).fetchone()
        if t is None:
            conn.execute(
                "INSERT INTO traces (trace_id, project, session_id, name, status, start_time, created_at) "
                "VALUES (?, ?, ?, ?, 'ok', ?, ?)",
                (trace_id, args.project, args.session, args.name, start, start),
            )
    else:
        trace_id = _new_id()
        conn.execute(
            "INSERT INTO traces (trace_id, project, session_id, name, status, start_time, created_at) "
            "VALUES (?, ?, ?, ?, 'ok', ?, ?)",
            (trace_id, args.project, args.session, args.name, start, start),
        )
    obs_id = _new_id()
    meta = json.dumps(_parse_meta(args.meta), ensure_ascii=False)
    conn.execute(
        "INSERT INTO observations (id, trace_id, parent_id, type, name, level, input, "
        "start_time, metadata) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)",
        (obs_id, trace_id, args.parent, args.type, args.name, "info", args.input, start, meta),
    )
    conn.commit()
    print(f"trace={trace_id} observation={obs_id}")
    return 0


def cmd_end(args, conn) -> int:
    end = _now_ms()
    row = conn.execute("SELECT * FROM observations WHERE id=?", (args.id,)).fetchone()
    if row is None:
        print(f"ERR: no observation {args.id}", file=sys.stderr)
        return 1
    start = row["start_time"] or end
    latency = end - start
    tokens_total = (args.tokens_in or 0) + (args.tokens_out or 0)
    conn.execute(
        "UPDATE observations SET output=?, tokens_in=?, tokens_out=?, tokens_total=?, "
        "cost=?, model=?, error=?, end_time=?, latency_ms=?, level=? WHERE id=?",
        (
            args.output,
            args.tokens_in or 0,
            args.tokens_out or 0,
            tokens_total,
            args.cost or 0,
            args.model or row["model"],
            args.error,
            end,
            latency,
            args.level,
            args.id,
        ),
    )
    # если ошибка — пометить трейс
    if args.error:
        conn.execute(
            "UPDATE traces SET status='error' WHERE trace_id=?",
            (row["trace_id"],),
        )
    conn.commit()
    print(f"observation {args.id} closed: latency={latency}ms tokens={tokens_total}")
    return 0


def cmd_tree(args, conn) -> int:
    if args.trace:
        obs = conn.execute(
            "SELECT * FROM observations WHERE trace_id=? ORDER BY start_time",
            (args.trace,),
        ).fetchall()
        title = f"Trace {args.trace}"
        if not obs:
            t = conn.execute("SELECT * FROM traces WHERE trace_id=?", (args.trace,)).fetchone()
            if t:
                print(f"[{title}] (no observations, trace only)")
                print(
                    f"  {t['name']} {t['project']} {t['status']} "
                    f"{t.get('start_time')}-{t.get('end_time')}"
                )
                return 0
            print(f"[{title}] not found"); return 1
    else:
        last = args.last
        obs = conn.execute(
            "SELECT * FROM observations ORDER BY start_time DESC LIMIT ?",
            (last * 20,),
        ).fetchall()
        obs = obs[::-1]
        title = f"Last {last} root observations"
    print(f"[{title}]")
    roots = [o for o in obs if not o["parent_id"]]
    by_parent: dict = {}
    for o in obs:
        by_parent.setdefault(o["parent_id"], []).append(o)
    for r in roots[: args.last]:
        _print_obs(r, 0)
        for ch in by_parent.get(r["id"], []):
            _print_obs(ch, 2)
    return 0


def _print_obs(o, indent: int) -> None:
    pad = " " * indent
    dur = f" {o['latency_ms']}ms" if o["latency_ms"] else ""
    tok = f" t{o['tokens_total']}" if o["tokens_total"] else ""
    lvl = f" [{o['level']}]" if o["level"] != "info" else ""
    err = f" !{o['error'][:60]}" if o["error"] else ""
    print(f"{pad}- {o['type']}:{o['name']}{lvl}{dur}{tok}{err}")


def cmd_traces(args, conn) -> int:
    sql = "SELECT * FROM traces WHERE 1=1"
    params: list = []
    if args.project:
        sql += " AND project=?"
        params.append(args.project)
    if args.errors:
        sql += " AND status='error'"
    sql += " ORDER BY start_time DESC LIMIT ?"
    params.append(args.last)
    rows = conn.execute(sql, params).fetchall()
    if not rows:
        print("(no traces)")
        return 0
    print(f"[traces, last {len(rows)}]")
    for r in rows:
        n = conn.execute(
            "SELECT COUNT(*) c, SUM(tokens_total) t FROM observations WHERE trace_id=?",
            (r["trace_id"],),
        ).fetchone()
        print(
            f"- {r['trace_id'][:12]} {r['project']:<8} {r['name'][:20]} {r['status']:<6} "
            f"obs={n['c']} tok={n['t']} s={r['session_id'] or '-'}"
        )
    return 0


def cmd_stats(args, conn) -> int:
    params: list = []
    conds: list = []
    if args.project:
        conds.append("project=?")
        params.append(args.project)
    where = (" WHERE " + " AND ".join(conds)) if conds else ""

    def _count(col: str) -> int:
        sql = f"SELECT COUNT(*) FROM observations {where}"
        if conds:
            sql += " AND " + col
        else:
            sql += " WHERE " + col
        return conn.execute(sql, params if conds else []).fetchone()[0]

    nt = conn.execute(f"SELECT COUNT(*) FROM traces {where}", params).fetchone()[0]
    no = conn.execute(f"SELECT COUNT(*) FROM observations {where}", params).fetchone()[0]
    ne = _count("level='error'")
    nllm = _count("type='llm'")
    ntool = _count("type='tool'")
    tokt = conn.execute(
        f"SELECT COALESCE(SUM(tokens_total),0) FROM observations {where}", params,
    ).fetchone()[0]
    print(f"traces={nt} observations={no} (llm={nllm} tool={ntool}) errors={ne} tokens={tokt}")

    projs = conn.execute(
        "SELECT project, COUNT(*) c FROM traces GROUP BY project ORDER BY c DESC"
    ).fetchall()
    for p in projs:
        print(f"  - {p['project']}: {p['c']}")
    return 0


def cmd_tail(args, conn) -> int:
    sql = "SELECT * FROM observations WHERE 1=1"
    params: list = []
    if args.type:
        sql += " AND type=?"
        params.append(args.type)
    if args.errors:
        sql += " AND (level='error' OR error IS NOT NULL)"
    sql += " ORDER BY start_time DESC LIMIT ?"
    params.append(args.last)
    rows = conn.execute(sql, params).fetchall()
    if not rows:
        print("(no observations)")
        return 0
    print(f"[last {len(rows)} observations]")
    for o in rows:
        print(
            f"- {o['start_time'] and _ts(o['start_time'])} {o['type']}:{o['name']} "
            f"trace={o['trace_id'][:12]} lvl={o['level']}"
        )


def _ts(ms: int) -> str:
    try:
        return datetime.fromtimestamp(ms / 1000).strftime("%H:%M:%S")
    except Exception:
        return str(ms)


def main() -> int:
    ap = argparse.ArgumentParser(description="Hermes local observability (traces/observations)")
    sub = ap.add_subparsers(dest="cmd", required=True)

    p = sub.add_parser("start", help="open an observation (and maybe a trace)")
    p.add_argument("name")
    p.add_argument("--trace", default="")
    p.add_argument("--parent", default="")
    p.add_argument("--type", choices=["llm", "tool", "span", "event"], default="span")
    p.add_argument("--session", default="")
    p.add_argument("--project", default="hermes")
    p.add_argument("--input", default="")
    p.add_argument("--meta", action="append", default=[])

    p = sub.add_parser("end", help="close an observation with result")
    p.add_argument("id")
    p.add_argument("--output", default="")
    p.add_argument("--tokens-in", type=int, default=0)
    p.add_argument("--tokens-out", type=int, default=0)
    p.add_argument("--model", default="")
    p.add_argument("--cost", type=float, default=0)
    p.add_argument("--error", default="")
    p.add_argument("--level", choices=["info", "warning", "error"], default="info")

    p = sub.add_parser("tree", help="show observation tree")
    p.add_argument("--trace", default="")
    p.add_argument("--last", type=int, default=10)

    p = sub.add_parser("traces", help="list traces")
    p.add_argument("--last", type=int, default=15)
    p.add_argument("--project", default="")
    p.add_argument("--errors", action="store_true")

    p = sub.add_parser("stats")
    p.add_argument("--project", default="")

    p = sub.add_parser("tail")
    p.add_argument("--last", type=int, default=15)
    p.add_argument("--type", default="")
    p.add_argument("--errors", action="store_true")

    args = ap.parse_args()
    conn = _db()
    try:
        return {
            "start": cmd_start,
            "end": cmd_end,
            "tree": cmd_tree,
            "traces": cmd_traces,
            "stats": cmd_stats,
            "tail": cmd_tail,
        }[args.cmd](args, conn)
    finally:
        conn.close()


if __name__ == "__main__":
    sys.exit(main())
```

### `hermes-search.py` — Поиск и извлечение (hsearch) — DDG-lite + webread

```python
#!/usr/bin/env python3
"""
hermes-search.py — локальный поиск и извлечение контента для Hermes Agent.

По мотивам Tavily / Exa / Firecrawl (search + extract), БЕЗ облачных ключей.
- search: веб-поиск через lite.duckduckgo.com (работает без ключа, HTTP 200),
  с возможностью подключить self-hosted SearXNG (search_api: url).
- extract: извлечение текста страницы в Markdown через curl (webread.py).

Команды:
  search  <query> [--max N] [--include-answer] [--depth basic|advanced]
          [--sites "site1.com,site2.com"]
  extract <url> [--max-chars N] [--markdown]
  sources                 # какие бекенды поиска доступны
"""
from __future__ import annotations

import argparse
import html
import json
import os
import re
import subprocess
import sys
import urllib.parse

UA = (
    "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 "
    "(KHTML, like Gecko) Chrome/126.0 Safari/537.36"
)

SEARXNG_URL = os.environ.get("HERMES_SEARXNG", "")  # напр. http://127.0.0.1:8888


# ---------- search backends ----------

def _search_ddg(query: str, max_results: int, sites: list[str]) -> list[dict]:
    """lite.duckduckgo.com scraping via urllib (no key)."""
    import urllib.parse
    import urllib.request

    if sites:
        query = " ".join([f"site:{s}" for s in sites]) + " " + query
    data = urllib.parse.urlencode({"q": query}).encode()
    req = urllib.request.Request(
        "https://lite.duckduckgo.com/lite/",
        data=data,
        headers={
            "User-Agent": UA,
            "Content-Type": "application/x-www-form-urlencoded",
        },
    )
    with urllib.request.urlopen(req, timeout=25) as resp:
        htmlsrc = resp.read().decode("utf-8", "ignore")
    # результат: <a rel="nofollow" href="URL" class='result-link'>TITLE</a>
    links = re.findall(
        r'<a[^>]*href="([^"]+)"[^>]*class=[\'"]result-link[\'"][^>]*>(.*?)</a>',
        htmlsrc, re.S | re.I,
    )
    results = []
    for href, title in links[:max_results]:
        href = href.strip()
        if not href.startswith("http"):
            continue
        title_t = re.sub(r"<[^>]+>", "", title)
        title_t = html.unescape(title_t).strip()
        results.append({"title": title_t, "url": href, "content": "", "score": None})
    return results


def _search_searxng(query: str, max_results: int, sites: list[str]) -> list[dict]:
    """Self-hosted SearXNG JSON search."""
    if not SEARXNG_URL:
        raise RuntimeError("HERMES_SEARXNG not set")
    if sites:
        query = " ".join([f"site:{s}" for s in sites]) + " " + query
    url = SEARXNG_URL.rstrip("/") + "/?format=json&q=" + urllib.parse.quote(query)
    p = subprocess.run(
        ["curl", "-s", "--max-time", "25", "-A", UA, url],
        capture_output=True, timeout=35,
    )
    data = json.loads(p.stdout.decode("utf-8", errors="ignore") or "{}")
    res = []
    for r in (data.get("results") or [])[:max_results]:
        res.append({
            "title": r.get("title", ""),
            "url": r.get("url", ""),
            "content": r.get("content", ""),
            "score": r.get("score"),
            "engine": r.get("engine"),
        })
    return res


# ---------- extract ----------

def extract_url(url: str, max_chars: int, markdown: bool) -> str:
    """Извлечь текст страницы через webread.py (curl + парсинг)."""
    args = ["python3", "/home/kkutuzov/webread.py", url, "--max-chars", str(max_chars)]
    if markdown:
        args.append("--markdown")
    p = subprocess.run(args, capture_output=True, text=True, timeout=60)
    if p.returncode != 0:
        return p.stderr.strip() or "WEBREAD_ERROR"
    return p.stdout.strip()


# ---------- cli ----------

def cmd_search(args) -> int:
    max_n = args.max
    sites = [s.strip() for s in (args.sites or "").split(",") if s.strip()]
    errors = []
    results = []
    # try searxng first if configured, then ddg
    if SEARXNG_URL:
        try:
            results = _search_searxng(args.query, max_n, sites)
        except Exception as e:  # noqa
            errors.append(f"searxng: {e}")
    if not results:
        try:
            results = _search_ddg(args.query, max_n, sites)
        except Exception as e:  # noqa
            errors.append(f"ddg: {e}")
    if not results:
        print("search: no results")
        for e in errors:
            print(f"  ! {e}", file=sys.stderr)
        return 1
    print(f"[search: {args.query}] ({len(results)} results)")
    for i, r in enumerate(results, 1):
        print(f"{i}. {r['title']}\n   {r['url']}\n   {r['content'][:200]}\n")
    return 0


def cmd_extract(args) -> int:
    text = extract_url(args.url, args.max_chars, args.markdown)
    print(text)
    return 0


def cmd_sources(args) -> int:
    print("search engines:")
    print(f"  - duckduckgo lite   (no key, always available)")
    if SEARXNG_URL:
        print(f"  - searxng           ({SEARXNG_URL})")
    else:
        print(f"  - searxng           (not configured — set HERMES_SEARXNG)")
    print("extract:")
    print("  - webread.py        (curl + HTML->text/Markdown)")
    return 0


def main() -> int:
    ap = argparse.ArgumentParser(description="Hermes local web search + extract")
    sub = ap.add_subparsers(dest="cmd", required=True)

    p = sub.add_parser("search")
    p.add_argument("query", nargs="+")
    p.add_argument("--max", type=int, default=6)
    p.add_argument("--include-answer", action="store_true")
    p.add_argument("--depth", choices=["basic", "advanced"], default="basic")
    p.add_argument("--sites", default="")

    p = sub.add_parser("extract")
    p.add_argument("url")
    p.add_argument("--max-chars", type=int, default=12000)
    p.add_argument("--markdown", action="store_true")

    p = sub.add_parser("sources")

    args = ap.parse_args()
    if args.cmd == "search":
        args.query = " ".join(args.query)
        return cmd_search(args)
    if args.cmd == "extract":
        return cmd_extract(args)
    if args.cmd == "sources":
        return cmd_sources(args)
    return 2


if __name__ == "__main__":
    sys.exit(main())
```

### `hermes-actions.py` — Интеграции/действия (hact) — реестр инструментов

```python
#!/usr/bin/env python3
"""
hermes-actions.py — локальный registrar интеграций/действий для Hermes Agent.

По мотивам Composio (toolkits + actions + connected accounts) и MCP
(name/schema/handler) — но без облака. Действия объявляются в JSON-схеме
(как MCP inputSchema), диспетчер выполняет handler, а "connected accounts"
эмулируются локальными ящиками/репозиториями в ~/.hermes-actions/.

Действия регистрируются в SQLite (name, description, input_schema, handler). Handler —
это функция Python, реализующая действие локально или через вызов CLI.

Команды (Hermes вызывает через терминал):
  list                          # все доступные инструменты (name + desc)
  show <name>                   # JSON-schema инструмента
  call <name> --arg key=value --arg k2=v2   # выполнить действие
  status                        # текущие "подключенные аккаунты" (локальные ящики)
  init                          # создать базовые демо-инструменты
"""
from __future__ import annotations

import argparse
import json
import os
import sqlite3
import sys
from datetime import datetime

DB_PATH = os.environ.get("HERMES_ACTIONS_DB", "/home/kkutuzov/.hermes-actions.db")
ACTIONS_DIR = os.environ.get("HERMES_ACTIONS_DIR", "/home/kkutuzov/.hermes-actions")


def _db() -> sqlite3.Connection:
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA journal_mode=WAL;")
    _ensure_schema(conn)
    return conn


def _ensure_schema(conn: sqlite3.Connection) -> None:
    conn.execute(
        """
        CREATE TABLE IF NOT EXISTS actions (
            name TEXT PRIMARY KEY,
            toolkit TEXT NOT NULL DEFAULT 'local',
            description TEXT NOT NULL DEFAULT '',
            input_schema TEXT NOT NULL DEFAULT '{}',
            output_schema TEXT NOT NULL DEFAULT '{}',
            handler TEXT NOT NULL DEFAULT '',
            enabled INTEGER NOT NULL DEFAULT 1
        );
        """
    )
    conn.execute(
        """
        CREATE TABLE IF NOT EXISTS accounts (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            toolkit TEXT NOT NULL,
            account TEXT NOT NULL,        -- напр. 'gmail-main', 'github-org'
            state TEXT NOT NULL DEFAULT '{}',  -- json, локальный "connected account"
            created_at TEXT,
            UNIQUE(toolkit, account)
        );
        """
    )
    conn.commit()


def _now() -> str:
    return datetime.now().strftime("%Y-%m-%dT%H:%M:%S")


# ---------------------------------------------------------------------------
# Handlers — локальная эмуляция интеграций (без внешних API)
# ---------------------------------------------------------------------------

def _h_git_status(args: dict, acct: dict | None) -> dict:
    # эмуляция GITHUB_GET_STATUS на локальном репо
    repo = args.get("repo") or os.getcwd()
    import subprocess
    r = subprocess.run(
        ["git", "status", "--short"], cwd=repo, capture_output=True, text=True, timeout=15,
    )
    if r.returncode != 0:
        return {"ok": False, "error": r.stderr.strip() or "not a git repo", "repo": repo}
    return {"ok": True, "repo": repo, "changes": r.stdout.splitlines() or []}


def _h_notes_add(args: dict, acct: dict | None) -> dict:
    """GMAIL_SEND / local notes: добавить запись в локальный ящик/заметку."""
    os.makedirs(ACTIONS_DIR, exist_ok=True)
    account = args.get("account") or "default"
    subject = args.get("subject") or args.get("title") or "note"
    body = args.get("body") or args.get("text") or ""
    path = os.path.join(ACTIONS_DIR, f"{account}.md")
    with open(path, "a", encoding="utf-8") as f:
        f.write(f"## [{_now()}] {subject}\n{body}\n\n")
    return {"ok": True, "stored_in": path, "subject": subject}


def _h_file_append(args: dict, acct: dict | None) -> dict:
    path = args.get("path")
    text = args.get("text", "")
    if not path:
        return {"ok": False, "error": "path required"}
    with open(path, "a", encoding="utf-8") as f:
        f.write(text + "\n")
    return {"ok": True, "path": path, "appended": len(text)}


def _h_file_read(args: dict, acct: dict | None) -> dict:
    path = args.get("path")
    if not path or not os.path.exists(path):
        return {"ok": False, "error": f"no such file: {path}"}
    with open(path, "r", encoding="utf-8", errors="ignore") as f:
        content = f.read()
    return {"ok": True, "path": path, "content": content[:2000]}


def _h_slack_msg(args: dict, acct: dict | None) -> dict:
    """SLACK_SEND / local: записать в ~/.hermes-actions/slack.log (эмуляция канала)."""
    os.makedirs(ACTIONS_DIR, exist_ok=True)
    channel = args.get("channel") or "general"
    text = args.get("text", "")
    log = os.path.join(ACTIONS_DIR, "slack.log")
    with open(log, "a", encoding="utf-8") as f:
        f.write(f"[{_now()}] #{channel}: {text}\n")
    return {"ok": True, "logged_to": log, "channel": channel, "text": text}


HANDLERS = {
    "git_status": _h_git_status,
    "notes_add": _h_notes_add,
    "file_append": _h_file_append,
    "file_read": _h_file_read,
    "slack_msg": _h_slack_msg,
}

# Базовые demo-инструменты (инициализация)
BASE_ACTIONS = [
    {
        "name": "github_status",
        "toolkit": "github",
        "description": "Показать статус локального git-репозитория (аналог GITHUB_GET_STATUS)",
        "input_schema": {"type": "object", "properties": {"repo": {"type": "string"}}},
        "handler": "git_status",
    },
    {
        "name": "gmail_send",
        "toolkit": "gmail",
        "description": "Сохранить сообщение в локальный ящик/заметки (аналог GMAIL_SEND)",
        "input_schema": {"type": "object", "properties": {
            "account": {"type": "string"}, "subject": {"type": "string"}, "body": {"type": "string"},
        }, "required": ["subject", "body"]},
        "handler": "notes_add",
    },
    {
        "name": "file_append",
        "toolkit": "filesystem",
        "description": "Добавить текст в конец файла",
        "input_schema": {"type": "object", "properties": {
            "path": {"type": "string"}, "text": {"type": "string"},
        }, "required": ["path", "text"]},
        "handler": "file_append",
    },
    {
        "name": "file_read",
        "toolkit": "filesystem",
        "description": "Прочитать файл (до 2000 символов)",
        "input_schema": {"type": "object", "properties": {"path": {"type": "string"}},
                         "required": ["path"]},
        "handler": "file_read",
    },
    {
        "name": "slack_send",
        "toolkit": "slack",
        "description": "Записать сообщение в локальный slack.log (аналог SLACK_POST_MESSAGE)",
        "input_schema": {"type": "object", "properties": {
            "channel": {"type": "string"}, "text": {"type": "string"},
        }, "required": ["text"]},
        "handler": "slack_msg",
    },
]


def cmd_init(args, conn) -> int:
    for a in BASE_ACTIONS:
        conn.execute(
            "INSERT OR IGNORE INTO actions (name, toolkit, description, input_schema, "
            "output_schema, handler, enabled) VALUES (?, ?, ?, ?, ?, ?, 1)",
            (a["name"], a["toolkit"], a["description"],
             json.dumps(a["input_schema"]), "{}", a["handler"]),
        )
    os.makedirs(ACTIONS_DIR, exist_ok=True)
    conn.commit()
    n = conn.execute("SELECT COUNT(*) c FROM actions").fetchone()["c"]
    print(f"actions: initialized {n} tools in {ACTIONS_DIR}")
    return 0


def cmd_list(args, conn) -> int:
    rows = conn.execute(
        "SELECT name, toolkit, description, enabled FROM actions WHERE 1=1"
        + ("" if not args.toolkit else " AND toolkit=?")
        + " ORDER BY toolkit, name",
        ([args.toolkit] if args.toolkit else []),
    ).fetchall() if args.toolkit else conn.execute(
        "SELECT name, toolkit, description, enabled FROM actions ORDER BY toolkit, name"
    ).fetchall()
    if not rows:
        print("(no actions — run 'init' to create defaults)")
        return 0
    print(f"[actions: {len(rows)}]")
    for r in rows:
        flag = "" if r["enabled"] else " [disabled]"
        print(f"- {r['toolkit']}:{r['name']} — {r['description']}{flag}")
    return 0


def cmd_show(args, conn) -> int:
    r = conn.execute("SELECT * FROM actions WHERE name=?", (args.name,)).fetchone()
    if not r:
        print(f"ERR: no action '{args.name}'", file=sys.stderr)
        return 1
    print(json.dumps({
        "name": r["name"], "toolkit": r["toolkit"], "description": r["description"],
        "inputSchema": json.loads(r["input_schema"] or "{}"),
        "outputSchema": json.loads(r["output_schema"] or "{}"),
    }, ensure_ascii=False, indent=2))
    return 0


def cmd_call(args, conn) -> int:
    r = conn.execute("SELECT * FROM actions WHERE name=? AND enabled=1", (args.name,)).fetchone()
    if not r:
        print(f"ERR: no enabled action '{args.name}'", file=sys.stderr)
        return 1
    kwargs = {}
    for kv in args.arg or []:
        if "=" in kv:
            k, v = kv.split("=", 1)
            kwargs[k] = v
    handler = r["handler"]
    fn = HANDLERS.get(handler)
    if fn is None:
        print(f"ERR: handler '{handler}' not registered", file=sys.stderr)
        return 1
    try:
        out = fn(kwargs, None)
    except Exception as e:  # noqa
        out = {"ok": False, "error": str(e)}
    print(json.dumps(out, ensure_ascii=False, indent=2))
    return 0 if out.get("ok", True) else 1


def cmd_status(args, conn) -> int:
    os.makedirs(ACTIONS_DIR, exist_ok=True)
    files = sorted(os.listdir(ACTIONS_DIR)) if os.path.isdir(ACTIONS_DIR) else []
    print(f"actions dir: {ACTIONS_DIR}")
    print("connected accounts (local):")
    if files:
        for f in files:
            print(f"  - {f}")
    else:
        print("  (empty — actions write here)")
    return 0


def main() -> int:
    ap = argparse.ArgumentParser(description="Hermes local actions/integrations registrar")
    sub = ap.add_subparsers(dest="cmd", required=True)

    p = sub.add_parser("init")
    p = sub.add_parser("list")
    p.add_argument("--toolkit", default="")
    p = sub.add_parser("show")
    p.add_argument("name")
    p = sub.add_parser("call")
    p.add_argument("name")
    p.add_argument("--arg", action="append", default=[])
    p = sub.add_parser("status")

    args = ap.parse_args()
    conn = _db()
    try:
        return {
            "init": cmd_init, "list": cmd_list, "show": cmd_show,
            "call": cmd_call, "status": cmd_status,
        }[args.cmd](args, conn)
    finally:
        conn.close()


if __name__ == "__main__":
    sys.exit(main())
```

### `webread.py` — Чтение веб-страниц (webread) — curl + HTML в текст

```python
#!/usr/bin/env python3
"""Read a web page as readable text using curl (bypasses Hermes cloud extract APIs).

Usage: python3 /home/kkutuzov/webread.py URL [--max-chars N]
Fetches the page with a real browser User-Agent, strips scripts/styles/tags,
normalizes whitespace, and prints the readable text to stdout.

Exit codes: 0 ok, 1 fetch failed, 2 parse/other error.
"""
import argparse
import html
import re
import subprocess
import sys


UA = (
    "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 "
    "(KHTML, like Gecko) Chrome/126.0 Safari/537.36"
)


def fetch(url: str, timeout: int = 30) -> bytes:
    # -L follow redirects, --max-filesize guard, --compressed
    cmd = [
        "curl", "-s", "-L", "--compressed",
        "--max-time", str(timeout),
        "-A", UA,
        url,
    ]
    p = subprocess.run(cmd, capture_output=True, timeout=timeout + 10)
    if p.returncode != 0:
        raise RuntimeError(f"curl exited {p.returncode}: {p.stderr.decode(errors='ignore')[:300]}")
    return p.stdout


def to_text(raw: bytes, max_chars: int) -> str:
    # Try utf-8, fall back to windows-1251 (common for legacy RU sites)
    for enc in ("utf-8", "windows-1251", "cp866", "koi8-r"):
        try:
            s = raw.decode(enc)
            break
        except (UnicodeDecodeError, LookupError):
            continue
    else:
        s = raw.decode("utf-8", errors="ignore")

    # Remove script/style/head
    s = re.sub(r"<(script|style|noscript|head|header|footer|nav)[^>]*>.*?</\1>", " ", s, flags=re.S | re.I)
    # Remove comments
    s = re.sub(r"<!--.*?-->", " ", s, flags=re.S)
    # Tag -> space
    s = re.sub(r"<[^>]+>", " ", s)
    # Decode entities
    s = html.unescape(s)
    # Collapse whitespace
    s = re.sub(r"[ \t]+", " ", s)
    s = re.sub(r"\n\s*\n+", "\n", s)
    s = s.strip()
    if not s:
        raise RuntimeError("no readable text extracted (page body empty)")
    return s[:max_chars] + ("…" if len(s) > max_chars else "")


def main() -> int:
    ap = argparse.ArgumentParser(description="Read a URL as text")
    ap.add_argument("url")
    ap.add_argument("--max-chars", type=int, default=12000)
    ap.add_argument("--timeout", type=int, default=30)
    args = ap.parse_args()

    if not re.match(r"^https?://", args.url):
        args.url = "https://" + args.url
    try:
        raw = fetch(args.url, args.timeout)
        text = to_text(raw, args.max_chars)
    except Exception as exc:  # noqa: BLE001
        print(f"WEBREAD_ERROR: {exc}", file=sys.stderr)
        return 1
    print(text)
    return 0


if __name__ == "__main__":
    sys.exit(main())
```
