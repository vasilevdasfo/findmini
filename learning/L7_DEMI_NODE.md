# L7 — DEMI-node: установка и первый запуск

> Полный гайд для Windows + WSL2 Ubuntu.
> Репозиторий: https://github.com/vasilevdasfo/demi-node
> Версия: `v0.1.0-alpha.1` (апрель 2026)

---

## 0. Что это такое и зачем

**DEMI-node** — это маленький P2P-узел, который ты запускаешь у себя на машине. Он умеет:

- напрямую соединяться с другой такой же нодой (у меня, у Бычковского-младшего, у любого у кого она установлена),
- принимать и отправлять сообщения end-to-end (ed25519 + Noise XX),
- хранить историю чата локально в SQLite,
- показывать веб-интерфейс на `http://localhost:4321`.

### Чем это отличается от Telegram

| Параметр | Telegram | DEMI-node |
|---|---|---|
| Где хранятся сообщения | На серверах Telegram (Дубай/Амстердам) | Локально у тебя в `~/.demi-node/chat.db` |
| Кто может прочитать историю | TG + тот у кого доступ к твоему аккаунту | Только ты (ключ `0600` на диске) |
| Метаданные (кто с кем когда) | Видит TG | Знаешь только ты и собеседник |
| Если упал сервер | Всё упало | Ничего не случилось — нет сервера |
| Нужен телефон / SMS | Да | Нет |
| Нужна регистрация | Да | Нет. Ключ создаётся локально, ник случайный |
| Переписка шифруется | Клиент-сервер (только в "секретных чатах" E2E) | Всегда end-to-end по умолчанию |

> 🟢 **Коротко:** DEMI — это «Telegram без Telegram». Нода живёт на твоём компьютере, соединяется с моей напрямую через DHT-сеть Hyperswarm.

### Для кого это сейчас

Alpha-версия. Работает, протестирована между моими двумя нодами и с парой friendly-юзеров. **Не** предназначена для life-safety сценариев (журналисты в недружественных юрисдикциях). Для нашей истории (ты, Бычковские, я) — рабочий инструмент.

---

## 1. Требования

| Что | Версия | Зачем |
|---|---|---|
| Windows 10 build 19041+ / Windows 11 | любая | хост |
| **WSL2** | включён, ядро ≥ 5.15 | Linux-слой |
| **Ubuntu** | 22.04 LTS или 24.04 | дистр |
| **Node.js** | ≥ 18 (лучше 20 LTS) | рантайм ноды |
| **git** | любой | клонировать репо |
| **Свободного диска** | ~200 MB | `node_modules` + база |
| **Свободной RAM** | ~150 MB на запущенную ноду | процесс + SQLite |
| **Сеть** | исходящий UDP/TCP на любые порты | Hyperswarm DHT |

🟡 **Windows Defender** часто блокирует исходящий UDP, который нужен Hyperswarm для hole-punching через NAT. В 90% случаев это лечится разрешением Node.js в «Брандмауэр Defender → Разрешить приложение». Если не лечится — смотри раздел **Troubleshooting → UDP blocked**.

---

## 2. Установка — шаг за шагом

Предполагаю, что WSL2 + Ubuntu + Node.js + git у тебя уже стоят (мы это прошли в L4/WINDOWS-гайдах). Если нет — сначала вернись туда.

### 2.1. Открой WSL

Запусти **Ubuntu** из меню Пуск. Окажешься в bash-шелле вида `alexander@DESKTOP-XXXX:~$`.

### 2.2. Проверь версии

```bash
node --version
# ожидаемый output: v20.x.x (или v18.x.x)

git --version
# ожидаемый output: git version 2.x.x

npm --version
# ожидаемый output: 10.x.x
```

Если `node --version` показывает что-то меньше `v18` — обнови:

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
```

### 2.3. Клонируй репо

```bash
cd ~
git clone https://github.com/vasilevdasfo/demi-node.git
cd demi-node
```

Ожидаемый output:

```
Cloning into 'demi-node'...
remote: Enumerating objects: ...
Receiving objects: 100% (...)
Resolving deltas: 100% (...)
```

### 2.4. Установи зависимости

```bash
npm install
```

⏱️ Это займёт 1–3 минуты. Увидишь что-то вроде:

```
added 187 packages in 1m 23s

41 packages are looking for funding
```

🟡 Если видишь warnings про `deprecated` или `peer dep` — не страшно, это нормально для alpha.

🔴 Если видишь **ERROR** про `better-sqlite3` или `node-gyp` — значит не хватает build-tools. Лечится так:

```bash
sudo apt-get update
sudo apt-get install -y build-essential python3
npm install
```

### 2.5. Первый запуск (он же — создание identity)

Нода сама создаст ключи при первом старте. Отдельной команды `identity create` нет — это частая путаница.

```bash
node src/index.js
```

Ожидаемый output (примерно):

```
[demi] generated identity
[demi] pubkey: 1a2b3c4d...  fp: 1a2b3c4d
[demi] nickname: swift-fox-42
[demi] transport: hyperswarm
[demi] UI:  http://localhost:4321
[demi] ready
```

🟢 Если увидел `[demi] ready` — всё работает. Оставь этот терминал открытым, нода должна быть запущена пока ты с ней работаешь.

**Что произошло под капотом:**
- В твоей домашке появилась папка `~/.demi-node/`.
- В ней лежат: `identity.key` (секрет, `0600`), `identity.pub`, `nickname`, `chat.db` (SQLite), `config.json`.
- Нода подключилась к публичной Hyperswarm DHT, начала слушать `127.0.0.1:4321`.

### 2.6. Проверь статус из другого терминала

Открой **второй** WSL-терминал (Ubuntu из Пуска ещё раз), и выполни:

```bash
cd ~/demi-node
node bin/demi.js status
```

Ожидаемый output:

```
Node is running
Nickname:    swift-fox-42
Fingerprint: 1a2b3c4d
Pubkey:      1a2b3c4d5e6f...<64 hex chars total>
Uptime:      37s
```

🟢 Видишь nickname и fingerprint — значит CLI-клиент тоже подключается к ноде.

---

## 3. Идентичность (identity) — что это и почему её нельзя терять

Identity = твои криптоключи (ed25519). Это **единственный способ** тебя узнать в сети. Собеседник добавляет в доверенные **pubkey**, не телефон и не ник.

### Где живёт

```
~/.demi-node/
├── identity.key       ← секретный ключ (NEVER share)
├── identity.pub       ← публичный ключ (можно показывать)
├── nickname           ← текстовый файл с ником
├── chat.db            ← история всех разговоров
└── config.json        ← порт UI, язык, rate-limits
```

> 🔴 **Критично: `identity.key` и `chat.db` — это вся твоя история в DEMI.**
> Потерял → сменил pubkey → все кто тебя добавили в доверенные видят тебя как нового незнакомого человека. Придётся заново делать pair.

### Бэкап — сделай СЕЙЧАС, до первого pair'а

```bash
mkdir -p ~/demi-backup
cp -a ~/.demi-node/identity.key ~/demi-backup/
cp -a ~/.demi-node/identity.pub ~/demi-backup/
cp -a ~/.demi-node/nickname ~/demi-backup/
chmod 600 ~/demi-backup/identity.key
```

Дополнительно — скопируй `identity.key` на USB-флешку или в зашифрованный архив (например `7z a -p...`).

### Восстановление из бэкапа

Представь: переустановил Windows, WSL снёс, всё с нуля. Восстановить identity:

```bash
mkdir -p ~/.demi-node
cp ~/demi-backup/identity.key ~/.demi-node/
cp ~/demi-backup/identity.pub ~/.demi-node/
cp ~/demi-backup/nickname ~/.demi-node/
chmod 600 ~/.demi-node/identity.key

cd ~/demi-node
node src/index.js
```

Нода увидит готовые ключи, не будет генерить новые, и ты появишься в сети под тем же pubkey. Все кто добавили тебя раньше — узнают.

🟡 **История чата `chat.db`** — это отдельный файл. Бэкап его тоже, если жалко переписку. Без него пара останется (по pubkey), но сообщения «до восстановления» будут видны только у собеседника.

---

## 4. Pair flow — как добавить меня в доверенные

Pair — это разовая церемония где два узла обмениваются pubkey'ами, подписями и поднимают защищённый канал. После этого вы соединяетесь автоматически при каждом запуске.

### Как это выглядит

1. Один создаёт 6-значный код вида `384-921`.
2. Передаёт другому через **любой** надёжный канал (Telegram, SMS, голос по телефону).
3. Второй вводит код у себя.
4. Ноды находят друг друга в DHT, обмениваются подписанными envelope'ами, валидируют, сохраняют в `chat.db`.
5. Код сгорает. TTL — 10 минут.

### Как получишь код от меня

Я напишу тебе в Telegram что-то вроде:

> Алекс, код: `384-921`. Введи в течение 10 минут.

Ты у себя в WSL запускаешь:

```bash
node bin/demi.js pair 384-921
```

Ожидаемый output:

```
Redeeming pair code 384-921...
✓ Paired with swift-dolphin-17 (fp: a1b2c3d4)
```

🟢 Всё, ты в доверенных у меня, я у тебя.

### Обратная сторона — ты создаёшь код для меня

```bash
node bin/demi.js pair --new
```

Output:

```
Pair code: 731-088

Share this code with your friend over a secure channel.
When they enter it, both nodes will connect automatically.
```

Копируешь `731-088`, отправляешь мне в Telegram. У меня 10 минут.

### Если код истёк (expired)

Вывод на моей стороне будет типа:

```
Failed: rpc timeout   (или "pair code expired")
```

Просто создаёшь новый — `pair --new` снова, присылаешь.

🟡 **Важно:** текущая версия alpha **не** поддерживает отмену кода (`pair --cancel`). Скоро добавится. Пока — если выдал не тому, подожди 10 минут, он сгорит сам.

---

## 5. Запуск ноды в «рабочем режиме»

У ноды есть два транспорта:

| Транспорт | Когда использовать |
|---|---|
| **hyperswarm** (default) | всё, кроме сценариев где Windows Defender режет UDP |
| **libp2p** (TCP + Noise) | fallback для сред где UDP/hyperswarm не работает |

### Обычный запуск (hyperswarm, default)

```bash
cd ~/demi-node
node src/index.js
```

### Запуск через libp2p (если hyperswarm не пробивает)

```bash
DEMI_TRANSPORT=libp2p node src/index.js
```

🟡 **Важный нюанс:** libp2p-ветка ещё дорабатывается — конкретно автоматический полный P2P-коннект между новыми узлами без заранее известных bootstrap-адресов сейчас работает не во всех сценариях. Если Defender режет hyperswarm, напиши мне — дам bootstrap-адрес своей ноды:

```bash
DEMI_LIBP2P_BOOTSTRAP=/ip4/SOME.IP/tcp/PORT/p2p/PEERID DEMI_TRANSPORT=libp2p node src/index.js
```

ETA fix — в ближайшие сутки, обновлю гайд когда landed.

### Фоновой запуск (чтобы не держать терминал открытым)

Вариант A — `nohup`:

```bash
cd ~/demi-node
nohup node src/index.js > /tmp/demi.log 2>&1 &
```

Процесс будет жить до перезагрузки WSL. Логи — `tail -f /tmp/demi.log`.

Остановить:

```bash
pkill -f "node src/index.js"
```

---

## 6. Observer UI — http://localhost:4321

Открой в обычном Windows-браузере (Chrome/Edge): **http://localhost:4321**

WSL2 пробрасывает `localhost` между Linux и Windows автоматически, поэтому браузер видит ноду.

### Что увидишь

**Экран 1 — Header / Identity**
Сверху твой nickname (`swift-fox-42`), fingerprint (`1a2b3c4d`), статус ноды (зелёный кружок = online). Слева переключатель языка RU/EN.

**Экран 2 — Peers**
Список людей с которыми ты в pair'е. Каждый — с индикатором `●` online / `○` offline, ник, fp-short, уровень trust (`trusted` / `seen`). По клику на peer открывается чат.

**Экран 3 — Chat**
Классический messenger-view. Слева — peers, справа — переписка с выбранным. Ввод снизу, Enter отправляет. Все сообщения сохраняются в `chat.db`, при рестарте ноды — всё на месте.

**Экран 4 — Pair panel**
Кнопка "Создать код" (выдаст 6-значный), поле "Ввести код" (куда вставить тот что прислали тебе). Всё что делает CLI через `demi pair`, можно сделать в UI.

🟡 **Сейчас в UI не работают** кнопки «Найти по нику» и «Профиль peer'а» — backend RPC для них ещё не landed. Скоро добавится.

---

## 7. CLI-шпаргалка

Все команды запускаются из папки `~/demi-node`. Нода должна быть запущена (`node src/index.js` в другом терминале или в фоне).

### Базовое

```bash
# статус ноды (работает ли, кто я)
node bin/demi.js status

# список peer'ов
node bin/demi.js peers
```

### Pair

```bash
# создать код
node bin/demi.js pair --new

# принять код
node bin/demi.js pair 384-921
```

### Чат

```bash
# отправить сообщение
node bin/demi.js send swift-dolphin-17 "Привет, это я"

# можно по pubkey вместо ника
node bin/demi.js send 1a2b3c4d5e6f...<64 hex> "Привет"

# прочитать историю (последние 20)
node bin/demi.js history swift-dolphin-17

# последние 100
node bin/demi.js history swift-dolphin-17 --last 100
```

### Паника

```bash
# ВНИМАНИЕ: стирает identity + всю историю + peers
# Восстановление только из бэкапа identity.key
node bin/demi.js wipe
# потом ввести YES (большими)

# без подтверждения (если знаешь что делаешь)
node bin/demi.js wipe --force
```

### Запуск ноды напрямую из CLI

```bash
node bin/demi.js start
# эквивалент node src/index.js
```

---

## 8. Environment variables (env)

Переменные окружения, которые можно подставлять перед `node src/index.js`:

| Переменная | Значение | Зачем |
|---|---|---|
| `DEMI_HOME` | путь, например `/tmp/demi-test` | запустить **вторую** ноду на той же машине для теста (другой identity, другой порт в своём `config.json`) |
| `DEMI_TRANSPORT` | `hyperswarm` (default) \| `libp2p` | переключение транспорта |
| `DEMI_LIBP2P_PORT` | число, напр. `0` | порт для libp2p TCP (0 = random) |
| `DEMI_LIBP2P_BOOTSTRAP` | `/ip4/X.X.X.X/tcp/PORT/p2p/PEERID,...` | явные bootstrap-узлы для libp2p |
| `DEMI_LIBP2P_MDNS` | `1` | включить mDNS (по умолчанию off — небезопасно в публичном WiFi) |

Пример:

```bash
# запустить тестовую ноду в отдельной директории с libp2p
DEMI_HOME=~/demi-test DEMI_TRANSPORT=libp2p node src/index.js
```

---

## 9. Troubleshooting — частые проблемы под Windows/WSL

### 9.1. Hyperswarm не коннектится (UDP blocked)

**Симптом:** нода стартует, `demi status` ок, но peer всегда `○ offline`, хотя у другой стороны — тоже офлайн.

**Причина:** Windows Defender режет исходящий UDP от Node.js.

**Фикс:**

1. Открой **Параметры Windows → Безопасность Windows → Брандмауэр и защита сети → Разрешить приложение через брандмауэр**.
2. Нажми «Изменить параметры», затем «Разрешить другое приложение».
3. Путь к Node.js в WSL найти сложно — проще разрешить `\\wsl$\Ubuntu\usr\bin\node` или добавить правило на весь подсеть WSL.
4. Перезапусти ноду.

**Альтернатива:** переключись на libp2p с bootstrap (см. раздел 5).

### 9.2. Порт 4321 занят

**Симптом:** при запуске `node src/index.js` видишь `Error: listen EADDRINUSE: 127.0.0.1:4321`.

**Причина:** нода уже запущена в другом терминале (или процесс повис).

**Фикс:**

```bash
# найти процесс
lsof -i :4321
# или
ps aux | grep "node src/index.js"

# убить все ноды
pkill -f "node src/index.js"

# перезапустить
node src/index.js
```

Если не хочешь убивать и нужно запустить вторую ноду — используй `DEMI_HOME` (см. раздел 8) и смени `uiPort` в её `config.json` на `4322`.

### 9.3. Pair code expired

**Симптом:** `Failed: pair code expired` или `rpc timeout` через 10+ минут.

**Причина:** TTL кода — 10 минут. Израсходовал — получай новый.

**Фикс:** попроси собеседника создать новый `pair --new`. Либо создай сам и перешли.

### 9.4. Identity потеряна

**Симптом:** `~/.demi-node/identity.key` исчез (снёс WSL, удалил случайно).

**Фикс:**
- Если делал бэкап (раздел 3) — восстанови копированием из `~/demi-backup`.
- Если нет — создавай новую identity (просто запусти `node src/index.js`, она появится). **Но:** pubkey будет другой, всем peer'ам придётся заново делать pair.

### 9.5. Hyperswarm timeout / очень долго подключается

**Симптом:** после старта ноды 30+ секунд ни одного peer-коннекта.

**Причина:** DHT bootstrap иногда медленный, особенно из-за провайдерских СОРМ-фильтров или двойного NAT (WSL NAT + роутер NAT).

**Фикс:**

```bash
# запуск с debug-логами
DEBUG=hyperswarm:* node src/index.js
```

Смотри что пишет. Если видно `dht: queried N nodes, 0 responded` — CORM/фаервол режет. Переключайся на libp2p.

### 9.6. mDNS не работает в WSL2

**Симптом:** локальное обнаружение на одной сети не работает.

**Причина:** WSL2 использует отдельный NAT-интерфейс, multicast туда не пробрасывается по умолчанию.

**Фикс:** не пытайся использовать mDNS в WSL — полагайся на hyperswarm DHT (он работает через NAT hole-punch) или на libp2p с явным bootstrap-адресом.

### 9.7. WSL2 NAT — peer видит тебя как другой IP

**Симптом:** ничего критичного, просто любопытно. Twoй peer видит твой endpoint как `172.x.x.x` (WSL internal), hole-punching всё равно работает через DHT.

**Фикс:** ничего. Работает как есть.

### 9.8. Нода стартует, но CLI пишет "Node not running on :4321"

**Симптом:** `node bin/demi.js status` → ошибка о недоступности, хотя нода запущена.

**Причина:** CLI и нода используют разные `DEMI_HOME` (разные профили).

**Фикс:** запускай и ноду и CLI без `DEMI_HOME` (или с одинаковым). Или убедись что в `~/.demi-node/config.json` порт не переопределён.

---

## 10. Когда что-то ломается — куда писать

У нас есть Telegram-группа **«Три поросёнка»** (Я + Евгений + Павел + ты). Пиши туда.

### Что приложить, чтобы я быстро помог

1. **Версия node:**
   ```bash
   node --version
   ```

2. **Версия ноды DEMI (коммит):**
   ```bash
   cd ~/demi-node
   git log -1 --oneline
   ```

3. **Последние строки stderr из ноды.** Если запускал в foreground — скопируй последние 30–50 строк. Если в фоне — пришли хвост лога:
   ```bash
   tail -n 50 /tmp/demi.log
   ```

4. **Если проблема в P2P-коннекте** — прогон с debug-логами:
   ```bash
   DEBUG=hyperswarm:* node src/index.js 2>&1 | tee /tmp/demi-debug.log
   ```
   И пришли `/tmp/demi-debug.log` (первые 200 строк).

5. **Что именно пытался сделать** — команда, ожидание, что получил.

🟢 Я обычно отвечаю в течение часа. В крайнем случае — задам уточняющие вопросы и пойдём дальше.

---

## Быстрый чек-лист «всё готово»

- [ ] `node --version` ≥ 18
- [ ] `git clone` прошёл
- [ ] `npm install` без ERROR
- [ ] `node src/index.js` видит `[demi] ready`
- [ ] `node bin/demi.js status` показывает nickname и fingerprint
- [ ] Бэкап `identity.key` сделан и лежит вне WSL
- [ ] В браузере открывается `http://localhost:4321`
- [ ] Получил от меня pair-код → `pair <код>` → видишь `Paired with ...`
- [ ] `send` прошёл, я ответил, `history` показывает обоих

Если все галки — ты полноценный DEMI-оператор. Добро пожаловать в сеть.

---

**Автор:** Дима + Demi
**Обновлено:** 23.04.2026
**Версия гайда:** 1.0
**Версия ноды на момент гайда:** `v0.1.0-alpha.1`
