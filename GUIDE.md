# findmini.org — инструкция Vlada

Сайт лежит здесь, https://findmini.org живёт через Vercel. Этот документ — всё что нужно чтобы:

- редактировать сайт самой (без программиста)
- подключить Claude Code для автоматизации (когда будет время)
- собрать скрипт анализа камер на серую tabby

---

## 1. Как менять сайт (без терминала)

Весь сайт — это **один файл** `index.html` (плюс папка `media/` с фото и видео).

### Через GitHub веб-редактор (5 минут на правку)

1. Открой https://github.com/vasilevdasfo/findmini
2. Кликни на `index.html`
3. Нажми карандаш ✏️ справа сверху
4. Меняй текст
5. Внизу страницы — кнопка **Commit changes** → зелёная **Commit**
6. Через **30 секунд** findmini.org обновится автоматически (Vercel деплоит сам)

**Что обычно правят в `index.html`:**
- Текст «$500 reward», «Lost since...», даты — простой поиск Ctrl+F по странице
- Номер телефона `520-273-3420` — поиск-замена
- Адрес `300 Palmetto Ave` — поиск-замена

### Заменить фото / добавить новое

1. На GitHub в репо findmini → папка **media**
2. Кнопка **Add file → Upload files** → перетаскиваешь файл
3. Commit
4. Если файл с **новым** именем — зайди в `index.html`, найди ближайшую `<img src="media/photo_...">` и поменяй имя на твой новый файл
5. Если перезаписала старый файл с тем же именем — ничего менять не надо, сайт сам подхватит

### Откатить если сломала

1. https://github.com/vasilevdasfo/findmini → вкладка **Commits**
2. Найди последний работавший commit (зелёная галка слева)
3. Кнопка `<>` справа от commit → Browse files → откроешь старую версию
4. Если совсем плохо — напиши в TG, верну за 30 сек

---

## 2. Hosting / DNS / SSL — справка

- **Хостинг:** Vercel (project `findmini`, owner `vasilevdasfos-projects`)
- **Домен:** findmini.org через Namecheap
- **DNS записи в Namecheap:**
  - CNAME `@` → `cname.vercel-dns.com.`
  - CNAME `www` → `cname.vercel-dns.com.`
- **SSL:** автоматически от Let's Encrypt через Vercel (обновляется сам)
- **Деплой:** при каждом коммите в `main` Vercel пересобирает за ~30 сек

Если что-то с доменом не работает — проверка одной командой в терминале:
```bash
dig +short A findmini.org
# должно показать IP типа 76.76.21.123 (Vercel edge)
```

---

## 3. Claude Code — что это и зачем

Claude Code = Claude (как чат на claude.ai) + прямой доступ к твоему компу + интернету. Видит файлы, пишет код, запускает команды, ходит в гугл, гитхаб, телеграм.

**Зачем тебе:**
- Менять сайт голосом: «добавь блок про FB-группу Pacifica Cats» — он сам пишет HTML и пушит
- Прогонять камеры через AI пакетно (см. раздел 5)
- Читать соцсети и шерстить посты на тему серой tabby
- Telegram-бот мониторинга на 30 минут работы

### Установка (когда будешь готова — займёт 10 мин)

```bash
# 1. Pro подписка на claude.ai ($20/мес или $100/год)
#    Реф от Дмитрия с бесплатной неделей: https://claude.ai/referral/VDIurtpiew
#    Если не открывается — напиши Диме, скинет актуальную

# 2. Установка Claude Code в терминале Mac
brew install --cask claude-code
# или скачать с https://claude.com/code

# 3. Логин (откроется браузер)
claude login

# 4. Зайти в папку проекта и запустить
cd ~/Documents
git clone https://github.com/vasilevdasfo/findmini.git
cd findmini
claude
```

После запуска `claude` — пишешь голосом или текстом задачу. Он сам разбирается.

### Базовые команды

| Команда | Что делает |
|---|---|
| `/init` | Создаёт CLAUDE.md (см. раздел 4) — память проекта |
| `/clear` | Очистить контекст (если запутался) |
| `/compact` | Сжать историю |
| `/help` | Список всех команд |
| `ESC` | Прервать что делает |
| `!команда` | Запустить shell-команду в чате |

---

## 4. Скиллы — как «скармливать» Claude правила

«Скилл» = текстовая инструкция, которую Claude читает каждый раз и применяет автоматически. Лежит в файле `CLAUDE.md` в корне проекта (или в `~/.claude/CLAUDE.md` для глобальных правил).

### Как создать свой первый скилл

В корне проекта (`~/Documents/findmini`) создай файл `CLAUDE.md`:

```markdown
# Проект findmini.org

## Что это
Сайт о потерянной кошке Mini в Pacifica, CA. Деплоится на Vercel автоматически из ветки `main`. Один файл `index.html`. Дизайн — светлый бежевый фон, акцент красный (#d4322a).

## Правила работы
- Перед каждой правкой `index.html` — снимай preview на 375×812 (iPhone), смотри что нет horizontal scroll
- После правки — `git add . && git commit -m "..." && git push` → через 30 сек обновится
- Все фото в `media/`, новые добавлять туда же. Сжимать до 1200px по длинной стороне (jpeg quality 80)
- Reward $200/$500 — на видном месте, не убирать
- Контактный телефон 520-273-3420 — не менять без подтверждения у меня
- Языки: EN/ES/ZH (Daly City — много латино и китайцев), не убирать
```

Сохрани — Claude в этой папке будет следовать этим правилам автоматически.

### Готовые скиллы которые тебе зайдут

Скопируй в `CLAUDE.md` после блока выше:

```markdown
## Скилл: проверка мобильной вёрстки перед пушем

Перед `git push` любого изменения index.html:
1. Открыть Chrome DevTools на 375×812 (iPhone)
2. Проверить `document.body.scrollWidth === 375` (нет horizontal overflow)
3. Если overflow есть — починить через clamp() на font-size, padding меньше, grid → 1fr на 720px

## Скилл: оптимизация фото

Если добавляю фото в media/:
1. Длинная сторона ≤1200px (`sips -Z 1200 photo.jpg`)
2. Quality 80 (`sips -s formatOptions 80`)
3. Total weight всей папки ≤5MB (видео не считаем)

## Скилл: безопасный коммит

Никогда не коммить:
- API-ключи (`OPENAI_API_KEY=...` → в `.env`, файл в `.gitignore`)
- Личные данные третьих лиц (имена-телефоны людей кто помогает искать кошку)
- Скриншоты камер с лицами прохожих (privacy)
```

### Куда ещё можно положить скиллы

- `~/.claude/CLAUDE.md` — глобально для всех проектов на твоём компе
- `~/.claude/skills/имя/SKILL.md` — отдельный модуль, Claude подгрузит когда нужен
- В тексте промта прямо в чате: «вот правило: ... теперь применяй» — он будет помнить в рамках сессии

Подробнее про скиллы: https://docs.claude.com/en/docs/claude-code/skills

---

## 5. Анализ камер — промт для Vision API

У тебя камеры наблюдения. Прогонять каждый кадр глазами — нереально. Решение: AI смотрит сам.

### Промт (универсальный, работает с GPT-4o, Claude, Gemini)

```
Ты — помощник по поиску потерянной кошки.

Кошка Mini: серо-коричневая tabby, 7-8 месяцев,
без ошейника, без белых пятен, чёрные тигровые полоски.

Я даю тебе кадр с уличной камеры.

Верни СТРОГО JSON:
{
  "cat_present": true | false,
  "confidence": 0-100,
  "matches_mini": true | false | "partial",
  "color_match": "grey-brown-tabby" | "other-tabby" | "non-tabby" | "no-cat",
  "size_estimate": "kitten" | "small-adult" | "adult" | null,
  "white_markings": true | false | null,
  "timestamp_visible": "..." | null,
  "notes": "одно короткое предложение"
}

Если cat_present=false — все остальные поля null.
Если matches_mini=true И confidence>70 — это ВАЖНОЕ совпадение.
```

### Какую модель использовать (дёшево → дорого)

| Модель | Цена за 1000 кадров | Когда брать |
|---|---|---|
| Claude Haiku 4.5 vision | ~$0.40 | Скрининг — пробежать всё что есть |
| GPT-4o-mini vision | ~$0.50 | Альтернатива Хайку, тоже скрининг |
| GPT-4o vision (high) | ~$15 | Финальная проверка только на тех кадрах где скрининг сказал «ВАЖНО» |
| Gemini 2.5 Flash | ~$0.30 | Самый дешёвый, тестируй точность |

### Стратегия экономии

1. **Режь камеру до 1 кадра в 30 секунд** — кошка не телепортируется. Из 30 fps × час = 108000 кадров → берём 120 кадров. Экономия 900×.
2. **Скрининг Хайку**: 120 кадров × $0.40/1000 = ~$0.05 за час записи
3. **Финал GPT-4o** только на тех где confidence>50: обычно 5-10 кадров → $0.15
4. **Всего:** ~$0.20 за час каждой камеры

### Скрипт-старт (Python)

```python
# pip install anthropic openai
import base64, json, os
from anthropic import Anthropic
client = Anthropic()  # читает ANTHROPIC_API_KEY из env

PROMPT = """[вставь промт сверху]"""

def check_frame(image_path: str) -> dict:
    with open(image_path, "rb") as f:
        b64 = base64.b64encode(f.read()).decode()
    r = client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=300,
        messages=[{
            "role": "user",
            "content": [
                {"type": "image", "source": {
                    "type": "base64", "media_type": "image/jpeg", "data": b64
                }},
                {"type": "text", "text": PROMPT}
            ]
        }]
    )
    txt = r.content[0].text
    # вырезаем JSON из ответа
    start, end = txt.find("{"), txt.rfind("}") + 1
    return json.loads(txt[start:end])

# Прогон папки
import glob
hits = []
for path in sorted(glob.glob("cam_dumps/*.jpg")):
    try:
        result = check_frame(path)
        if result.get("matches_mini") and result.get("confidence", 0) > 50:
            hits.append({"file": path, **result})
            print(f"⭐ {path}: confidence={result['confidence']}, {result['notes']}")
    except Exception as e:
        print(f"err {path}: {e}")

# Сохраняем все находки
with open("hits.json", "w") as f:
    json.dump(hits, f, indent=2, ensure_ascii=False)
```

### Если хочешь Telegram-уведомления при находке

Когда `matches_mini=true && confidence>70` — пушнуть в твой телеграм:

```python
import requests
TG_TOKEN = os.environ["TG_BOT_TOKEN"]   # созданный @BotFather
TG_CHAT = os.environ["TG_CHAT_ID"]      # твой user_id

def notify(file_path, result):
    with open(file_path, "rb") as photo:
        requests.post(
            f"https://api.telegram.org/bot{TG_TOKEN}/sendPhoto",
            data={"chat_id": TG_CHAT,
                  "caption": f"⭐ возможно Mini\nconf={result['confidence']}\n{result['notes']}"},
            files={"photo": photo})
```

Бот создаётся за 2 минуты у `@BotFather` в TG.

### Резка кадров из видео

```bash
# 1 кадр каждые 30 сек:
ffmpeg -i camera_record.mp4 -vf "fps=1/30" cam_dumps/frame_%04d.jpg
```

---

## 6. Где взять API ключи

| Сервис | Где получить | Цена для тебя |
|---|---|---|
| Anthropic (Claude) | https://console.anthropic.com → Settings → API Keys | $5 кредит при регистрации, потом по факту |
| OpenAI | https://platform.openai.com/api-keys | $5 кредит, потом ~$10/мес типичные расходы |
| Google AI (Gemini) | https://aistudio.google.com/apikey | бесплатный tier есть |
| Telegram Bot | https://t.me/BotFather → /newbot | бесплатно |

**Важно:** ключи никогда не коммить в git. В коде читать через `os.environ["OPENAI_API_KEY"]`, локально хранить в `~/.zshrc` или `.env` (с `.env` в `.gitignore`).

---

## 7. Что делать когда упёрлась

1. Опиши задачу Claude максимально подробно — он сам разберётся
2. Если Claude путается — `/clear` и начни заново, в этот раз короче
3. Если совсем не получается — скрин в TG @Posbitcoin, разберём
4. Если идея «а можно ли...» — да, скорее всего можно. Спрашивай в чате с Claude, он скажет реализуемо или нет

---

*Этот guide обновляется. Последнее обновление: 2026-05-03. Если нашла ошибку — открой issue на GitHub или напиши в TG.*
