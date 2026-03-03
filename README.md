# Generate Images Bot v9 + Video (poll mode)

Telegram-бот на n8n для генерации изображений и видео с хранением пользователей, контекста и результатов в Supabase.

## Архитектура

```
Telegram Bot
    ↓
n8n Workflow (router + image/video pipelines)
    ├─ OpenRouter API (image generation/edit)
    ├─ MiniMax API (video generation, poll mode)
    └─ Supabase (Postgres + Storage)
```

## Что нового в v9

- Добавлен workflow-файл `Generate Images Bot v9 + Video (poll mode) (1).json`.
- Добавлены режимы `Text2Video` и `Image2Video` через MiniMax (`MiniMax-Hailuo-02`).
- Реализован polling статуса видео (`Generate Video` -> `Wait 30s` -> `Query Video Status`).
- Добавлен прогресс-бар генерации видео в Telegram-сообщении (редактирование одного сообщения).
- Расширена схема БД:
  - `ImageUsers.generation_type` (`image`/`video`)
  - `ImageHistory.image_model` (например, `image: openai/gpt-5-image-mini`).
- Главное меню теперь включает выбор типа генерации (image/video) и моделей.

## Возможности

- Генерация изображений по тексту.
- Редактирование изображений с учетом активного контекста.
- Загрузка фото в контекст с сохранением в Supabase Storage (`photos`).
- Генерация видео:
  - `Text2Video` (по промпту),
  - `Image2Video` (по промпту + первое изображение из контекста).
- Очистка активного контекста (`/clear_context` и кнопка `Новая генерация`).
- Выбор модели прямо в inline-меню.
- Сохранение результатов (image/video) в `ImageHistory`.

## Команды бота

| Команда | Описание |
|---------|----------|
| `/start` | Регистрация/проверка пользователя + установка команд через `setMyCommands` |
| `/menu` | Показ главного меню |
| `/clear_context` | Очистка активных записей контекста/результатов |
| `/help` | Справка *(ветка есть, полноценный help пока не заполнен)* |

## Inline-кнопки

| Кнопка | Действие |
|--------|----------|
| Новая генерация | Очищает активный контекст, переключает на `Fast` |
| Продолжить редактирование | Закрывает меню и возвращает в текущий контекст |
| ChatGPT mini | `openai/gpt-5-image-mini` + `generation_type=image` |
| Nano Banana | `google/gemini-2.5-flash-image` + `generation_type=image` |
| Nano Banana Pro | `google/gemini-3-pro-image-preview` + `generation_type=image` |
| Fast | `black-forest-labs/flux.2-klein-4b` + `generation_type=image` |
| Text2Video | `MiniMax-Hailuo-02` + `generation_type=video` |
| Image2Video | `MiniMax-Hailuo-02` + `generation_type=video` |

## Используемые модели

### Image

| Модель | Назначение |
|--------|-----------|
| `openai/gpt-5-image-mini` | Быстрая недорогая генерация |
| `google/gemini-2.5-flash-image` | Универсальная генерация/редактирование |
| `google/gemini-3-pro-image-preview` | Продвинутая генерация/редактирование |
| `black-forest-labs/flux.2-klein-4b` | Быстрый режим (default для нового пользователя) |

### Video

| Модель | Назначение |
|--------|-----------|
| `MiniMax-Hailuo-02` | Генерация видео (t2v/i2v) с polling статуса |

## Структура базы данных (Supabase)

### Таблица `ImageUsers`

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | int | Первичный ключ |
| `telegram_id` | string | Telegram ID пользователя |
| `user_name` | string | Username пользователя |
| `last_sign_in_at` | timestamp/string | Время последнего `/start` |
| `model` | string | Текущая выбранная модель |
| `generation_type` | string | `image` или `video` |

### Таблица `ImageHistory`

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | int | Первичный ключ |
| `user_id` | int | FK -> `ImageUsers.id` |
| `prompt` | string | Промпт пользователя |
| `image_id` | string | ID объекта в Supabase Storage |
| `image_name` | string | Key объекта в bucket `photos` |
| `image_status` | string | `active` или `cleared` |
| `image_type` | string | `context` или `result` |
| `usage` | json/string | Usage от OpenRouter (для image-результатов) |
| `image_model` | string | Снимок режима и модели (`<generation_type>: <model>`) |

## Поток обработки (коротко)

1. `Telegram Trigger` принимает `message`/`callback_query`.
2. `Set Params` нормализует входные данные.
3. Проверяется пользователь в `ImageUsers`:
   - `/start` создает пользователя (если нет) и выставляет Telegram-команды.
4. Роутинг:
   - команды: `/start`, `/menu`, `/clear_context`, `/help`;
   - типы событий: `photo`, `text`, `file`, `callback`.
5. Ветка image:
   - загружается активный контекст из `ImageHistory`,
   - изображения конвертируются в base64,
   - отправляется запрос в OpenRouter,
   - результат отправляется в Telegram и сохраняется в Storage + `ImageHistory`.
6. Ветка video:
   - формируется payload для MiniMax,
   - создается task (`/video_generation`),
   - выполняется polling статуса (`Wait 30s`, до лимита попыток),
   - при успехе видео отправляется в Telegram и сохраняется в Storage + `ImageHistory`.

## Необходимые credentials в n8n

| Credential | Тип | Назначение |
|-----------|-----|-----------|
| `Telegram Text2Image2Video` | Telegram API | Основной бот |
| `OpenRouter for Image-Video Generator` | OpenRouter API | Генерация изображений |
| `Bearer Auth MinMax` | HTTP Bearer Auth | Генерация/опрос статуса видео MiniMax |
| `Supabase for ImageVideoGenerator` | Supabase API | Работа с таблицами |
| `Header Auth for ImageVideoGenerator` | HTTP Header Auth | Доступ к Supabase Storage |

Также нужен env-параметр:

- `TELEGRAM_BOT_TOKEN` — используется узлом `Create main menu` для вызова `setMyCommands`.

## Установка и настройка

1. Импортируйте в n8n файл `Generate Images Bot v9 + Video (poll mode) (1).json`.
2. Создайте Telegram-бота через [@BotFather](https://t.me/BotFather).
3. Создайте проект Supabase:
   - таблицы `ImageUsers` и `ImageHistory`,
   - bucket `photos`.
4. Получите ключи:
   - OpenRouter API key,
   - MiniMax API key.
5. Настройте credentials в n8n (из таблицы выше).
6. Добавьте env-переменную `TELEGRAM_BOT_TOKEN`.
7. Активируйте workflow.

## Примечания по текущей реализации

- Для image-ответа в Telegram отправляется только первое изображение из массива ответа модели.
- При `/clear_context` workflow очищает активные записи и возвращает модель `Fast`.
- Для video используется polling-режим с паузой `30s` между запросами статуса.
- В `v9 (poll mode)` для Storage-запросов используется `Header Auth for ImageVideoGenerator`; credential `VseLLM Images` может встречаться в экспорте как неактивный legacy-след.

## Статус

- [x] Регистрация пользователя и автокоманды на `/start`
- [x] Контекст из загруженных изображений
- [x] Генерация изображений с выбором модели
- [x] Генерация видео (`Text2Video`/`Image2Video`) через MiniMax
- [x] Polling статуса видео с прогрессом в Telegram
- [x] Сохранение context/result в Supabase
- [x] Очистка активного контекста (`/clear_context`)
- [x] Базовая обработка ошибок (image/video)
- [ ] Полноценное наполнение `/help`
