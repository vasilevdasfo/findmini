# 🎙 L4 — Голос и Telegram-боты (за выходные)

> Цель недели: ты диктуешь голосовое в Telegram — через 20 секунд идея лежит текстом в твоём проекте.
> Это точка где Claude становится по-настоящему полезным: не надо печатать.

---

## 📦 Архитектура (чтобы понимать что к чему)

```
Твой голос в Telegram
       ↓ (Telegram Bot API)
Твой бот (Python)
       ↓ (ffmpeg: конвертация ogg → wav)
Groq Whisper (распознавание речи, бесплатно)
       ↓ (текст)
Anthropic Claude (разбор + категоризация)
       ↓
Файл в проекте: 3_ЗАДАЧИ/📥_INBOX.md
```

5 компонентов: бот + ffmpeg + Whisper + Claude + файловая система. Каждый — отдельно ставится и тестируется.

---

## 🔧 Шаг 1. Установить зависимости (20 мин)

### ffmpeg
Нужен чтобы конвертировать голосовые (Telegram отдаёт `.ogg` Opus) в формат для Whisper.

```bash
brew install ffmpeg
ffmpeg -version   # должен показать версию
```

На Windows: скачать с https://www.gyan.dev/ffmpeg/builds/, распаковать, добавить в PATH.

⚠️ **Важно:** на Mac не удаляй ffmpeg — боты Димы держат статический бинарник в `04_CYBOS/ffmpeg` как бекап, потому что brew иногда обновляется несовместимо.

### Python 3.11+
```bash
python3 --version   # должно быть 3.11 или выше
```

Если меньше:
```bash
brew install python@3.12
```

### Библиотеки
```bash
pip3 install python-telegram-bot groq anthropic
```

Если жалуется на system python — лучше поставь `pyenv` или виртуальное окружение.

---

## 🔑 Шаг 2. Получить ключи (15 мин)

### Groq (бесплатно, быстро)
1. https://console.groq.com → Sign up (Google OAuth)
2. API Keys → Create key → скопируй (начинается с `gsk_...`)
3. Модель которая тебе нужна: `whisper-large-v3` (распознаёт русский отлично)

### Anthropic API (платно, но копейки)
1. https://console.anthropic.com
2. API Keys → Create key (начинается с `sk-ant-...`)
3. Пополни баланс на $5 — хватит на месяц активного использования

Альтернатива: **Deepgram Nova-2** — платный, но с диаризацией (понимает кто говорит, если несколько людей в голосовом). Для одиночной диктовки Groq лучше — бесплатный.

### Telegram Bot
1. Открой @BotFather → `/newbot`
2. Имя: что угодно (напр. `Alex Voice Inbox`)
3. Username: должен кончаться на `_bot` (напр. `alex_voice_inbox_bot`)
4. Сохрани **токен** (формат `1234567890:AAH...`)

---

## 💾 Шаг 3. Хранение ключей

**НЕ в коде. НЕ в git.** Один из двух вариантов:

### Вариант А: macOS Keychain (безопаснее)
```bash
security add-generic-password -a "$USER" -s "groq_api" -w "gsk_..."
security add-generic-password -a "$USER" -s "anthropic_api" -w "sk-ant-..."
security add-generic-password -a "$USER" -s "tg_bot_voice" -w "1234567890:AAH..."
```

Достать в Python:
```python
import subprocess
def get_secret(name):
    return subprocess.check_output(
        ["security", "find-generic-password", "-s", name, "-w"]
    ).decode().strip()

GROQ_KEY = get_secret("groq_api")
```

### Вариант Б: `.env` файл (проще, но не в git!)
```
GROQ_API_KEY=gsk_...
ANTHROPIC_API_KEY=sk-ant-...
TG_BOT_TOKEN=1234567890:AAH...
```

В `.gitignore`:
```
.env
*.session
```

И в Python:
```python
from dotenv import load_dotenv
load_dotenv()
import os
GROQ_KEY = os.getenv("GROQ_API_KEY")
```

---

## 🤖 Шаг 4. Первый бот — голос → текст → файл (30 мин)

Попроси Claude Code:

```
Сделай Telegram-бота voice_inbox_bot.py по такому ТЗ:

1. Слушает мой Telegram-бот (токен в keychain service "tg_bot_voice")
2. Принимает голосовые сообщения
3. Скачивает .ogg, конвертирует через ffmpeg в wav 16kHz mono
4. Отправляет в Groq Whisper (whisper-large-v3), язык=ru
5. Получает транскрипт
6. Добавляет в начало файла ~/projects/inbox.md:
   ## YYYY-MM-DD HH:MM
   {transcript}
   ---
7. Отвечает в чат «✓ записано, X символов»

Используй python-telegram-bot, groq библиотеку.
Ключи через security find-generic-password.
Обработай ошибки: если Groq вернул пустое — написать «пусто, повтори громче».
```

Claude Code сгенерирует код. Прочитай его, не запускай слепо. Пойми что он делает.

Запусти:
```bash
python3 voice_inbox_bot.py
```

Открой свой бот в Telegram, пришли голосовое. Через ~15 сек должен прийти ответ «✓ записано».

---

## 🧠 Шаг 5. Добавить Claude для разбора (следующий день)

Расширяем бота: транскрипт идёт не сырым в inbox, а сначала к Claude, который:
- Выделяет сущности (клиент, проект, deadline)
- Категоризирует (задача / идея / встреча / рефлексия)
- Форматирует в markdown

ТЗ Claude Code:

```
Расширь voice_inbox_bot.py:

После получения транскрипта от Whisper — отправь его в Claude API
(модель claude-sonnet-4-7) с промптом:

«Ты разбираешь голосовую заметку. Верни JSON:
{
  "category": "task|idea|meeting|reflection",
  "title": "короткий заголовок 5-8 слов",
  "text": "очищенный текст без слов-паразитов",
  "entities": {"clients": [...], "projects": [...], "people": [...]},
  "deadline": "YYYY-MM-DD или null",
  "priority": "high|med|low"
}
ТЕКСТ: {transcript}»

Результат положи в файл по категории:
- task → 3_ЗАДАЧИ/📥_INBOX.md
- idea → 3_ЗАДАЧИ/💡_IDEAS.md
- meeting → 3_ЗАДАЧИ/📅_MEETINGS.md
- reflection → 3_ЗАДАЧИ/💭_REFLECTION.md

В ответе боту укажи: категория + заголовок + куда положил.
```

---

## 🚀 Шаг 6. Автозапуск (чтобы работало без терминала)

### macOS — launchd
Файл `~/Library/LaunchAgents/com.alex.voicebot.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.alex.voicebot</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/bin/python3</string>
    <string>/Users/YOU/projects/voice_inbox_bot.py</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key><string>/tmp/voicebot.log</string>
  <key>StandardErrorPath</key><string>/tmp/voicebot.err</string>
</dict>
</plist>
```

Загрузить:
```bash
launchctl load ~/Library/LaunchAgents/com.alex.voicebot.plist
```

Проверить что бот живой:
```bash
launchctl list | grep voicebot
tail -f /tmp/voicebot.log
```

На Linux — `systemd service`, на Windows — `NSSM` или Task Scheduler.

---

## ⚠️ Грабли на которые наступишь

1. **«Whisper вернул на английском»** — не указал `language="ru"` в Groq запросе.
2. **«ffmpeg not found»** — после brew install перезапусти терминал, или укажи полный путь `/opt/homebrew/bin/ffmpeg`.
3. **«Голосовое > 1 мин, бот молчит»** — Groq таймаут 30 сек по дефолту, поставь 120. Для длинных голосовых — режь на куски.
4. **«Дубликаты сообщений»** — два процесса бота одновременно. Убей старый через `ps aux | grep voice_inbox`.
5. **«Telegram rate limit»** — не шли больше 20 сообщений в минуту. Добавь `await asyncio.sleep(1)` между ответами.
6. **«Claude возвращает текст вместо JSON»** — используй `response_format={"type":"json_object"}` или парси markdown code block.

---

## 📋 Чек что L4 пройден

- [ ] ffmpeg, Python 3.11+, библиотеки стоят
- [ ] Ключи в Keychain (или `.env` + `.gitignore`)
- [ ] Голосовое → транскрипт работает (Groq Whisper)
- [ ] Транскрипт → разбор Claude → категоризация в файлы
- [ ] launchd автозапуск — бот переживает перезагрузку мака

Когда дойдёшь — продиктуй короткое голосовое в свой бот и скинь в эту группу скриншот результата. Поздравим 🎉

---

**Автор:** Дима (набил 4 бота, знает где больно) + Demi
