# PROJECT_CONTEXT.md — Калькулятор пакетов EVA (Telegram Bot)

## Обзор проекта
Калькулятор расчёта стоимости пакетов-слайдеров EVA, встроенный в Telegram бота.
Клиенты открывают калькулятор через кнопку в Telegram канале, заполняют параметры и копируют расчёт для отправки менеджеру.

## Ключевые данные

| Параметр | Значение |
|----------|----------|
| GitHub репо | `https://github.com/andkuv001-ui/calaulator-pak.git` |
| Web App URL | `https://andkuv001-ui.github.io/calaulator-pak/` |
| Bot username | `@calculator_eva_bot` |
| Bot Token | `8915403878:AAHvHUgEEyQyWqhEbW6NmwRSvwDmTEphdqw` |
| Канал | `@ZipDoy` (ZIP & DOY | Продажа упаковки) |
| User ID (andkuv1) | `385207085` |
| User ID (andkuv001) | `5477173725` |
| Chat ID канала | `-1002424392113` |
| ADMIN_CHAT_ID | `385207085` |
| Менеджер | `@andkuv1` |

## Структура файлов

```
калькулятор клиентский пакеты ЕВА/
├── index.html          # Калькулятор (Telegram Web App) — ~290 строк
├── bot.py              # Telegram бот — 192 строки
├── requirements.txt    # python-telegram-bot==21.3, python-dotenv==1.0.1
├── Dockerfile          # Docker-образ для деплоя на Coolify
├── Procfile            # worker: python bot.py (для Render/Coolify)
├── runtime.txt         # python-3.11.9
├── .env                # BOT_TOKEN=..., CHANNEL_CHAT_ID=..., ADMIN_CHAT_ID=385207085
├── .env.example        # Шаблон
├── .gitignore          # .env, __pycache__, *.pyc, .DS_Store, venv/
└── PROJECT_CONTEXT.md  # Этот файл
```

## Как запустить локально

```bash
cd "/Users/andrejkuvsinov/Desktop/калькул клиентский без печати"
pip3 install -r requirements.txt
python3 bot.py
```

## Деплой (Coolify на VPS)

Бот развёрнут через **Coolify** на VPS и работает 24/7.

### Настройки Coolify
- **Build Pack**: Dockerfile
- **Dockerfile Location**: /Dockerfile
- **Port**: не нужен (бот работает через polling, не через веб-сервер)

### Environment Variables (в Coolify)
| Переменная | Значение |
|---|---|
| `BOT_TOKEN` | `8915403878:AAHvHUgEEyQyWqhEbW6NmwRSvwDmTEphdqw` |
| `CHANNEL_CHAT_ID` | `-1002424392113` |
| `ADMIN_CHAT_ID` | `385207085` |

## Архитектура бота (bot.py)

### Команды
- `/start` — только в личных сообщениях (PRIVATE). Показывает ReplyKeyboard с кнопкой Web App.
- `/post` — работает везде:
  - **Личка** → публикует калькулятор в канал `@ZipDoy` и закрепляет
  - **Группа** → админ публикует калькулятор в группу
  - **Канал** → админ публикует калькулятор в канал

### Три типа клавиатур
1. **ReplyKeyboardMarkup** (личные сообщения) — кнопка внизу экрана, `web_app=WebAppInfo`
2. **InlineKeyboardButton с web_app** (группы) — кнопка прямо под сообщением, `web_app=WebAppInfo`
3. **InlineKeyboardButton с url** (каналы) — кнопка открывает URL в браузере, `url=WEB_APP_URL`

> **Важно:** В каналах Telegram НЕ поддерживаются `web_app` кнопки (`Button_type_invalid`). Используется обычная URL-кнопка.

### Обработка WebApp данных
- `handle_web_app_data` — получает данные из калькулятора
- В личке: отправляет подтверждение клиенту + пересылает в `ADMIN_CHAT_ID`
- В группе/канале: публикует заявку прямо там

### Фильтры
```python
CommandHandler("start", ..., filters=filters.ChatType.PRIVATE)
CommandHandler("post", post_calculator)  # без фильтра — работает везде
MessageHandler(filters.StatusUpdate.WEB_APP_DATA, handle_web_app_data)
```

## Калькулятор (index.html)

### Продукт
Пакети-слайдеры EVA матовые двух толщин:
- **EVA матовые 70 мкм** — вертикальные (24 размера) + горизонтальные (2 размера)
- **EVA матовые 60 мкм** — вертикальные (24 размера) + горизонтальные (4 размера)

### Данные (hardcode в JS)
- `BAGS` — цены пакетов по типам (slider_eva_70, slider_eva_60), ориентациям (vertical, horizontal) и размерам
- `TYPES` — 2 типа пакетов
- Нет печати, нет доп. услуг — простой расчёт: цена за шт. × количество

### Особенности расчёта
- Единая цена за штуку (нет градации по тиражу)
- Минимальный заказ: 100 шт.
- Максимум: 99 999 шт.
- При итого >= 30 000 ₽ — скидка устанавливается индивидуально (табло)

### Telegram Web App интеграция
```javascript
var tgApp = window.Telegram && window.Telegram.WebApp;
if (tgApp) { tgApp.ready(); tgApp.expand(); }
```

### Кнопка «Скопировать расчёт»
- В Telegram Web App: `tgApp.sendData(text)` → бот получает заявку
- В браузере (канал): fallback на `navigator.clipboard.writeText(text)`
- Текст расчёта содержит: тип, ориентацию, размер, количество, итого + контакт менеджера `@andkuv1`

### Warning-блоки
- 🚚 Доставка не включена в стоимость
- 💰 При заказе от 30 000 ₽ — индивидуальные скидки

## Известные особенности

1. **Канал vs Браузер**: В канале калькулятор открывается в браузере (URL-кнопка), поэтому `sendData()` не работает — текст копируется в буфер, клиент отправляет в личку менеджеру
2. **GitHub Pages**: Калькулятор хостится на GitHub Pages
3. **Бот работает 24/7**: Развёрнут через Coolify на VPS (Dockerfile, polling)
4. **Канал не группа**: Каналы обрабатываются через `channel_post` updates
5. **ADMIN_CHAT_ID**: Настроен на `385207085` (andkuv1) — бот пересылает заявки из лички
6. **`/post` в личке**: Бот принимает `/post` в личном чате и публикует калькулятор в канал `@ZipDoy` с закреплением
7. **`drop_pending_updates=True`**: При запуске бот сбрасывает старые необработанные обновления

## Связанные проекты

| Проект | Репо | Бот | Описание |
|--------|------|-----|----------|
| Калькулятор пакетов с печатью | `calculator-klient` | `@calculator_klient_bot` | Пакеты с печатью (4 типа, доп. услуги, градация по тиражу) |
| **Калькулятор пакетов EVA** | `calaulator-pak` | `@calculator_eva_bot` | Пакеты-слайдеры EVA (2 типа, без печати) |

## Git история

```
80c9226 Add Dockerfile for Coolify deployment
dad3178 Add Procfile and runtime.txt for Render deployment
cd392fa fix: remove send button, keep only copy to clipboard with manager contact
b659f56 fix: add ADMIN_CHAT_ID, improve clipboard feedback with manager contact
50f0f2a fix: update Web App URL to match repo name calaulator-pak
92aae88 feat: Telegram Web App calculator for EVA slider bags (initial)
```

## Что было решено

- **Button_type_invalid**: В каналах нельзя использовать `web_app` кнопки — заменено на `url`
- **Одна кнопка**: Убрана кнопка «Отправить заявку менеджеру», оставлена только «Скопировать расчёт» — в канале sendData() не работает
- **Контакт менеджера**: В скопированный текст добавлен `@andkuv1` чтобы клиент знал куда отправить
- **ADMIN_CHAT_ID**: Настроен для пересылки заявок из лички в админский чат
- **`/post` в личке**: Добавлена возможность публиковать калькулятор в канал из личного чата с ботом (удобнее, чем искать команду в канале)
- **`drop_pending_updates`**: При запуске бот сбрасывает старые обновления, чтобы не обрабатывать «зависшие» команды
- **Деплой 24/7**: Бот развёрнут через Coolify на VPS с Dockerfile, работает постоянно
