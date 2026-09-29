# 2. Асинхронность и asyncio

Бот постоянно ждёт: сообщений от пользователей, ответов от базы, ответов от Telegram. Если бы он ждал каждого ответа последовательно, то, пока один пользователь скачивает файл, все остальные висели бы в очереди. Асинхронность решает эту проблему.

Асинхронная программа умеет **приостанавливать** выполнение одной задачи, пока она ждёт чего-то (например, ответа по сети), и в это время выполнять другую задачу. Один поток, один процесс — но задачи переключаются между собой так быстро, что для пользователя это выглядит как параллельная работа.

## Event loop

<div align="center">
    <img width="400" height="240" alt="image" src="https://github.com/user-attachments/assets/df09753f-160e-4d5b-85ff-c99f32b376e0" />
</div>

В основе всего — **event loop** (цикл событий). Это бесконечный цикл, который следит за задачами и решает, какую из них запустить прямо сейчас. Когда задача встречает `await`, она говорит event loop'у: «Я жду результата, займись чем-нибудь другим». Loop переключается на другую задачу. Когда результат готов — возвращается к первой.

Схематично это выглядит так:

<div align="center">
    <img width="700" height="400" alt="image" src="https://github.com/user-attachments/assets/2d667d0e-55ee-4a0d-b036-edfdd3535bd2" />
</div>


Важная деталь: event loop **однопоточный**. Всё выполняется в одном потоке, просто задачи чередуются. Это значит, что **если вы заблокируете поток, встанет всё**.

## async и await

Для асинхронности в Python есть два ключевых слова. `async def` создаёт **корутину** — функцию, которую можно приостанавливать. `await` — это точка, где функция отдаёт управление event loop'у и ждёт результата другой корутины.

```python
import asyncio

async def say_hello():
    print("Привет")
    await asyncio.sleep(1)
    print("Мир")

asyncio.run(say_hello())
```

`asyncio.run()` запускает event loop, выполняет корутину и закрывает loop, когда та завершится. В реальном боте `asyncio.run()` вызывается один раз — в `main()`, а внутри уже запускается polling, который крутится бесконечно.

## Что блокирует event loop

Главная ловушка для новичков: кажется, что если функция асинхронная, то всё хорошо. Но это не так. Если внутри корутины вызвать **синхронную** функцию, которая выполняется долго, event loop встанет.

Вот что **нельзя** делать внутри хендлеров:

```python
import time
import requests
import psycopg2

@router.message(Command("bad"))
async def bad_handler(message: Message):
    time.sleep(5)                      # блокирует loop на 5 секунд
    response = requests.get("...")     # блокирует loop на время запроса
    conn = psycopg2.connect("...")     # синхронный драйвер БД
    heavy_computation()                # тяжёлые вычисления
```

Пока выполняются эти вызовы, **бот не отвечает никому** — ни этому пользователю, ни остальным. Внешне это выглядит как «бот завис».

## Как не блокировать loop

Для каждой блокирующей операции есть асинхронная альтернатива.

| Блокирующий вызов | Асинхронная замена |
|---|---|
| `time.sleep(n)` | `await asyncio.sleep(n)` |
| `requests.get(...)` | `await aiohttp.get(...)` или `httpx.AsyncClient` |
| `psycopg2` | `asyncpg` или `sqlalchemy.ext.asyncio` |
| `open(...).read()` для больших файлов | `await asyncio.to_thread(...)` |
| Тяжёлые вычисления (PIL, ML) | `await asyncio.to_thread(...)` |

asyncio.to_thread() запускает синхронную функцию в отдельном потоке и возвращает управление event loop'у. Это спасение для библиотек, у которых нет асинхронных версий.

**Оговорка про GIL**. Python-потоки не дают настоящего параллелизма для CPU-bound задач: GIL (Global Interpreter Lock) не позволит двум потокам одновременно выполнять Python-код. to_thread() отлично работает для I/O (сеть, диск, БД) — там поток большую часть времени ждёт. Но для тяжёлых вычислений (например, обработка изображений через PIL, обучение модели) используйте ProcessPoolExecutor или выносите задачу в отдельный воркер — иначе потоки будут бороться за GIL, и выигрыша не будет.

```python
import asyncio

def heavy_computation(data):
    # какая-то долгая синхронная логика
    return data * 2

@router.message(Command("compute"))
async def compute_handler(message: Message):
    result = await asyncio.to_thread(heavy_computation, 42)
    await message.answer(f"Результат: {result}")
```

## Параллельный запуск задач

Часто нужно запустить несколько асинхронных операций одновременно. Для этого есть `asyncio.gather()`.

```python
import asyncio

async def fetch_user(user_id: int) -> str:
    await asyncio.sleep(1)  # имитация запроса к БД
    return f"User {user_id}"

async def main():
    # Последовательно — 3 секунды
    u1 = await fetch_user(1)
    u2 = await fetch_user(2)
    u3 = await fetch_user(3)

    # Параллельно — 1 секунда
    results = await asyncio.gather(
        fetch_user(1),
        fetch_user(2),
        fetch_user(3),
    )
    print(results)  # ['User 1', 'User 2', 'User 3']

asyncio.run(main())
```

`asyncio.gather()` запускает все корутины одновременно и ждёт завершения всех. Порядок результатов соответствует порядку аргументов, даже если задачи завершились в другом порядке.

## Фоновые задачи

Иногда нужно запустить задачу и не ждать её — например, отправить уведомление через минуту. Для этого используется `asyncio.create_task()`.

```python
import asyncio
from aiogram import Bot, F
from aiogram.filters import Command
from aiogram.types import Message

async def background_notification(bot: Bot, chat_id: int):
    await asyncio.sleep(5)
    await bot.send_message(chat_id, "Фоновое уведомление!")

@router.message(Command("notify"))
async def cmd_notify(message: Message, bot: Bot):
    asyncio.create_task(background_notification(bot, message.chat.id))
    await message.answer("Уведомление придёт через 5 секунд")
```

Пользователь сразу получает ответ, а фоновая задача продолжает работать независимо. Если она упадёт с ошибкой — вы об этом не узнаете, если не обернёте в `try/except` и не залогируете.

## Типичные ошибки

**Ошибка 1. Забыть `await`.**

```python
async def send_welcome(message: Message):
    message.answer("Привет!")  # вернёт корутину, но не выполнит её
```

Python выдаст предупреждение `coroutine was never awaited`, но код не выполнится. Всегда проверяйте, что перед вызовом асинхронной функции стоит `await`.

**Ошибка 2. Использовать `time.sleep` вместо `asyncio.sleep`.**

Мы уже разобрали это выше, но повторимся — это самая частая ошибка новичков в aiogram. Если бот иногда «зависает» на пару секунд, ищите `time.sleep`.

**Ошибка 3. Запускать долгую задачу через `create_task` без обработки ошибок.**

```python
asyncio.create_task(might_fail())  # ошибки потеряются

# правильно
task = asyncio.create_task(might_fail())
task.add_done_callback(lambda t: t.exception() and logging.error(t.exception()))
```

**Ошибка 4. Создавать event loop вручную.**

```python
loop = asyncio.new_event_loop()  # не нужно
asyncio.set_event_loop(loop)
loop.run_until_complete(main())

# правильно
asyncio.run(main())
```

В aiogram вы вообще не должны управлять loop'ом вручную — фреймворк делает это за вас в `dp.start_polling()`.

## Когда асинхронности недостаточно

Иногда задача настолько тяжёлая (обработка видео, обучение модели), что её нельзя выполнить даже в отдельном потоке — она съест все ресурсы. В таких случаях используют **отдельные воркеры**: бот кладёт задачу в очередь (Redis, RabbitMQ, Celery), а отдельный процесс её обрабатывает. Бот при этом остаётся отзывчивым.

Это выходит за рамки курса, но полезно знать, что такой вариант существует.

**Не спешите с воркерами.** Для бота на несколько сотен или даже тысяч
пользователей очередь задач (Celery, ARQ, RabbitMQ) не нужна — это
оверинжиниринг. Отдельный воркер оправдан, когда:

- рассылка идёт часами и забивает лимиты Telegram;
- задача занимает минуты (обработка видео, обучение модели);
- нужно гарантировать выполнение задачи при падении бота.

Во всех остальных случаях хватит `asyncio.to_thread()` для тяжёлых
синхронных вызовов и `asyncio.create_task()` для фоновых операций.

## Совет

Если сомневаетесь, блокирует ли какая-то функция event loop — оберните её в `asyncio.to_thread()`. Это почти всегда безопасно (кроме случаев, когда функция сама использует event loop), и вы точно не заблокируете бота. Накладные расходы на создание потока минимальны по сравнению с зависшим ботом.
