# Generate Images Bot v10 + Video (poll mode)

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

## Что нового в v10

- Добавлен workflow-файл `Generate Images Bot v10 + Video (poll mode).json`.
- Обновлена логика проверки типа генерации: в условии используется стабильный доступ
  `$('Set User Params').first().json.tg.generation_type` (вместо `.item`).
- Удален дублирующий Telegram-узел ошибки видео (`Error Video1`) для упрощения ветки ошибок.
- Обновлены служебные идентификаторы и layout узлов после переэкспорта workflow из n8n.

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

## Ключи и переменные

### n8n Credentials

- `Telegram Text2Image2Video` — Telegram Bot API token.
- `OpenRouter for Image-Video Generator` — OpenRouter API key.
- `Bearer Auth MinMax` — MiniMax API bearer token.
- `Supabase for ImageVideoGenerator` — Supabase URL + API key (server-side).
- `Header Auth for ImageVideoGenerator` — заголовки для Supabase Storage API.

### ENV-переменные

- `TELEGRAM_BOT_TOKEN` — токен Telegram бота.
- `SUPABASE_URL` — URL проекта Supabase.
- `SUPABASE_ANON_KEY` — только для публичных клиентских сценариев (если нужны).
- `SUPABASE_SERVICE_ROLE_KEY` — только backend/n8n, никогда в клиенте.
- `OPENROUTER_API_KEY` — ключ OpenRouter.
- `MINIMAX_API_KEY` — ключ MiniMax.

### Правила безопасности для ключей

- Не хранить секреты в `README`, workflow JSON и исходном коде.
- Использовать placeholders: `YOUR_SUPABASE_URL`, `YOUR_API_KEY`.
- Добавить `.env*` в `.gitignore`.
- Не отправлять ключи в Telegram, логи и тексты ошибок.
- Делать ротацию ключей регулярно или при подозрении на утечку.

## ToDo (Supabase: безопасность, очистка, оптимизация)

### 1) Безопасность Storage

- [x] Проверить статус bucket `photos` (`public` true/false).
- [x] Переключить bucket `photos` в приватный режим (`public = false`).
- [X] Убрать любые публичные URL для файлов (если есть).
- [ ] Использовать только signed URL с коротким TTL (60-300 сек) для временного доступа.
- [ ] Стандартизировать пути объектов: `photos/user_<telegram_id>/<timestamp>_<filename>`.

### 2) Политики доступа (RLS / Storage)

- [ ] Включить/проверить RLS для `storage.objects`.
- [ ] Добавить policy на `SELECT` только для файлов текущего пользователя.
- [ ] Добавить policy на `INSERT` только в префикс текущего пользователя.
- [ ] Добавить policy на `DELETE` только своих файлов.
- [ ] Проверить, что workflow не позволяет читать/удалять чужие `image_name`.

### 3) Ключи и секреты

- [ ] Использовать `service_role` только в backend/n8n.
- [ ] Не использовать `service_role` в клиенте/фронте/боте у пользователя.
- [ ] Хранить ключи только в credentials/env (не в workflow JSON и не в git).
- [ ] Ротация ключей раз в N месяцев или при подозрении на утечку.
- [ ] Проверить, что в логах/ошибках не печатаются ключи.

### 4) Очистка истории и Storage

- [ ] Оставить soft-delete (`image_status = 'cleared'`) для UX.
- [ ] Добавить hard-delete job (cron): удалять файлы из Storage для `cleared`.
- [ ] Добавить TTL для старых записей (например 30 дней) и автоочистку.
- [ ] После удаления файлов удалять строки из `ImageHistory`.
- [ ] Добавить dry-run режим очистки для безопасного теста.

### 5) Оптимизация БД

- [ ] Проверить уникальный индекс `ImageUsers(telegram_id)`.
- [ ] Добавить индекс `ImageHistory(user_id, image_status, created_at desc)`.
- [ ] Ограничить число активных элементов контекста на пользователя (например 5-10).
- [ ] Периодически архивировать/удалять старые `result`, если они не нужны.

### 6) Наблюдаемость и контроль

- [ ] Логировать размер загружаемых файлов и итоговый расход Storage.
- [ ] Добавить алерт при резком росте Storage/числа объектов.
- [ ] Добавить алерт при частых ошибках Supabase API.
- [ ] Добавить дашборд: активные пользователи, число генераций, объем хранения.

## Установка и настройка

1. Импортируйте в n8n файл `Generate Images Bot v10 + Video (poll mode).json`.
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
- В `v10 (poll mode)` для Storage-запросов используется `Header Auth for ImageVideoGenerator`; credential `VseLLM Images` может встречаться в экспорте как неактивный legacy-след.

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
