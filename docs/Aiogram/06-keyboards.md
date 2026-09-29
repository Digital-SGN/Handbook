# 6. Клавиатуры

Клавиатура — это способ дать пользователю выбор без необходимости печатать текст. В Telegram их два типа, и они решают разные задачи.

## Два типа клавиатур

| Характеристика | Reply-клавиатура | Inline-клавиатура |
|---|---|---|
| Где отображается | Внизу экрана, вместо системной | Прикреплена к сообщению |
| Что отправляет при нажатии | Текст как обычное сообщение | `callback_query` с `callback_data` |
| Исчезает ли при прокрутке | Нет, видна всегда | Да, уходит вместе с сообщением |
| Можно ли поставить несколько | Нет, одна на чат | Да, у каждого сообщения своя |
| Для чего лучше | Навигация, главное меню | Контекстные действия |

Запомните главное отличие: **Reply-клавиатура отправляет текст**, **Inline-клавиатура отправляет callback**. Отсюда — все остальные различия.

## Reply-клавиатура

Reply-клавиатура заменяет системную клавиатуру внизу экрана. Когда пользователь нажимает на кнопку, её текст отправляется в чат как обычное сообщение.

```python
from aiogram.types import ReplyKeyboardMarkup, KeyboardButton

kb = ReplyKeyboardMarkup(
    keyboard=[
        [KeyboardButton(text="Профиль")],
        [KeyboardButton(text="Мероприятия"), KeyboardButton(text="Помощь")],
    ],
    resize_keyboard=True,
)
```

Reply-клавиатура состоит из рядов кнопок. Каждый внутренний список — это один ряд. Параметр `resize_keyboard=True` делает кнопки компактнее — без него они растягиваются на всю ширину.

Отправить клавиатуру пользователю:

```python
@router.message(CommandStart())
async def cmd_start(message: Message):
    await message.answer("Выбери раздел:", reply_markup=kb)
```

Reply-клавиатура остаётся видимой, пока её не уберут или не заменят. Её удобно использовать для навигации: постоянное меню с командами, которые всегда под рукой.

### Обработка нажатий

Так как Reply-кнопка отправляет текст, обрабатывается она как обычное сообщение:

```python
@router.message(F.text == "Профиль")
async def show_profile(message: Message):
    await message.answer("Твой профиль: ...")

@router.message(F.text == "Мероприятия")
async def show_events(message: Message):
    await message.answer("Список мероприятий: ...")
```

Именно поэтому `F.text == "Профиль"` — стандартный способ ловить нажатия Reply-кнопок.

### Особенности

**Нельзя отправить клавиатуру отдельно от сообщения.** Reply-клавиатура всегда прикреплена к сообщению, после которого она появляется.

**Кнопки могут быть разного размера.** Обычная кнопка занимает всю ширину ряда. Чтобы кнопки делили ряд, просто поместите их в один список.

**Reply-клавиатура не защищена от подделки.** Пользователь может ввести текст, совпадающий с текстом кнопки, вручную — и хендлер сработает. Для критичных действий используйте Inline-кнопки.

### Удаление

Чтобы убрать Reply-клавиатуру, отправьте `ReplyKeyboardRemove()`:

```python
from aiogram.types import ReplyKeyboardRemove

@router.message(Command("hide"))
async def cmd_hide(message: Message):
    await message.answer("Клавиатура скрыта", reply_markup=ReplyKeyboardRemove())
```

После этого у пользователя вернётся системная клавиатура.

## Inline-клавиатура

Inline-клавиатура прикрепляется к конкретному сообщению. При нажатии кнопки текст **не отправляется** в чат — вместо этого бот получает `callback_query` с указанными данными.

```python
from aiogram.utils.keyboard import InlineKeyboardBuilder

builder = InlineKeyboardBuilder()
builder.button(text="Записаться", callback_data="register:42")
builder.button(text="Отменить", callback_data="cancel:42")
builder.adjust(2)

kb = builder.as_markup()
```

`builder.adjust(2)` означает: укладывать по 2 кнопки в ряд. Можно управлять раскладкой точнее:

```python
builder.adjust(1)        # все кнопки в столбик
builder.adjust(3)        # по 3 в ряд
builder.adjust(2, 1)     # первый ряд — 2 кнопки, второй — 1
```

Отправить Inline-клавиатуру:

```python
await message.answer("Выбери действие:", reply_markup=kb)
```

Inline-клавиатура удобна для контекстных действий: кнопка «Записаться» рядом с конкретным мероприятием, кнопка «Удалить» рядом с конкретной записью. Пользователь видит действие и сразу может его выполнить.

### Обработка нажатий

Inline-кнопки обрабатываются через `callback_query`:

```python
from aiogram.types import CallbackQuery

@router.callback_query(F.data.startswith("register:"))
async def on_register(callback: CallbackQuery):
    event_id = int(callback.data.split(":")[1])
    await callback.answer("Записал!")
    await callback.message.edit_text("Ты записан на мероприятие")
```

**Важно:** всегда вызывайте `callback.answer()`, иначе у пользователя на кнопке будут висеть «часики».

### Редактирование сообщения

Одно из главных преимуществ Inline-кнопок — можно менять сообщение прямо на месте. Пользователь нажимает кнопку — текст сообщения обновляется, а не появляется новое.

```python
@router.callback_query(F.data == "next")
async def on_next(callback: CallbackQuery):
    await callback.message.edit_text(
        "Страница 2",
        reply_markup=new_keyboard,
    )
    await callback.answer()
```

Полезные методы:

- `callback.message.edit_text(...)` — заменить текст.
- `callback.message.edit_reply_markup(...)` — заменить только клавиатуру.
- `callback.message.delete()` — удалить сообщение.

Если попытаться изменить текст на такой же — Telegram вернёт ошибку `message is not modified`. Оборачивайте такие вызовы в `try/except` или проверяйте, что текст действительно другой.

### Специальные кнопки

Кроме обычных callback-кнопок, Inline-клавиатура поддерживает:

**URL-кнопка** — открывает ссылку:

```python
from aiogram.types import InlineKeyboardButton

builder.button(text="Наш сайт", url="https://example.com")
```

**WebApp-кнопка** — открывает мини-приложение:

```python
builder.button(text="Открыть приложение", web_app=WebAppInfo(url="https://app.example.com"))
```

**Switch Inline** — переключает пользователя в другой чат:

```python
builder.button(text="Написать в поддержку", switch_inline_query="помогите")
```

**Pay** — для платежей через Telegram Stars.

В `InlineKeyboardBuilder` для них есть отдельные методы: `.button(...)` универсальный, но можно и явно: `.url(...)`, `.web_app(...)`, `.switch_inline_query(...)`.

## callback_data

`callback_data` — это строка, которую Telegram присылает боту при нажатии. У неё есть лимит в **64 байта**. Если данных больше, используют короткие идентификаторы или JSON в укороченном виде.

Часто используют формат `action:entity_id`. Например:

- `register:42` — записаться на мероприятие 42.
- `cancel:42` — отменить регистрацию.
- `profile:edit` — редактировать профиль.
- `page:3` — перейти на страницу 3.

В хендлере эти данные разбираются:

```python
@router.callback_query(F.data.startswith("register:"))
async def on_register(callback: CallbackQuery):
    event_id = int(callback.data.split(":")[1])
    await callback.answer("Записал!")
```

### Если данных больше 64 байт

Часто хочется положить в `callback_data` несколько полей — например,
ID пользователя, ID мероприятия и действие. Если строка не влезает,
используют один из двух приёмов.

**Приём 1: короткий идентификатор.** Генерируем случайный токен,
сохраняем все данные на сервере, в кнопке — только токен.

```python
import secrets

token = secrets.token_urlsafe(8)  # ~11 символов
await cache.set(f"cb:{token}", {"event_id": 42, "action": "register"}, ttl=3600)
builder.button(text="Записаться", callback_data=f"cb:{token}")
```

В хендлере:

```python
@router.callback_query(F.data.startswith("cb:"))
async def on_action(callback: CallbackQuery):
    token = callback.data.split(":", 1)[1]
    payload = await cache.get(f"cb:{token}")
    if payload is None:
        await callback.answer("Кнопка устарела", show_alert=True)
        return
    # payload["event_id"], payload["action"] — используем
```

Плюс: влезает что угодно. Минус: нужен кеш (Redis), данные живут ограниченное время.

**Приём 2: сжатие полей.** Если все поля — числа, их можно упаковать
в компактную строку через `base64` или просто через разделитель.

```python
import base64
import struct

# 4 байта user_id + 4 байта event_id + 1 байт action = 9 байт → ~12 в base64
packed = base64.urlsafe_b64encode(
    struct.pack(">IIB", user_id, event_id, action_code)
).decode()
builder.button(text="Записаться", callback_data=packed)
```

Это работает без внешнего хранилища, но код сложнее и требует
аккуратной распаковки с проверкой длины. Для большинства ботов
первый приём проще и надёжнее.

### Валидация callback_data

`callback_data` — это **пользовательский ввод**. Её можно подделать или испортить. Всегда проверяйте данные перед использованием:

```python
@router.callback_query(F.data.startswith("register:"))
async def on_register(callback: CallbackQuery, db):
    try:
        event_id = int(callback.data.split(":")[1])
    except (IndexError, ValueError):
        await callback.answer("Некорректные данные", show_alert=True)
        return

    # Проверяем, что мероприятие существует
    event = await db.fetchrow("SELECT id FROM events WHERE id = $1", event_id)
    if not event:
        await callback.answer("Мероприятие не найдено", show_alert=True)
        return

    # ... записываем
    await callback.answer("Записал!")
```

Это чуть многословнее, но защищает от подделки callback'ов и ошибок в коде.

## Что выбрать

**Reply-клавиатура — для навигации.** Постоянные разделы бота, главное меню, переходы между режимами. Она видна всегда, поэтому подходит для глобальных действий.

**Inline-клавиатура — для конкретных действий.** Реакция на конкретное сообщение, работа с конкретной сущностью. Она исчезает, когда пользователь пролистывает чат вверх, поэтому не подходит для постоянных меню.

Часто оба типа используются вместе: Reply-клавиатура как основное меню, Inline — как кнопки внутри сообщений.

Пример гибридного бота:

```python
@router.message(CommandStart())
async def cmd_start(message: Message):
    # Reply-клавиатура: постоянное меню
    await message.answer("Главное меню:", reply_markup=main_menu_kb)

@router.message(F.text == "Мероприятия")
async def show_events(message: Message):
    # Inline-клавиатура: кнопки рядом с конкретными событиями
    for event in await get_events():
        builder = InlineKeyboardBuilder()
        builder.button(text="Записаться", callback_data=f"register:{event.id}")
        await message.answer(
            f"<b>{event.title}</b>\n{event.description}",
            reply_markup=builder.as_markup(),
        )
```

##  Совет

Не пихайте в одну клавиатуру десять кнопок — Telegram отобразит их некрасиво, а пользователю будет сложно ориентироваться. Лучше разбивайте на несколько экранов с навигацией «Назад / Вперёд».

И ещё: для Inline-клавиатур используйте `InlineKeyboardBuilder` — он гораздо удобнее ручного конструирования `InlineKeyboardMarkup` из списка списков. `adjust()` решает почти все задачи раскладки.
