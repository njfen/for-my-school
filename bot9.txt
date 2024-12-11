import asyncio
import json
import os

from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import Command
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import StatesGroup, State
from aiogram.filters import StateFilter

BOT_TOKEN = "7502227825:AAHojDwaqC33asGh9aesF7Bn0YlT43HGke4"

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher(storage=MemoryStorage())

# Загрузка профилей пользователей
try:
    with open("user_profiles.json", "r", encoding="utf-8") as file:
        user_profiles = json.load(file)
except FileNotFoundError:
    user_profiles = {}

# Загрузка мероприятий
try:
    with open("events.json", "r", encoding="utf-8") as file:
        events_data = json.load(file)
except FileNotFoundError:
    events_data = {"events": []}

admin_user_ids = [925413569]  # Укажите id администраторов

class ContactAdminStates(StatesGroup):
    waiting_for_user_question = State()

class CreateEventStates(StatesGroup):
    waiting_for_title = State()
    waiting_for_dates = State()
    waiting_for_description = State()
    waiting_for_notification_days = State()

class EditEventStates(StatesGroup):
    choose_field = State()
    edit_title = State()
    edit_dates = State()
    edit_description = State()
    edit_notification_days = State()

# Инициализация поля participants в мероприятиях, если его нет
for e in events_data["events"]:
    if "participants" not in e:
        e["participants"] = []

olympiad_info = {
    "rus_lang": """...""",
    # Остальные предметы скрыты для краткости, оставьте их прежними
    "literature": """...""",
    "english": """...""",
    "math": """...""",
    "informatics": """...""",
    "history": """...""",
    "social_studies": """...""",
    "geography": """...""",
    "biology": """...""",
    "chemistry": """...""",
    "physics": """...""",
    "pe": """...""",
    "art": """...""",
    "music": """...""",
    "tech": """...""",
    "obzh": """..."""
}

def main_menu():
    return types.InlineKeyboardMarkup(
        inline_keyboard=[
            [types.InlineKeyboardButton(text="Профиль", callback_data="profile")],
            [types.InlineKeyboardButton(text="Школьные мероприятия", callback_data="school_events")],
            [types.InlineKeyboardButton(text="Олимпиады", callback_data="olympiads")],
            [types.InlineKeyboardButton(text="Связаться с организаторами", callback_data="contact_admins")]
        ]
    )

def save_events():
    with open("events.json", "w", encoding="utf-8") as f:
        json.dump(events_data, f, ensure_ascii=False, indent=4)

@dp.message(Command("start"))
async def cmd_start(message: types.Message):
    user_id = str(message.from_user.id)
    if user_id in user_profiles:
        await message.answer("Вы уже зарегистрированы. Выберите действие:", reply_markup=main_menu())
    else:
        await message.answer("Добро пожаловать! Введите данные в формате: Имя Фамилия Класс")

@dp.message(Command("help"))
async def cmd_help(message: types.Message):
    help_text = (
        "Справка по боту:\n\n"
        "Используйте кнопки меню для навигации.\n"
        "/start - начало.\n"
        "/help - эта справка.\n"
        "Для связи с организаторами нажмите соответствующую кнопку в меню."
    )
    await message.answer(help_text)

@dp.message(Command("reply"))
async def admin_reply(message: types.Message):
    if message.from_user.id not in admin_user_ids:
        await message.answer("У вас нет прав для этой команды.")
        return
    parts = message.text.split(" ", 2)
    if len(parts) < 3:
        await message.answer("Неверный формат. Используйте: /reply <user_id> <текст ответа>")
        return
    user_id, reply_text = parts[1], parts[2]
    await bot.send_message(user_id, f"Ответ от организатора:\n{reply_text}")

@dp.callback_query(F.data == "go_home")
async def go_home(callback_query: types.CallbackQuery):
    await callback_query.message.edit_text("Выберите действие:", reply_markup=main_menu())

@dp.callback_query(F.data == "contact_admins")
async def contact_admins(callback_query: types.CallbackQuery, state: FSMContext):
    kb = types.InlineKeyboardMarkup(inline_keyboard=[
        [types.InlineKeyboardButton(text="В начало", callback_data="go_home")]
    ])
    await callback_query.message.edit_text("Опишите ваш вопрос для организаторов:", reply_markup=kb)
    await state.set_state(ContactAdminStates.waiting_for_user_question)

@dp.message(StateFilter(ContactAdminStates.waiting_for_user_question))
async def handle_admin_question(message: types.Message, state: FSMContext):
    user_id = str(message.from_user.id)
    question = message.text
    await state.clear()
    await message.answer("Ваш вопрос отправлен организаторам. Ожидайте ответа.")
    for admin_id in admin_user_ids:
        await bot.send_message(admin_id, f"Вопрос от {user_id}:\n{question}\nОтветьте командой: /reply {user_id} <ответ>")

@dp.message(StateFilter(None))
async def handle_message(message: types.Message, state: FSMContext):
    user_id = str(message.from_user.id)
    if user_id not in user_profiles:
        try:
            name, surname, grade = message.text.split(" ")
            user_profiles[user_id] = {"name": name, "surname": surname, "grade": grade}
            with open("user_profiles.json", "w", encoding="utf-8") as file:
                json.dump(user_profiles, file, ensure_ascii=False, indent=4)
            await message.answer(f"Данные сохранены! Добро пожаловать, {name}.\nВыберите действие:", reply_markup=main_menu())
        except ValueError:
            await message.answer("Некорректный формат. Введите: Имя Фамилия Класс")
    else:
        await message.answer("Вы уже зарегистрированы. Используйте кнопки для взаимодействия.")

@dp.callback_query(F.data == "profile")
async def show_profile(callback_query: types.CallbackQuery):
    user_id = str(callback_query.from_user.id)
    profile = user_profiles.get(user_id, {})
    if profile:
        profile_buttons = types.InlineKeyboardMarkup(
            inline_keyboard=[
                [types.InlineKeyboardButton(text="Изменить данные", callback_data="edit_profile")],
                [types.InlineKeyboardButton(text="В начало", callback_data="go_home")]
            ]
        )
        await callback_query.message.edit_text(
            f"Ваш профиль:\nИмя: {profile.get('name')}\nФамилия: {profile.get('surname')}\nКласс: {profile.get('grade')}",
            reply_markup=profile_buttons
        )
    else:
        await callback_query.message.edit_text(
            "Профиль не найден. Попробуйте зарегистрироваться.",
            reply_markup=types.InlineKeyboardMarkup(inline_keyboard=[
                [types.InlineKeyboardButton(text="В начало", callback_data="go_home")]
            ])
        )

@dp.callback_query(F.data == "olympiads")
async def olympiads_menu(callback_query: types.CallbackQuery):
    olympiad_buttons = [
        [types.InlineKeyboardButton(text="Русский язык", callback_data="rus_lang")],
        [types.InlineKeyboardButton(text="Литература", callback_data="literature")],
        [types.InlineKeyboardButton(text="Английский язык", callback_data="english")],
        [types.InlineKeyboardButton(text="Математика", callback_data="math")],
        [types.InlineKeyboardButton(text="Информатика", callback_data="informatics")],
        [types.InlineKeyboardButton(text="История", callback_data="history")],
        [types.InlineKeyboardButton(text="Обществознание", callback_data="social_studies")],
        [types.InlineKeyboardButton(text="География", callback_data="geography")],
        [types.InlineKeyboardButton(text="Биология", callback_data="biology")],
        [types.InlineKeyboardButton(text="Химия", callback_data="chemistry")],
        [types.InlineKeyboardButton(text="Физика", callback_data="physics")],
        [types.InlineKeyboardButton(text="Физическая культура", callback_data="pe")],
        [types.InlineKeyboardButton(text="Искусство", callback_data="art")],
        [types.InlineKeyboardButton(text="Музыка", callback_data="music")],
        [types.InlineKeyboardButton(text="Технология", callback_data="tech")],
        [types.InlineKeyboardButton(text="ОБЖ", callback_data="obzh")],
        [types.InlineKeyboardButton(text="В начало", callback_data="go_home")]
    ]
    await callback_query.message.edit_text("Выберите предмет:", reply_markup=types.InlineKeyboardMarkup(inline_keyboard=olympiad_buttons))

@dp.callback_query(F.data.in_(olympiad_info.keys()))
async def show_olympiad_info(callback_query: types.CallbackQuery):
    olympiad = callback_query.data
    text = olympiad_info.get(olympiad, "Информация отсутствует.")
    back_button = types.InlineKeyboardMarkup(
        inline_keyboard=[
            [types.InlineKeyboardButton(text="В начало", callback_data="go_home")]
        ]
    )
    await callback_query.message.edit_text(text, reply_markup=back_button)

def events_menu_buttons(user_id: str):
    buttons = []
    for event in events_data["events"]:
        view_button = types.InlineKeyboardButton(text=event["title"], callback_data=f"view_event:{event['id']}")
        buttons.append([view_button])
    if int(user_id) in admin_user_ids:
        buttons.append([types.InlineKeyboardButton(text="Добавить мероприятие", callback_data="add_event")])
    buttons.append([types.InlineKeyboardButton(text="В начало", callback_data="go_home")])
    return types.InlineKeyboardMarkup(inline_keyboard=buttons)

@dp.callback_query(F.data == "school_events")
async def show_events(callback_query: types.CallbackQuery):
    user_id = str(callback_query.from_user.id)
    if events_data["events"]:
        await callback_query.message.edit_text("Список мероприятий:", reply_markup=events_menu_buttons(user_id))
    else:
        await callback_query.message.edit_text("Мероприятий пока нет.", reply_markup=events_menu_buttons(user_id))

def get_event_by_id(event_id):
    return next((e for e in events_data["events"] if e["id"] == event_id), None)

@dp.callback_query(F.data.startswith("view_event:"))
async def view_event(callback_query: types.CallbackQuery):
    event_id = int(callback_query.data.split(":")[1])
    event = get_event_by_id(event_id)
    if event:
        text = (f"Название: {event['title']}\n"
                f"Даты: {event['dates']}\n"
                f"Описание: {event['description']}\n"
                f"Напоминание за {event.get('notification_days_before', 'не указано')} дн. до мероприятия")

        buttons = []

        user_id = str(callback_query.from_user.id)
        is_creator = (user_id == event["creator"])
        is_participant = (user_id in event.get("participants", []))

        # Кнопки для организатора
        if is_creator:
            # Редактирование и удаление
            buttons.append([types.InlineKeyboardButton(text="Редактировать мероприятие", callback_data=f"edit_event:{event_id}")])
            buttons.append([types.InlineKeyboardButton(text="Удалить мероприятие", callback_data=f"delete_event:{event_id}")])
            # Кнопка получить список участников
            buttons.append([types.InlineKeyboardButton(text="Список зарегистрировавшихся", callback_data=f"event_participants:{event_id}")])
        else:
            # Кнопки для обычного пользователя
            if is_participant:
                buttons.append([types.InlineKeyboardButton(text="Отменить регистрацию", callback_data=f"unregister_event:{event_id}")])
            else:
                buttons.append([types.InlineKeyboardButton(text="Зарегистрироваться", callback_data=f"register_event:{event_id}")])

        buttons.append([types.InlineKeyboardButton(text="В начало", callback_data="go_home")])
        markup = types.InlineKeyboardMarkup(inline_keyboard=buttons)
        await callback_query.message.edit_text(text, reply_markup=markup)
    else:
        await callback_query.answer("Мероприятие не найдено.")

@dp.callback_query(F.data.startswith("register_event:"))
async def register_event(callback_query: types.CallbackQuery):
    event_id = int(callback_query.data.split(":")[1])
    event = get_event_by_id(event_id)
    user_id = str(callback_query.from_user.id)

    if event:
        participants = event.get("participants", [])
        if user_id not in participants:
            participants.append(user_id)
            event["participants"] = participants
            save_events()
            await callback_query.answer("Успешная регистрация!")
            await view_event(callback_query)  # Обновляем отображение
        else:
            await callback_query.answer("Вы уже зарегистрированы.")
    else:
        await callback_query.answer("Мероприятие не найдено.")

@dp.callback_query(F.data.startswith("unregister_event:"))
async def unregister_event(callback_query: types.CallbackQuery):
    event_id = int(callback_query.data.split(":")[1])
    event = get_event_by_id(event_id)
    user_id = str(callback_query.from_user.id)

    if event:
        participants = event.get("participants", [])
        if user_id in participants:
            participants.remove(user_id)
            event["participants"] = participants
            save_events()
            await callback_query.answer("Вы отменили регистрацию.")
            await view_event(callback_query)  # Обновляем отображение
        else:
            await callback_query.answer("Вы не были зарегистрированы.")
    else:
        await callback_query.answer("Мероприятие не найдено.")

@dp.callback_query(F.data.startswith("event_participants:"))
async def event_participants(callback_query: types.CallbackQuery):
    event_id = int(callback_query.data.split(":")[1])
    event = get_event_by_id(event_id)
    if event:
        participants = event.get("participants", [])
        if participants:
            # Формируем список зарегистрировавшихся с именем, фамилией, классом
            lines = []
            for p_id in participants:
                user_data = user_profiles.get(p_id, {})
                name = user_data.get("name", "Имя?")
                surname = user_data.get("surname", "Фамилия?")
                grade = user_data.get("grade", "Класс?")
                lines.append(f"{name} {surname}, {grade}")

            participant_list = "\n".join(lines)
            text = f"Список зарегистрировавшихся:\n{participant_list}"
        else:
            text = "Нет участников."

        # Добавляем кнопку "В начало"
        back_btn = types.InlineKeyboardMarkup(
            inline_keyboard=[
                [types.InlineKeyboardButton(text="В начало", callback_data="go_home")]
            ]
        )

        await callback_query.message.edit_text(text, reply_markup=back_btn)
    else:
        await callback_query.answer("Мероприятие не найдено.")

@dp.callback_query(F.data.startswith("delete_event:"))
async def delete_event(callback_query: types.CallbackQuery):
    event_id = int(callback_query.data.split(":")[1])
    event = get_event_by_id(event_id)
    if event:
        if callback_query.from_user.id == int(event["creator"]):
            events_data["events"] = [e for e in events_data["events"] if e["id"] != event_id]
            save_events()
            await callback_query.answer("Мероприятие удалено.")
            await show_events(callback_query)
        else:
            await callback_query.answer("У вас нет прав удалять это мероприятие.", show_alert=True)
    else:
        await callback_query.answer("Мероприятие не найдено.", show_alert=True)

@dp.callback_query(F.data == "add_event")
async def add_event(callback_query: types.CallbackQuery, state: FSMContext):
    if callback_query.from_user.id in admin_user_ids:
        await state.set_state(CreateEventStates.waiting_for_title)
        await callback_query.message.edit_text("Введите название мероприятия:")
    else:
        await callback_query.answer("У вас нет прав для добавления мероприятий.", show_alert=True)

@dp.message(StateFilter(CreateEventStates.waiting_for_title))
async def create_event_title(message: types.Message, state: FSMContext):
    await state.update_data(title=message.text)
    await state.set_state(CreateEventStates.waiting_for_dates)
    await message.answer("Введите даты проведения мероприятия:")

@dp.message(StateFilter(CreateEventStates.waiting_for_dates))
async def create_event_dates(message: types.Message, state: FSMContext):
    await state.update_data(dates=message.text)
    await state.set_state(CreateEventStates.waiting_for_description)
    await message.answer("Введите описание мероприятия:")

@dp.message(StateFilter(CreateEventStates.waiting_for_description))
async def create_event_description(message: types.Message, state: FSMContext):
    await state.update_data(description=message.text)
    await state.set_state(CreateEventStates.waiting_for_notification_days)

    kb = types.InlineKeyboardMarkup(inline_keyboard=[
        [types.InlineKeyboardButton(text=str(i), callback_data=f"choose_notify:{i}") for i in range(1,8)]
    ])
    await message.answer("За сколько дней до мероприятия напомнить?", reply_markup=kb)

@dp.callback_query(StateFilter(CreateEventStates.waiting_for_notification_days), F.data.startswith("choose_notify:"))
async def choose_notify_day(callback_query: types.CallbackQuery, state: FSMContext):
    days = int(callback_query.data.split(":")[1])
    data = await state.get_data()
    new_event = {
        "id": len(events_data["events"]) + 1,
        "title": data["title"],
        "dates": data["dates"],
        "description": data["description"],
        "notification_days_before": days,
        "creator": str(callback_query.from_user.id),
        "participants": []
    }
    events_data["events"].append(new_event)
    save_events()

    await state.clear()
    await callback_query.message.edit_text("Мероприятие успешно добавлено!")

@dp.callback_query(F.data.startswith("edit_event:"))
async def edit_event_menu(callback_query: types.CallbackQuery, state: FSMContext):
    event_id = int(callback_query.data.split(":")[1])
    event = get_event_by_id(event_id)
    if not event:
        await callback_query.answer("Мероприятие не найдено.")
        return

    if callback_query.from_user.id != int(event["creator"]):
        await callback_query.answer("У вас нет прав редактировать это мероприятие.", show_alert=True)
        return

    await state.update_data(event_id=event_id)
    await state.set_state(EditEventStates.choose_field)
    kb = types.InlineKeyboardMarkup(inline_keyboard=[
        [types.InlineKeyboardButton("Редактировать название", callback_data="edit_field:title")],
        [types.InlineKeyboardButton("Редактировать даты", callback_data="edit_field:dates")],
        [types.InlineKeyboardButton("Редактировать описание", callback_data="edit_field:description")],
        [types.InlineKeyboardButton("Редактировать дни до напоминания", callback_data="edit_field:notify_days")],
        [types.InlineKeyboardButton("В начало", callback_data="go_home")]
    ])
    await callback_query.message.edit_text("Что хотите отредактировать?", reply_markup=kb)

@dp.callback_query(StateFilter(EditEventStates.choose_field), F.data.startswith("edit_field:"))
async def edit_field_select(callback_query: types.CallbackQuery, state: FSMContext):
    field = callback_query.data.split(":")[1]
    if field == "title":
        await state.set_state(EditEventStates.edit_title)
        await callback_query.message.edit_text("Введите новое название мероприятия:")
    elif field == "dates":
        await state.set_state(EditEventStates.edit_dates)
        await callback_query.message.edit_text("Введите новые даты:")
    elif field == "description":
        await state.set_state(EditEventStates.edit_description)
        await callback_query.message.edit_text("Введите новое описание:")
    elif field == "notify_days":
        await state.set_state(EditEventStates.edit_notification_days)
        kb = types.InlineKeyboardMarkup(inline_keyboard=[
            [types.InlineKeyboardButton(text=str(i), callback_data=f"edit_notify:{i}") for i in range(1,8)]
        ])
        await callback_query.message.edit_text("Выберите новые дни напоминания:", reply_markup=kb)

async def update_event_data(event_id, key, value):
    for event in events_data["events"]:
        if event["id"] == event_id:
            event[key] = value
            break
    save_events()

@dp.message(StateFilter(EditEventStates.edit_title))
async def edit_title(message: types.Message, state: FSMContext):
    new_title = message.text
    data = await state.get_data()
    event_id = data["event_id"]
    await update_event_data(event_id, "title", new_title)
    await state.clear()
    await message.answer("Название обновлено.")

@dp.message(StateFilter(EditEventStates.edit_dates))
async def edit_dates(message: types.Message, state: FSMContext):
    new_dates = message.text
    data = await state.get_data()
    event_id = data["event_id"]
    await update_event_data(event_id, "dates", new_dates)
    await state.clear()
    await message.answer("Даты обновлены.")

@dp.message(StateFilter(EditEventStates.edit_description))
async def edit_description(message: types.Message, state: FSMContext):
    new_description = message.text
    data = await state.get_data()
    event_id = data["event_id"]
    await update_event_data(event_id, "description", new_description)
    await state.clear()
    await message.answer("Описание обновлено.")

@dp.callback_query(StateFilter(EditEventStates.edit_notification_days), F.data.startswith("edit_notify:"))
async def edit_notify_days(callback_query: types.CallbackQuery, state: FSMContext):
    days = int(callback_query.data.split(":")[1])
    data = await state.get_data()
    event_id = data["event_id"]
    await update_event_data(event_id, "notification_days_before", days)
    await state.clear()
    await callback_query.message.edit_text("Дни до напоминания обновлены.")

async def main():
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
