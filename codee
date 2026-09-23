import asyncio
import logging
from datetime import datetime
import aiosqlite

from aiogram import Bot, Dispatcher, F, Router
from aiogram.filters import Command, CommandStart
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.types import Message

# ================= CONFIGURATION =================
BOT_TOKEN = "8667336977:AAG_7r3OYP1tt_dy1_J3hbY2xwzc6gKdJhA"  # Вставь сюда токен от @BotFather
DB_NAME = "kwit_health.db"
# =================================================

logging.basicConfig(level=logging.INFO)
bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()
router = Router()
dp.include_router(router)


# --- Состояния для FSM (Настройка и Ввод) ---
class SetupForm(StatesGroup):
    weight_start = State()
    weight_target = State()
    daily_calories_limit = State()


class DailyLogForm(StatesGroup):
    calories = State()
    weight_today = State()


# --- Инициализация базы данных ---
async def init_db():
    async with aiosqlite.connect(DB_NAME) as db:
        await db.execute("""
            CREATE TABLE IF NOT EXISTS user_profile (
                user_id INTEGER PRIMARY KEY,
                weight_start REAL,
                weight_current REAL,
                weight_target REAL,
                daily_limit INTEGER,
                streak INTEGER DEFAULT 0,
                jokers_left INTEGER DEFAULT 2,
                last_log_date TEXT
            )
        """)
        await db.execute("""
            CREATE TABLE IF NOT EXISTS daily_logs (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                date TEXT,
                calories INTEGER,
                weight REAL,
                is_success INTEGER
            )
        """)
        await db.commit()


# --- Команда: /start ---
@router.message(CommandStart())
async def cmd_start(message: Message, state: FSMContext):
    async with aiosqlite.connect(DB_NAME) as db:
        async with db.execute("SELECT user_id FROM user_profile WHERE user_id = ?", (message.from_user.id,)) as cursor:
            user = await cursor.fetchone()

    if user:
        await message.answer(
            "👋 Привет! Твой профиль уже настроен.\n\n"
            "Команды:\n"
            "📝 /log — Записать калории и вес за сегодня\n"
            "📊 /status — Посмотреть твой Kwit-прогресс и здоровье\n"
            "⚙️ /reset — Сбросить профиль и начать заново"
        )
    else:
        await message.answer("Здорово! Давай настроим твой профиль.\n\nВведи твой **начальный вес** (в кг), например: 170")
        await state.set_state(SetupForm.weight_start)


# --- Шаги настройки профиля ---
@router.message(SetupForm.weight_start)
async def process_start_weight(message: Message, state: FSMContext):
    try:
        w_start = float(message.text.replace(",", "."))
        await state.update_data(weight_start=w_start)
        await message.answer("Принято! Какая у тебя **цель по весу** (в кг)? Например: 100")
        await state.set_state(SetupForm.weight_target)
    except ValueError:
        await message.answer("Пожалуйста, введи число (например, 170 или 169.5)")


@router.message(SetupForm.weight_target)
async def process_target_weight(message: Message, state: FSMContext):
    try:
        w_target = float(message.text.replace(",", "."))
        await state.update_data(weight_target=w_target)
        await message.answer("Какой у тебя **дневной лимит калорий** для дефицита? Например: 2300")
        await state.set_state(SetupForm.daily_calories_limit)
    except ValueError:
        await message.answer("Пожалуйста, введи число.")


@router.message(SetupForm.daily_calories_limit)
async def process_cal_limit(message: Message, state: FSMContext):
    try:
        cal_limit = int(message.text)
        data = await state.get_data()

        async with aiosqlite.connect(DB_NAME) as db:
            await db.execute(
                """
                INSERT OR REPLACE INTO user_profile
                (user_id, weight_start, weight_current, weight_target, daily_limit, streak, jokers_left)
                VALUES (?, ?, ?, ?, ?, 0, 2)
            """,
                (message.from_user.id, data["weight_start"], data["weight_start"], data["weight_target"], cal_limit),
            )
            await db.commit()

        await state.clear()
        await message.answer(
            "🎉 Профиль успешно создан!\n\n"
            "Теперь каждый вечер отправляй команду /log, чтобы зафиксировать калории."
        )
    except ValueError:
        await message.answer("Пожалуйста, введи целое число.")


# --- Команда: /log (Ежедневный ввод) ---
@router.message(Command("log"))
async def cmd_log(message: Message, state: FSMContext):
    await message.answer("Сколько **калорий** ты наел за сегодня?")
    await state.set_state(DailyLogForm.calories)


@router.message(DailyLogForm.calories)
async def process_daily_calories(message: Message, state: FSMContext):
    try:
        cals = int(message.text)
        await state.update_data(calories=cals)
        await message.answer("Введи твой **сегодняшний вес** (в кг). Если не взвешивался, напиши 0:")
        await state.set_state(DailyLogForm.weight_today)
    except ValueError:
        await message.answer("Введи число калорий.")


@router.message(DailyLogForm.weight_today)
async def process_daily_weight(message: Message, state: FSMContext):
    try:
        weight_input = float(message.text.replace(",", "."))
        data = await state.get_data()
        cals = data["calories"]
        today_str = datetime.now().strftime("%Y-%m-%d")

        async with aiosqlite.connect(DB_NAME) as db:
            async with db.execute("SELECT * FROM user_profile WHERE user_id = ?", (message.from_user.id,)) as cursor:
                profile = await cursor.fetchone()

            if not profile:
                await message.answer("Сначала пройди настройку с помощью /start")
                await state.clear()
                return

            user_id, w_start, w_current, w_target, limit, streak, jokers, last_log = profile

            # Обновляем вес, если пользователь ввел значение больше 0
            if weight_input > 0:
                w_current = weight_input

            # Проверка лимита с допуском +5%
            max_allowed = limit * 1.05
            is_success = cals <= max_allowed

            if is_success:
                streak += 1
                result_text = f"🔥 **Отлично! День засчитан.**\nСтрик увеличен: **{streak} дн.** 🚀"
            else:
                if jokers > 0:
                    jokers -= 1
                    result_text = (
                        f"⚠️ Ты превысил лимит ({cals} / {limit} ккал), но сработал **Джокер (Праздник)**!\n"
                        f"Стрик сохранен: **{streak} дн.** (Осталось джокеров в этом месяце: {jokers})"
                    )
                else:
                    streak = 0
                    result_text = f"💔 Стрик сгорел... Ты наел {cals} ккал (лимит {limit}). Начинаем сначала, не сдавайся!"

            # Сохранение профиля и лога
            await db.execute(
                """
                UPDATE user_profile
                SET weight_current = ?, streak = ?, jokers_left = ?, last_log_date = ?
                WHERE user_id = ?
            """,
                (w_current, streak, jokers, today_str, message.from_user.id),
            )

            await db.execute(
                """
                INSERT INTO daily_logs (user_id, date, calories, weight, is_success)
                VALUES (?, ?, ?, ?, ?)
            """,
                (message.from_user.id, today_str, cals, w_current, 1 if is_success else 0),
            )

            await db.commit()

        await state.clear()
        await message.answer(f"{result_text}\n\nПосмотреть общую статистику: /status")

    except ValueError:
        await message.answer("Введи число.")


# --- Команда: /status (Метрики Kwit) ---
@router.message(Command("status"))
async def cmd_status(message: Message):
    async with aiosqlite.connect(DB_NAME) as db:
        async with db.execute("SELECT * FROM user_profile WHERE user_id = ?", (message.from_user.id,)) as cursor:
            profile = await cursor.fetchone()

    if not profile:
        await message.answer("Профиль не найден. Напиши /start")
        return

    user_id, w_start, w_current, w_target, limit, streak, jokers, last_log = profile

    dropped_kg = max(0.0, w_start - w_current)
    excess_weight = max(1.0, w_start - w_target)

    # === РАСЧЕТЫ И ФОРМУЛЫ ===
    # 1. Разгрузка коленей: ~4 кг снижения нагрузки на 1 кг веса при ходьбе
    knee_relief = round(dropped_kg * 4, 1)

    # 2. Улучшение сердца %: отношение сброшенного веса к избыточному
    heart_improve = round(min(100.0, (dropped_kg / excess_weight) * 100), 1)

    # 3. Эквивалент в пачках масла (200 г в пачке)
    butter_packs = int(dropped_kg // 0.2)

    # 4. Добавленные дни жизни: ~2 дня за 1 кг сброшенного веса при высоком ИМТ
    life_days = round(dropped_kg * 2, 1)

    # Прогресс-бар из 10 кубиков
    progress_pct = min(1.0, dropped_kg / excess_weight)
    filled_blocks = int(progress_pct * 10)
    progress_bar = "🟩" * filled_blocks + "⬜" * (10 - filled_blocks)

    status_msg = (
        f"🏆 **ТВОЙ КВИТ-ПРОГРЕСС ПОХУДЕНИЯ**\n"
        f"────────────────────────\n"
        f"🔥 **Текущий стрик:** {streak} дней подряд\n"
        f"🃏 **Джокеры (праздники):** {jokers} шт.\n\n"
        f"⚖️ **Вес:** {w_start} кг ➔ **{w_current} кг** (Сброшено: **-{dropped_kg:.1f} кг**)\n"
        f"🎯 **Цель:** {w_target} кг\n"
        f"Прогресс: [{progress_bar}] {progress_pct*100:.1f}%\n\n"
        f"🩺 **ВЛИЯНИЕ НА ЗДОРОВЬЕ:**\n"
        f"❤️ Сердечно-сосудистый риск снижен на: **{heart_improve}%**\n"
        f"🦵 Разгрузка коленей и спины при шаге: **-{knee_relief} кг**\n"
        f"⏳ Возвращено дней активной жизни: **+{life_days} дн.**\n"
        f"🧈 Сброшено жира в масле: **{butter_packs} пачек** (по 200г)\n"
        f"────────────────────────\n"
        f"Записать сегодняшний день: /log"
    )

    await message.answer(status_msg, parse_mode="Markdown")


# --- Команда: /reset ---
@router.message(Command("reset"))
async def cmd_reset(message: Message, state: FSMContext):
    async with aiosqlite.connect(DB_NAME) as db:
        await db.execute("DELETE FROM user_profile WHERE user_id = ?", (message.from_user.id,))
        await db.commit()
    await state.clear()
    await message.answer("Профиль сброшен. Напиши /start для повторной настройки.")


# --- Точка входа ---
async def main():
    await init_db()
    print("Бот успешно запущен!")
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())

