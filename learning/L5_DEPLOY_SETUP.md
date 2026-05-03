# 🚀 L5 — Deploy pipeline (сайт в продакшн за 10 минут голосом)

> Цель: ты говоришь «сделай лендинг под X, запушь» → через час клиент кликает по живому URL.
> Дима делает так каждую неделю — это не магия, это один раз собранный конвейер.

---

## 🧩 Стек (почему именно так)

```
Claude Code пишет HTML
    ↓
git commit → GitHub (хранилище кода)
    ↓
Vercel (автодеплой из main) → живой URL
    ↓
Namecheap/Cloudflare (домен → Vercel IP)
    ↓
клиент видит https://твой-домен
```

**Почему не WordPress / Tilda / Webflow:**
- Claude не умеет их редактировать. Ты будешь кликать мышкой — медленно.
- HTML + GitHub + Vercel = всё редактируется текстом, Claude видит всё.

**Почему Vercel а не GitHub Pages:**
- Vercel быстрее, custom domains бесплатно, preview-URL на каждый PR.
- GitHub Pages норм для простых случаев — Дима держит там `promstroy-sites`.

---

## 🔧 Шаг 1. GitHub (15 мин)

1. https://github.com → Sign up (тот же email что везде).
2. Settings → Developer settings → **Personal Access Tokens → Tokens (classic)** → Generate new.
   - Scope: `repo` (всё), `workflow`.
   - Expiration: 1 год.
   - Сохрани токен — больше не покажут.
3. Сохрани в Keychain:
   ```bash
   security add-generic-password -a "$USER" -s "github_token" -w "ghp_..."
   ```
4. Настрой git identity (один раз):
   ```bash
   git config --global user.name "Alexander X"
   git config --global user.email "alex@yourmail.com"
   ```

---

## 🌐 Шаг 2. Vercel (10 мин)

1. https://vercel.com → Sign up **через GitHub** (важно — тогда автоконнект).
2. Import Repository → выбери свой репо.
3. Framework preset: **Other** (для чистого HTML) или Next.js если делаешь Next.
4. Deploy. Получаешь `yourproject-xxxx.vercel.app`.
5. **Custom domain:** Settings → Domains → добавь `yourdomain.com`.

**Автодеплой:** каждый `git push` в `main` → Vercel автоматически выкатывает. 30-60 сек.
**Preview:** каждый push в любую ветку → свой URL типа `yourproject-git-feature.vercel.app`. Можно показывать клиенту до мержа.

---

## 🌍 Шаг 3. Домен (Namecheap / Cloudflare)

### Namecheap (проще для новичка)
1. Купи домен ($8-15/год).
2. Domain List → Manage → Advanced DNS.
3. Удали дефолтные записи.
4. Добавь:
   ```
   A     @      76.76.21.21       (IP Vercel)
   CNAME www    cname.vercel-dns.com
   ```
5. Жди 5-30 минут до распространения.

### Cloudflare (лучше — кэш, SSL, защита)
1. Добавь домен → Cloudflare выдаст 2 nameserver.
2. В Namecheap → Domain → Nameservers → Custom DNS → вставь cloudflare nameservers.
3. В Cloudflare DNS:
   ```
   A     @      76.76.21.21       Proxied
   CNAME www    cname.vercel-dns.com   Proxied
   ```

Дима держит на Namecheap. Cloudflare добавляет когда нужна защита от DDoS/бот-трафика.

---

## ✅ Шаг 4. Mobile-deploy-check (ОБЯЗАТЕЛЬНЫЙ гейт)

**Правило Димы:** перед КАЖДЫМ `git push` сайта — проверяем на мобилке. 70% людей откроют с телефона. Ломается мобилка — ломается конверсия.

### Быстрый чек через Claude Preview MCP
В Claude Code:
```
Используй preview_resize с preset=mobile (375×812), открой текущий сайт на localhost,
проверь document.body.scrollWidth === 375 (не должно быть горизонтального скролла),
сделай скриншот hero + main content + footer.
Покажи мне.
```

Что должно быть:
- ✅ `scrollWidth === 375` — никакого `overflow-x`
- ✅ Hero читается без зума (font-size ≥ 16px)
- ✅ Кнопки тапабельные (min 44×44px)
- ✅ Формы — поля на всю ширину, input ≥ 16px (иначе iOS зумит)
- ✅ Изображения сжимаются (не торчат вправо)

Если что-то не ОК — **не пушь**. Чинь в CSS:
- `font-size: clamp(14px, 4vw, 18px)` — адаптивные шрифты
- `grid-template-columns: 1fr` на ширине `< 720px`
- `padding: clamp(16px, 5vw, 48px)`

---

## 🎯 Шаг 5. Conversion-audit (15 пунктов)

Перед отдачей клиенту — механический чек-лист. Один из них — сразу жалоба.

```markdown
## Hero
- [ ] Заголовок: кто + что делает + для кого (одно предложение)
- [ ] Подзаголовок: конкретная выгода / цифра
- [ ] CTA-кнопка выше fold (до первой прокрутки)
- [ ] Один основной CTA, максимум два
- [ ] Цифра в hero (опыт / клиенты / деньги)

## Trust
- [ ] Логотипы клиентов / партнёров
- [ ] Testimonials с именами и фото (2-5)
- [ ] Кейсы с цифрами
- [ ] Фото эксперта, не сток

## Конверсия
- [ ] Форма лаконичная (3-4 поля max)
- [ ] Telegram/WhatsApp кнопка (быстрый канал)
- [ ] «Что будет дальше» после отправки
- [ ] NFA-дисклеймер если фин-тематика

## Мобильный
- [ ] Hamburger меню
- [ ] Кликабельные телефоны `tel:`
- [ ] Inline CTA между секциями (не только в hero)
- [ ] Формы с правильным type (email, tel)
```

---

## 🛠 Шаг 6. CLAUDE.md проекта сайта

В корне каждого клиентского репо — `CLAUDE.md`:

```markdown
# ПРОЕКТ: Лендинг для [клиент]

**Домен:** client-domain.com
**Vercel project:** client-landing
**GitHub:** github.com/alex/client-landing

## Деплой
git push origin main
Vercel сам выкатит за 60 сек.
Проверить: https://client-domain.com

## Перед пушем — ОБЯЗАТЕЛЬНО
1. Mobile-deploy-check (375×812, scrollWidth === 375)
2. Conversion-audit (15 пунктов)
3. Грамматика: прогнать через Claude «проверь орфографию и пунктуацию»

## НЕ трогать
- /assets/client-photos/ — оригиналы клиента
- favicon.ico — утверждён клиентом

## Стек
- Vanilla HTML + CSS, без фреймворков
- GSAP для анимаций
- Плавающая форма внизу на мобилке
```

---

## 📦 Шаг 7. Шаблон-стартер (делаешь один раз, потом копируешь)

Рекомендую собрать себе `~/templates/landing-starter/`:
```
landing-starter/
├── index.html          ← базовая структура (hero, about, services, testimonials, form, footer)
├── styles.css          ← design tokens (цвета, шрифты, spacing) + responsive
├── script.js           ← форма → Telegram Bot API / Formspree
├── .gitignore          ← .env, .DS_Store, node_modules
├── CLAUDE.md           ← шаблон для клиентского проекта
├── vercel.json         ← redirects, headers
└── README.md
```

Когда новый клиент:
```bash
cp -r ~/templates/landing-starter ~/projects/NEW_CLIENT
cd ~/projects/NEW_CLIENT
git init && git add . && git commit -m "init"
# → через Claude: «адаптируй под клиента [описание], деплой»
```

---

## ⚠️ Типичные ошибки

1. **Запушил секретный ключ в публичный репо** — GitHub прислал алерт. Revoke ключ немедленно, перегенери, в `.gitignore` добавь. Используй `git-secrets` или `gitleaks` в pre-commit.
2. **Домен не резолвится 2 часа** — подожди до 24 ч (DNS propagation). `dig yourdomain.com` проверит.
3. **Vercel деплоит, но показывает старое** — hard refresh (Cmd+Shift+R), проверь Deployments в Vercel dashboard.
4. **SSL не работает** — в Vercel → Domains → кликни Refresh, обычно само через 5 мин.
5. **Мобильный overflow** — 99% случаев: `width: 100vw` вместо `100%`, или таблица без `overflow-x: auto`.

---

## ✅ Чек что L5 пройден

- [ ] GitHub токен в Keychain, git identity настроен
- [ ] Vercel подключён к GitHub, автодеплой работает
- [ ] Один домен подключён и резолвится через HTTPS
- [ ] Mobile-deploy-check прогон хотя бы 1 раз — понял как делать
- [ ] Conversion-audit 15 пунктов — чек-лист у тебя под рукой
- [ ] Шаблон `landing-starter` собран — следующий клиент стартует за 5 минут

Когда дойдёшь — скинь в группу URL первого лендинга. Разберём вместе, дадим фидбек.

---

**Автор:** Дима (деплоит по 2-3 сайта в неделю) + Demi
