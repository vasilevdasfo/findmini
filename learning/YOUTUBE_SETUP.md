# 📺 YouTube-транскрипты как у Димы

> Кидаешь линк → через 15 секунд готовый summary + ключевые цитаты + задачи.
> Без этого Дима 5 раз в день копировал бы видео в OpenAI — с этим бот делает сам.

---

## 🧩 Архитектура (двухуровневая с fallback)

```
YouTube URL
    ↓
Шаг 1: Supadata API (платно, быстро)
    ├─ ОК? → возвращаем транскрипт
    └─ Упал/пусто? → Шаг 2
Шаг 2: youtube-transcript-api (бесплатно, Python)
    ├─ Есть субтитры? → берём их
    └─ Нет субтитров? → Шаг 3
Шаг 3: yt-dlp скачивает аудио → Groq Whisper
    └─ Транскрибируем сами
       ↓
Шаг 4: Claude — summary + цитаты + задачи + темы
       ↓
Файл в vault / ответ в Telegram
```

**Почему не один сервис:**
- Supadata платный, но забирает даже авто-субтитры, которые YouTube прячет для API.
- `youtube-transcript-api` — бесплатно, но часто баню IP или видео заблокировано.
- Whisper — последний рубеж, работает всегда, но тратит минуты.

Три слоя = 99% попаданий.

---

## 🔧 Установка (15 мин)

### Зависимости Python
```bash
pip3 install youtube-transcript-api yt-dlp groq anthropic requests
```

### Supadata API (опционально, но рекомендуется)
1. https://supadata.ai → Sign up
2. Dashboard → API key
3. Бесплатный тариф: 100 запросов/мес. Платный: $10/мес за 1000 запросов.
4. Сохрани в Keychain:
   ```bash
   security add-generic-password -a "$USER" -s "supadata_api" -w "sup_..."
   ```

### yt-dlp (для аудио-скачивания когда субтитров нет)
```bash
brew install yt-dlp
yt-dlp --version
```

Groq + Anthropic ключи — уже поставил в L4.

---

## 💻 Минимальный скрипт (можешь дать Claude Code как ТЗ)

```python
import re, subprocess, requests
from pathlib import Path

def get_secret(name):
    return subprocess.check_output(
        ["security", "find-generic-password", "-s", name, "-w"]
    ).decode().strip()

def extract_video_id(url: str) -> str:
    m = re.search(r"(?:v=|youtu\.be/|/shorts/)([A-Za-z0-9_-]{11})", url)
    return m.group(1) if m else ""

# ─── Шаг 1: Supadata ────────────────────────────
def try_supadata(video_id: str) -> str:
    try:
        r = requests.get(
            "https://api.supadata.ai/v1/youtube/transcript",
            params={"videoId": video_id, "text": "true"},
            headers={"x-api-key": get_secret("supadata_api")},
            timeout=30,
        )
        if r.status_code == 200:
            return r.json().get("content", "")
    except Exception:
        pass
    return ""

# ─── Шаг 2: youtube-transcript-api ──────────────
def try_yt_api(video_id: str) -> str:
    try:
        from youtube_transcript_api import YouTubeTranscriptApi
        api = YouTubeTranscriptApi()
        for langs in [["ru"], ["en"], ["ru", "en"]]:
            try:
                items = api.fetch(video_id, languages=langs)
                return " ".join(i.text for i in items if i.text)
            except Exception:
                continue
    except Exception:
        pass
    return ""

# ─── Шаг 3: Whisper через yt-dlp ────────────────
def try_whisper(url: str) -> str:
    out = Path(f"/tmp/{extract_video_id(url)}.m4a")
    subprocess.run([
        "yt-dlp", "-f", "bestaudio", "-o", str(out),
        "--extract-audio", "--audio-format", "m4a", url
    ], check=True)

    from groq import Groq
    g = Groq(api_key=get_secret("groq_api"))
    with open(out, "rb") as f:
        res = g.audio.transcriptions.create(
            file=(out.name, f.read()),
            model="whisper-large-v3",
            language="ru",
            temperature=0.0,
        )
    out.unlink(missing_ok=True)
    return res.text

# ─── Главная функция ────────────────────────────
def transcribe_youtube(url: str) -> str:
    vid = extract_video_id(url)
    if not vid:
        raise ValueError("Не YouTube URL")

    for fn, label in [
        (lambda: try_supadata(vid), "Supadata"),
        (lambda: try_yt_api(vid), "yt-api"),
        (lambda: try_whisper(url), "Whisper"),
    ]:
        text = fn()
        if text and len(text) > 50:
            print(f"[✓ {label}] {len(text)} символов")
            return text
    raise RuntimeError("Ни один метод не сработал")
```

---

## 🧠 Claude превращает транскрипт в summary

Добавь финальный шаг — отправь сырой транскрипт в Claude:

```python
from anthropic import Anthropic

def summarize(transcript: str, url: str) -> str:
    a = Anthropic(api_key=get_secret("anthropic_api"))
    msg = a.messages.create(
        model="claude-sonnet-4-7",
        max_tokens=2000,
        messages=[{"role": "user", "content": f"""Разбери YouTube-видео. Сделай markdown:

# {{title — угадай или «Видео»}}

**URL:** {url}

## TL;DR
3-5 предложений сути.

## Ключевые идеи (5-8 пунктов)
- ...

## Цитаты (3-5 самых сильных)
> «...»

## Задачи / To-do (если упомянуты)
- [ ] ...

## Темы для дальнейшего изучения
- ...

---
ТРАНСКРИПТ:
{transcript}
"""}]
    )
    return msg.content[0].text
```

---

## 🤖 Интеграция в Telegram-бот (как у Димы)

К уже существующему voice-inbox боту из L4 добавь обработчик текстовых сообщений с YouTube-ссылкой:

```python
async def handle_text(update, context):
    text = update.message.text
    # YouTube?
    if re.search(r"(youtube\.com|youtu\.be)/", text):
        url = re.search(r"https?://\S+", text).group()
        await update.message.reply_text("⏳ Обрабатываю видео...")
        try:
            transcript = transcribe_youtube(url)
            summary = summarize(transcript, url)
            # Сохраняем
            vid = extract_video_id(url)
            Path(f"~/vault/Transcripts/{vid}.md").expanduser().write_text(summary)
            # Краткий ответ в чат
            preview = summary.split("\n\n")[1][:500]  # TL;DR
            await update.message.reply_text(f"✓ Готово\n\n{preview}")
        except Exception as e:
            await update.message.reply_text(f"❌ {e}")
```

---

## 📂 Куда класть результат

Дима складывает в `5_БАЗА_ЗНАНИЙ/Transcripts/YYYY-MM-DD_youtube_{video_id}.md`.

Преимущество: потом можно через grep найти всё что смотрел по теме, или скормить Claude несколько транскриптов разом для cross-reference.

**Reco структуры папок:**
```
~/vault/Transcripts/
├── youtube/
│   └── 2026-04-23_abc123.md
├── meetings/       ← Deepgram с диаризацией
├── voice/          ← твои голосовые
└── INDEX.md        ← авто-обновляемый список
```

---

## ⚠️ Грабли

1. **Видео приватное / ограничено по региону** — `yt-dlp` попросит cookies. Генеришь через браузер-плагин, сохраняешь в `cookies.txt`, добавляешь `--cookies cookies.txt` к yt-dlp.
2. **Shorts короче 60 сек часто без субтитров** — сразу идут в Whisper, не трать попытки на API.
3. **Стримы часовые+** — `yt-dlp` отдаёт m4a на 100 МБ. Whisper 25 МБ лимит. Режь на куски:
   ```bash
   ffmpeg -i long.m4a -f segment -segment_time 900 -c copy part_%03d.m4a
   ```
   900 сек = 15 мин, обычно укладывается в 25 МБ при речи.
4. **Автосубтитры YouTube кривые** — без пунктуации, с опечатками. Перед Claude прогоняй через regex-чистку (убрать повторы слов).
5. **Supadata бесплатный тариф = 100/мес** — экономь на тесты, используй fallback.

---

## ✅ Чек что YouTube-настройка прошла

- [ ] `yt-dlp --version` работает
- [ ] `youtube-transcript-api` ставится и fetch'ит тестовое видео
- [ ] (Опционально) Supadata API ключ в Keychain
- [ ] Скрипт `transcribe_youtube()` для любого URL возвращает текст
- [ ] Claude превращает транскрипт в `TL;DR + идеи + цитаты + задачи`
- [ ] Бот реагирует на YouTube-ссылку в Telegram и присылает summary

---

## 🎁 Бонус — как Дима это использует в работе

1. **Конкурентная разведка:** присылает в бот интервью конкурента → через 30 сек готовый summary с цитатами.
2. **Клиентские презентации:** клиент скинул видео по теме → перед встречей прочитал саммари, не смотрел 40 мин.
3. **Контент-машина:** своё же YouTube-интервью → транскрипт → Claude нарезает в 5 постов для LinkedIn/TG.
4. **Крипто-DD:** видео с Binance/OKX → выделяются цифры, сроки, имена → в DD-отчёт клиенту.

Ты поймёшь где пригодится как только запустишь. Главное — сделать базовый pipeline. Всё остальное — надстройки.

---

**Автор:** Дима + Demi
