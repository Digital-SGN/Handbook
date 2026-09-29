<div align="center">
  <img width="1000" height="540" alt="image" src="https://github.com/user-attachments/assets/d513a8a9-fdee-498e-8aba-d7ec0e61c1bb" />

</div>

# Руководство по Telegram-ботам на aiogram

Полный курс по созданию Telegram-ботов на Python с использованием aiogram: от первого бота до продакшена.

## Содержание

### Введение

0. [Быстрый старт: первый бот за 10 минут](00-quickstart.md)
   — Установка aiogram, токен у BotFather, `.env`, минимальный эхо-бот, типичные ошибки при запуске.
   
### Основы

1. [Что такое Telegram-бот и как он работает](01-what-is-a-bot.md)
   — Bot API, Update, Polling vs Webhook, BotFather и токен.
2. [Асинхронность и asyncio](02-asyncio-basics.md)
   — Event loop, `async`/`await`, что блокирует loop, `asyncio.gather`.
3. [Архитектура aiogram: Bot, Dispatcher, Router](03-architecture.md)
   — Три ключевых объекта, иерархия роутеров, точка входа.

### Логика бота

4. [Хендлеры и фильтры](04-handlers-and-filters.md)
   — Виды событий, встроенные и кастомные фильтры, порядок проверки.
5. [Магические фильтры F](05-magic-filters.md)
   — Условия в декораторе, комбинации, частые ошибки.
6. [Клавиатуры](06-keyboards.md)
   — Reply vs Inline, `callback_data`, валидация.
7. [Машина состояний FSM](07-fsm.md)
   — Пошаговые диалоги, `StatesGroup`, Redis как хранилище.
8. [Мидлвари](08-middlewares.md)
   — Outer vs inner, антифлуд, инъекция зависимостей, проверка подписки.

### Данные и автоматизация

9. [Работа с базой данных](09-database.md)
   — PostgreSQL, asyncpg, пул соединений, SQLAlchemy, миграции.
10. [Планировщик задач](10-scheduler.md)
    — APScheduler, триггеры, идемпотентность, мониторинг.
11. [Деплой](11-deploy.md)
    — Amvera, Vercel, VPS, Docker, systemd, секреты.

### Дополнительно

12. [Форматирование текста и отправка медиа](12-formatting-and-media.md)
    — HTML-разметка, `FSInputFile`, альбомы, скачивание файлов.
13. [Тестирование ботов](13-testing.md)
    — Unit-тесты, `feed_raw_update`, моки, CI/CD.
14. [Безопасность](14-security.md)
    — Токен, SQL-инъекции, `callback_data`, secret_token, чек-лист.
15. [Продвинутые темы](15-advanced-topics.md)
    — Inline-режим, Mini Apps, платежи, локализация, микросервисы.
16. [Собираем своего бота](16-building-bot.md)
    — Практическая глава, где мы создадим своего бота для регистраций, используя все прошлые знания.

## Полезные ссылки

- [Документация aiogram](https://docs.aiogram.dev/)
- [Примеры aiogram на GitHub](https://github.com/aiogram/aiogram/tree/dev-3.x/examples)
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [Awesome aiogram](https://github.com/aiogram/awesome-aiogram)
