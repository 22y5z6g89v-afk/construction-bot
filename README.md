# construction-bot
Аренда спецтехники и работа по благоустройству и асфальтированию
import os
from aiogram import Bot, Dispatcher, F
from aiogram.types import (
    Message, CallbackQuery,
    InlineKeyboardMarkup, InlineKeyboardButton,
    KeyboardButton, ReplyKeyboardMarkup
)
from aiogram.filters import Command
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.context import FSMContext
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.types import ChatInviteLink
import asyncio

# === НАСТРОЙКИ ===
BOT_TOKEN = "7759368717:AAFXcE3TJgfE6zO0dAlXeM0Oyx_TW7qwExI"
ADMIN_ID = 123456789          # Твой ID (6986157999)
EXECUTORS_CHAT_ID = -1001234567890  # ID чата с исполнителями (Your group ID is: -5080559951)

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher(storage=MemoryStorage())

# === СОСТОЯНИЯ ===
class TechForm(StatesGroup):
    tech_type = State()
    duration = State()
    city = State()
    phone = State()

class WorkerForm(StatesGroup):
    work_type = State()
    city = State()
    phone = State()

# === КОМАНДА /start ===
@dp.message(Command("start"))
async def cmd_start(message: Message):
    kb = ReplyKeyboardMarkup(
        keyboard=[
            [KeyboardButton(text="🚛 Аренда техники")],
            [KeyboardButton(text="👷 Нужна бригада")]
        ],
        resize_keyboard=True,
        one_time_keyboard=True
    )
    await message.answer(
        "Если вы хотите заказать спецтехнику в аренду или требуется бригада для выполнения работы — выберите вариант ниже:",
        reply_markup=kb
    )

# === АРЕНДА ТЕХНИКИ ===
@dp.message(F.text == "🚛 Аренда техники")
async def tech_start(message: Message, state: FSMContext):
    await message.answer("Какая техника требуется? (например: экскаватор, бетономешалка, манипулятор)")
    await state.set_state(TechForm.tech_type)

@dp.message(TechForm.tech_type)
async def tech_type(message: Message, state: FSMContext):
    await state.update_data(tech_type=message.text)
    await message.answer("На какой срок? (например: 3 дня, 2 недели)")
    await state.set_state(TechForm.duration)

@dp.message(TechForm.duration)
async def tech_duration(message: Message, state: FSMContext):
    await state.update_data(duration=message.text)
    await message.answer("В каком городе?")
    await state.set_state(TechForm.city)

@dp.message(TechForm.city)
async def tech_city(message: Message, state: FSMContext):
    await state.update_data(city=message.text)
    await message.answer("Укажите ваш номер телефона для связи:")
    await state.set_state(TechForm.phone)

@dp.message(TechForm.phone)
async def tech_phone(message: Message, state: FSMContext):
    data = await state.get_data()
    summary = (
        "🚛 <b>Заявка на аренду техники</b>\n\n"
        f"• Техника: {data['tech_type']}\n"
        f"• Срок: {message.text}\n"
        f"• Город: {data['city']}\n"
        f"• Телефон: {message.text}\n"
        f"• Заказчик: @{message.from_user.username or '—'} (ID: {message.from_user.id})"
    )
    await send_application(summary, message.from_user.id, "tech")
    await message.answer("✅ Спасибо! Ваша заявка отправлена. Ожидайте предложения от исполнителей.")
    await state.clear()

# === НУЖНА БРИГАДА ===
@dp.message(F.text == "👷 Нужна бригада")
async def worker_start(message: Message, state: FSMContext):
    await message.answer("Какой вид работ нужно выполнить? (например: отделка, демонтаж, земляные работы)")
    await state.set_state(WorkerForm.work_type)

@dp.message(WorkerForm.work_type)
async def worker_type(message: Message, state: FSMContext):
    await state.update_data(work_type=message.text)
    await message.answer("В каком городе?")
    await state.set_state(WorkerForm.city)

@dp.message(WorkerForm.city)
async def worker_city(message: Message, state: FSMContext):
    await state.update_data(city=message.text)
    await message.answer("Укажите ваш номер телефона для связи:")
    await state.set_state(WorkerForm.phone)

@dp.message(WorkerForm.phone)
async def worker_phone(message: Message, state: FSMContext):
    data = await state.get_data()
    summary = (
        "👷 <b>Заявка на бригаду</b>\n\n"
        f"• Работы: {data['work_type']}\n"
        f"• Город: {data['city']}\n"
        f"• Телефон: {message.text}\n"
        f"• Заказчик: @{message.from_user.username or '—'} (ID: {message.from_user.id})"
    )
    await send_application(summary, message.from_user.id, "worker")
    await message.answer("✅ Спасибо! Ваша заявка отправлена. Ожидайте предложений от исполнителей.")
    await state.clear()

# === ОТПРАВКА ЗАЯВКИ ===
async def send_application(text: str, user_id: int, req_type: str):
    # 1. Отправить тебе (админу)
    await bot.send_message(ADMIN_ID, text, parse_mode="HTML")

    # 2. Отправить в чат исполнителей с кнопкой "Принять заказ"
    keyboard = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✅ Принять заказ", callback_data=f"accept_{req_type}_{user_id}")]
    ])
    await bot.send_message(EXECUTORS_CHAT_ID, text, reply_markup=keyboard, parse_mode="HTML")

# === ОБРАБОТКА КНОПКИ "ПРИНЯТЬ ЗАКАЗ" ===
@dp.callback_query(F.data.startswith("accept_"))
async def handle_accept(callback: CallbackQuery):
    _, req_type, customer_id = callback.data.split("_")
    customer_id = int(customer_id)

    executor = callback.from_user
    message_to_admin = (
        f"🔔 <b>Исполнитель откликнулся!</b>\n\n"
        f"• Тип: {'Техника' if req_type == 'tech' else 'Бригада'}\n"
        f"• Исполнитель: @{executor.username or '—'} (ID: {executor.id})\n"
        f"• Имя: {executor.full_name}"
    )

    # Отправить тебе (админу)
    await bot.send_message(ADMIN_ID, message_to_admin, parse_mode="HTML")

    # Уведомить исполнителя
    await callback.answer("✅ Вы откликнулись! Администратор скоро свяжется с вами.")
    await callback.message.edit_reply_markup(reply_markup=None)  # убрать кнопку

# === ЗАПУСК ===
async def main():
    print("✅ Бот запущен и готов принимать заявки!")
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())


