from aiohttp import web
import asyncio
import os
import sqlite3

DB_PATH = "diamonds_bot.db"  # путь к базе

conn = sqlite3.connect(DB_PATH)
cursor = conn.cursor()

# Проверяем, есть ли колонка commands_blocked
cursor.execute("PRAGMA table_info(users)")
columns = [row[1] for row in cursor.fetchall()]

if "commands_blocked" not in columns:
    cursor.execute("""
        ALTER TABLE users
        ADD COLUMN commands_blocked INTEGER DEFAULT 0
    """)
    print("✅ Колонка commands_blocked добавлена!")
else:
    print("ℹ️ Колонка commands_blocked уже существует")

conn.commit()
conn.close()

from telegram.ext import JobQueue
import sqlite3

DB_PATH = "diamonds_bot.db"

def reset_game_locks():
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    # 🌾 Сброс фарма
    cursor.execute("UPDATE users SET active_farm = 0")

    # 💣 Сброс минного поля
    cursor.execute("UPDATE users SET active_mines_game = 0")

    # 🔺 Сброс пирамиды
    cursor.execute("UPDATE users SET pyramid_active = 0")

    conn.commit()
    conn.close()
    print("[OK] Все игровые блокировки сброшены (farm, mines, pyramid)")

import time
import traceback
from telegram.error import RetryAfter, TimedOut, NetworkError
import sys
import os

# Устанавливаем локали в Python
os.environ['LANG'] = 'en_US.UTF-8'
os.environ['LC_ALL'] = 'en_US.UTF-8'

# Настраиваем stdout в UTF-8
sys.stdout.reconfigure(encoding='utf-8')

import sqlite3
import random
from datetime import datetime, timedelta
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, ContextTypes, CallbackQueryHandler, MessageHandler, filters

TOKEN = "7971701925:AAGr7HiOrCcnKeIxMjrLzisiX4LpSLMg-b4"
ADMIN_IDS = [7689411893]  # список админов
ADMIN_USER_ID = 7689411893

# Обработчик команды изменения здоровья
async def set_farm_health(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    if user_id not in ADMIN_IDS:
        await update.message.reply_text("❌ У вас нет прав для этой команды.")
        print(f"Отклонена команда изменения здоровья от {user_id} (не админ)")
        return

    if len(context.args) != 2:
        await update.message.reply_text("❌ Использование: /set_farm_health <target_user_id> <new_health>")
        return

    try:
        target_user_id = int(context.args[0])
        new_health = int(context.args[1])
    except ValueError:
        await update.message.reply_text("❌ ID пользователя и здоровье должны быть числами.")
        return

    conn = sqlite3.connect("database.db")
    cursor = conn.cursor()
    cursor.execute(
        "UPDATE farm_status SET farm_health = ? WHERE user_id = ?",
        (new_health, target_user_id)
    )
    conn.commit()
    conn.close()

    await update.message.reply_text(f"✅ Здоровье фермы пользователя {target_user_id} изменено на {new_health}.")
    print(f"Админ {user_id} изменил здоровье фермы {target_user_id} на {new_health}")

# Глобальные словари для хранения временных данных
farm_data = {}  # Для фарма алмазов
strawberry_farm_data = {}  # Для хранения временных данных фермы
russian_roulette_data = {}  # Для хранения состояния русской рулетки

def init_db():
    conn = sqlite3.connect('diamonds_bot.db')
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS users (
            user_id INTEGER PRIMARY KEY,
            player_id INTEGER UNIQUE,
            username TEXT,
            first_name TEXT,
            diamonds INTEGER DEFAULT 0,
            points INTEGER DEFAULT 0,
            last_diamond_time TEXT,
            upgrade_level INTEGER DEFAULT 0,
            registration_date TEXT,
            strawberries INTEGER DEFAULT 0,
            farm_level INTEGER DEFAULT 0,
            last_zombie_attack TEXT,
            zombie_status INTEGER DEFAULT 0,
            last_strawberry_update TEXT,
            last_bonus_time TEXT,
            tomatoes INTEGER DEFAULT 0,
            coins INTEGER DEFAULT 0
        )
    ''')

    # Проверяем и добавляем отсутствующие колонки
    columns = [
        ('registration_date', 'TEXT'),
        ('strawberries', 'INTEGER DEFAULT 0'),
        ('farm_level', 'INTEGER DEFAULT 0'),
        ('last_zombie_attack', 'TEXT'),
        ('zombie_status', 'INTEGER DEFAULT 0'),
        ('last_strawberry_update', 'TEXT'),
        ('last_bonus_time', 'TEXT'),
        ('tomatoes', 'INTEGER DEFAULT 0'),
        ('coins', 'INTEGER DEFAULT 0')
    ]
    for column, column_type in columns:
        try:
            cursor.execute(f"SELECT {column} FROM users LIMIT 1")
        except sqlite3.OperationalError:
            print(f"🔄 Добавляем колонку {column} в таблицу users...")
            cursor.execute(f'ALTER TABLE users ADD COLUMN {column} {column_type}')

    conn.commit()
    conn.close()

def get_next_player_id():
    conn = sqlite3.connect('diamonds_bot.db')
    cursor = conn.cursor()
    cursor.execute('SELECT MAX(player_id) FROM users')
    max_id = cursor.fetchone()[0]
    conn.close()
    return (max_id or 0) + 1

def get_user(user_id, username=None, first_name=None):
    conn = sqlite3.connect('diamonds_bot.db')
    cursor = conn.cursor()

    cursor.execute('''
        SELECT user_id, player_id, username, first_name, diamonds, points, last_diamond_time,
               upgrade_level, registration_date, strawberries, farm_level, last_zombie_attack,
               zombie_status, last_strawberry_update, last_bonus_time, tomatoes, grapes, rubies, coins,
               farms_destroyed, clan_id, wood, stone, iron, titanium
        FROM users WHERE user_id = ?
    ''', (user_id,))
    user = cursor.fetchone()

    if not user:
        player_id = get_next_player_id()
        registration_date = datetime.now().isoformat()
        last_strawberry_update = datetime.now().isoformat()
        cursor.execute('''
            INSERT INTO users (
                user_id, player_id, username, first_name,
                diamonds, points, last_diamond_time, upgrade_level, registration_date,
                strawberries, farm_level, last_zombie_attack, zombie_status,
                last_strawberry_update, last_bonus_time, tomatoes, grapes, rubies, coins,
                farms_destroyed, clan_id, wood, stone, iron, titanium
            )
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        ''', (
            user_id, player_id, username, first_name,
            50, 10, 0, 0, registration_date,
            0, 0, None, 0,
            last_strawberry_update, None, 0, 10, 0, 0,
            0, None, 0, 0, 0, 0  # farms_destroyed=0, clan_id=None, wood=0, stone=0, iron=0, titanium=0
        ))
        conn.commit()

        cursor.execute('''
            SELECT user_id, player_id, username, first_name, diamonds, points, last_diamond_time,
                   upgrade_level, registration_date, strawberries, farm_level, last_zombie_attack,
                   zombie_status, last_strawberry_update, last_bonus_time, tomatoes, grapes, rubies, coins,
                   farms_destroyed, clan_id, wood, stone, iron, titanium
            FROM users WHERE user_id = ?
        ''', (user_id,))
        user = cursor.fetchone()
    else:
        cursor.execute('UPDATE users SET username = ?, first_name = ? WHERE user_id = ?', (username, first_name, user_id))
        conn.commit()

    conn.close()
    return user

def find_user_by_player_id(player_id):
    conn = sqlite3.connect('diamonds_bot.db')
    cursor = conn.cursor()
    cursor.execute('SELECT user_id, player_id, username, first_name, diamonds, points, last_diamond_time, upgrade_level, registration_date, strawberries, farm_level, last_zombie_attack, zombie_status, last_strawberry_update, last_bonus_time, tomatoes, grapes, coins FROM users WHERE player_id = ?', (player_id,))
    user = cursor.fetchone()
    conn.close()
    return user

def get_all_users():
    conn = sqlite3.connect('diamonds_bot.db')
    cursor = conn.cursor()
    cursor.execute('''
        SELECT
            user_id, player_id, username, first_name, diamonds, points, last_diamond_time,
            upgrade_level, registration_date, strawberries, farm_level, last_zombie_attack,
            zombie_status, last_strawberry_update, last_bonus_time, tomatoes, coins, farms_destroyed,
            wood, stone, iron, titanium,
            sawmill, quarry, smelter, factory, mine, last_factory_update,
            solved_riddles, grapes, rubies
        FROM users
        ORDER BY player_id
    ''')
    users = cursor.fetchall()
    conn.close()
    return users

def get_users_count():
    conn = sqlite3.connect('diamonds_bot.db')
    cursor = conn.cursor()
    cursor.execute('SELECT COUNT(*) FROM users')
    count = cursor.fetchone()[0]
    conn.close()
    return count

def update_user(
    user_id,
    diamonds=None,
    points=None,
    last_diamond_time=None,
    upgrade_level=None,
    strawberries=None,
    farm_level=None,
    last_zombie_attack=None,
    zombie_status=None,
    last_strawberry_update=None,
    last_bonus_time=None,
    tomatoes=None,
    grapes=None,
    rubies=None,
    coins=None,
    wood=None,
    stone=None,
    iron=None,
    titanium=None,
    sawmill=None,
    quarry=None,
    smelter=None,
    factory=None,
    mine=None,
    last_factory_update=None
):
    try:
        with sqlite3.connect("diamonds_bot.db", timeout=10) as conn:
            cursor = conn.cursor()

            update_fields = []
            values = []

            if diamonds is not None:
                update_fields.append("diamonds = ?")
                values.append(diamonds)
            if points is not None:
                update_fields.append("points = ?")
                values.append(points)
            if last_diamond_time is not None:
                update_fields.append("last_diamond_time = ?")
                values.append(last_diamond_time)
            if upgrade_level is not None:
                update_fields.append("upgrade_level = ?")
                values.append(upgrade_level)
            if strawberries is not None:
                update_fields.append("strawberries = ?")
                values.append(strawberries)
            if farm_level is not None:
                update_fields.append("farm_level = ?")
                values.append(farm_level)
            if last_zombie_attack is not None:
                update_fields.append("last_zombie_attack = ?")
                values.append(last_zombie_attack)
            if zombie_status is not None:
                update_fields.append("zombie_status = ?")
                values.append(zombie_status)
            if last_strawberry_update is not None:
                update_fields.append("last_strawberry_update = ?")
                values.append(last_strawberry_update)
            if last_bonus_time is not None:
                update_fields.append("last_bonus_time = ?")
                values.append(last_bonus_time)
            if tomatoes is not None:
                update_fields.append("tomatoes = ?")
                values.append(tomatoes)
            if grapes is not None:
                update_fields.append("grapes = ?")
                values.append(grapes)
            if rubies is not None:
                update_fields.append("rubies = rubies + ?")
                values.append(rubies)
            if coins is not None:
                update_fields.append("coins = coins + ?")  # прибавляем к старым монетам
                values.append(coins)
            if wood is not None:
                update_fields.append("wood = wood + ?")
                values.append(wood)
            if stone is not None:
                update_fields.append("stone = stone + ?")
                values.append(stone)
            if iron is not None:
                update_fields.append("iron = iron + ?")
                values.append(iron)
            if titanium is not None:
                update_fields.append("titanium = titanium + ?")
                values.append(titanium)

            # === Новые поля для заводов ===
            if sawmill is not None:
                update_fields.append("sawmill = sawmill + ?")
                values.append(sawmill)
            if quarry is not None:
                update_fields.append("quarry = quarry + ?")
                values.append(quarry)
            if smelter is not None:
                update_fields.append("smelter = smelter + ?")
                values.append(smelter)
            if factory is not None:
                update_fields.append("factory = factory + ?")
                values.append(factory)
            if mine is not None:
                update_fields.append("mine = mine + ?")
                values.append(mine)
            if last_factory_update is not None:
                update_fields.append("last_factory_update = ?")
                values.append(last_factory_update)

            if update_fields:
                values.append(user_id)
                cursor.execute(f'UPDATE users SET {", ".join(update_fields)} WHERE user_id = ?', values)
                conn.commit()

    except Exception as e:
        print(f"❌ ERROR in update_user: {e}")

ADMIN_USER_ID = 7689411893

async def block_commands_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_USER_ID:
        return

    if not context.args or not context.args[0].isdigit():
        await update.message.reply_text("❌ Использование: /block_commands <player_id>")
        return

    target_player_id = int(context.args[0])

    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    cursor.execute(
        "UPDATE users SET commands_blocked = 1 WHERE player_id = ?",
        (target_player_id,)
    )

    if cursor.rowcount == 0:
        await update.message.reply_text("❌ Игрок не найден")
    else:
        await update.message.reply_text(
            f"⛔ Игрок с ID {target_player_id} НАВСЕГДА заблокирован"
        )

    conn.commit()
    conn.close()

async def unblock_commands_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_USER_ID:
        return

    if not context.args or not context.args[0].isdigit():
        await update.message.reply_text("❌ Использование: /unblock_commands <player_id>")
        return

    target_player_id = int(context.args[0])

    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute(
        "UPDATE users SET commands_blocked = 0 WHERE player_id = ?",
        (target_player_id,)
    )
    conn.commit()
    conn.close()

    await update.message.reply_text(
        f"✅ Игрок с ID {target_player_id} разблокирован"
    )

import numpy as np
import asyncio
import random
import sqlite3
from datetime import datetime

from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application,
    CommandHandler,
    CallbackQueryHandler,
    MessageHandler,
    filters,
    ContextTypes,
)

# ================== КОНФИГ ==================
DB_PATH = "crash_bot.db"
CRASH_TICK = 0.35       # интервал между обновлениями множителя
CRASH_STEP = 0.05       # шаг роста множителя за тик
MAX_X = 5.0
WAITING_TIME = 30       # секунд ожидания ставок

active_crash_games = {}  # chat_id → game_data
# ============================================


async def init_db():
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute('''
        CREATE TABLE IF NOT EXISTS users (
            user_id INTEGER PRIMARY KEY,
            diamonds INTEGER DEFAULT 1000
        )
    ''')
    conn.commit()
    conn.close()


async def crash_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.effective_chat.id

    if chat_id in active_crash_games:
        await update.message.reply_text("❌ В этом чате уже идёт игра Crash!")
        return

    r = random.random()

    if r < 0.30:
        crash_x = random.uniform(1.00, 1.02)

    elif r < 0.60:
        t = random.random() ** 3.0
        crash_x = 1.05 + t * (2.0 - 1.05)

    else:
        if random.random() < 0.85:
            crash_x = random.uniform(2.0, 2.15)
        else:
            crash_x = 2.15 + np.random.exponential(scale=0.8)

    crash_x = min(crash_x, MAX_X)

    game = {
        "phase": "waiting",
        "players": {},
        "multiplier": 1.0,
        "crash_x": crash_x,
        "message_id": None,
        "start_time": datetime.now(),
        "auto_start_job": None
    }

    keyboard = [
        [
            InlineKeyboardButton("💎 Войти", callback_data="crash_join"),
            InlineKeyboardButton("🚀 Начать сейчас", callback_data="crash_start")
        ]
    ]

    msg = await update.message.reply_text(
        "🎰 *CRASH*\n\n"
        "Нажми 💎 Войти и укажи ставку\n"
        f"⏳ Автостарт через **{WAITING_TIME}** сек\n"
        "Или начни раньше кнопкой «Начать сейчас»",
        parse_mode="Markdown",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

    game["message_id"] = msg.message_id
    active_crash_games[chat_id] = game

    # Запускаем автостарт
    job = context.job_queue.run_once(
        auto_start_callback,
        when=WAITING_TIME,
        chat_id=chat_id,
        name=f"crash_auto_{chat_id}"
    )
    game["auto_start_job"] = job


async def auto_start_callback(context: ContextTypes.DEFAULT_TYPE):
    job = context.job
    chat_id = job.chat_id
    game = active_crash_games.get(chat_id)

    if not game or game["phase"] != "waiting":
        return

    if not game["players"]:
        try:
            await context.bot.edit_message_text(
                chat_id=chat_id,
                message_id=game["message_id"],
                text="⏰ Время вышло\nНикто не сделал ставку → игра отменена"
            )
        except:
            pass
        active_crash_games.pop(chat_id, None)
        return

    game["phase"] = "running"
    await start_crash_game(chat_id, context)


async def join_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    chat_id = query.message.chat.id
    game = active_crash_games.get(chat_id)

    if not game or game["phase"] != "waiting":
        await query.answer("❌ Игра уже началась или закончилась!", show_alert=True)
        return

    context.user_data["crash_bet_pending"] = True
    await query.message.reply_text(
        "💎 Введи сумму ставки (целое число):",
        reply_to_message_id=query.message.message_id
    )


async def start_now_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    chat_id = query.message.chat.id
    game = active_crash_games.get(chat_id)

    if not game or game["phase"] != "waiting":
        return

    if not game["players"]:
        await query.answer("❌ Нет игроков!", show_alert=True)
        return

    # Отмена автостарта
    if game.get("auto_start_job"):
        game["auto_start_job"].schedule_removal()
        game["auto_start_job"] = None

    game["phase"] = "running"
    await start_crash_game(chat_id, context)


async def bet_input_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.user_data.get("crash_bet_pending"):
        return

    chat_id = update.effective_chat.id
    user_id = update.effective_user.id
    text = update.message.text.strip()

    try:
        bet = int(text)
        if bet <= 0:
            raise ValueError
    except ValueError:
        await update.message.reply_text("❌ Нужно ввести положительное целое число!")
        return

    context.user_data["crash_bet_pending"] = False

    game = active_crash_games.get(chat_id)
    if not game or game["phase"] != "waiting":
        await update.message.reply_text("❌ Игра уже началась или закончилась")
        return

    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("SELECT diamonds FROM users WHERE user_id = ?", (user_id,))
    row = c.fetchone()

    if not row:
        c.execute("INSERT INTO users (user_id, diamonds) VALUES (?, 1000)", (user_id,))
        diamonds = 1000
    else:
        diamonds = row[0]

    if diamonds < bet:
        conn.close()
        await update.message.reply_text("❌ Недостаточно алмазов!")
        return

    c.execute("UPDATE users SET diamonds = diamonds - ? WHERE user_id = ?", (bet, user_id))
    conn.commit()
    conn.close()

    game["players"][user_id] = {
        "bet": bet,
        "cashed": False,
        "cashout": None,
        "name": update.effective_user.first_name or f"ID{user_id}"
    }

    await update.message.reply_text(f"✅ Ты вошёл в игру! Ставка: {bet}💎")

async def start_crash_game(chat_id: int, context: ContextTypes.DEFAULT_TYPE):
    game = active_crash_games[chat_id]
    game["phase"] = "running"
    game["multiplier"] = 1.0

    keyboard = [[InlineKeyboardButton("💰 ЗАБРАТЬ", callback_data="crash_cashout")]]

    msg = await context.bot.edit_message_text(
        chat_id=chat_id,
        message_id=game["message_id"],
        text="🚀 Игра началась!\nМножитель: **1.00x**",
        parse_mode="Markdown",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

    # Запускаем фоновую задачу для обновления множителя
    asyncio.create_task(run_crash_multiplier(chat_id, msg, context))


async def run_crash_multiplier(chat_id: int, msg, context):
    game = active_crash_games[chat_id]
    last_text = ""
    while game["multiplier"] < game["crash_x"] and game["phase"] == "running":
        await asyncio.sleep(0.55)
        game["multiplier"] = round(game["multiplier"] + CRASH_STEP, 2)

        new_text = f"🚀 **CRASH**\n\nМножитель: **{game['multiplier']:.2f}x**"

        if new_text != last_text:
            try:
                await context.bot.edit_message_text(
                    chat_id=chat_id,
                    message_id=msg.message_id,
                    text=new_text,
                    parse_mode="Markdown",
                    reply_markup=InlineKeyboardMarkup(
                        [[InlineKeyboardButton("💰 ЗАБРАТЬ", callback_data="crash_cashout")]]
                    )
                )
                last_text = new_text
            except Exception as e:
                print(f"Ошибка редактирования в игре {chat_id}: {e}")
                await asyncio.sleep(2.0)

    # КРАШ
    await crash_end(chat_id, context)

async def cashout_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query

    # Отвечаем Telegram сразу — обязательно!
    await query.answer()

    chat_id = query.message.chat.id
    user_id = query.from_user.id

    game = active_crash_games.get(chat_id)
    if not game:
        await query.answer("❌ Игра уже завершена", show_alert=True)
        return

    if game["phase"] != "running":
        await query.answer("❌ Игра не активна", show_alert=True)
        return

    if game["multiplier"] >= game["crash_x"]:
        await query.answer("💥 Уже краш! Опоздал 😔", show_alert=True)
        return

    player = game["players"].get(user_id)
    if not player:
        await query.answer("❌ Ты не в игре", show_alert=True)
        return

    if player["cashed"]:
        await query.answer("✅ Ты уже забрал!", show_alert=True)
        return

    # Антифлуд (защита от спама нажатий)
    lock_key = f"cashout_lock_{chat_id}_{user_id}"
    if context.user_data.get(lock_key):
        return
    context.user_data[lock_key] = True

    try:
        current_x = game["multiplier"]
        player["cashed"] = True
        player["cashout"] = current_x

        win = int(player["bet"] * current_x)

        # Обновляем баланс
        conn = sqlite3.connect(DB_PATH)
        c = conn.cursor()
        c.execute("UPDATE users SET diamonds = diamonds + ? WHERE user_id = ?", (win, user_id))
        conn.commit()
        conn.close()

        # Личное уведомление (алерт)
        await query.answer(
            f"✅ Забрал на {current_x:.2f}x!\n+{win} 💎",
            show_alert=True
        )

        # Главное — отправляем сообщение В ЧАТ для всех
        username = query.from_user.username
        display_name = f"@{username}" if username else query.from_user.first_name

        await context.bot.send_message(
            chat_id=chat_id,
            text=f"🎉 **{display_name}** забрал на **{current_x:.2f}x**! "
                 f"+{win} 💎 (ставка {player['bet']}💎)",
            parse_mode="Markdown"
        )

    except Exception as e:
        print(f"Ошибка в cashout: {e}")
        await query.answer("⚠️ Ошибка... Попробуй ещё раз", show_alert=True)

    finally:
        # Снимаем лок
        await asyncio.sleep(1.2)
        context.user_data.pop(lock_key, None)

async def crash_end(chat_id: int, context: ContextTypes.DEFAULT_TYPE):
    game = active_crash_games.get(chat_id)
    if not game:
        return

    lines = [f"💥 **КРАШ** на **{game['crash_x']:.2f}x**\n"]

    for player in game["players"].values():
        if player["cashed"]:
            lines.append(f"✅ {player['name']} — {player['cashout']:.2f}x")
        else:
            lines.append(f"❌ {player['name']} — проиграл {player['bet']}💎")

    if not any(p["cashed"] for p in game["players"].values()):
        lines.append("\nНикто не успел забрать 😢")

    try:
        await context.bot.edit_message_text(
            chat_id=chat_id,
            message_id=game["message_id"],
            text="\n".join(lines),
            parse_mode="Markdown"
        )
    except:
        pass

    active_crash_games.pop(chat_id, None)

MAX_BANK_LEVEL = 12

BANK_LEVELS = {
    1: {"diamonds": 5000, "coins": 750_000, "upgrade_diamonds": 0, "upgrade_coins": 0},
    2: {"diamonds": 15_000, "coins": 2_250_000, "upgrade_diamonds": 10_000, "upgrade_coins": 1_500_000},
    3: {"diamonds": 30_000, "coins": 4_500_000, "upgrade_diamonds": 20_000, "upgrade_coins": 3_000_000},
    4: {"diamonds": 60_000, "coins": 9_000_000, "upgrade_diamonds": 40_000, "upgrade_coins": 6_000_000},
    5: {"diamonds": 120_000, "coins": 18_000_000, "upgrade_diamonds": 80_000, "upgrade_coins": 12_000_000},
    6: {"diamonds": 250_000, "coins": 37_500_000, "upgrade_diamonds": 150_000, "upgrade_coins": 22_500_000},
    7: {"diamonds": 400_000, "coins": 60_000_000, "upgrade_diamonds": 250_000, "upgrade_coins": 37_500_000},
    8: {"diamonds": 600_000, "coins": 90_000_000, "upgrade_diamonds": 400_000, "upgrade_coins": 60_000_000},
    9: {"diamonds": 900_000, "coins": 135_000_000, "upgrade_diamonds": 650_000, "upgrade_coins": 97_500_000},
    10: {"diamonds": 1_300_000, "coins": 195_000_000, "upgrade_diamonds": 1_000_000, "upgrade_coins": 150_000_000},
    11: {"diamonds": 2_000_000, "coins": 300_000_000, "upgrade_diamonds": 1_600_000, "upgrade_coins": 240_000_000},
    12: {"diamonds": 3_000_000, "coins": 450_000_000, "upgrade_diamonds": 2_500_000, "upgrade_coins": 375_000_000},
}

# ------------------------------
# Показ личного банка
# ------------------------------
async def personal_bank_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args=None):
    try:
        user_id = update.effective_user.id
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("SELECT coins, diamonds, bank_level, bank_coins, bank_diamonds FROM users WHERE user_id = ?", (user_id,))
        row = cursor.fetchone()
        if not row:
            await update.message.reply_text("❌ Пользователь не найден.")
            conn.close()
            return
        coins, diamonds, level, bank_coins, bank_diamonds = row
        conn.close()

        max_coins = BANK_LEVELS[level]["coins"]
        max_diamonds = BANK_LEVELS[level]["diamonds"]

        text = (
            f"🏦 <b>Личный банк</b> (уровень {level}):\n"
            f"💎 Алмазы: {bank_diamonds}/{max_diamonds}\n"
            f"💰 Монеты: {bank_coins}/{max_coins}"
        )
        await update.message.reply_text(text, parse_mode="HTML")
    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при показе банка: {str(e)}")
        print(f"ERROR in personal_bank_command: {str(e)}")

# ------------------------------
# Внести ресурсы в банк
# ------------------------------
async def deposit_bank_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args=None):
    try:
        args = update.message.text.strip().split()

        if len(args) < 4:  # тгз внести <монеты/алмазы> <количество>
            await update.message.reply_text("❌ Используйте: тгз внести <монеты/алмазы> <количество>")
            return

        resource = args[2].lower()  # монеты или алмазы
        try:
            amount = int(args[3])
            if amount <= 0:
                raise ValueError
        except:
            await update.message.reply_text("❌ Количество должно быть положительным числом.")
            return

        user_id = update.effective_user.id
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("SELECT coins, diamonds, bank_level, bank_coins, bank_diamonds FROM users WHERE user_id = ?", (user_id,))
        row = cursor.fetchone()
        if not row:
            await update.message.reply_text("❌ Пользователь не найден.")
            conn.close()
            return

        coins, diamonds, level, bank_coins, bank_diamonds = row
        max_coins = BANK_LEVELS[level]["coins"]
        max_diamonds = BANK_LEVELS[level]["diamonds"]

        if resource == "монеты":
            if coins < amount:
                await update.message.reply_text("❌ У вас недостаточно монет.")
                conn.close()
                return
            if bank_coins + amount > max_coins:
                await update.message.reply_text(f"❌ В банке не хватает места. Макс: {max_coins}")
                conn.close()
                return
            coins -= amount
            bank_coins += amount
        elif resource == "алмазы":
            if diamonds < amount:
                await update.message.reply_text("❌ У вас недостаточно алмазов.")
                conn.close()
                return
            if bank_diamonds + amount > max_diamonds:
                await update.message.reply_text(f"❌ В банке не хватает места. Макс: {max_diamonds}")
                conn.close()
                return
            diamonds -= amount
            bank_diamonds += amount
        else:
            await update.message.reply_text("❌ Укажите ресурс: монеты или алмазы.")
            conn.close()
            return

        cursor.execute(
            "UPDATE users SET coins = ?, diamonds = ?, bank_coins = ?, bank_diamonds = ? WHERE user_id = ?",
            (coins, diamonds, bank_coins, bank_diamonds, user_id)
        )
        conn.commit()
        conn.close()
        await update.message.reply_text(f"✅ Вы внесли {amount} {resource} в банк.")

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при внесении: {str(e)}")
        print(f"ERROR in deposit_bank_command: {str(e)}")


# ------------------------------
# Снять ресурсы из банка
# ------------------------------
async def withdraw_bank_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args=None):
    try:
        args = update.message.text.strip().split()

        if len(args) < 4:  # тгз снять <монеты/алмазы> <количество>
            await update.message.reply_text("❌ Используйте: тгз снять <монеты/алмазы> <количество>")
            return

        resource = args[2].lower()  # монеты или алмазы
        try:
            amount = int(args[3])
            if amount <= 0:
                raise ValueError
        except:
            await update.message.reply_text("❌ Количество должно быть положительным числом.")
            return

        user_id = update.effective_user.id
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("SELECT coins, diamonds, bank_coins, bank_diamonds FROM users WHERE user_id = ?", (user_id,))
        row = cursor.fetchone()
        if not row:
            await update.message.reply_text("❌ Пользователь не найден.")
            conn.close()
            return

        coins, diamonds, bank_coins, bank_diamonds = row

        if resource == "монеты":
            if bank_coins < amount:
                await update.message.reply_text("❌ В банке недостаточно монет.")
                conn.close()
                return
            coins += amount
            bank_coins -= amount
        elif resource == "алмазы":
            if bank_diamonds < amount:
                await update.message.reply_text("❌ В банке недостаточно алмазов.")
                conn.close()
                return
            diamonds += amount
            bank_diamonds -= amount
        else:
            await update.message.reply_text("❌ Укажите ресурс: монеты или алмазы.")
            conn.close()
            return

        cursor.execute(
            "UPDATE users SET coins = ?, diamonds = ?, bank_coins = ?, bank_diamonds = ? WHERE user_id = ?",
            (coins, diamonds, bank_coins, bank_diamonds, user_id)
        )
        conn.commit()
        conn.close()
        await update.message.reply_text(f"✅ Вы сняли {amount} {resource} из банка.")

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при снятии: {str(e)}")
        print(f"ERROR in withdraw_bank_command: {str(e)}")

import sqlite3
import time
from telegram import Update
from telegram.ext import ContextTypes

# ------------------------------
# Конфиги
# ------------------------------
# Перезарядка в секундах для каждого уровня (индекс = текущий уровень перед апгрейдом)
BANK_COOLDOWNS = [
    10*60,      # lvl 1 -> 2 : 10 минут
    30*60,      # lvl 2 -> 3 : 30 минут
    60*60,      # lvl 3 -> 4 : 1 час
    4*3600,     # lvl 4 -> 5 : 4 часа
    8*3600,     # lvl 5 -> 6 : 8 часов
    24*3600,    # lvl 6 -> 7 : 24 часа
    2*86400,    # lvl 7 -> 8 : 2 дня
    4*86400,    # lvl 8 -> 9 : 4 дня
    8*86400,    # lvl 9 -> 10 : 8 дней
    16*86400,   # lvl 10 -> 11 : 16 дней
    32*86400,   # lvl 11 -> 12 : 32 дня
]

# ------------------------------
# Форматирование времени
# ------------------------------
def format_time(seconds: int) -> str:
    days, seconds = divmod(seconds, 86400)
    hours, seconds = divmod(seconds, 3600)
    minutes, seconds = divmod(seconds, 60)
    parts = []
    if days > 0: parts.append(f"{days}д")
    if hours > 0: parts.append(f"{hours}ч")
    if minutes > 0: parts.append(f"{minutes}м")
    if seconds > 0: parts.append(f"{seconds}с")
    return " ".join(parts) if parts else "0с"

# ------------------------------
# Команда улучшения банка
# ------------------------------
async def upgrade_bank_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args=None):
    try:
        user_id = update.effective_user.id
        conn = sqlite3.connect(DB_PATH)
        conn.row_factory = sqlite3.Row
        cursor = conn.cursor()
        cursor.execute("SELECT coins, diamonds, bank_level, bank_cooldown FROM users WHERE user_id = ?", (user_id,))
        row = cursor.fetchone()
        if not row:
            await update.message.reply_text("❌ Пользователь не найден.")
            conn.close()
            return

        coins, diamonds, level, bank_cooldown = row["coins"], row["diamonds"], row["bank_level"], row["bank_cooldown"]

        # Проверка перезарядки
        now = int(time.time())
        if bank_cooldown > now:
            remaining = bank_cooldown - now
            await update.message.reply_text(f"🏦 Банк строится! Оставшееся время: {format_time(remaining)}")
            conn.close()
            return

        if level >= MAX_BANK_LEVEL:
            await update.message.reply_text("🏦 Ваш банк уже максимально прокачан.")
            conn.close()
            return

        next_level = level + 1
        cost_coins = BANK_LEVELS[next_level]["upgrade_coins"]
        cost_diamonds = BANK_LEVELS[next_level]["upgrade_diamonds"]

        if coins < cost_coins or diamonds < cost_diamonds:
            await update.message.reply_text(f"❌ Недостаточно ресурсов: нужно {cost_coins} монет и {cost_diamonds} алмазов.")
            conn.close()
            return

        # Списание ресурсов
        coins -= cost_coins
        diamonds -= cost_diamonds

        # Установка новой перезарядки
        cooldown_seconds = BANK_COOLDOWNS[level]  # индекс = текущий уровень до апгрейда
        new_cooldown = now + cooldown_seconds

        cursor.execute(
            "UPDATE users SET coins = ?, diamonds = ?, bank_level = ?, bank_cooldown = ? WHERE user_id = ?",
            (coins, diamonds, next_level, new_cooldown, user_id)
        )
        conn.commit()
        conn.close()

        await update.message.reply_text(f"✅ Банк улучшен до уровня {next_level}!\n⏱ Следующее улучшение будет доступно через {format_time(cooldown_seconds)}")

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при улучшении банка: {str(e)}")
        print(f"ERROR in upgrade_bank_command: {str(e)}")

import random
from telegram import Update
from telegram.ext import ContextTypes

async def chance(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    user_data = get_user(user.id, user.username, user.first_name)

    if user_data is None or len(user_data) < 24:
        await update.message.reply_text("❌ Пользователь не найден!")
        return

    user_id, player_id, username, first_name, diamonds, points, *rest = user_data

    message = update.message.text.lower().split()

    # тгз шанс <процент> <ставка>
    if len(message) < 4 or message[:2] != ["тгз", "шанс"]:
        await update.message.reply_text("❌ Использование: тгз шанс <процент> <ставка>")
        return

    try:
        chance = int(message[2])
        bet = int(message[3])
    except ValueError:
        await update.message.reply_text("❌ Процент и ставка должны быть числами!")
        return

    if chance < 20 or chance > 80:
        await update.message.reply_text("❌ Шанс должен быть от 20 до 80%")
        return

    if bet < 700:
        await update.message.reply_text("❌ Минимальная ставка: 700")
        return

    if bet > diamonds:
        await update.message.reply_text(f"❌ Недостаточно алмазов! У вас: {diamonds}")
        return

    # ===== коэффициент =====
    multiplier = round(0.75 / (chance / 100), 2)

    # ===== бросок =====
    roll = random.randint(10, 100)

    if roll <= chance:
        # выигрыш
        win = int(bet * multiplier)
        new_balance = diamonds - bet + win
        update_user(user_id, diamonds=new_balance)

        await update.message.reply_text(
            f"🎯 Успех!\n"
            f"Ваш шанс: {chance}%\n"
            f"Коэффициент: x{multiplier}\n"
            f"💎 Вы выиграли: {win}\n"
            f"Баланс: {new_balance}"
        )
    else:
        # проигрыш
        new_balance = diamonds - bet
        update_user(user_id, diamonds=new_balance)

        await update.message.reply_text(
            f"💥 Не повезло!\n"
            f"Ваш шанс: {chance}%\n"
            f"❌ Вы проиграли {bet} 💎\n"
            f"Баланс: {new_balance}"
        )

import asyncio
import random
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ContextTypes

# Эмодзи дверей по уровням
LEVEL_DOORS = {
    1: ["🚪", "🚪", "🚪", "🚪"],
    2: ["🗿", "🗿", "🗿", "🗿"],
    3: ["🛕", "🛕", "🛕", "🛕"],
    4: ["🏛️", "🏛️", "🏛️", "🏛️"],
    5: ["🏰", "🏰", "🏰", "🏰"],
    6: ["🎪", "🎪", "🎪", "🎪"],
    7: ["🕌", "🕌", "🕌", "🕌"],
    8: ["🏩", "🏩", "🏩", "🏩"],
}

PYR_COEF = {
1: 1.00,
2: 1.09,
3: 1.47,
4: 2.10,
5: 3.19,
6: 5.14,
7: 8.86,
8: 16.41
}

# ================================
# Запуск игры
# ================================
async def pyramid(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    user_id = user.id

    # Прямая проверка флага pyramid_active из базы
    conn = sqlite3.connect("diamonds_bot.db")  # ← подставь свой путь к базе
    cursor = conn.cursor()
    cursor.execute("SELECT diamonds, pyramid_active FROM users WHERE user_id = ?", (user_id,))
    row = cursor.fetchone()
    conn.close()

    if not row:
        await update.message.reply_text("❌ Пользователь не найден в базе")
        return

    diamonds, pyramid_active = row

    if pyramid_active == 1:
        await update.message.reply_text(
            "❌ У вас уже запущена пирамида!\n"
            "Сначала завершите её или заберите выигрыш."
        )
        return

    # Парсинг ставки
    message_text = update.message.text.strip().lower().split()
    if len(message_text) < 3 or message_text[:2] != ["тгз", "пирамида"]:
        await update.message.reply_text("❌ Использование: тгз пирамида <ставка>")
        return

    try:
        bet = int(message_text[2])
    except ValueError:
        await update.message.reply_text("❌ Ставка должна быть числом!")
        return

    if bet < 1400:
        await update.message.reply_text("❌ Минимальная ставка: 1400 алмазов!")
        return

    if bet > diamonds:
        await update.message.reply_text(f"❌ Недостаточно алмазов! У вас: {diamonds}")
        return

    # Списываем ставку + ставим флаг активной игры
    conn = sqlite3.connect("diamonds_bot.db")
    cursor = conn.cursor()
    cursor.execute(
        "UPDATE users SET diamonds = ?, pyramid_active = 1 WHERE user_id = ?",
        (diamonds - bet, user_id)
    )
    conn.commit()
    conn.close()

    # Сохраняем состояние в context.user_data
    context.user_data["pyramid"] = {
        "user_id": user_id,
        "bet": bet,
        "current_win": bet,
        "level": 1,
        "start_balance": diamonds
    }

    await send_pyramid_level(update, context)

async def send_pyramid_level(obj, context):
    """
    Отправляет/обновляет сообщение с текущим уровнем пирамиды
    obj может быть либо Update (первое сообщение), либо CallbackQuery (обновление)
    """
    game = context.user_data.get("pyramid")
    if not game:
        text = "❌ Игра не найдена или завершена"
        if isinstance(obj, Update):
            await obj.message.reply_text(text)
        else:
            await obj.edit_message_text(text)
        return

    level = game["level"]
    current_win = game["current_win"]

    lvl = game["level"]
    bet = game["bet"]

    next_coef = PYR_COEF.get(lvl + 1, PYR_COEF[8])
    next_win_approx = int(bet * next_coef)

    doors = LEVEL_DOORS.get(level, LEVEL_DOORS[8])

    text = (
        f"🏛️ <b>Пирамида удачи</b>\n\n"
        f"Уровень: <b>{level}</b> из 8\n"
        f"💎 Текущий выигрыш: <b>{current_win}</b>\n"
        f"Следующий ≈ <b>{next_win_approx}</b>\n\n"
        f"Выбери дверь:"
    )

    # Единый стиль callback_data для всех дверей
    keyboard = [
        [
            InlineKeyboardButton(f"{doors[0]}  1", callback_data=f"pyr_d_{level}_1"),
            InlineKeyboardButton(f"{doors[1]}  2", callback_data=f"pyr_d_{level}_2"),
        ],
        [
            InlineKeyboardButton(f"{doors[2]}  3", callback_data=f"pyr_d_{level}_3"),
            InlineKeyboardButton(f"{doors[3]}  4", callback_data=f"pyr_d_{level}_4"),
        ],
        [
            InlineKeyboardButton("💰 ЗАБРАТЬ", callback_data="pyr_take")
        ]
    ]

    reply_markup = InlineKeyboardMarkup(keyboard)

    # Отправка или редактирование
    if isinstance(obj, Update):
        await obj.message.reply_text(
            text,
            reply_markup=reply_markup,
            parse_mode="HTML",
            disable_web_page_preview=True
        )
    else:  # CallbackQuery
        await obj.edit_message_text(
            text,
            reply_markup=reply_markup,
            parse_mode="HTML",
            disable_web_page_preview=True
        )

async def pyramid_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    data = query.data
    user_id = query.from_user.id

    if not data.startswith("pyr_"):
        return

    # Проверка активности игры из базы
    conn = sqlite3.connect("diamonds_bot.db")
    cursor = conn.cursor()
    cursor.execute("SELECT pyramid_active FROM users WHERE user_id = ?", (user_id,))
    row = cursor.fetchone()
    conn.close()

    if not row or row[0] != 1:
        context.user_data.pop("pyramid", None)
        await query.edit_message_text("❌ Игра уже завершена или не активна")
        return

    game = context.user_data.get("pyramid")
    if not game or game["user_id"] != user_id:
        await query.answer("Это не ваша игра", show_alert=True)
        return

    # ─────────────── ЗАБРАТЬ ───────────────
    if data == "pyr_take":
        win = game["current_win"]

        conn = sqlite3.connect("diamonds_bot.db")
        cursor = conn.cursor()
        cursor.execute("SELECT diamonds FROM users WHERE user_id = ?", (user_id,))
        current_balance = cursor.fetchone()[0]

        new_balance = current_balance + win
        cursor.execute(
            "UPDATE users SET diamonds = ?, pyramid_active = 0 WHERE user_id = ?",
            (new_balance, user_id)
        )
        conn.commit()
        conn.close()

        await query.edit_message_text(
            f"💰 <b>Вы успешно забрали!</b>\n\n"
            f"Выигрыш: {win:,} 💎\n"
            f"Новый баланс: {new_balance:,} 💎",
            parse_mode="HTML"
        )
        context.user_data.pop("pyramid", None)
        return

    # ─────────────── ВЫБОР ДВЕРИ ───────────────
    if not data.startswith("pyr_d_"):
        await query.answer("Неизвестная кнопка", show_alert=True)
        return

    try:
        _, _, level_str, door_str = data.split("_")
        selected_level = int(level_str)
        door = int(door_str)
    except Exception:
        await query.answer("Ошибка обработки кнопки", show_alert=True)
        return

    # Мягкая проверка уровня — не даём нажимать старые
    if selected_level < game["level"]:
        await query.answer("Этот уровень уже пройден", show_alert=True)
        return

    # Если уровень выше текущего — тоже отклоняем (на всякий случай)
    if selected_level > game["level"]:
        await query.answer("Этот уровень ещё не открыт", show_alert=True)
        return

    # Шанс проигрыша
    lose_chance = 0.22 + (game["level"] - 1) * 0.04

    if random.random() < lose_chance:
        conn = sqlite3.connect("diamonds_bot.db")
        cursor = conn.cursor()
        cursor.execute("UPDATE users SET pyramid_active = 0 WHERE user_id = ?", (user_id,))
        conn.commit()
        conn.close()

        await query.edit_message_text(
            f"💥 Дверь оказалась ловушкой!\n"
            f"Вы потеряли ставку {game['bet']:,} 💎",
            parse_mode="HTML"
        )
        context.user_data.pop("pyramid", None)
        return

    # Успех → следующий уровень
    lvl = game["level"]
    game["level"] += 1

    prev_coef = PYR_COEF.get(lvl, 1)
    curr_coef = PYR_COEF.get(game["level"], 1)

    game["current_win"] = int(game["current_win"] * (curr_coef / prev_coef))

    # Если дошли до конца — можно автоматически завершить
    if game["level"] > 8:
        win = game["current_win"]
        conn = sqlite3.connect("diamonds_bot.db")
        cursor = conn.cursor()
        cursor.execute("SELECT diamonds FROM users WHERE user_id = ?", (user_id,))
        current_balance = cursor.fetchone()[0]
        new_balance = current_balance + win
        cursor.execute(
            "UPDATE users SET diamonds = ?, pyramid_active = 0 WHERE user_id = ?",
            (new_balance, user_id)
        )
        conn.commit()
        conn.close()

        await query.edit_message_text(
            f"🏆 <b>Поздравляем! Вы прошли всю пирамиду!</b>\n\n"
            f"Выигрыш: {win:,} 💎\n"
            f"Новый баланс: {new_balance:,} 💎",
            parse_mode="HTML"
        )
        context.user_data.pop("pyramid", None)
        return

    await send_pyramid_level(query, context)

import asyncio

async def dice_even_odd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    user_data = get_user(user.id, user.username, user.first_name)

    if user_data is None or len(user_data) < 24:
        await update.message.reply_text("❌ Пользователь не найден!")
        return

    (
        user_id, player_id, username, first_name, diamonds, points,
        *rest
    ) = user_data

    message_text = update.message.text.strip().lower().split()

    # Проверка команды
    if len(message_text) < 4 or message_text[:2] != ["тгз", "кубы"]:
        await update.message.reply_text(
            "❌ Использование:\n"
            "тгз кубы чёт 500\n"
            "тгз кубы нечёт 500\n"
            "тгз кубы 1-6 500"
        )
        return

    # ===== Нормализация: ё → е =====
    mode = message_text[2].replace("ё", "е")

    mode = message_text[2].lower()

    if "ё" in mode or "ë" in mode:
        await update.message.reply_text(
            "❌ Используйте только 'чет' или 'нечет' с буквой Е!"
        )
        return

    try:
        bet = int(message_text[3])
    except ValueError:
        await update.message.reply_text("❌ Ставка должна быть числом!")
        return

    if bet < 1400:
        await update.message.reply_text("❌ Минимальная ставка: 1400 алмазов!")
        return

    if bet > diamonds:
        await update.message.reply_text(
            f"❌ Недостаточно алмазов! У вас: {diamonds}, ставка: {bet}"
        )
        return

    # ===== Бросок настоящего кубика Telegram 🎲 =====
    dice_message = await update.message.reply_dice(emoji="🎲")
    dice_value = dice_message.dice.value
    dice_even = dice_value % 2 == 0

    # ⏳ Пауза, чтобы игрок увидел результат
    await asyncio.sleep(4)

    win = False
    multiplier = 0
    result_text = ""

    # ===== Режим ЧЁТ / НЕЧЁТ =====
    if mode in ["чет", "нечет"]:
        win = (mode == "чет" and dice_even) or (mode == "нечет" and not dice_even)
        if win:
            multiplier = 1.7
            result_text = "Вы угадали!"

    # ===== Режим ТОЧНОГО ЧИСЛА =====
    else:
        try:
            chosen_number = int(mode)
        except ValueError:
            await update.message.reply_text("❌ Нужно выбрать: чёт, нечёт или число 1–6!")
            return

        if chosen_number < 1 or chosen_number > 6:
            await update.message.reply_text("❌ Число должно быть от 1 до 6!")
            return

        if dice_value == chosen_number:
            win = True
            multiplier = 3.5
            result_text = "🎯 Точное попадание!"

    # ===== Объявление результата =====
    if win:
        win_amount = int(bet * multiplier)
        new_diamonds = diamonds - bet + win_amount
        update_user(user_id, diamonds=new_diamonds)

        await update.message.reply_text(
            f"🎲 Выпало {dice_value}! {result_text}\n"
            f"💎 Выигрыш: {win_amount} алмазов (x{multiplier})\n"
            f"💎 Новый баланс: {new_diamonds}"
        )
    else:
        new_diamonds = diamonds - bet
        update_user(user_id, diamonds=new_diamonds)

        await update.message.reply_text(
            f"🎲 Выпало {dice_value}.\n"
            f"❌ Вы проиграли {bet} алмазов.\n"
            f"💎 Новый баланс: {new_diamonds}"
        )

import asyncio
from telegram import Update
from telegram.ext import ContextTypes

async def basketball(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    user_data = get_user(user.id, user.username, user.first_name)

    if user_data is None or len(user_data) < 24:
        await update.message.reply_text("❌ Пользователь не найден!")
        return

    (
        user_id, player_id, username, first_name, diamonds, points,
        *rest
    ) = user_data

    message_text = update.message.text.strip().lower().split()

    # Проверка команды
    if len(message_text) < 3 or message_text[:2] != ["тгз", "баскетбол"]:
        await update.message.reply_text("❌ Использование: тгз баскетбол <ставка>")
        return

    # Проверка ставки
    try:
        bet = int(message_text[2])
    except ValueError:
        await update.message.reply_text("❌ Ставка должна быть числом!")
        return

    if bet < 1400:
        await update.message.reply_text("❌ Минимальная ставка: 1400 алмазов!")
        return

    if bet > diamonds:
        await update.message.reply_text(
            f"❌ Недостаточно алмазов! У вас: {diamonds}, ставка: {bet}"
        )
        return

    # ===== Анимация броска мяча 🏀 =====
    dice_message = await update.message.reply_dice(emoji="🏀")
    await asyncio.sleep(4)  # ждём, пока анимация пройдёт

    # ===== Получаем значение анимации =====
    dice_value = dice_message.dice.value

    # ===== Определяем попадание или промах =====
    if dice_value in [4, 5]:  # Попадание
        multiplier = 2.1
        win_amount = int(bet * multiplier)
        new_diamonds = diamonds - bet + win_amount
        update_user(user_id, diamonds=new_diamonds)

        await update.message.reply_text(
            f"🏀 Мяч попал в кольцо!\n"
            f"💎 Вы выиграли: {win_amount} алмазов (x{multiplier})\n"
            f"💎 Новый баланс: {new_diamonds}"
        )
    else:  # 1,2,3 → Промах
        new_diamonds = diamonds - bet
        update_user(user_id, diamonds=new_diamonds)

        await update.message.reply_text(
            f"🏀 Мяч промахнулся!\n"
            f"❌ Вы проиграли {bet} алмазов.\n"
            f"💎 Новый баланс: {new_diamonds}"
        )

import asyncio
from telegram import Update
from telegram.ext import ContextTypes

async def football(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    user_data = get_user(user.id, user.username, user.first_name)

    if user_data is None or len(user_data) < 24:
        await update.message.reply_text("❌ Пользователь не найден!")
        return

    (
        user_id, player_id, username, first_name, diamonds, points,
        *rest
    ) = user_data

    message_text = update.message.text.strip().lower().split()

    # Проверка команды
    if len(message_text) < 3 or message_text[:2] != ["тгз", "футбол"]:
        await update.message.reply_text("❌ Использование: тгз футбол <ставка>")
        return

    # Проверка ставки
    try:
        bet = int(message_text[2])
    except ValueError:
        await update.message.reply_text("❌ Ставка должна быть числом!")
        return

    if bet < 1400:
        await update.message.reply_text("❌ Минимальная ставка: 1400 алмазов!")
        return

    if bet > diamonds:
        await update.message.reply_text(
            f"❌ Недостаточно алмазов! У вас: {diamonds}, ставка: {bet}"
        )
        return

    # ===== Анимация удара мяча ⚽ =====
    dice_message = await update.message.reply_dice(emoji="⚽")
    dice_value = dice_message.dice.value
    await asyncio.sleep(4)  # ждём анимацию

    # ===== Определяем результат и множитель =====
    if dice_value in [1, 2]:  # промах
        multiplier = 0
        result_text = "⚽ Промах!"
    elif dice_value == 3:  # слабый гол
        multiplier = 0.8
        result_text = "⚽ Слабый гол!"
    elif dice_value == 4:  # средний гол
        multiplier = 1.4
        result_text = "⚽ Средний гол!"
    elif dice_value == 5:  # сильный гол
        multiplier = 2
        result_text = "⚽ Сильный гол!"
    else:  # на всякий случай (6)
        multiplier = 0
        result_text = "⚽ Промах!"

    # ===== Расчет выигрыша =====
    if multiplier > 0:
        win_amount = int(bet * multiplier)
        new_diamonds = diamonds - bet + win_amount
        update_user(user_id, diamonds=new_diamonds)

        await update.message.reply_text(
            f"{result_text}\n"
            f"💎 Вы выиграли: {win_amount} алмазов (x{multiplier})\n"
            f"💎 Новый баланс: {new_diamonds}"
        )
    else:
        new_diamonds = diamonds - bet
        update_user(user_id, diamonds=new_diamonds)

        await update.message.reply_text(
            f"{result_text}\n"
            f"❌ Вы проиграли {bet} алмазов.\n"
            f"💎 Новый баланс: {new_diamonds}"
        )

import sqlite3
import time
from telegram import Update
from telegram.ext import ContextTypes

DB_PATH = "diamonds_bot.db"

import sqlite3
import html
from telegram.ext import Application, CommandHandler

# Определяем DB_PATH если ещё нет
DB_PATH = 'diamonds_bot.db'

# Функция debug_command
async def debug_command(update, context):
    user_id = update.effective_user.id
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    # Получаем структуру таблицы
    cursor.execute("PRAGMA table_info(users)")
    columns = cursor.fetchall()

    # Получаем данные пользователя
    cursor.execute("SELECT * FROM users WHERE user_id = ?", (user_id,))
    user_data = cursor.fetchone()

    text = "📊 **Структура таблицы users:**\n"
    for col in columns:
        text += f"{col[0]}. {col[1]} ({col[2]})\n"

    text += "\n📋 **Ваши данные:**\n"
    for i, value in enumerate(user_data):
        col_name = columns[i][1] if i < len(columns) else f"unknown_{i}"
        text += f"[{i}] {col_name} = {value}\n"

    conn.close()
    await update.message.reply_text(f"<pre>{html.escape(text)}</pre>", parse_mode='HTML')

FORGE_PRODUCTION = {
    "sword_wood": {"cost": {"wood": 27000}},
    "sword_stone": {"cost": {"stone": 24000}},
    "sword_iron": {"cost": {"iron": 8000}},
    "sword_titanium": {"cost": {"titanium": 4000}},
    "sword_diamond": {"cost": {"diamonds": 800}},

    "armor_wood": {"cost": {"wood": 27000}},
    "armor_stone": {"cost": {"stone": 24000}},
    "armor_iron": {"cost": {"iron": 8000}},
    "armor_titanium": {"cost": {"titanium": 4000}},
    "armor_diamond": {"cost": {"diamonds": 800}},
}

ITEM_NAMES = {
    "sword_wood": "🪵 Деревянный меч",
    "sword_stone": "🪨 Каменный меч",
    "sword_iron": "🔩 Железный меч",
    "sword_titanium": "⚙ Титановый меч",
    "sword_diamond": "💎 Алмазный меч",

    "armor_wood": "🪵 Деревянная броня",
    "armor_stone": "🪨 Каменная броня",
    "armor_iron": "🔩 Железная броня",
    "armor_titanium": "⚙ Титановая броня",
    "armor_diamond": "💎 Алмазная броня"
}

async def forge_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user_id = update.effective_user.id
        now = int(time.time())

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        cursor.execute("""
            SELECT forge_level, last_forge_update,
                   wood, stone, iron, titanium, diamonds,
                   sword_wood, sword_stone, sword_iron, sword_titanium, sword_diamond,
                   armor_wood, armor_stone, armor_iron, armor_titanium, armor_diamond
            FROM users WHERE user_id=?
        """, (user_id,))

        row = cursor.fetchone()

        if not row:
            await update.message.reply_text("❌ Вы не зарегистрированы.")
            conn.close()
            return

        (
            forge_level, last_update,
            wood, stone, iron, titanium, diamonds,
            sword_wood, sword_stone, sword_iron, sword_titanium, sword_diamond,
            armor_wood, armor_stone, armor_iron, armor_titanium, armor_diamond
        ) = row

        # Первый запуск
        if last_update == 0:
            cursor.execute("UPDATE users SET last_forge_update=? WHERE user_id=?", (now, user_id))
            conn.commit()
            conn.close()
            await update.message.reply_text("🛠 Кузница запущена!")
            return

        seconds_passed = now - last_update
        hours = seconds_passed // 3600

        if hours <= 0:
            await update.message.reply_text("⌛ Кузница ещё работает.")
            conn.close()
            return

        resources = {
            "wood": wood,
            "stone": stone,
            "iron": iron,
            "titanium": titanium,
            "diamonds": diamonds
        }

        produced_text = []

        # 🔥 уровень 0 — производит всё
        for item, info in FORGE_PRODUCTION.items():

            cursor.execute(
                f"SELECT prod_{item} FROM users WHERE user_id=?",
                (user_id,)
            )
            enabled = cursor.fetchone()[0]

            if enabled == 0:
                continue  # ❌ снаряжение отключено

            # считаем сколько можем сделать
            items_per_hour = forge_level + 1
            possible = hours * items_per_hour

            for res, cost in info["cost"].items():
                possible = min(possible, resources[res] // cost)

            if possible <= 0:
                continue

            # списываем ресурсы
            for res, cost in info["cost"].items():
                resources[res] -= cost * possible

            # добавляем предметы
            cursor.execute(
                f"UPDATE users SET {item} = {item} + ? WHERE user_id=?",
                (possible, user_id)
            )

            produced_text.append(f"{ITEM_NAMES[item]} × {possible}")

        # сохраняем ресурсы
        cursor.execute("""
            UPDATE users
            SET wood=?, stone=?, iron=?, titanium=?, diamonds=?, last_forge_update=?
            WHERE user_id=?
        """, (
            resources["wood"], resources["stone"], resources["iron"],
            resources["titanium"], resources["diamonds"], now, user_id
        ))

        conn.commit()
        conn.close()

        if produced_text:
            text = "\n".join(produced_text)
        else:
            text = "❌ Недостаточно ресурсов."

        await update.message.reply_text(f"🛠 Кузница произвела:\n\n{text}")

    except Exception as e:
        print(e)
        await update.message.reply_text(f"Ошибка: {e}")

async def towers_list_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = (
        "🏰 *Башни защиты*\n\n"

        "1️⃣ 🔥 *Огненная башня*\n"
        "💰 Стоимость: 2 000 000 монет + 66 000 алмазов\n"
        "🛡 Прочность: 18 000\n"
        "⚔ Урон: 300 по каждому зомби\n"
        "✨ Эффект:\n"
        "🔥 *Испепеление* — 15% шанс мгновенно уничтожить 1 атакующего зомби\n\n"

        "2️⃣ ☠️ *Ядовитая башня*\n"
        "💰 Стоимость: 2 000 000 монет + 66 000 алмазов\n"
        "🛡 Прочность: 18 000\n"
        "⚔ Урон: 300 по каждому зомби\n"
        "✨ Эффект:\n"
        "☣️ *Токсин* — яд на всех атакующих зомби\n"
        "➕ Доп. урон: 150 по каждому зомби\n\n"

        "3️⃣ 🔮 *Магическая башня*\n"
        "💰 Стоимость: 2 000 000 монет + 66 000 алмазов\n"
        "🛡 Прочность: 18 000\n"
        "⚔ Урон: 300 по каждому зомби\n"
        "✨ Эффект:\n"
        "✨ *Ослабление магией* — урон зомби −30%\n\n"

        "4️⃣ 🏹 *Арбалетная башня*\n"
        "💰 Стоимость: 2 000 000 монет + 66 000 алмазов\n"
        "🛡 Прочность: 18 000\n"
        "⚔ Урон: 300 по каждому зомби\n"
        "✨ Эффект:\n"
        "🎯 *Проникающий болт* — 50% шанс нанести дополнительные 300 урона по всем зомби\n\n"

        "5️⃣ 🕯️ *Проклинающая башня*\n"
        "💰 Стоимость: 2 000 000 монет + 66 000 алмазов\n"
        "🛡 Прочность: 18 000\n"
        "⚔ Урон: 300 по каждому зомби\n"
        "✨ Эффект:\n"
        "🩸 *Проклятие слабости* — урон по зомби +25%\n\n"
    )

    await update.message.reply_text(text, parse_mode="Markdown")

async def buy_tower_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_data = get_user(user.id, user.username, user.first_name)

        if user_data is None or len(user_data) < 25:
            await update.message.reply_text("❌ Пользователь не найден!")
            return

        (
            user_id, player_id, username, first_name, diamonds, points,
            last_diamond_time, upgrade_level, registration_date, strawberries,
            farm_level, last_zombie_attack, zombie_status, last_strawberry_update,
            last_bonus_time, tomatoes, coins, farms_destroyed, clan_id,
            *rest
        ) = user_data

        message_text = update.message.text.strip().lower().split()

        # Проверка формата команды
        if len(message_text) != 4 or message_text[:3] != ["тгз", "купить", "башню"]:
            await update.message.reply_text("❌ Использование: тгз купить башню <номер>")
            return

        # Извлекаем номер башни
        try:
            tower_number = int(message_text[3])
        except ValueError:
            await update.message.reply_text("❌ Номер башни должен быть числом!")
            return

        # Настройки башен
        TOWERS = {
            1: {"name": "огненная", "coins": 2_000_000, "diamonds": 66_000, "column": "tower_fire", "health": 18000},
            2: {"name": "ядовитая", "coins": 2_000_000, "diamonds": 66_000, "column": "tower_poison", "health": 18000},
            3: {"name": "магическая", "coins": 2_000_000, "diamonds": 66_000, "column": "tower_magic", "health": 18000},
            4: {"name": "арбалетная", "coins": 2_000_000, "diamonds": 66_000, "column": "tower_crossbow", "health": 18000},
            5: {"name": "проклинающая", "coins": 2_000_000, "diamonds": 66_000, "column": "tower_curse", "health": 18000},
        }

        if tower_number not in TOWERS:
            await update.message.reply_text("❌ Неверный номер башни. Доступно: 1-5")
            return

        tower = TOWERS[tower_number]
        column = tower["column"]
        cost_coins = tower["coins"]
        cost_diamonds = tower["diamonds"]
        tower_health = tower["health"]

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # Получаем ресурсы, здоровье и проверяем наличие башни
        cursor.execute(f"""
            SELECT coins, diamonds, {column}, max_farm_health, new_farm_health
            FROM users WHERE user_id = ?
        """, (user_id,))
        row = cursor.fetchone()

        if not row:
            await update.message.reply_text("❌ Вы не зарегистрированы.")
            conn.close()
            return

        coins, diamonds, has_tower, max_health, new_health = row

        if has_tower:
            await update.message.reply_text(f"❌ У вас уже есть башня '{tower['name']}'.")
            conn.close()
            return

        if coins < cost_coins or diamonds < cost_diamonds:
            await update.message.reply_text(
                f"❌ Недостаточно ресурсов.\nМонет: {coins}/{cost_coins}\nАлмазов: {diamonds}/{cost_diamonds}"
            )
            conn.close()
            return

        # Списываем ресурсы, добавляем башню и увеличиваем здоровье фермы
        cursor.execute(f"""
            UPDATE users
            SET coins = coins - ?,
                diamonds = diamonds - ?,
                {column} = 1,
                max_farm_health = max_farm_health + ?,
                new_farm_health = new_farm_health + ?
            WHERE user_id = ?
        """, (cost_coins, cost_diamonds, tower_health, tower_health, user_id))

        conn.commit()
        conn.close()

        await update.message.reply_text(
            f"✅ Вы купили башню '{tower['name']}'!\n"
            f"🏰 Здоровье фермы увеличено на {tower_health}"
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка: {e}")

import sqlite3
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ContextTypes

# Пример данных прокачки кузницы
LEVELS_PER_PAGE = 5
MAX_FORGE_LEVEL = 9
FORGE_LEVEL_COST = 384000

async def forge_upgrade_info_command(update: Update, context: ContextTypes.DEFAULT_TYPE, page: int = 0):
    try:
        user_id = update.effective_user.id

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("SELECT forge_level FROM users WHERE user_id=?", (user_id,))
        row = cursor.fetchone()
        conn.close()

        if not row:
            await update.message.reply_text("❌ Вы не зарегистрированы!")
            return

        user_level = row[0]

        text = "🛠 <b>Уровни кузницы</b>\n\n"

        start = page * LEVELS_PER_PAGE
        end = min(start + LEVELS_PER_PAGE, MAX_FORGE_LEVEL + 1)

        for lvl in range(start, end):

            items_per_hour = lvl + 1

            # Стоимость уровня
            if lvl == 0:
                cost_text = "Бесплатно"
            else:
                cost_value = lvl * FORGE_LEVEL_COST
                cost_text = f"{cost_value:,} железа".replace(",", " ")

            status = ""
            if lvl == user_level:
                status = " ⭐ ТЕКУЩИЙ"
            elif lvl < user_level:
                status = " ✅"

            text += f"<b>Уровень {lvl}</b>{status}\n"
            text += f"⛓️ Стоимость: {cost_text}\n"
            text += f"⚔ Производство: {items_per_hour} шт/час\n\n"

        # кнопки страниц
        keyboard = []

        if page > 0:
            keyboard.append(InlineKeyboardButton("⬅️ Назад", callback_data=f"forge_page_{page-1}"))

        if end <= MAX_FORGE_LEVEL:
            keyboard.append(InlineKeyboardButton("➡️ Вперед", callback_data=f"forge_page_{page+1}"))

        reply_markup = InlineKeyboardMarkup([keyboard]) if keyboard else None

        if update.callback_query:
            await update.callback_query.edit_message_text(
                text=text,
                parse_mode="HTML",
                reply_markup=reply_markup
            )
        else:
            await update.message.reply_text(
                text,
                parse_mode="HTML",
                reply_markup=reply_markup
            )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка: {e}")

async def forge_upgrade_page_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    try:
        page = int(query.data.split("_")[-1])
    except:
        page = 0

    await forge_upgrade_info_command(update, context, page)

def check_global_diamond_limit(user_id: int) -> bool:
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    # Проверяем алмазы
    cursor.execute("SELECT diamonds, shop_banned FROM users WHERE user_id = ?", (user_id,))
    row = cursor.fetchone()

    if row:
        diamonds, shop_banned = row
        # Если достигнут лимит, включаем вечный запрет
        if diamonds >= 7_000_000 and shop_banned == 0:
            cursor.execute("UPDATE users SET shop_banned = 1 WHERE user_id = ?", (user_id,))
            conn.commit()
            shop_banned = 1
    conn.close()

    return shop_banned == 1

async def chat_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    cursor.execute("""
        SELECT username, player_id, text
        FROM chat_messages
        ORDER BY id DESC
        LIMIT 15
    """)
    rows = cursor.fetchall()
    conn.close()

    if not rows:
        await update.message.reply_text("💭 Чат пуст")
        return

    text = "🗨 <b>Чат:</b>\n\n"

    for username, player_id, msg in reversed(rows):
        text += f"👤 <b>{username}</b> [ID {player_id}]\n"
        text += f"💬 {msg}\n\n"

    await update.message.reply_text(text, parse_mode="HTML")

import re
import time
import sqlite3
from telegram import Update
from telegram.ext import ContextTypes

async def message_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    user_id = user.id

    if not context.args:
        await update.message.reply_text("❌ Укажи текст сообщения")
        return

    text = " ".join(context.args)
    if len(text) > 30:
        await update.message.reply_text("❌ Максимум 30 символов")
        return

    if re.search(r'[A-Za-z]', text):
        await update.message.reply_text("❌ Английские буквы запрещены!")
        return

    now = int(time.time())

    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    cursor.execute(
        "SELECT message_cooldown, player_id FROM users WHERE user_id = ?",
        (user_id,)
    )
    row = cursor.fetchone()

    if not row:
        conn.close()
        await update.message.reply_text("❌ Пользователь не найден")
        return

    cooldown_until, player_id = row
    cooldown_until = cooldown_until or 0

    if now < cooldown_until:
        conn.close()
        await update.message.reply_text(
            f"⏳ Подожди {cooldown_until - now} сек."
        )
        return

    username = user.username or user.first_name or "Без ника"

    cursor.execute("""
        INSERT INTO chat_messages (user_id, player_id, username, text, timestamp)
        VALUES (?, ?, ?, ?, ?)
    """, (
        user_id,
        player_id,
        username,
        text,
        now
    ))

    cursor.execute(
        "UPDATE users SET message_cooldown = ? WHERE user_id = ?",
        (now + 45, user_id)
    )

    conn.commit()
    conn.close()

    await update.message.reply_text("💬 Сообщение отправлено")

async def upgrade_forge_command(update, context):
    try:
        user_id = update.effective_user.id

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # Получаем уровень кузницы и железо
        cursor.execute("SELECT forge_level, iron FROM users WHERE user_id=?", (user_id,))
        row = cursor.fetchone()

        if not row:
            await update.message.reply_text("❌ Вы не зарегистрированы!")
            conn.close()
            return

        forge_level, iron = row

        # Проверка максимального уровня
        if forge_level >= MAX_FORGE_LEVEL:
            await update.message.reply_text("🏆 Кузница уже максимального уровня!")
            conn.close()
            return

        # Стоимость следующего уровня
        next_level = forge_level + 1
        upgrade_cost = next_level * FORGE_LEVEL_COST

        # Проверка железа
        if iron < upgrade_cost:
            await update.message.reply_text(
                f"❌ Недостаточно железа!\n"
                f"⛓️ Нужно: {upgrade_cost:,}".replace(",", " ")
            )
            conn.close()
            return

        # Улучшаем кузницу
        new_iron = iron - upgrade_cost

        cursor.execute("""
            UPDATE users
            SET forge_level=?, iron=?
            WHERE user_id=?
        """, (next_level, new_iron, user_id))

        conn.commit()
        conn.close()

        await update.message.reply_text(
            f"🛠 Кузница улучшена до уровня {next_level}!\n"
            f"⚔ Производство: {next_level + 1} шт/час"
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка: {e}")

EQUIPMENT_MAP = {
    1: ("sword_wood", "Деревянные мечи"),
    2: ("sword_stone", "Каменные мечи"),
    3: ("sword_iron", "Железные мечи"),
    4: ("sword_titanium", "Титановые мечи"),
    5: ("sword_diamond", "Алмазные мечи"),
    6: ("armor_wood", "Деревянная броня"),
    7: ("armor_stone", "Каменная броня"),
    8: ("armor_iron", "Железная броня"),
    9: ("armor_titanium", "Титановая броня"),
    10: ("armor_diamond", "Алмазная броня"),
}

async def disable_equipment_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args):
    if not args or not args[0].isdigit():
        await update.message.reply_text("❌ Укажи номер снаряжения (1–10)")
        return

    num = int(args[0])
    if num not in EQUIPMENT_MAP:
        await update.message.reply_text("❌ Номер должен быть от 1 до 10")
        return

    user_id = update.effective_user.id
    item, name = EQUIPMENT_MAP[num]

    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute(
        f"UPDATE users SET prod_{item}=0 WHERE user_id=?",
        (user_id,)
    )
    conn.commit()
    conn.close()

    await update.message.reply_text(f"⛔ Производство отключено: {name}")

async def enable_equipment_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args):
    if not args or not args[0].isdigit():
        await update.message.reply_text("❌ Укажи номер снаряжения (1–10)")
        return

    num = int(args[0])
    if num not in EQUIPMENT_MAP:
        await update.message.reply_text("❌ Номер должен быть от 1 до 10")
        return

    user_id = update.effective_user.id
    item, name = EQUIPMENT_MAP[num]

    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute(
        f"UPDATE users SET prod_{item}=1 WHERE user_id=?",
        (user_id,)
    )
    conn.commit()
    conn.close()

    await update.message.reply_text(f"✅ Производство включено: {name}")

def get_upgrade_info(level):
    upgrade_levels = {
        0: {"cost": 0, "min_diamonds": 20, "max_diamonds": 24},
        1: {"cost": 449, "min_diamonds": 24, "max_diamonds": 28},
        2: {"cost": 898, "min_diamonds": 28, "max_diamonds": 32},
        3: {"cost": 1347, "min_diamonds": 32, "max_diamonds": 36},
        4: {"cost": 1797, "min_diamonds": 36, "max_diamonds": 42},
        5: {"cost": 2246, "min_diamonds": 42, "max_diamonds": 48},
        6: {"cost": 2695, "min_diamonds": 48, "max_diamonds": 54},
        7: {"cost": 3144, "min_diamonds": 54, "max_diamonds": 62},
        8: {"cost": 3594, "min_diamonds": 62, "max_diamonds": 70},
        9: {"cost": 4043, "min_diamonds": 70, "max_diamonds": 78},
        10: {"cost": 4492, "min_diamonds": 78, "max_diamonds": 88},
        11: {"cost": 4941, "min_diamonds": 88, "max_diamonds": 98},
        12: {"cost": 5391, "min_diamonds": 98, "max_diamonds": 108},
        13: {"cost": 5840, "min_diamonds": 108, "max_diamonds": 120},
        14: {"cost": 6289, "min_diamonds": 120, "max_diamonds": 132},
        15: {"cost": 6738, "min_diamonds": 132, "max_diamonds": 144},
        16: {"cost": 7188, "min_diamonds": 144, "max_diamonds": 148},
        17: {"cost": 7637, "min_diamonds": 148, "max_diamonds": 172},
        18: {"cost": 8086, "min_diamonds": 172, "max_diamonds": 186},
        19: {"cost": 8535, "min_diamonds": 186, "max_diamonds": 202},
        20: {"cost": 8985, "min_diamonds": 202, "max_diamonds": 218},
    }

    print(f"DEBUG: get_upgrade_info called with level={level}, returning {upgrade_levels.get(level)}")
    return upgrade_levels.get(level)

QUEST_REQUIREMENTS = {
    1: {"zombie_id": 1, "need": 1500, "text": "Накопите 1500 обычных зомби"},
    2: {"zombie_id": 2, "need": 600,  "text": "Накопите 600 бегунов"},
    3: {"zombie_id": 3, "need": 300,  "text": "Накопите 300 взрывунов"},
    4: {"zombie_id": 4, "need": 200,  "text": "Накопите 200 невидимок"},
    5: {"zombie_id": 5, "need": 188,  "text": "Накопите 188 плювак"}
}

async def quests_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    if check_global_diamond_limit(user_id):
        await update.message.reply_text("❌ У вас заблокированы любые действия.")

    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    # --- Создаём таблицу, если нет ---
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS user_quests (
            user_id INTEGER PRIMARY KEY,
            q1 INTEGER DEFAULT 0,
            q2 INTEGER DEFAULT 0,
            q3 INTEGER DEFAULT 0,
            q4 INTEGER DEFAULT 0,
            q5 INTEGER DEFAULT 0
        )
    """)

    # --- Создаём запись пользователя, если нет ---
    cursor.execute("""
        INSERT INTO user_quests (user_id)
        VALUES (?) ON CONFLICT(user_id) DO NOTHING
    """, (user_id,))

    # --- Получаем прогресс зомби пользователя ---
    cursor.execute("""
        SELECT zombie_id, quantity FROM user_zombies
        WHERE user_id = ?
    """, (user_id,))
    zombie_rows = cursor.fetchall()
    zombie_owned = {z_id: qty for z_id, qty in zombie_rows}

    # --- Забираем выполненные квесты ---
    cursor.execute("SELECT q1, q2, q3, q4, q5 FROM user_quests WHERE user_id = ?", (user_id,))
    q = cursor.fetchone()
    if q is None:
        q = (0, 0, 0, 0, 0)  # если записи нет, считаем все квесты невыполненными

    # --- Вычисляем скидку до закрытия соединения ---
    cursor.execute("SELECT zombie_discount FROM users WHERE user_id = ?", (user_id,))
    discount_row = cursor.fetchone()
    discount = discount_row[0] if discount_row else q.count(1) * 10

    conn.close()  # закрываем соединение только после всех запросов

    # --- Формируем текст ---
    text = "📜 <b>Список квестов</b>\n"
    text += f"🎁 Текущая скидка: <b>{discount}%</b>\n\n"

    for quest_id, quest in QUEST_REQUIREMENTS.items():
        need = quest["need"]
        have = zombie_owned.get(quest["zombie_id"], 0)
        status = "✅ Выполнено" if q[quest_id - 1] == 1 else "❌ Не выполнено"
        left = max(0, need - have)

        text += (
            f"<b>{quest_id}) {quest['text']}</b>\n"
            f"➡️ Куплено: <b>{have}/{need}</b>\n"
            f"📌 Статус: {status}\n"
        )

        if left > 0:
            text += f"⏳ Осталось купить: {left}\n"

        text += "🎁 Награда: -10% стоимости всех зомби\n\n"

    await update.message.reply_text(text, parse_mode="HTML")

ARMORY_ITEMS = {
    "swords": [
        ("sword_wood", "🪵 Деревянный меч", "⚔️ +90 урона", "🪵 54 000 дерева"),
        ("sword_stone", "🪨 Каменный меч", "⚔️ +99 урона", "🪨 48 000 камня"),
        ("sword_iron", "🔩 Железный меч", "⚔️ +108 урона", "🔩 16 000 железа"),
        ("sword_titanium", "⚙ Титановый меч", "⚔️ +120 урона", "⚙ 8 000 титана"),
        ("sword_diamond", "💎 Алмазный меч", "⚔️ +264 урона", "💎 800 алмазов"),
        ("sword_ruby", "♦️ Рубиновый меч", "⚔️ +825 урона", "♦️ 1 рубин"),
    ],
    "armor": [
        ("armor_wood", "🪵 Деревянная броня", "❤️ +270 здоровья", "🪵 54 000 дерева"),
        ("armor_stone", "🪨 Каменная броня", "❤️ +297 здоровья", "🪨 48 000 камня"),
        ("armor_iron", "🔩 Железная броня", "❤️ +327 здоровья", "🔩 16 000 железа"),
        ("armor_titanium", "⚙ Титановая броня", "❤️ +359 здоровья", "⚙ 8 000 титана"),
        ("armor_diamond", "💎 Алмазная броня", "❤️ +790 здоровья", "💎 800 алмазов"),
        ("armor_ruby", "♦️ Рубиновая броня", "❤️ +2470 здоровья", "♦️ 1 рубин"),
    ]
}

def build_armory_text(user_row, mode: str):
    text = ""

    if mode == "swords":
        text += "🗡 МЕЧИ ДЛЯ ЗОМБИ Ф\n\n"
        for i, (col, name, bonus, cost) in enumerate(ARMORY_ITEMS["swords"], 1):
            qty = user_row[col]
            text += (
                f"{i}) {name}\n"
                f"{bonus}\n"
                f"💰 Стоимость: {cost}\n"
                f"🗒️ Количество: {qty}\n\n"
            )

    if mode == "armor":
        text += "🛡 БРОНЯ ДЛЯ ЗОМБИ Ф\n\n"
        for i, (col, name, bonus, cost) in enumerate(ARMORY_ITEMS["armor"], 7):
            qty = user_row[col]
            text += (
                f"{i}) {name}\n"
                f"{bonus}\n"
                f"💰 Стоимость: {cost}\n"
                f"🗒️ Количество: {qty}\n\n"
            )

    return text

async def armory_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id

    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    cursor = conn.cursor()

    cursor.execute("""
        SELECT
            sword_wood, sword_stone, sword_iron, sword_titanium, sword_diamond, sword_ruby,
            armor_wood, armor_stone, armor_iron, armor_titanium, armor_diamond, armor_ruby
        FROM users WHERE user_id = ?
    """, (user_id,))
    row = cursor.fetchone()
    conn.close()

    if not row:
        await update.message.reply_text("❌ Пользователь не найден.")
        return

    text = build_armory_text(row, "swords")

    keyboard = [
        [
            InlineKeyboardButton("🗡 Мечи", callback_data="armory_swords"),
            InlineKeyboardButton("🛡 Броня", callback_data="armory_armor"),
        ]
    ]

    await update.message.reply_text(
        text,
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

async def armory_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    user_id = query.from_user.id
    mode = query.data.replace("armory_", "")

    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    cursor = conn.cursor()

    cursor.execute("""
        SELECT
            sword_wood, sword_stone, sword_iron, sword_titanium, sword_diamond, sword_ruby,
            armor_wood, armor_stone, armor_iron, armor_titanium, armor_diamond, armor_ruby
        FROM users WHERE user_id = ?
    """, (user_id,))
    row = cursor.fetchone()
    conn.close()

    if not row:
        await query.edit_message_text("❌ Пользователь не найден.")
        return

    text = build_armory_text(row, mode)

    keyboard = [
        [
            InlineKeyboardButton("🗡 Мечи", callback_data="armory_swords"),
            InlineKeyboardButton("🛡 Броня", callback_data="armory_armor"),
        ]
    ]

    await query.edit_message_text(
        text=text,
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

EQUIPMENT = {
    1: {
        "name": "🪵 Деревянный меч",
        "type": "sword",
        "column": "sword_wood",
        "cost": {"wood": 54000},
    },
    2: {
        "name": "🪨 Каменный меч",
        "type": "sword",
        "column": "sword_stone",
        "cost": {"stone": 48000},
    },
    3: {
        "name": "🔩 Железный меч",
        "type": "sword",
        "column": "sword_iron",
        "cost": {"iron": 16000},
    },
    4: {
        "name": "⚙ Титановый меч",
        "type": "sword",
        "column": "sword_titanium",
        "cost": {"titanium": 8000},
    },
    5: {
        "name": "💎 Алмазный меч",
        "type": "sword",
        "column": "sword_diamond",
        "cost": {"diamonds": 800},
    },
    6: {
        "name": "♦️ Рубиновый меч",
        "type": "sword",
        "column": "sword_ruby",
        "cost": {"rubies": 1},
    },

    7: {
        "name": "🪵 Деревянная броня",
        "type": "armor",
        "column": "armor_wood",
        "cost": {"wood": 54000},
    },
    8: {
        "name": "🪨 Каменная броня",
        "type": "armor",
        "column": "armor_stone",
        "cost": {"stone": 48000},
    },
    9: {
        "name": "🔩 Железная броня",
        "type": "armor",
        "column": "armor_iron",
        "cost": {"iron": 16000},
    },
    10: {
        "name": "⚙ Титановая броня",
        "type": "armor",
        "column": "armor_titanium",
        "cost": {"titanium": 8000},
    },
    11: {
        "name": "💎 Алмазная броня",
        "type": "armor",
        "column": "armor_diamond",
        "cost": {"diamonds": 800},
    },
    12: {
        "name": "♦️ Рубиновая броня",
        "type": "armor",
        "column": "armor_ruby",
        "cost": {"rubies": 1},
    },
}

async def buy_equipment_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args):
    try:
        if len(args) < 2:
            await update.message.reply_text(
                "❌ Использование:\n"
                "тгз купить снаряжение <номер> <количество>"
            )
            return

        number = int(args[1])
        quantity = int(args[2]) if len(args) >= 3 else 1

        if quantity <= 0:
            await update.message.reply_text("❌ Количество должно быть больше 0.")
            return

        if number not in EQUIPMENT:
            await update.message.reply_text("❌ Неверный номер снаряжения.")
            return

        equip = EQUIPMENT[number]
        user_id = update.effective_user.id

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # Получаем ресурсы игрока
        cursor.execute("""
            SELECT wood, stone, iron, titanium, diamonds, rubies, {}
            FROM users WHERE user_id = ?
        """.format(equip["column"]), (user_id,))
        row = cursor.fetchone()

        if not row:
            conn.close()
            await update.message.reply_text("❌ Вы не зарегистрированы.")
            return

        resources = {
            "wood": row[0],
            "stone": row[1],
            "iron": row[2],
            "titanium": row[3],
            "diamonds": row[4],
            "rubies": row[5],
        }
        owned = row[6]

        # Проверка ресурсов
        for res, cost in equip["cost"].items():
            total_cost = cost * quantity
            if resources.get(res, 0) < total_cost:
                conn.close()
                await update.message.reply_text(
                    f"❌ Недостаточно ресурса: {res}\n"
                    f"Нужно: {total_cost}, у вас: {resources.get(res, 0)}"
                )
                return

        # Списание ресурсов
        for res, cost in equip["cost"].items():
            resources[res] -= cost * quantity

        owned += quantity

        cursor.execute(f"""
            UPDATE users SET
                wood = ?, stone = ?, iron = ?, titanium = ?,
                diamonds = ?, rubies = ?,
                {equip["column"]} = ?
            WHERE user_id = ?
        """, (
            resources["wood"],
            resources["stone"],
            resources["iron"],
            resources["titanium"],
            resources["diamonds"],
            resources["rubies"],
            owned,
            user_id
        ))

        conn.commit()
        conn.close()

        await update.message.reply_text(
            f"✅ Покупка успешна!\n\n"
            f"{equip['name']}\n"
            f"📦 Куплено: {quantity}\n"
            f"🗒️ Всего: {owned}"
        )

    except ValueError:
        await update.message.reply_text("❌ Номер и количество должны быть числами.")
    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка: {e}")
        print(f"[BUY_EQUIPMENT ERROR] {e}")

async def equip_zombie_command(update, context, args):
    # тгз снарядить зомби <equip_id> <zombie_id> <qty>
    if len(args) != 4 or args[0].lower() != "зомби":
        await update.message.reply_text(
            "❌ Пример:\nтгз снарядить зомби <снаряжение_id> <зомби_id> <количество>"
        )
        return

    try:
        equip_id = int(args[1])
        zombie_id = int(args[2])
        qty = int(args[3])
    except ValueError:
        await update.message.reply_text("❌ ID и количество должны быть числами.")
        return

    if qty <= 0:
        await update.message.reply_text("❌ Количество должно быть больше 0.")
        return

    info = EQUIPMENT.get(equip_id)
    if not info:
        await update.message.reply_text("❌ Неверный ID снаряжения.")
        return

    equip_type = info["type"]   # sword / armor
    col = info["column"]
    name = info["name"]
    user_id = update.effective_user.id

    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    # 1️⃣ Проверяем количество зомби
    cursor.execute("""
        SELECT quantity FROM user_zombies
        WHERE user_id = ? AND zombie_id = ?
    """, (user_id, zombie_id))
    row = cursor.fetchone()

    if row is None or row[0] <= 0:
        conn.close()
        await update.message.reply_text("❌ У вас нет зомби с таким ID.")
        return

    total_zombies = row[0]

    # 2️⃣ Сколько уже экипировано этим типом
    cursor.execute("""
        SELECT COALESCE(quantity, 0)
        FROM zombie_equipment
        WHERE user_id = ? AND zombie_id = ? AND equip_type = ?
    """, (user_id, zombie_id, equip_type))
    eq_row = cursor.fetchone()
    equipped = eq_row[0] if eq_row else 0

    free = total_zombies - equipped

    if free <= 0:
        conn.close()
        await update.message.reply_text(
            f"❌ Все зомби уже имеют {equip_type}."
        )
        return

    if qty > free:
        conn.close()
        await update.message.reply_text(
            f"❌ Не хватает свободных зомби.\n"
            f"Свободно: {free}, нужно: {qty}"
        )
        return

    # 3️⃣ Проверка наличия снаряжения
    cursor.execute(f"""
        SELECT {col} FROM users WHERE user_id = ?
    """, (user_id,))
    row = cursor.fetchone()

    if row is None:
        conn.close()
        await update.message.reply_text(
            "❌ Ошибка профиля игрока (нет записи в users)."
        )
        return

    have = row[0] or 0

    if have < qty:
        conn.close()
        await update.message.reply_text(
            f"❌ Недостаточно снаряжения.\nУ вас: {have}, нужно: {qty}"
        )
        return

    # 4️⃣ Списание снаряжения
    cursor.execute(
        f"UPDATE users SET {col} = {col} - ? WHERE user_id = ?",
        (qty, user_id)
    )

    cursor.execute("""
        INSERT INTO zombie_equipment (user_id, zombie_id, equip_type, equipment_id, quantity)
        VALUES (?, ?, ?, ?, ?)
        ON CONFLICT(user_id, zombie_id, equip_type, equipment_id)
        DO UPDATE SET quantity = quantity + excluded.quantity
    """, (user_id, zombie_id, equip_type, equip_id, qty))

    conn.commit()
    conn.close()

    await update.message.reply_text(
        f"✅ Снаряжение установлено!\n\n"
        f"🧟 Зомби ID: {zombie_id}\n"
        f"🎒 {name} ×{qty}"
    )

import random
import time
import sqlite3
from telegram import Update
from telegram.ext import ContextTypes

SNOWBALL_COOLDOWN = 2  # 5 минут
DB_PATH = "diamonds_bot.db"

async def throw_snowball_command(update: Update, context: ContextTypes.DEFAULT_TYPE, a):
    try:
        user = update.effective_user
        user_id = user.id

        # ---------- аргументы ----------
        # a = ["снежок", "<player_id>"]
        if not a or len(a) < 2 or not a[1].isdigit():
            await update.message.reply_text(
                "❌ Использование:\nтгз бросить снежок <ID игрока>"
            )
            return

        target_player_id = int(a[1])

        # ---------- атакующий ----------
        user_data = get_user(user_id)
        if not user_data:
            await update.message.reply_text("❌ Вы не зарегистрированы.")
            return

        attacker_player_id = (
            user_data["player_id"] if isinstance(user_data, dict) else user_data[1]
        )

        if attacker_player_id == target_player_id:
            await update.message.reply_text("❌ Нельзя бросить снежок в себя.")
            return

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        now = int(time.time())

        # ---------- кулдаун ----------
        cursor.execute(
            "SELECT snowball_cooldown FROM users WHERE user_id = ?",
            (user_id,)
        )
        row = cursor.fetchone()

        if row and row[0] and row[0] > now:
            remaining = row[0] - now
            await update.message.reply_text(
                f"⏳ Снежок ещё не готов!\n"
                f"Попробуй через {remaining // 60} мин {remaining % 60} сек."
            )
            conn.close()
            return

        cursor.execute(
            "SELECT snowballs FROM users WHERE user_id = ?",
            (user_id,)
        )
        row = cursor.fetchone()

        snowballs = row[0] if row and row[0] is not None else 0

        if snowballs < 1:
            await update.message.reply_text("❄️ У вас нет снежков!")
            conn.close()
            return

        # ---------- цель ----------
        cursor.execute(
            "SELECT user_id, diamonds, points FROM users WHERE player_id = ?",
            (target_player_id,)
        )
        target = cursor.fetchone()

        if not target:
            conn.close()
            await update.message.reply_text("❌ Игрок с таким ID не найден.")
            return

        target_user_id, target_diamonds, target_points = target

        # ---------- шанс ----------
        roll = random.randint(1, 100)

        if roll <= 10:
            diamonds = 0
            points = 0
            desc = "❄️ Промах… снежок ушёл в молоко 😬"
        elif roll <= 70:
            diamonds = 50
            points = 0
            desc = "❄️💥 Неплохое попадание! Снежок прилетел точно в спину."
        elif roll <= 95:
            diamonds = 150
            points = 2
            desc = "❄️🔥 Отличный бросок! Чётко по цели."
        else:
            diamonds = 750
            points = 10
            desc = "❄️👑 ЛЕГЕНДАРНО! Идеальный снежок. Цель уничтожена морально."

        # ---------- защита от минуса ----------
        diamonds = min(diamonds, target_diamonds)
        points = min(points, target_points)

        # ---------- обновление БД ----------
        cursor.execute(
            "UPDATE users SET diamonds = diamonds - ?, points = points - ? WHERE user_id = ?",
            (diamonds, points, target_user_id)
        )

        cursor.execute(
            """
            UPDATE users
            SET diamonds = diamonds + ?,
                points = points + ?,
                snowballs = snowballs - 1,
                snowball_cooldown = ?
            WHERE user_id = ?
            """,
            (diamonds, points, now + SNOWBALL_COOLDOWN, user_id)
        )

        conn.commit()
        conn.close()

        # ---------- сообщение атакующему ----------
        await update.message.reply_text(
            f"{desc}\n\n"
            f"🎯 Цель: игрок ID {target_player_id}\n"
            f"💎 Алмазы: +{diamonds}\n"
            f"⭐ Очки: +{points}"
            f"❄️ Снежков осталось: {snowballs - 1}"
        )

        # ---------- сообщение цели ----------
        await context.bot.send_message(
            chat_id=target_user_id,
            text=(
                "❄️ <b>В вас прилетел снежок!</b>\n"
                f"🎯 Нападавший ID: {attacker_player_id}\n"
                f"💎 Потеряно алмазов: {diamonds}\n"
                f"⭐ Потеряно очков: {points}"
            ),
            parse_mode="HTML"
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка: {e}")
        print("ERROR in throw_snowball_command:", e)

async def snowmaker_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user_id = update.effective_user.id
        now = int(time.time())

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        cursor.execute("""
            SELECT snowballs, COALESCE(last_snowmaker_update, 0)
            FROM users WHERE user_id=?
        """, (user_id,))
        data = cursor.fetchone()

        if not data:
            await update.message.reply_text("❌ Вы не зарегистрированы!")
            conn.close()
            return

        snowballs, last_update = data

        # первый запуск
        if last_update == 0:
            cursor.execute(
                "UPDATE users SET last_snowmaker_update=? WHERE user_id=?",
                (now, user_id)
            )
            conn.commit()
            conn.close()
            await update.message.reply_text("❄️ Снежкодел запущен! Производство началось.")
            return

        seconds = now - last_update
        if seconds < 60:
            await update.message.reply_text("⌛ Снежкодел ещё не произвёл снежки.")
            conn.close()
            return

        hours = seconds / 3600
        produced = int(hours * 2)

        if produced <= 0:
            await update.message.reply_text("⌛ Пока ничего не произведено.")
            conn.close()
            return

        cursor.execute("""
            UPDATE users
            SET snowballs = snowballs + ?, last_snowmaker_update=?
            WHERE user_id=?
        """, (produced, now, user_id))

        conn.commit()
        conn.close()

        await update.message.reply_text(
            f"❄️ <b>Снежкодел отработал!</b>\n"
            f"➕ Получено снежков: {produced}\n"
            f"🎯 Всего снежков: {snowballs + produced}",
            parse_mode="HTML"
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка снежкодела: {e}")
        print(f"ERROR snowmaker_command: {e}")

async def mines_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    user_id = user.id

    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    cursor.execute("SELECT wood, stone, iron, titanium, diamonds FROM users WHERE user_id = ?", (user_id,))
    row = cursor.fetchone()
    wood, stone, iron, titanium, diamonds = row

    text = (
        "💣 <b>Мины для защиты фермы</b>\n"
        "Мины являются одноразовыми + срабатывают при нехватке урона турелей\n\n"
        "1) 🪨 <b>Капкан</b>\n"
        "├ 💰 Стоимость: 3300 камня\n"
        "└ ❤️ Урон: +5\n\n"
        "2) ⛓️ <b>Мина</b>\n"
        "├ 💰 Стоимость: 3300 железа\n"
        "└ ❤️ Урон: +18\n\n"
        "3) 🔩 <b>Морская мина</b>\n"
        "├ 💰 Стоимость: 3300 титана\n"
        "└ ❤️ Урон: +75\n\n"
        "🪄 Для установки мины введите:\n"
        "<code>тгз установить мину (номер)</code>\n\n"
        "💎 <b>Ваш баланс ресурсов:</b>\n"
        f"🪵 Дерево: {wood}\n"
        f"🪨 Камень: {stone}\n"
        f"⛓ Железо: {iron}\n"
        f"🔩 Титан: {titanium}\n"
        f"💎 Алмазы: {diamonds}\n"
    )

    await update.message.reply_text(text, parse_mode="HTML")
    conn.close()

async def set_mine_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    user_id = user.id

    args = update.message.text.strip().split()
    if len(args) < 5:  # теперь нужен номер мины + количество
        await update.message.reply_text("❌ Используйте: тгз установить мину (номер мины) (количество)")
        return

    try:
        mine_id = int(args[-2])  # номер мины
        amount = int(args[-1])   # количество мин
        if amount <= 0:
            raise ValueError
    except:
        await update.message.reply_text("❌ Номер мины и количество должны быть числами.")
        return

    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    cursor.execute("SELECT stone, iron, titanium FROM users WHERE user_id = ?", (user_id,))
    row = cursor.fetchone()
    stone, iron, titanium = row

    # 1) Капкан
    if mine_id == 1:
        cost = 3300 * amount
        if stone < cost:
            await update.message.reply_text("❌ Недостаточно камня.")
            conn.close()
            return
        cursor.execute(
            "UPDATE users SET stone = stone - ?, mine_trap = mine_trap + ? WHERE user_id = ?",
            (cost, amount, user_id)
        )
        text = f"🪨 Вы установили <b>{amount}× Капкан</b> (+5 урона каждая)"

    # 2) Мина
    elif mine_id == 2:
        cost = 3300 * amount
        if iron < cost:
            await update.message.reply_text("❌ Недостаточно железа.")
            conn.close()
            return
        cursor.execute(
            "UPDATE users SET iron = iron - ?, mine_iron = mine_iron + ? WHERE user_id = ?",
            (cost, amount, user_id)
        )
        text = f"⛓ Вы установили <b>{amount}× Мину</b> (+18 урона каждая)"

    # 3) Морская мина
    elif mine_id == 3:
        cost = 3300 * amount
        if titanium < cost:
            await update.message.reply_text("❌ Недостаточно титана.")
            conn.close()
            return
        cursor.execute(
            "UPDATE users SET titanium = titanium - ?, mine_sea = mine_sea + ? WHERE user_id = ?",
            (cost, amount, user_id)
        )
        text = f"🔩 Вы установили <b>{amount}× Морскую мину</b> (+75 урона каждая)"

    else:
        await update.message.reply_text("❌ Такой мины нет.")
        conn.close()
        return

    conn.commit()
    conn.close()
    await update.message.reply_text(text, parse_mode="HTML")

async def clan_bank_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user_id = update.effective_user.id
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # Получаем клан игрока
        cursor.execute("SELECT clan_id FROM users WHERE user_id = ?", (user_id,))
        row = cursor.fetchone()
        if not row or not row[0]:
            conn.close()
            await update.message.reply_text("❌ Вы не состоите в клане.")
            return
        clan_id = row[0]

        # Получаем банк из таблицы clans
        cursor.execute("SELECT wood, stone, iron, titanium FROM clans WHERE id = ?", (clan_id,))
        bank = cursor.fetchone()
        if not bank:
            bank = (0, 0, 0, 0)

        wood, stone, iron, titanium = bank
        conn.close()

        text = (
            "🏦 <b>Банк клана</b>:\n"
            f"🌲 Дерева: {wood}\n"
            f"🪨 Камня: {stone}\n"
            f"⛓️ Железа: {iron}\n"
            f"⚙️ Титана: {titanium}"
        )

        await update.message.reply_text(text, parse_mode="HTML")

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при показе банка: {str(e)}")
        print(f"ERROR in clan_bank_command: {str(e)}")

async def deposit_to_clan_bank(update: Update, context: ContextTypes.DEFAULT_TYPE, args=None):
    try:
        user_id = update.effective_user.id
        if not args or len(args) < 5:
            await update.message.reply_text("❌ Укажите команду в формате: тгз положить в банк <ресурс> <число>")
            return

        resource = args[3].lower()
        amount = int(args[4])

        if resource not in ["дерево", "камень", "железо", "титан"]:
            await update.message.reply_text("❌ Неверный ресурс. Доступные: дерево, камень, железо, титан.")
            return
        if amount <= 0:
            await update.message.reply_text("❌ Количество должно быть положительным числом.")
            return

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        cursor.execute("SELECT clan_id FROM users WHERE user_id = ?", (user_id,))
        row = cursor.fetchone()
        if not row or not row[0]:
            conn.close()
            await update.message.reply_text("❌ Вы не состоите в клане.")
            return

        clan_id = row[0]

        column_map = {"дерево": "wood", "камень": "stone", "железо": "iron", "титан": "titanium"}
        column = column_map[resource]

        # Проверяем, хватает ли у игрока
        cursor.execute(f"SELECT {column} FROM users WHERE user_id = ?", (user_id,))
        user_amount = cursor.fetchone()[0]
        if user_amount < amount:
            conn.close()
            await update.message.reply_text("❌ У вас недостаточно ресурсов.")
            return

        # Списываем у игрока и добавляем в банк клана
        cursor.execute(f"UPDATE users SET {column} = {column} - ? WHERE user_id = ?", (amount, user_id))
        cursor.execute(f"UPDATE clans SET {column} = {column} + ? WHERE id = ?", (amount, clan_id))

        conn.commit()
        conn.close()

        await update.message.reply_text(f"✅ Вы положили {amount} {resource} в банк клана.")

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при внесении: {str(e)}")
        print(f"ERROR in deposit_to_clan_bank: {str(e)}")

async def withdraw_from_clan_bank(update: Update, context: ContextTypes.DEFAULT_TYPE, args=None):
    try:
        user_id = update.effective_user.id
        if not args or len(args) < 5:
            await update.message.reply_text("❌ Укажите команду в формате: тгз вывести из банка <ресурс> <число>")
            return

        resource = args[3].lower()
        amount = int(args[4])

        if resource not in ["дерево", "камень", "железо", "титан"]:
            await update.message.reply_text("❌ Неверный ресурс. Доступные: дерево, камень, железо, титан.")
            return
        if amount <= 0:
            await update.message.reply_text("❌ Количество должно быть положительным числом.")
            return

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        cursor.execute("SELECT clan_id FROM users WHERE user_id = ?", (user_id,))
        row = cursor.fetchone()
        if not row or not row[0]:
            conn.close()
            await update.message.reply_text("❌ Вы не состоите в клане.")
            return
        clan_id = row[0]

        column_map = {"дерево": "wood", "камень": "stone", "железо": "iron", "титан": "titanium"}
        column = column_map[resource]

        cursor.execute(f"SELECT {column} FROM clans WHERE id = ?", (clan_id,))
        bank_amount = cursor.fetchone()[0]
        if bank_amount < amount:
            conn.close()
            await update.message.reply_text("❌ В банке недостаточно ресурсов.")
            return

        # Снимаем из банка и добавляем игроку
        cursor.execute(f"UPDATE clans SET {column} = {column} - ? WHERE id = ?", (amount, clan_id))
        cursor.execute(f"UPDATE users SET {column} = {column} + ? WHERE user_id = ?", (amount, user_id))

        conn.commit()
        conn.close()

        await update.message.reply_text(f"✅ Вы вывели {amount} {resource} из банка клана.")

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при выводе: {str(e)}")
        print(f"ERROR in withdraw_from_clan_bank: {str(e)}")

FACTORY_ALIASES = {
    "лесопилка": "лесопилка", "лесопилки": "лесопилка",
    "каменоломня": "каменоломня", "каменоломни": "каменоломня",
    "плавильня": "плавильня", "плавильни": "плавильня",
    "фабрика": "фабрика", "фабрики": "фабрика",
    "карьер": "карьер", "карьеры": "карьер"
}

# --- Включить завод ---
async def enable_factory_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args=None):
    try:
        if not args:
            await update.message.reply_text("❌ Укажите завод: тгз включить лесопилки")
            return

        raw = args[0].lower()
        factory_name = FACTORY_ALIASES.get(raw)
        if not factory_name:
            await update.message.reply_text("❌ Неизвестный завод!")
            return

        col_count = FACTORIES_INFO[factory_name]["column"]
        col_enabled = f"{col_count}_enabled"
        user_id = update.effective_user.id

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        cursor.execute(f"SELECT {col_enabled} FROM users WHERE user_id = ?", (user_id,))
        row = cursor.fetchone()
        if not row:
            conn.close()
            await update.message.reply_text("❌ Вы не зарегистрированы!")
            return

        if row[0] == 1:
            conn.close()
            await update.message.reply_text(f"❌ {FACTORY_DISPLAY[factory_name]} уже включена!")
            return

        cursor.execute(f"UPDATE users SET {col_enabled} = 1 WHERE user_id = ?", (user_id,))
        conn.commit()
        conn.close()

        await update.message.reply_text(f"✅ {FACTORY_DISPLAY[factory_name]} включена! Производство активировано.")
    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при включении завода: {e}")
        print(f"ERROR in enable_factory_command: {e}")

# --- Отключить завод ---
async def disable_factory_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args=None):
    try:
        if not args:
            await update.message.reply_text("❌ Укажите завод: тгз отключить лесопилки")
            return

        raw = args[0].lower()
        factory_name = FACTORY_ALIASES.get(raw)
        if not factory_name:
            await update.message.reply_text("❌ Неизвестный завод!")
            return

        col_count = FACTORIES_INFO[factory_name]["column"]
        col_enabled = f"{col_count}_enabled"
        user_id = update.effective_user.id

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        cursor.execute(f"SELECT {col_enabled} FROM users WHERE user_id = ?", (user_id,))
        row = cursor.fetchone()
        if not row:
            conn.close()
            await update.message.reply_text("❌ Вы не зарегистрированы!")
            return

        if row[0] == 0:
            conn.close()
            await update.message.reply_text(f"❌ {FACTORY_DISPLAY[factory_name]} уже отключена!")
            return

        cursor.execute(f"UPDATE users SET {col_enabled} = 0 WHERE user_id = ?", (user_id,))
        conn.commit()
        conn.close()

        await update.message.reply_text(f"✅ {FACTORY_DISPLAY[factory_name]} отключена! Производство остановлено.")
    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при отключении завода: {e}")
        print(f"ERROR in disable_factory_command: {e}")

# 🔒 Закрыть клан (только владелец)
async def close_clan_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args=None):
    try:
        user_id = update.effective_user.id
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # Проверяем, является ли пользователь владельцем клана
        cursor.execute("SELECT id FROM clans WHERE owner_id = ?", (user_id,))
        clan = cursor.fetchone()

        if not clan:
            conn.close()
            await update.message.reply_text("❌ Вы не владелец клана.")
            return

        clan_id = clan[0]
        cursor.execute("UPDATE clans SET is_closed = 1 WHERE id = ?", (clan_id,))
        conn.commit()
        conn.close()

        await update.message.reply_text("🔒 Клан успешно закрыт! Теперь в него нельзя вступить.")
    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при закрытии клана: {str(e)}")
        print(f"ERROR in close_clan_command: {str(e)}")


# 🔓 Открыть клан (только владелец)
async def open_clan_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args=None):
    try:
        user_id = update.effective_user.id
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # Проверяем, является ли пользователь владельцем клана
        cursor.execute("SELECT id FROM clans WHERE owner_id = ?", (user_id,))
        clan = cursor.fetchone()

        if not clan:
            conn.close()
            await update.message.reply_text("❌ Вы не владелец клана.")
            return

        clan_id = clan[0]
        cursor.execute("UPDATE clans SET is_closed = 0 WHERE id = ?", (clan_id,))
        conn.commit()
        conn.close()

        await update.message.reply_text("✅ Клан снова открыт! Теперь в него можно вступать.")
    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при открытии клана: {str(e)}")
        print(f"ERROR in open_clan_command: {str(e)}")

def get_farm_info(level):
    farms = {
        0: {"cost": 0, "income_min": 20, "income_max": 24},
        1: {"cost": 449, "income_min": 24, "income_max": 28},
        2: {"cost": 898, "income_min": 28, "income_max": 32},
        3: {"cost": 1347, "income_min": 32, "income_max": 36},
        4: {"cost": 1797, "income_min": 36, "income_max": 42},
        5: {"cost": 2246, "income_min": 42, "income_max": 48},
        6: {"cost": 2695, "income_min": 48, "income_max": 54},
        7: {"cost": 3144, "income_min": 54, "income_max": 62},
        8: {"cost": 3594, "income_min": 62, "income_max": 70},
        9: {"cost": 4043, "income_min": 70, "income_max": 78},
        10: {"cost": 4492, "income_min": 78, "income_max": 88},
        11: {"cost": 4941, "income_min": 88, "income_max": 98},
        12: {"cost": 5391, "income_min": 98, "income_max": 108},
        13: {"cost": 5840, "income_min": 108, "income_max": 120},
        14: {"cost": 6289, "income_min": 120, "income_max": 132},
        15: {"cost": 6738, "income_min": 132, "income_max": 144},
        16: {"cost": 7188, "income_min": 144, "income_max": 148},
        17: {"cost": 7637, "income_min": 148, "income_max": 172},
        18: {"cost": 8086, "income_min": 172, "income_max": 186},
        19: {"cost": 8535, "income_min": 186, "income_max": 202},
        20: {"cost": 8985, "income_min": 202, "income_max": 218},
    }

    return farms.get(level, farms[0])

# ✅ Просмотр улучшений клана
async def clan_upgrades_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    # Проверяем, состоит ли игрок в клане
    cursor.execute("SELECT clan_id FROM users WHERE user_id = ?", (user_id,))
    row = cursor.fetchone()
    if not row or not row[0]:
        conn.close()
        await update.message.reply_text("❌ Вы не состоите в клане.")
        return

    clan_id = row[0]

    # Получаем уровень улучшения клана
    cursor.execute("SELECT defense_upgrade_level FROM clans WHERE id = ?", (clan_id,))
    level = cursor.fetchone()[0]

    upgrades = [
        ("🪵 Дерево", 200_000, 5),
        ("🪵 Дерево", 400_000, 10),
        ("🪵 Дерево", 800_000, 15),
        ("🪨 Камень", 1_200_000, 20),
        ("🪨 Камень", 2_000_000, 25),
        ("🪨 Камень", 3_000_000, 30),
        ("⛓ Железо", 1_500_000, 35),
        ("⛓ Железо", 2_000_000, 40),
        ("⚙️ Титан", 500_000, 50),
        ("⚙️ Титан", 1_000_000, 60),
        ("⚙️ Титан", 1_500_000, 70),
        ("⚙️ Титан", 2_000_000, 80)
    ]

    text = "🏰 <b>Улучшения клана</b>\n\n"
    if level >= len(upgrades):
        text += "🔝 Клан имеет максимальный уровень улучшений!\n"
    else:
        res, cost, bonus = upgrades[level]
        text += (
            f"📈 Текущий уровень: {level}\n"
            f"💸 Текущая скидка: {bonus if level > 0 else 0}%\n"
            f"🔜 Следующее улучшение:\n"
            f" ├ 💰 Стоимость: {cost:,} {res}\n"
            f" └ 💎 Скидка на баррикады: {bonus}%\n\n"
            f"Для покупки введите: <b>тгз улучшить клан</b>"
        )

    conn.close()
    await update.message.reply_text(text, parse_mode="HTML")


# ✅ Покупка улучшения клана
async def upgrade_clan_defense_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    # Проверяем, состоит ли игрок в клане
    cursor.execute("SELECT clan_id FROM users WHERE user_id = ?", (user_id,))
    row = cursor.fetchone()
    if not row or not row[0]:
        conn.close()
        await update.message.reply_text("❌ Вы не состоите в клане.")
        return

    clan_id = row[0]

    # Проверяем владельца и уровень
    cursor.execute("SELECT owner_id, defense_upgrade_level FROM clans WHERE id = ?", (clan_id,))
    data = cursor.fetchone()
    if not data:
        conn.close()
        await update.message.reply_text("❌ Клан не найден.")
        return

    cursor.execute("SELECT defense_upgrade_level FROM clans WHERE id = ?", (clan_id,))
    row = cursor.fetchone()
    level = row[0] if row else 0  # если пусто, считаем уровень 0

    upgrades = [
        ("wood", 200_000, "🪵 Дерево", 5),
        ("wood", 400_000, "🪵 Дерево", 10),
        ("wood", 800_000, "🪵 Дерево", 15),
        ("stone", 1_200_000, "🪨 Камень", 20),
        ("stone", 2_000_000, "🪨 Камень", 25),
        ("stone", 3_000_000, "🪨 Камень", 30),
        ("iron", 1_500_000, "⛓ Железо", 35),
        ("iron", 2_000_000, "⛓ Железо", 40),
        ("titanium", 500_000, "⚙️ Титан", 50),
        ("titanium", 1_000_000, "⚙️ Титан", 60),
        ("titanium", 1_500_000, "⚙️ Титан", 70),
        ("titanium", 2_000_000, "⚙️ Титан", 80)
    ]

    if level >= len(upgrades):
        conn.close()
        await update.message.reply_text("🔝 Клан уже полностью улучшен!")
        return

    res_key, cost, res_emoji, bonus = upgrades[level]

    # Проверяем ресурс у владельца
    cursor.execute(f"SELECT {res_key} FROM users WHERE user_id = ?", (user_id,))
    current_amount = cursor.fetchone()[0]
    if current_amount < cost:
        conn.close()
        await update.message.reply_text(f"❌ Недостаточно ресурса. Нужно {cost:,} {res_emoji}, у вас {current_amount:,}.")
        return

    # Списываем и обновляем уровень
    cursor.execute(f"UPDATE users SET {res_key} = {res_key} - ? WHERE user_id = ?", (cost, user_id))
    cursor.execute("UPDATE clans SET defense_upgrade_level = defense_upgrade_level + 1 WHERE id = ?", (clan_id,))

    conn.commit()
    conn.close()

    await update.message.reply_text(
        f"🏰 Клан улучшен до уровня {level + 1}!\n"
        f"💸 Потрачено: {cost:,} {res_emoji}\n"
        f"📉 Новая скидка на баррикады: {bonus}% для всех членов клана."
    )

def get_rank(points):
    ranks = [
        ("Новичок 🟢", 0, 99, 100),
        ("Ученик 🔵", 100, 299, 300),
        ("Паломник 🎒", 300, 499, 500),
        ("Адепт 📖", 500, 799, 800),
        ("Опытный 🟣", 800, 1199, 1200),
        ("Странник 🧭", 1200, 1699, 1700),
        ("Ловкач 🎯", 1700, 1799, 1800),
        ("Счастливчик 🍀", 1800, 2399, 2400),
        ("Кулинар 👨‍🍳", 2400, 2999, 3000),
        ("Первооткрыватель 🗺️", 3000, 3799, 3800),
        ("Археолог 🏺", 3800, 3899, 3900),
        ("Эксперт 💡", 3900, 4699, 4700),
        ("Везунчик 🎲", 4700, 5699, 5700),
        ("Зверолов 🐾", 5700, 5799, 5800),
        ("Алхимик 🧪", 5800, 6799, 6800),
        ("Эксперт-оружейник 🛠️", 6800, 7999, 8000),
        ("Инженер ⚙️", 8000, 8099, 8100),
        ("Коллекционер 📦", 8100, 9299, 9300),
        ("Виртуоз 🎵", 9300, 10699, 10700),
        ("Ассасин 🔪", 10700, 12199, 12200),
        ("Кладоискатель 💰", 12200, 13799, 13800),
        ("Мастер 🟠", 13800, 15499, 15500),
        ("Покровитель 🤝", 15500, 17299, 17300),
        ("Меценат 💰", 17300, 17399, 17400),
        ("Стратег 🧠", 17400, 19199, 19200),
        ("Тактик 🎯", 19200, 19299, 19300),
        ("Лидер 👑", 19300, 21199, 21200),
        ("Дипломат 🕊️", 21200, 21299, 21300),
        ("Чемпион 🏆", 21300, 22299, 22300),
        ("Хранитель 🛡️", 22300, 22399, 22400),
        ("Гладиатор ⚔️", 22400, 23599, 23600),
        ("Магнат 🏦", 23600, 24799, 24800),
        ("Завоеватель 🚩", 24800, 26099, 26100),
        ("Истребитель 👹", 26100, 27499, 27500),
        ("Легенда 🔴", 27500, 28999, 29000),
        ("Безумец 🤪", 29000, 30599, 30600),
        ("Мудрец 📜", 30600, 32299, 32300),
        ("Летописец 📜", 32300, 32399, 32400),
        ("Мистик 🔮", 32400, 32499, 32500),
        ("Сновидец 😴", 32500, 35499, 35500),  # Исправлено с 33000 на 32500 для непрерывности
        ("Гуру 🙏", 35500, 37999, 38000),
        ("Архимаг 🧙", 38000, 40999, 41000),
        ("Созидатель 🏗️", 41000, 41099, 41100),
        ("Превосходящий ✨", 41100, 42399, 42400),
        ("Бог 💎", 42400, float('inf'), float('inf'))
    ]

    for rank_number, (rank_name, min_points, max_points, next_points) in enumerate(ranks, start=1):
        if min_points <= points <= max_points:
            return rank_name, next_points, rank_number
    return "Бог 💎", float('inf'), len(ranks)

async def ranks_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        ranks_text = """🏆 Система рангов:

• Новичок 🟢: 0 - 99 очков
• Ученик 🔵: 100 - 299 очков
• Паломник 🎒: 300 - 499 очков
• Адепт 📖: 500 - 799 очков
• Опытный 🟣: 800 - 1199 очков
• Странник 🧭: 1200 - 1699 очков
• Ловкач 🎯: 1700 - 1799 очков
• Счастливчик 🍀: 1800 - 2399 очков
• Кулинар 👨‍🍳: 2400 - 2999 очков
• Первооткрыватель 🗺️: 3000 - 3799 очков
• Археолог 🏺: 3800 - 3899 очков
• Эксперт 💡: 3900 - 4699 очков
• Везунчик 🎲: 4700 - 5699 очков
• Зверолов 🐾: 5700 - 5799 очков
• Алхимик 🧪: 5800 - 6799 очков
• Эксперт-оружейник 🛠️: 6800 - 7999 очков
• Инженер ⚙️: 8000 - 8099 очков
• Коллекционер 📦: 8100 - 9299 очков
• Виртуоз 🎵: 9300 - 10699 очков
• Ассасин 🔪: 10700 - 12199 очков
• Кладоискатель 💰: 12200 - 13799 очков
• Мастер 🟠: 13800 - 15499 очков
• Покровитель 🤝: 15500 - 17299 очков
• Меценат 💰: 17300 - 17399 очков
• Стратег 🧠: 17400 - 19199 очков
• Тактик 🎯: 19200 - 19299 очков
• Лидер 👑: 19300 - 21199 очков
• Дипломат 🕊️: 21200 - 21299 очков
• Чемпион 🏆: 21300 - 22299 очков
• Хранитель 🛡️: 22300 - 22399 очков
• Гладиатор ⚔️: 22400 - 23599 очков
• Магнат 🏦: 23600 - 24799 очков
• Завоеватель 🚩: 24800 - 26099 очков
• Истребитель 👹: 26100 - 27499 очков
• Легенда 🔴: 27500 - 28999 очков
• Безумец 🤪: 29000 - 30599 очков
• Мудрец 📜: 30600 - 32299 очков
• Летописец 📜: 32300 - 32399 очков
• Мистик 🔮: 32400 - 32499 очков
• Сновидец 😴: 33000 - 35499 очков
• Гуру 🙏: 35500 - 37999 очков
• Архимаг 🧙: 38000 - 40999 очков
• Созидатель 🏗️: 41000 - 41099 очков
• Превосходящий ✨: 41100 - 42399 очков
• Бог 💎: 42400+ очков"""

        await update.message.reply_text(ranks_text)

    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка при показе рангов: {str(e)}")

def get_rank_damage_bonus(rank_number: int) -> int:
    bonus = 0
    for i in range(1, rank_number + 1):
        tier = (i - 1) // 5 + 1   # каждые 5 рангов
        bonus += min(tier, 9)    # максимум 9% за ранг
    return bonus

# ✅ Прокачка клана (увеличение лимита участников)
async def upgrade_clan_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args=None):
    try:
        user_id = update.effective_user.id

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # Проверяем, состоит ли пользователь в клане
        cursor.execute("SELECT clan_id FROM users WHERE user_id = ?", (user_id,))
        row = cursor.fetchone()
        if not row or not row[0]:
            conn.close()
            await update.message.reply_text("❌ Вы не состоите в клане.")
            return

        clan_id = row[0]

        # Проверяем, является ли пользователь владельцем клана
        cursor.execute("SELECT owner_id, upgrade_level FROM clans WHERE id = ?", (clan_id,))
        row = cursor.fetchone()
        if not row:
            conn.close()
            await update.message.reply_text("❌ Клан не найден.")
            return

        owner_id, upgrade_level = row
        if owner_id != user_id:
            conn.close()
            await update.message.reply_text("❌ Только владелец клана может прокачивать клан.")
            return

        # Определяем стоимость прокачки
        upgrade_level = upgrade_level or 0
        upgrade_costs = [
            ("wood", 500_000, "🪵 500 000 дерева"),
            ("stone", 500_000, "⛏ 500 000 камня"),
            ("iron", 500_000, "⛓ 500 000 железа"),
            ("titanium", 500_000, "⚙ 500 000 титана")
        ]

        if upgrade_level >= len(upgrade_costs):
            conn.close()
            await update.message.reply_text("🔝 Клан уже полностью прокачан!")
            return

        resource, cost, cost_text = upgrade_costs[upgrade_level]

        # Проверяем ресурсы владельца
        cursor.execute(f"SELECT {resource} FROM users WHERE user_id = ?", (user_id,))
        amount = cursor.fetchone()[0]
        if amount < cost:
            conn.close()
            await update.message.reply_text(f"❌ Недостаточно ресурса. Нужно {cost_text}, у вас {amount}.")
            return

        # Списываем ресурсы и увеличиваем лимит
        cursor.execute(f"UPDATE users SET {resource} = {resource} - ? WHERE user_id = ?", (cost, user_id))
        cursor.execute("UPDATE clans SET max_members = max_members + 2, upgrade_level = upgrade_level + 1 WHERE id = ?", (clan_id,))

        conn.commit()
        conn.close()

        await update.message.reply_text(
            f"🏰 Клан улучшен до уровня {upgrade_level + 1}!\n"
            f"👥 Лимит участников увеличен на 2.\n"
            f"💸 Потрачено: {cost_text}"
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при прокачке клана: {str(e)}")
        print(f"ERROR in upgrade_clan_command: {str(e)}")

from telegram import Update
from telegram.ext import ContextTypes
import sqlite3

# ====== Команда администратора: список пользователей ======
async def admin_list_users(update: Update, context: ContextTypes.DEFAULT_TYPE):
    ADMIN_ID = 7689411893  # <-- Укажи свой Telegram ID

    # Проверка прав администратора
    if update.effective_user.id != ADMIN_ID:
        await update.message.reply_text("❌ У вас нет прав администратора!")
        return

    try:
        conn = sqlite3.connect('diamonds_bot.db')  # <-- используй свою БД
        cursor = conn.cursor()
        cursor.execute('SELECT player_id, username, first_name, diamonds, points FROM users ORDER BY player_id')
        users = cursor.fetchall()
        conn.close()
    except Exception as e:
        await update.message.reply_text(f"⚠️ Ошибка при работе с БД: {e}")
        return

    if not users:
        await update.message.reply_text("❌ Нет зарегистрированных пользователей!")
        return

    users_text = "👥 Список пользователей:\n\n"
    for i, user in enumerate(users, 1):
        player_id, username, first_name, diamonds, points = user
        username_display = f"@{username}" if username else "нет"
        users_text += (
            f"{i}. 🆔 {player_id} | 👤 {first_name}\n"
            f"   💎 {diamonds} | ⭐ {points} | {username_display}\n\n"
        )

    # Если сообщение слишком длинное — разбиваем на части
    for part in [users_text[i:i + 4000] for i in range(0, len(users_text), 4000)]:
        await update.message.reply_text(part)

async def admin_change_id(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != 7689411893:
        await update.message.reply_text("❌ У вас нет прав администратора!")
        return
    args = update.message.text.split()
    if len(args) != 3:
        await update.message.reply_text("❌ Использование: /change_id [текущий_id] [новый_id]")
        return
    try:
        current_id = int(args[1])
        new_id = int(args[2])
        conn = sqlite3.connect('database.db')
        cursor = conn.cursor()
        cursor.execute('SELECT user_id FROM users WHERE player_id = ?', (current_id,))
        result = cursor.fetchone()
        if not result:
            await update.message.reply_text(f"❌ Игрок с Player ID {current_id} не найден!")
            conn.close()
            return
        user_id = result[0]
        cursor.execute('UPDATE users SET player_id = ? WHERE user_id = ?', (new_id, user_id))
        conn.commit()
        conn.close()
        await update.message.reply_text(f"✅ Player ID изменён с {current_id} на {new_id}")
    except ValueError:
        await update.message.reply_text("❌ IDs должны быть числами!")
    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка: {str(e)}")
        print(f"ERROR in admin_change_id: {str(e)}")

async def admin_add_diamonds(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != 7689411893:
        await update.message.reply_text("❌ У вас нет прав администратора!")
        return
    args = update.message.text.split()
    if len(args) != 3:
        await update.message.reply_text("❌ Использование: /add_diamonds [player_id] [количество]")
        return
    try:
        player_id = int(args[1])
        amount = int(args[2])
        if amount < 0:
            await update.message.reply_text("❌ Количество должно быть положительным!")
            return
        conn = sqlite3.connect('database.db')
        cursor = conn.cursor()
        cursor.execute('SELECT user_id FROM users WHERE player_id = ?', (player_id,))
        result = cursor.fetchone()
        if not result:
            await update.message.reply_text(f"❌ Игрок с Player ID {player_id} не найден!")
            conn.close()
            return
        user_id = result[0]
        cursor.execute('UPDATE users SET diamonds = diamonds + ? WHERE user_id = ?', (amount, user_id))
        conn.commit()
        conn.close()
        await update.message.reply_text(f"✅ Добавлено {amount} алмазов игроку с Player ID {player_id}")
    except ValueError:
        await update.message.reply_text("❌ Player ID и количество должны быть числами!")
    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка: {str(e)}")
        print(f"ERROR in admin_add_diamonds: {str(e)}")

async def admin_remove_diamonds(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != 7689411893:
        await update.message.reply_text("❌ У вас нет прав администратора!")
        return
    args = update.message.text.split()
    if len(args) != 3:
        await update.message.reply_text("❌ Использование: /remove_diamonds [player_id] [количество]")
        return
    try:
        player_id = int(args[1])
        amount = int(args[2])
        if amount < 0:
            await update.message.reply_text("❌ Количество должно быть положительным!")
            return
        conn = sqlite3.connect('database.db')
        cursor = conn.cursor()
        cursor.execute('SELECT user_id FROM users WHERE player_id = ?', (player_id,))
        result = cursor.fetchone()
        if not result:
            await update.message.reply_text(f"❌ Игрок с Player ID {player_id} не найден!")
            conn.close()
            return
        user_id = result[0]
        cursor.execute('UPDATE users SET diamonds = MAX(0, diamonds - ?) WHERE user_id = ?', (amount, user_id))
        conn.commit()
        conn.close()
        await update.message.reply_text(f"✅ Удалено {amount} алмазов у игрока с Player ID {player_id}")
    except ValueError:
        await update.message.reply_text("❌ Player ID и количество должны быть числами!")
    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка: {str(e)}")
        print(f"ERROR in admin_remove_diamonds: {str(e)}")

async def admin_add_points(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_USER_ID:
        await update.message.reply_text("❌ У вас нет прав администратора!")
        return
    args = update.message.text.split()
    if len(args) != 3:
        await update.message.reply_text("❌ Использование: /add_points [player_id] [количество]")
        return
    try:
        player_id = int(args[1])
        amount = int(args[2])
        if amount < 0:
            await update.message.reply_text("❌ Количество должно быть положительным!")
            return
        conn = sqlite3.connect('database.db')
        cursor = conn.cursor()
        cursor.execute('SELECT user_id FROM users WHERE player_id = ?', (player_id,))
        result = cursor.fetchone()
        if not result:
            await update.message.reply_text(f"❌ Игрок с Player ID {player_id} не найден!")
            conn.close()
            return
        user_id = result[0]
        cursor.execute('UPDATE users SET points = points + ? WHERE user_id = ?', (amount, user_id))
        conn.commit()
        conn.close()
        await update.message.reply_text(f"✅ Добавлено {amount} очков игроку с Player ID {player_id}")
    except ValueError:
        await update.message.reply_text("❌ Player ID и количество должны быть числами!")
    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка: {str(e)}")
        print(f"ERROR in admin_add_points: {str(e)}")

async def admin_remove_points(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != 7689411893:
        await update.message.reply_text("❌ У вас нет прав администратора!")
        return
    args = update.message.text.split()
    if len(args) != 3:
        await update.message.reply_text("❌ Использование: /remove_points [player_id] [количество]")
        return
    try:
        player_id = int(args[1])
        amount = int(args[2])
        if amount < 0:
            await update.message.reply_text("❌ Количество должно быть положительным!")
            return
        conn = sqlite3.connect('database.db')
        cursor = conn.cursor()
        cursor.execute('SELECT user_id FROM users WHERE player_id = ?', (player_id,))
        result = cursor.fetchone()
        if not result:
            await update.message.reply_text(f"❌ Игрок с Player ID {player_id} не найден!")
            conn.close()
            return
        user_id = result[0]
        cursor.execute('UPDATE users SET points = MAX(0, points - ?) WHERE user_id = ?', (amount, user_id))
        conn.commit()
        conn.close()
        await update.message.reply_text(f"✅ Удалено {amount} очков у игрока с Player ID {player_id}")
    except ValueError:
        await update.message.reply_text("❌ Player ID и количество должны быть числами!")
    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка: {str(e)}")
        print(f"ERROR in admin_remove_points: {str(e)}")

async def admin_user_info(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != 7689411893:
        await update.message.reply_text("❌ У вас нет прав администратора!")
        return
    args = update.message.text.split()
    if len(args) != 2:
        await update.message.reply_text("❌ Использование: /user_info [player_id]")
        return
    try:
        player_id = int(args[1])
        conn = sqlite3.connect('database.db')
        cursor = conn.cursor()
        cursor.execute('SELECT * FROM users WHERE player_id = ?', (player_id,))
        user_data = cursor.fetchone()
        conn.close()
        if not user_data:
            await update.message.reply_text(f"❌ Игрок с Player ID {player_id} не найден!")
            return
        user_id, player_id, username, first_name, diamonds, points, last_diamond_time, upgrade_level, registration_date, strawberries, farm_level, last_zombie_attack, zombie_status, last_strawberry_update, last_bonus_time, tomatoes, coins = user_data
        info_text = (
            f"👤 Информация о пользователе (Player ID: {player_id}):\n"
            f"Имя: {first_name}\nUsername: @{username if username else 'нет'}\n"
            f"💎 Алмазы: {diamonds}\n⭐ Очки: {points}\n"
            f"⚡ Уровень фарма: {upgrade_level}\n🍓 Клубники: {strawberries}\n"
            f"🍅 Помидоры: {tomatoes}\n💰 Монеты: {coins}\n"
            f"🚜 Уровень фермы: {farm_level}\n📅 Регистрация: {registration_date}"
        )
        await update.message.reply_text(info_text)
    except ValueError:
        await update.message.reply_text("❌ Player ID должен быть числом!")
    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка: {str(e)}")
        print(f"ERROR in admin_user_info: {str(e)}")

async def admin_set_upgrade_level(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != 7689411893:
        await update.message.reply_text("❌ У вас нет прав администратора!")
        return
    args = update.message.text.split()
    if len(args) != 3:
        await update.message.reply_text("❌ Использование: /set_upgrade_level [player_id] [уровень]")
        return
    try:
        player_id = int(args[1])
        level = int(args[2])
        if level < 0 or level > 15:
            await update.message.reply_text("❌ Уровень должен быть от 0 до 15!")
            return
        conn = sqlite3.connect('database.db')
        cursor = conn.cursor()
        cursor.execute('SELECT user_id FROM users WHERE player_id = ?', (player_id,))
        result = cursor.fetchone()
        if not result:
            await update.message.reply_text(f"❌ Игрок с Player ID {player_id} не найден!")
            conn.close()
            return
        user_id = result[0]
        cursor.execute('UPDATE users SET upgrade_level = ? WHERE user_id = ?', (level, user_id))
        conn.commit()
        conn.close()
        await update.message.reply_text(f"✅ Уровень фарма для Player ID {player_id} установлен на {level}")
    except ValueError:
        await update.message.reply_text("❌ Player ID и уровень должны быть числами!")
    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка: {str(e)}")
        print(f"ERROR in admin_set_upgrade_level: {str(e)}")

# ✅ Бонус клана (индивидуальная перезарядка)
from datetime import datetime, timezone
import sqlite3, time

CLAN_BONUS_STRAWBERRIES = 10000
CLAN_BONUS_TOMATOES = 300000
DB_PATH = "diamonds_bot.db"

async def clan_bonus_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    # Проверяем, состоит ли пользователь в клане
    cursor.execute("SELECT clan_id, last_clan_bonus, username FROM users WHERE user_id = ?", (user_id,))
    row = cursor.fetchone()
    if not row or not row[0]:
        conn.close()
        await update.message.reply_text("❌ Вы не состоите в клане.")
        return

    clan_id, last_bonus, username = row
    now = datetime.now(timezone.utc)
    cooldown_hours = 24

    # Проверяем кулдаун только у вызвавшего
    if last_bonus:
        try:
            last_bonus_dt = datetime.fromisoformat(last_bonus.replace('Z', '+00:00'))
            if last_bonus_dt.tzinfo is None:
                last_bonus_dt = last_bonus_dt.replace(tzinfo=timezone.utc)
            hours_passed = (now - last_bonus_dt).total_seconds() / 3600
        except Exception:
            hours_passed = float('inf')
    else:
        hours_passed = float('inf')

    if hours_passed < cooldown_hours:
        remaining_time = (cooldown_hours - hours_passed) * 3600
        hours, remainder = divmod(int(remaining_time), 3600)
        minutes = remainder // 60
        conn.close()
        await update.message.reply_text(f"❌ Бонус можно получить через {hours} ч {minutes} мин.")
        return

    # Получаем всех участников клана
    cursor.execute("SELECT user_id FROM users WHERE clan_id = ?", (clan_id,))
    members = cursor.fetchall()
    if not members:
        conn.close()
        await update.message.reply_text("❌ Участников клана нет.")
        return

    # Выдаём бонус всем участникам клана
    for (member_id,) in members:
        cursor.execute("""
            UPDATE users
            SET strawberries = COALESCE(strawberries, 0) + ?,
                tomatoes = COALESCE(tomatoes, 0) + ?
            WHERE user_id = ?
        """, (CLAN_BONUS_STRAWBERRIES, CLAN_BONUS_TOMATOES, member_id))

    # Ставим кулдаун только тому, кто вызвал команду
    cursor.execute("UPDATE users SET last_clan_bonus = ? WHERE user_id = ?", (now.isoformat(), user_id))

    conn.commit()
    conn.close()

    # Отправляем сообщение
    await update.message.reply_text(
        f"🎁 Бонус клана активирован участником <b>{username}</b>!\n\n"
        f"🍓 Всем выдано: {CLAN_BONUS_STRAWBERRIES} клубники\n"
        f"🍅 Всем выдано: {CLAN_BONUS_TOMATOES} помидоров\n"
        f"⏱️ Твой следующий бонус доступен через 24 часа.",
        parse_mode="HTML"
    )

import sqlite3
from telegram import Update
from telegram.ext import ContextTypes

DB_PATH = "diamonds_bot.db"

# Каталог турелей с ценами в 2 раза меньше
TURRETS = {
    1: {
        "name": "🪵 Деревянная турель",
        "price_type": "wood",
        "price": 60000,
        "damage": 30,
        "db_column": "turret_wood"
    },
    2: {
        "name": "🧱 Каменная турель",
        "price_type": "stone",
        "price": 75000,
        "damage": 50,
        "db_column": "turret_stone"
    },
    3: {
        "name": "⛓ Железная турель",
        "price_type": "iron",
        "price": 25000,
        "damage": 70,
        "db_column": "turret_iron"
    },
    4: {
        "name": "🔩 Титановая турель",
        "price_type": "titanium",
        "price": 15000,
        "damage": 150,
        "db_column": "turret_titanium"
    },
    5: {
        "name": "🚀 Ракетная турель",
        "price_type": "diamonds",
        "price": 3750,
        "damage": 1300,
        "db_column": "turret_rocket"
    },
}

async def turrets_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Показывает каталог всех турелей"""
    reply = "🎯 <b>Турели для защиты фермы</b>\n\n"

    for tid, turret in TURRETS.items():
        reply += (
            f"{tid}) {turret['name']}\n"
            f" ├ 💰 Стоимость: {turret['price']} {turret['price_type'].capitalize()}\n"
            f" └ ❤️ Урон: +{turret['damage']}\n\n"
        )

    reply += "🪄 Для установки турели введите:\n<code>тгз поставить турель (номер)</code>"

    # Показываем баланс игрока
    user = update.effective_user
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute(
        "SELECT wood, stone, iron, titanium, diamonds FROM users WHERE user_id = ?",
        (user.id,),
    )
    res = cursor.fetchone() or (0, 0, 0, 0, 0)
    conn.close()

    reply += (
        "\n\n💎 <b>Ваш баланс ресурсов:</b>\n"
        f"🌲 Дерево: {res[0]}\n"
        f"🪨 Камень: {res[1]}\n"
        f"⛓ Железо: {res[2]}\n"
        f"🔩 Титан: {res[3]}\n"
        f"💎 Алмазы: {res[4]}"
    )

    await update.message.reply_text(reply, parse_mode="HTML")


async def buy_turret_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Покупка турели: тгз поставить турель [номер]"""
    user = update.effective_user
    user_data = get_user(user.id)
    if not user_data:
        await update.message.reply_text("❌ Вы не зарегистрированы.")
        return

    parts = update.message.text.strip().lower().split()
    if len(parts) != 4 or parts[:3] != ["тгз", "поставить", "турель"]:
        await update.message.reply_text("❌ Использование: тгз поставить турель [номер]")
        return

    try:
        turret_id = int(parts[3])
    except ValueError:
        await update.message.reply_text("❌ Номер турели должен быть числом!")
        return

    if turret_id not in TURRETS:
        await update.message.reply_text("❌ Турель с таким номером не существует.")
        return

    turret = TURRETS[turret_id]
    price_column = turret["price_type"]
    price_amount = turret["price"]
    db_column = turret["db_column"]

    # Проверяем ресурсы пользователя
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute(f"SELECT {price_column} FROM users WHERE user_id = ?", (user.id,))
    row = cursor.fetchone()
    user_resource = row[0] if row else 0

    if user_resource < price_amount:
        await update.message.reply_text(
            f"❌ Недостаточно ресурсов.\n"
            f"Нужно: {price_amount} ({price_column.capitalize()})"
        )
        conn.close()
        return

    # Списываем ресурсы и добавляем турель
    cursor.execute(f"""
        UPDATE users SET
            {price_column} = {price_column} - ?,
            {db_column} = {db_column} + 1
        WHERE user_id = ?
    """, (price_amount, user.id))
    conn.commit()
    conn.close()

    await update.message.reply_text(
        f"✅ Турель установлена!\n\n"
        f"{turret['name']} успешно размещена на ферме 🏰\n"
        f"💰 Потрачено: {price_amount} ({price_column})"
    )

async def users_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        users_count = get_users_count()
        await update.message.reply_text(f"📊 Всего пользователей: {users_count}")
    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при получении количества пользователей: {str(e)}")

import sqlite3
from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, filters, CallbackContext

DB_PATH = "bot_chats.db"  # База для хранения chat_id
ALLOWED_USER_ID = 8398161117  # Твой Telegram ID

# --- Сохраняем chat_id ---
def save_chat_id(chat_id):
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute("CREATE TABLE IF NOT EXISTS chats (chat_id INTEGER UNIQUE)")
    cursor.execute("INSERT OR IGNORE INTO chats (chat_id) VALUES (?)", (chat_id,))
    conn.commit()
    conn.close()

# --- Регистрация чата (для старых и новых чатов) ---
async def register_chat(update: Update, context: CallbackContext):
    chat_id = update.effective_chat.id
    save_chat_id(chat_id)
    await update.message.reply_text("✅ Чат зарегистрирован для рассылок.")

# --- Команда для удаления чата из базы рассылки ---
async def unregister_chat(update: Update, context: CallbackContext):
    chat_id = update.effective_chat.id
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute("DELETE FROM chats WHERE chat_id = ?", (chat_id,))
    conn.commit()
    conn.close()
    await update.message.reply_text("❌ Чат удалён из базы рассылок.")

# --- Отправка сообщения во все чаты ---
async def send_message_to_all_chats(message, context: CallbackContext):
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute("SELECT chat_id FROM chats")
    chats = cursor.fetchall()
    conn.close()

    for chat in chats:
        chat_id = chat[0]
        try:
            await context.bot.send_message(chat_id=chat_id, text=message)
        except Exception as e:
            print(f"Ошибка при отправке в {chat_id}: {e}")

# --- Команда для рассылки сообщений ---
async def send_message(update: Update, context: CallbackContext):
    if update.effective_user.id != ALLOWED_USER_ID:
        await update.message.reply_text("❌ У вас нет прав на выполнение этой команды.")
        return

    message = ' '.join(context.args)
    if message:
        await send_message_to_all_chats(message, context)
        await update.message.reply_text(f"✅ Сообщение отправлено во все чаты: {message}")
    else:
        await update.message.reply_text("⚠️ Укажите сообщение для отправки.")

async def farm_callback_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        query = update.callback_query
        await query.answer()

        callback_data = query.data
        user_id = int(callback_data.split('_')[-1])

        if query.from_user.id != user_id:
            await query.answer("❌ Вы не можете взаимодействовать с чужой фермой!", show_alert=True)
            return

        user_data = get_user(user_id, query.from_user.username, query.from_user.first_name)
        user_id, player_id, username, first_name, diamonds, points, last_diamond_time, upgrade_level, registration_date, strawberries, farm_level, last_zombie_attack, zombie_status, last_strawberry_update, last_bonus_time, tomatoes, coins = user_data

        if callback_data.startswith('defend_zombies_'):
            if not zombie_status:
                await query.message.edit_text("🧟 На ферме нет зомби!")
                return

            update_user(user_id, zombie_status=0, last_zombie_attack=datetime.now().isoformat())
            await query.message.edit_text("🧟 Вы успешно отбились от зомби!")

        elif callback_data.startswith('collect_harvest_'):
            if user_id not in strawberry_farm_data:
                await query.message.edit_text("❌ Данные фермы устарели или не найдены!")
                return

            if strawberry_farm_data[user_id]['harvest_collected']:
                await query.message.edit_text("❌ Урожай уже собран!")
                return

            farm_strawberries = strawberry_farm_data[user_id]['farm_strawberries']
            farm_tomatoes = strawberry_farm_data[user_id]['farm_tomatoes']
            new_strawberries = strawberries + farm_strawberries
            new_tomatoes = tomatoes + farm_tomatoes
            update_user(user_id, strawberries=new_strawberries, tomatoes=new_tomatoes, last_strawberry_update=datetime.now().isoformat())

            strawberry_farm_data[user_id]['harvest_collected'] = True
            del strawberry_farm_data[user_id]

            await query.message.edit_text(
                f"🍓 Вы собрали {farm_strawberries} клубники и {farm_tomatoes} помидоров!\n"
                f"🍓 Новый баланс: {new_strawberries} клубники\n"
                f"🍅 Новый баланс: {new_tomatoes} помидоров"
            )

    except Exception as e:
        await query.message.edit_text(f"❌ Ошибка при обработке фермы: {str(e)}")

from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import CommandHandler, CallbackQueryHandler, ContextTypes, ApplicationBuilder

# 🌟 Главное меню доната
async def donate_menu(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [
        [InlineKeyboardButton("💎 Алмазы", callback_data="donate_diamonds")],
        [InlineKeyboardButton("💰 Монеты", callback_data="donate_coins")],
        [InlineKeyboardButton("♦️ Рубины", callback_data="donate_rubies")]
    ]
    await update.message.reply_text(
        "⭐️ Выберите, что хотите купить:",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

async def donate_diamonds_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    await query.message.reply_text(
        "💎 Алмазы:\n"
        "• 37 500 алмазов → 25 ⭐️ / 25 ₽\n"
        "• 80 000 алмазов → 50 ⭐️ / 50 ₽\n"
        "• 170 000 алмазов → 100 ⭐️ / 100 ₽\n"
        "• 380 000 алмазов → 200 ⭐️ / 200 ₽\n"
        "• 800 000 алмазов → 400 ⭐️ / 400 ₽\n\n"
        "• Разбан игрока → 75 ⭐️ / 75 ₽\n"
        "За покупкой — @A3APT_MEPTB"
    )

async def donate_coins_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    await query.message.reply_text(
        "💰 Монеты:\n"
        "• 1 100 000 монет → 25 ⭐️ / 25 ₽\n"
        "• 2 700 000 монет → 50 ⭐️ / 50 ₽\n"
        "• 6 400 000 монет → 100 ⭐️ / 100 ₽\n"
        "• 16 800 000 монет → 200 ⭐️ / 200 ₽\n"
        "• 41 600 000 монет → 400 ⭐️ / 400 ₽\n\n"
        "• Разбан игрока → 75 ⭐️ / 75 ₽\n"
        "За покупкой — @A3APT_MEPTB"
    )

async def donate_rubies_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    await query.message.reply_text(
        "♦️ Рубины:\n"
        "• 10 рубинов → 25 ⭐️ / 25 ₽\n"
        "• 25 рубинов → 50 ⭐️ / 50 ₽\n"
        "• 60 рубинов → 100 ⭐️ / 100 ₽\n"
        "• 160 рубинов → 200 ⭐️ / 200 ₽\n"
        "• 400 рубинов → 400 ⭐️ / 400 ₽\n\n"
        "• Разбан игрока → 75 ⭐️ / 75 ₽\n"
        "За покупкой — @A3APT_MEPTB"
    )

async def buy_farm_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_data = get_user(user.id, user.username, user.first_name)

        if user_data is None or len(user_data) < 25:
            await update.message.reply_text("❌ Пользователь не найден!")
            return

        (
            user_id, player_id, username, first_name, diamonds, points,
            last_diamond_time, upgrade_level, registration_date, strawberries,
            farm_level, last_zombie_attack, zombie_status, last_strawberry_update,
            last_bonus_time, tomatoes, coins, farms_destroyed, clan_id,
            *rest
        ) = user_data

        message_text = update.message.text.strip().lower().split()

        # Проверка формата команды
        if len(message_text) != 4 or message_text[:3] != ["тгз", "купить", "фарм"]:
            await update.message.reply_text("❌ Использование: тгз купить фарм [уровень]")
            return

        # Извлекаем уровень
        try:
            level = int(message_text[3])
        except ValueError:
            await update.message.reply_text("❌ Уровень должен быть числом!")
            return

        # Проверка диапазона уровней
        if level not in range(0, 21):  # ✅ теперь уровни 0–20
            await update.message.reply_text("❌ Уровень должен быть от 0 до 20!")
            return

        if level != upgrade_level + 1:
            await update.message.reply_text(f"❌ Можно купить только следующий уровень фарма ({upgrade_level + 1})!")
            return

        # Получаем инфо об апгрейде
        upgrade_info = get_upgrade_info(level)
        if not upgrade_info:
            await update.message.reply_text("❌ Указан некорректный уровень фарма!")
            return

        cost = upgrade_info["cost"]
        if cost > diamonds:
            await update.message.reply_text(f"❌ Недостаточно алмазов! Нужно: {cost}, у вас: {diamonds}")
            return

        # Списываем алмазы
        new_diamonds = diamonds - cost
        update_user(user_id, diamonds=new_diamonds, upgrade_level=level)

        await update.message.reply_text(
            f"✅ Фарм алмазов успешно улучшен до уровня {level}!\n"
            f"💎 Потрачено: {cost} алмазов\n"
            f"💎 Остаток: {new_diamonds}\n"
            f"📈 Новый доход: {upgrade_info['min_diamonds']}-{upgrade_info['max_diamonds']} алмазов"
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при покупке фарма: {str(e)}")
        print(f"ERROR in buy_farm_command: {str(e)}")

from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import ContextTypes, CallbackQueryHandler

LEVELS_PER_PAGE = 5  # по 5 уровней на странице

def build_upgrade_page(upgrade_level, diamonds, page: int):
    """Возвращает текст и клавиатуру для указанной страницы."""
    start = page * LEVELS_PER_PAGE
    end = start + LEVELS_PER_PAGE
    total_levels = 21  # уровни 0–20
    info_text = f"⚡ Текущий уровень фарма: {upgrade_level}\n\n"

    for level in range(start, min(end, total_levels)):
        upgrade_info = get_upgrade_info(level)
        is_current = "✅ Текущий уровень" if level == upgrade_level else ""
        if upgrade_info:
            info_text += (
                f"Уровень {level}:\n"
                f"💎 Стоимость: {upgrade_info['cost']}\n"
                f"📈 Доход: {upgrade_info['min_diamonds']}-{upgrade_info['max_diamonds']} алмазов\n"
                f"{is_current}\n\n"
            )
        else:
            info_text += f"Уровень {level}: данные не найдены\n\n"

    info_text += f"💎 Ваш баланс: {diamonds}\n📋 Используйте: тгз купить фарм [уровень]"

    # Кнопки навигации
    buttons = []
    if page > 0:
        buttons.append(InlineKeyboardButton("◀ Назад", callback_data=f"upgrade_page_{page-1}"))
    if end < total_levels:
        buttons.append(InlineKeyboardButton("Вперёд ▶", callback_data=f"upgrade_page_{page+1}"))

    keyboard = InlineKeyboardMarkup([buttons]) if buttons else None
    return info_text, keyboard


async def upgrade_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Обработка команды 'тгз прокачка'"""
    try:
        user = update.effective_user
        user_data = get_user(user.id, user.username, user.first_name)

        if not user_data or len(user_data) < 24:
            await update.message.reply_text("❌ Ошибка: пользователь не найден!")
            return

        (
            user_id, player_id, username, first_name, diamonds, points,
            last_diamond_time, upgrade_level, registration_date, strawberries,
            farm_level, last_zombie_attack, zombie_status, last_strawberry_update,
            last_bonus_time, tomatoes, coins, farms_destroyed, clan_id,
            *rest
        ) = user_data

        message_text = update.message.text.strip().lower().split()
        if len(message_text) != 2 or message_text != ["тгз", "прокачка"]:
            await update.message.reply_text("❌ Использование: тгз прокачка")
            return

        # Старт с первой страницы (0)
        page = 0
        text, keyboard = build_upgrade_page(upgrade_level, diamonds, page)
        await update.message.reply_text(text, reply_markup=keyboard)

    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка при просмотре прокачки: {e}")
        print(f"ERROR in upgrade_command: {e}")


async def upgrade_page_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Обработка нажатий на кнопки страниц"""
    query = update.callback_query
    await query.answer()

    try:
        user = query.from_user
        user_data = get_user(user.id, user.username, user.first_name)

        if not user_data or len(user_data) < 23:
            await query.edit_message_text("❌ Ошибка: пользователь не найден!")
            return

        (
            user_id, player_id, username, first_name, diamonds, points,
            last_diamond_time, upgrade_level, registration_date, strawberries,
            farm_level, last_zombie_attack, zombie_status, last_strawberry_update,
            last_bonus_time, tomatoes, coins, farms_destroyed, clan_id,
            *rest
        ) = user_data

        # Определяем страницу из callback_data
        data = query.data
        if not data.startswith("upgrade_page_"):
            return

        page = int(data.split("_")[-1])
        text, keyboard = build_upgrade_page(upgrade_level, diamonds, page)

        await query.edit_message_text(text, reply_markup=keyboard)

    except Exception as e:
        await query.edit_message_text(f"❌ Ошибка при обработке страницы: {e}")
        print(f"ERROR in upgrade_page_callback: {e}")

async def russian_roulette_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_data = get_user(user.id, user.username, user.first_name)

        if not user_data or len(user_data) < 24:
            await update.message.reply_text("❌ Пользователь не найден!")
            return

        (
            user_id, player_id, username, first_name, diamonds, points,
            last_diamond_time, upgrade_level, registration_date,
            strawberries, farm_level, last_zombie_attack, zombie_status,
            last_strawberry_update, last_bonus_time, tomatoes, coins,
            farms_destroyed, clan_id, *rest
        ) = user_data

        # Проверка — если игра уже активна
        if user_id in russian_roulette_data:
            await update.message.reply_text("❌ У вас уже есть активная игра в рулетку! Завершите её перед новой.")
            return

        message_text = update.message.text.strip().lower().split()
        if len(message_text) != 4 or message_text[:3] != ["тгз", "русская", "рулетка"]:
            await update.message.reply_text("❌ Использование: тгз русская рулетка [ставка]")
            return

        try:
            bet = int(message_text[3])
        except ValueError:
            await update.message.reply_text("❌ Ставка должна быть числом!")
            return

        if bet < 700:
            await update.message.reply_text("❌ Минимальная ставка: 700 алмазов!")
            return

        if bet > diamonds:
            await update.message.reply_text(f"❌ Недостаточно алмазов! У вас: {diamonds}, ставка: {bet}")
            return

        # Генерация барабана
        revolver = [False] * 6 + [True]
        random.shuffle(revolver)
        russian_roulette_data[user_id] = {
            'bet': bet,
            'shots_fired': 0,
            'revolver': revolver,
            'current_diamonds': diamonds - bet,
            'locked': False
        }

        # Сразу списываем ставку
        new_diamonds = diamonds - bet
        update_user(user_id, diamonds=new_diamonds)

        rules_text = f"""Русская рулетка началась!

В барабане 6 ячеек, 1 из них - смертельная. Вы можете остановиться в любой момент и забрать выигрыш.

Ваша ставка: {bet} алмазов
Остаток: {new_diamonds} алмазов

Нажмите "Продолжить" для первого выстрела!"""

        keyboard = [
            [
                InlineKeyboardButton("🔫 Продолжить", callback_data=f"rr_continue_{user_id}"),
                InlineKeyboardButton("💰 Забрать выигрыш", callback_data=f"rr_stop_{user_id}")
            ]
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)

        await update.message.reply_text(rules_text, reply_markup=reply_markup)

    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка в русской рулетке: {str(e)}")
        print(f"ERROR in russian_roulette_command: {str(e)}")


async def russian_roulette_callback_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        query = update.callback_query
        await query.answer()

        callback_data = query.data
        user_id = int(callback_data.split('_')[-1])

        if query.from_user.id != user_id:
            await query.answer("❌ Вы не можете играть в чужую игру!", show_alert=True)
            return

        if user_id not in russian_roulette_data:
            await query.message.edit_text("❌ Данные игры устарели или не найдены!")
            return

        game_data = russian_roulette_data[user_id]

        # защита от двойного клика
        if game_data.get("locked", False):
            await query.answer("⏳ Подождите! Ход уже обрабатывается.", show_alert=True)
            return
        game_data["locked"] = True

        bet = game_data['bet']
        shots_fired = game_data['shots_fired']
        revolver = game_data['revolver']
        current_diamonds = game_data['current_diamonds']

        # Проверяем свежие данные из базы
        fresh_data = get_user(user_id, query.from_user.username, query.from_user.first_name)
        if fresh_data and len(fresh_data) >= 23:
            diamonds = fresh_data[4]
        else:
            diamonds = current_diamonds

        # Игрок решил забрать выигрыш
        if callback_data.startswith('rr_stop_'):
            win_multiplier = {0: 1, 1: 1.08, 2: 1.25, 3: 1.5, 4: 1.8, 5: 2.1}.get(shots_fired, 0)
            win_amount = int(bet * win_multiplier)

            fresh = get_user(user_id, query.from_user.username, query.from_user.first_name)
            current_real = fresh[4]  # ТЕКУЩИЕ АЛМАЗЫ ИЗ БАЗЫ

            new_diamonds = current_real + win_amount

            update_user(user_id, diamonds=new_diamonds)
            del russian_roulette_data[user_id]

            await query.message.edit_text(
                f"💰 Вы забрали выигрыш!\n"
                f"🎁 Чистый выигрыш: {win_amount} алмазов\n"
                f"💎 Текущий баланс: {new_diamonds}"
            )
            return

        # Игрок продолжает
        elif callback_data.startswith('rr_continue_'):
            shots_fired += 1
            game_data['shots_fired'] = shots_fired

            # Смертельный выстрел
            if revolver[shots_fired - 1]:
                update_user(user_id, diamonds=current_diamonds)
                del russian_roulette_data[user_id]
                await query.message.edit_text(
                    f"💥 Выстрел {shots_fired}: Поражение! Вы проиграли.\n"
                    f"💸 Потеряно: {bet} алмазов\n"
                    f"💎 Текущий баланс: {current_diamonds}"
                )
                return

            # Если выжил все 5 выстрелов
            if shots_fired >= 5:
                win_amount = int(bet * 2.1)

                fresh = get_user(user_id, query.from_user.username, query.from_user.first_name)
                current_real = fresh[4]

                new_diamonds = current_real + win_amount

                update_user(user_id, diamonds=new_diamonds)
                del russian_roulette_data[user_id]

                await query.message.edit_text(
                    f"🎉 Поздравляем! Вы выжили все 5 выстрелов!\n"
                    f"🎁 Чистый выигрыш: {win_amount} алмазов\n"
                    f"💎 Текущий баланс: {new_diamonds}"
                )
                return

            # Если игрок выжил, но игра продолжается
            encouragement_phrases = [
                "Отличное начало! Готовы рискнуть еще?",
                "Удача на вашей стороне! Не упустите свой шанс!",
                "Смелее, еще один шаг к победе!",
                "Вы везунчик! Продолжайте!"
            ]
            encouragement = random.choice(encouragement_phrases)
            win_multiplier = {1: 1.08, 2: 1.25, 3: 1.5, 4: 1.8, 5: 2.1}.get(shots_fired, 0)

            keyboard = [
                [
                    InlineKeyboardButton("🔫 Продолжить", callback_data=f"rr_continue_{user_id}"),
                    InlineKeyboardButton("💰 Забрать выигрыш", callback_data=f"rr_stop_{user_id}")
                ]
            ]
            reply_markup = InlineKeyboardMarkup(keyboard)

            game_data["locked"] = False

            await query.message.edit_text(
                f"🔫 Выстрел {shots_fired}: Вы выжили!\n"
                f"💰 Текущий возможный выигрыш: {int(bet * win_multiplier)} алмазов\n"
                f"{encouragement}",
                reply_markup=reply_markup
            )

    except Exception as e:
        await query.message.edit_text(f"❌ Ошибка в русской рулетке: {str(e)}")
        print(f"ERROR in russian_roulette_callback_handler: {str(e)}")

import random
import sqlite3
from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import ContextTypes

DB_PATH = "diamonds_bot.db"
MINES_COUNT = 5
FIELD_SIZE = 5
MIN_BET = 700

active_mines_games = {}

def calculate_multiplier(safe_cells):
    return round(1 + safe_cells * 0.102 + (safe_cells ** 2) * 0.054, 2)

async def minesweeper_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args=None):
    try:
        user = update.effective_user
        user_id = user.id

        if not args or len(args) < 1:
            await update.message.reply_text("❌ Укажите ставку: тгз минное поле <ставка>\nМинимальная ставка: 300")
            return

        try:
            bet = int(args[0])
        except ValueError:
            await update.message.reply_text("❌ Ставка должна быть числом!")
            return

        if bet < MIN_BET:
            await update.message.reply_text(f"❌ Минимальная ставка: {MIN_BET}")
            return

        # Проверяем баланс и активную игру
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("SELECT diamonds, active_mines_game FROM users WHERE user_id = ?", (user_id,))
        row = cursor.fetchone()
        if not row or row[0] < bet:
            conn.close()
            await update.message.reply_text(f"❌ Недостаточно алмазов для ставки!")
            return

        if row[1]:  # Если уже активная игра
            conn.close()
            await update.message.reply_text("❌ У тебя уже есть активная игра! Заверши её перед новой ставкой.")
            return

        # Списываем ставку и включаем флаг активной игры
        cursor.execute("UPDATE users SET diamonds = diamonds - ?, active_mines_game = 1 WHERE user_id = ?", (bet, user_id))
        conn.commit()
        conn.close()

        # Создаём игру
        mine_positions = random.sample(range(FIELD_SIZE ** 2), MINES_COUNT)
        active_mines_games[user_id] = {
            "mines": mine_positions,
            "opened": [],
            "alive": True,
            "bet": bet
        }

        # Клавиатура
        keyboard = [
            [InlineKeyboardButton("⬜", callback_data=f"mine_{user_id}_{r * FIELD_SIZE + c}") for c in range(FIELD_SIZE)]
            for r in range(FIELD_SIZE)
        ]
        keyboard.append([InlineKeyboardButton("💎 Забрать", callback_data=f"mine_take_{user_id}")])

        await update.message.reply_text(
            f"💣 *Минное поле 5×5!*\nСтавка: {bet} 💎\nОткрой клетки!\n💎 За каждую выжившую награда растёт.",
            parse_mode="Markdown",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка в минном поле: {str(e)}")
        print(f"ERROR in minesweeper_command: {str(e)}")

async def minesweeper_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    user_id = query.from_user.id
    data = query.data

    # Игнорируем нажатие на "мертвые" кнопки
    if data == "none":
        return

    if data.startswith("mine_take_"):
        _, target_user = data.split("_take_")
        target_user = int(target_user)
    else:
        _, target_user, cell_id = data.split("_")
        target_user = int(target_user)

    if target_user != user_id:
        await query.answer("Это не твоя игра!", show_alert=True)
        return

    if target_user not in active_mines_games:
        await query.edit_message_text("❌ У тебя нет активной игры! Напиши: тгз минное поле <ставка>", parse_mode="Markdown")
        return

    game = active_mines_games[user_id]

    if data.startswith("mine_take_"):
        safe_cells = len(game["opened"])
        multiplier = calculate_multiplier(safe_cells)
        reward = int(game["bet"] * multiplier)

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("UPDATE users SET diamonds = diamonds + ?, active_mines_game = 0 WHERE user_id = ?", (reward, user_id))
        conn.commit()
        conn.close()

        del active_mines_games[user_id]
        await query.edit_message_text(f"💎 Ты забрал {reward} алмазов!\nВыживших клеток: {safe_cells}")
        return

    # Клетка
    cell_id = int(cell_id)
    if cell_id in game["opened"]:
        return

    if cell_id in game["mines"]:
        game["alive"] = False

        keyboard = []
        for r in range(FIELD_SIZE):
            row = []
            for c in range(FIELD_SIZE):
                i = r * FIELD_SIZE + c
                if i in game["mines"]:
                    emoji = "💥" if i == cell_id else "💣"
                elif i in game["opened"]:
                    emoji = "🟩"
                else:
                    emoji = "⬜"
                # Кнопки после конца игры не активны
                row.append(InlineKeyboardButton(emoji, callback_data="none"))
            keyboard.append(row)

        # Сбрасываем флаг активной игры
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("UPDATE users SET active_mines_game = 0 WHERE user_id = ?", (user_id,))
        conn.commit()
        conn.close()

        await query.edit_message_text(
            "💥 Ты попал на мину! Все мины показаны ниже:",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )
        del active_mines_games[user_id]
        return

    game["opened"].append(cell_id)

    keyboard = []
    for r in range(FIELD_SIZE):
        row = []
        for c in range(FIELD_SIZE):
            i = r * FIELD_SIZE + c
            row.append(InlineKeyboardButton("🟩" if i in game["opened"] else "⬜", callback_data=f"mine_{user_id}_{i}"))
        keyboard.append(row)
    keyboard.append([InlineKeyboardButton("💎 Забрать", callback_data=f"mine_take_{user_id}")])

    multiplier = calculate_multiplier(len(game["opened"]))
    await query.edit_message_text(
        f"✅ Клетка {len(game['opened'])} безопасна!\nВыживших: {len(game['opened'])}\n💎 Потенциальная награда: {int(game['bet'] * multiplier)} (x{multiplier:.2f})",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

async def coin_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_data = get_user(user.id, user.username, user.first_name)

        if user_data is None or len(user_data) < 24:
            await update.message.reply_text("❌ Пользователь не найден!")
            return

        (
            user_id, player_id, username, first_name, diamonds, points,
            last_diamond_time, upgrade_level, registration_date, strawberries,
            farm_level, last_zombie_attack, zombie_status, last_strawberry_update,
            last_bonus_time, tomatoes, coins, farms_destroyed, clan_id, *rest
        ) = user_data

        message_text = update.message.text.strip().lower().split()

        # Проверка команды
        if len(message_text) < 4 or message_text[:2] != ["тгз", "монетка"]:
            await update.message.reply_text("❌ Использование: тгз монетка [орёл/решка] [ставка]")
            return

        side_choice = message_text[2]
        if side_choice not in ["орёл", "решка"]:
            await update.message.reply_text("❌ Нужно выбрать сторону: орёл или решка!")
            return

        try:
            bet = int(message_text[3])
        except (ValueError, IndexError):
            await update.message.reply_text("❌ Ставка должна быть числом!")
            return

        if bet < 1400:
            await update.message.reply_text("❌ Минимальная ставка: 1400 алмазов!")
            return

        if bet > diamonds:
            await update.message.reply_text(f"❌ Недостаточно алмазов! У вас: {diamonds}, ставка: {bet}")
            return

        # Рандомим результат
        import random
        golden_chance = random.randint(2, 100) == 2  #5% шанс золотой монеты
        coin_side = random.choice(["орёл", "решка"])

        if golden_chance:
            win_amount = bet * 5
            new_diamonds = diamonds - bet + win_amount
            update_user(user_id, diamonds=new_diamonds)
            await update.message.reply_text(
                f"✨🪙 Золотая монета! Удача на вашей стороне!\n"
                f"Ставка: {bet} алмазов\n"
                f"Выигрыш: {win_amount} алмазов (x5)\n"
                f"💎 Новый баланс: {new_diamonds}"
            )
            return

        if coin_side == side_choice:
            # Победа: ставка возвращается +70%
            win_amount = int(bet * 1.70)
            new_diamonds = diamonds - bet + win_amount
            update_user(user_id, diamonds=new_diamonds)
            await update.message.reply_text(
                f"🎉 Выпал {coin_side}, вы угадали!\n"
                f"💰 Выигрыш: {win_amount} алмазов (+70%)\n"
                f"💎 Новый баланс: {new_diamonds}"
            )
        else:
            # Проигрыш
            new_diamonds = diamonds - bet
            update_user(user_id, diamonds=new_diamonds)
            await update.message.reply_text(
                f"😔 Выпал {coin_side}, вы не угадали!\n"
                f"💸 Потеряно: {bet} алмазов\n"
                f"💎 Новый баланс: {new_diamonds}"
            )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка в игре монетка: {str(e)}")
        print(f"ERROR in coin_command: {str(e)}")

import asyncio
import random
from telegram import Update
from telegram.ext import ContextTypes

async def roulette_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_data = get_user(user.id, user.username, user.first_name)

        if user_data is None or len(user_data) < 24:
            await update.message.reply_text("❌ Пользователь не найден!")
            return

        (
            user_id, player_id, username, first_name, diamonds, points,
            last_diamond_time, upgrade_level, registration_date, strawberries,
            farm_level, last_zombie_attack, zombie_status, last_strawberry_update,
            last_bonus_time, tomatoes, coins, farms_destroyed, clan_id,
            *rest
        ) = user_data

        message_text = update.message.text.strip().lower().split()

        # Проверка формата команды
        if len(message_text) != 4 or message_text[:2] != ["тгз", "рулетка"]:
            await update.message.reply_text("❌ Использование: тгз рулетка [ставка] [красное/чёрное/зелёное]")
            return

        try:
            amount = int(message_text[2])
            if amount < 350:
                await update.message.reply_text("❌ Минимальная ставка: 350 алмазов!")
                return
        except (ValueError, IndexError):
            await update.message.reply_text("❌ Ставка должна быть числом!")
            return

        color = message_text[3]
        if color not in ["красное", "чёрное", "зелёное"]:
            await update.message.reply_text("❌ Цвет должен быть: красное, чёрное или зелёное!")
            return

        if amount > diamonds:
            await update.message.reply_text(f"❌ Недостаточно алмазов! У вас: {diamonds}")
            return

        # Анимация рулетки
        phrases = [
            "🎰 Рулетка крутится...",
            "⚡ Шарик ускоряется...",
            "🔄 Всё быстрее и быстрее...",
            "⏳ Шарик замедляется...",
            "👀 Почти остановился..."
        ]

        spinning_message = await update.message.reply_text(random.choice(phrases))
        last_text = spinning_message.text

        for _ in range(2):  # сменим текст 2 раза
            await asyncio.sleep(random.uniform(1.0, 1.5))
            new_text = random.choice([p for p in phrases if p != last_text])
            last_text = new_text
            await spinning_message.edit_text(new_text)

        await asyncio.sleep(random.uniform(1.5, 3.5))  # финальная пауза

        # Логика рулетки
        outcomes = ["красное"] * 18 + ["чёрное"] * 18 + ["зелёное"]
        result = random.choice(outcomes)

        # Эмодзи для результата
        color_emoji = {
            "красное": "❤️",
            "чёрное": "⚫",
            "зелёное": "🎲"
        }[result]

        if result == color:
            if color in ["красное", "чёрное"]:
                winnings = int(amount * 1.7)
            else:  # зелёное 🎲
                winnings = int(amount * 9)

            new_diamonds = diamonds - amount + winnings
            result_text = (
                f"🎉 Победа! Выпало: {color_emoji} {result}\n"
                f"💰 Выигрыш: {winnings} алмазов\n"
                f"💎 Новый баланс: {new_diamonds}"
            )
        else:
            new_diamonds = diamonds - amount
            result_text = (
                f"💥 Поражение! Выпало: {color_emoji} {result}\n"
                f"💸 Потеряно: {amount} алмазов\n"
                f"💎 Остаток: {new_diamonds}"
            )

        # Обновляем БД
        update_user(user_id, diamonds=new_diamonds)

        # Избегаем ошибки "Message is not modified"
        if result_text == last_text:
            result_text += " "

        # Финальный результат
        await spinning_message.edit_text(result_text)

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при игре в рулетку: {str(e)}")
        print(f"ERROR in roulette_command: {str(e)}")

import time
from datetime import datetime, timezone
from telegram import InlineKeyboardButton, InlineKeyboardMarkup
import random
import sqlite3
from telegram.ext import Application, CommandHandler, MessageHandler, CallbackQueryHandler, filters
from telegram import Update
from telegram.ext import ContextTypes

# Глобальная переменная для русской рулетки
russian_roulette_data = {}

import sqlite3

def init_db():
    conn = sqlite3.connect("diamonds_bot.db")
    cursor = conn.cursor()

    # ✅ Таблица пользователей
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER PRIMARY KEY,
        player_id INTEGER UNIQUE,
        username TEXT,
        first_name TEXT,
        diamonds INTEGER DEFAULT 0,
        points INTEGER DEFAULT 0,
        last_diamond_time TEXT,
        upgrade_level INTEGER DEFAULT 0,
        registration_date TEXT,
        strawberries INTEGER DEFAULT 0,
        farm_level INTEGER DEFAULT 0,
        last_zombie_attack TEXT,
        zombie_status BOOLEAN DEFAULT FALSE,
        last_strawberry_update TEXT,
        last_bonus_time TEXT,
        tomatoes INTEGER DEFAULT 0,
        coins INTEGER DEFAULT 0,
        farms_destroyed INTEGER DEFAULT 0,
        clan_id INTEGER,
        FOREIGN KEY (clan_id) REFERENCES clans(id)
    )
    """)

    # ✅ Проверка и добавление недостающих колонок
    cursor.execute("PRAGMA table_info(users)")
    columns = [row[1] for row in cursor.fetchall()]

    if "last_reduce_cd" not in columns:
        cursor.execute("ALTER TABLE users ADD COLUMN last_reduce_cd TEXT")
        print("✅ Колонка last_reduce_cd добавлена!")

    if "farms_destroyed" not in columns:
        cursor.execute("ALTER TABLE users ADD COLUMN farms_destroyed INTEGER DEFAULT 0")

    if "clan_id" not in columns:
        cursor.execute("ALTER TABLE users ADD COLUMN clan_id INTEGER")

    # ✅ Новые ресурсы
    if "wood" not in columns:
        cursor.execute("ALTER TABLE users ADD COLUMN wood INTEGER DEFAULT 0")
    if "stone" not in columns:
        cursor.execute("ALTER TABLE users ADD COLUMN stone INTEGER DEFAULT 0")
    if "iron" not in columns:
        cursor.execute("ALTER TABLE users ADD COLUMN iron INTEGER DEFAULT 0")
    if "titanium" not in columns:
        cursor.execute("ALTER TABLE users ADD COLUMN titanium INTEGER DEFAULT 0")

    # ✅ Таблица кланов
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS clans (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        clan_name TEXT UNIQUE NOT NULL,
        owner_id INTEGER NOT NULL
    )
    """)

    # ✅ Таблица участников кланов
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS clan_members (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        clan_id INTEGER NOT NULL,
        user_id INTEGER NOT NULL,
        FOREIGN KEY (clan_id) REFERENCES clans(id),
        FOREIGN KEY (user_id) REFERENCES users(user_id)
    )
    """)

    # ✅ Таблица статуса фермы
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS farm_status (
        user_id INTEGER PRIMARY KEY,
        farm_health INTEGER DEFAULT 100,
        max_farm_health INTEGER DEFAULT 100,
        last_farm_attack TEXT,
        FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
    )
    """)

    # ✅ Таблица масок
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS user_masks (
        user_id INTEGER,
        mask_id INTEGER,
        PRIMARY KEY (user_id, mask_id),
        FOREIGN KEY (user_id) REFERENCES users(user_id)
    )
    """)

    # ✅ Таблица зомби
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS user_zombies (
        user_id INTEGER,
        zombie_id INTEGER,
        quantity INTEGER DEFAULT 0,
        PRIMARY KEY (user_id, zombie_id),
        FOREIGN KEY (user_id) REFERENCES users(user_id)
    )
    """)

    # ✅ Таблица лога изменений
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS change_log (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        player_id INTEGER,
        changed_by_user_id INTEGER,
        resource_type TEXT,
        old_value INTEGER,
        new_value INTEGER,
        timestamp TEXT,
        FOREIGN KEY (user_id) REFERENCES users(user_id)
    )
    """)

    conn.commit()
    conn.close()
    print("✅ База данных инициализирована (ничего не удалено).")

# 🔹 Функция для получения информации о масках
def get_mask_info(mask_id):
    masks = {
        1: {"name": "Опыт", "effect": "На 1 очко больше с команды тгз фарм", "cost": 12999},
        2: {"name": "Старатель", "effect": "При команде тгз фарм выдаёт дополнительно 32000 монет", "cost": 17999},
        3: {"name": "Бонус", "effect": "На 5250 больше алмазов с команды тгз ежедневный бонус", "cost": 20999},
        4: {"name": "Фермер", "effect": "При команде тгз фарм выдаёт дополнительно 3000 клубники и 90000 помидоров", "cost": 24999},
        5: {"name": "Шахтёр", "effect": "На 400 алмазов больше с команды тгз фарм", "cost": 32999},
        6: {"name": "Живучесть", "effect": "Выдаёт дополнительные 55 здоровья фермы при команде тгз ежедневный бонус", "cost": 16999},
        7: {"name": "Садовник", "effect": "Выдаёт дополнительные 58 000 клубники при команде тгз ежедневный бонус", "cost": 23199},
        8: {"name": "Томат", "effect": "Выдаёт дополнительные 1 700 000 помидоров при команде тгз ежедневный бонус", "cost": 25299},
        9: {"name": "Зомбификатор", "effect": "Выдаёт дополнительные 5 зомби Невидимок (ID: 4) при команде тгз ежедневный бонус", "cost": 28899},
        10: {"name": "Титановый", "effect": "Выдаёт дополнительные 36 000 титана при команде тгз ежедневный бонус", "cost": 36899},
        11: {"name": "Знаменосец", "effect": "Стоимость создать клан в 5 раз дешевле + Добавляет 200000 дерева и 200000 камня к фарму", "cost": 22499},
        12: {"name": "Обнаружение", "effect": "Показывает ID и имя игрока, который напал на вашу ферму", "cost": 40799},
        13: {"name": "Вестник", "effect": "Выдаёт +22 элитных разведчиков (ID 14) при тгз ежедневный бонус", "cost": 15699},
        14: {"name": "Берсеркер", "effect": "Выдаёт +3 зомби Берсерка (ID 20) при тгз ежедневный бонус", "cost": 39299},
        15: {"name": "Турельщик", "effect": "Выдаёт 3 ракетных турели при тгз ежедневный бонус", "cost": 288999},

        # === НОВЫЕ МАСКИ ===
        16: {"name": "Слизняк", "effect": "При тгз фарм выдаёт 2 зомби Слизняка (ID 19)", "cost": 27799},
        17: {"name": "Подрыватель", "effect": "При тгз фарм выдаёт 2 дополнительные морские мины", "cost": 55299},
        18: {"name": "Заяц", "effect": "При тгз фарм шанс 20% получить 1 зомби Зайку (ID 12)", "cost": 197999},
        19: {"name": "Арбалетчик", "effect": "При тгз фарм выдаёт 2 дополнительные деревянные турели", "cost": 141999},
    }
    print(f"DEBUG: get_mask_info called with mask_id={mask_id}, returning {masks.get(mask_id)}")
    return masks.get(mask_id)

# Функция для получения информации об уровнях фермы
def get_farm_info(farm_level):
    farm_levels = {
        0: {"cost": 0, "strawberries_per_hour": 50, "tomatoes_per_hour": 1500, "grapes_per_hour": 5},
        1: {"cost": 36000, "strawberries_per_hour": 55, "tomatoes_per_hour": 1650, "grapes_per_hour": 5.5},
        2: {"cost": 39600, "strawberries_per_hour": 60.5, "tomatoes_per_hour": 1815, "grapes_per_hour": 6},
        3: {"cost": 43560, "strawberries_per_hour": 66.5, "tomatoes_per_hour": 1995, "grapes_per_hour": 6.5},
        4: {"cost": 47916, "strawberries_per_hour": 73, "tomatoes_per_hour": 2190, "grapes_per_hour": 7.5},
        5: {"cost": 52707.5, "strawberries_per_hour": 80.5, "tomatoes_per_hour": 2415, "grapes_per_hour": 8},
        6: {"cost": 57978.5, "strawberries_per_hour": 88.5, "tomatoes_per_hour": 2655, "grapes_per_hour": 9},
        7: {"cost": 63776, "strawberries_per_hour": 97.5, "tomatoes_per_hour": 2925, "grapes_per_hour": 9.5},
        8: {"cost": 70153.5, "strawberries_per_hour": 107, "tomatoes_per_hour": 3210, "grapes_per_hour": 10.5},
        9: {"cost": 77169, "strawberries_per_hour": 118, "tomatoes_per_hour": 3540, "grapes_per_hour": 12},
        10: {"cost": 84886, "strawberries_per_hour": 129.5, "tomatoes_per_hour": 3885, "grapes_per_hour": 13},
        11: {"cost": 93374.5, "strawberries_per_hour": 142.5, "tomatoes_per_hour": 4275, "grapes_per_hour": 14},
        12: {"cost": 102712, "strawberries_per_hour": 157, "tomatoes_per_hour": 4710, "grapes_per_hour": 15.5},
        13: {"cost": 112983, "strawberries_per_hour": 172.5, "tomatoes_per_hour": 5175, "grapes_per_hour": 17.5},
        14: {"cost": 124281.5, "strawberries_per_hour": 190, "tomatoes_per_hour": 5700, "grapes_per_hour": 19},
        15: {"cost": 136709.5, "strawberries_per_hour": 209, "tomatoes_per_hour": 6270, "grapes_per_hour": 21},
        16: {"cost": 150380.5, "strawberries_per_hour": 230, "tomatoes_per_hour": 6900, "grapes_per_hour": 23},
        17: {"cost": 165418.5, "strawberries_per_hour": 253, "tomatoes_per_hour": 7590, "grapes_per_hour": 25.5},
        18: {"cost": 181960.5, "strawberries_per_hour": 278.5, "tomatoes_per_hour": 8355, "grapes_per_hour": 28},
        19: {"cost": 200156.5, "strawberries_per_hour": 306, "tomatoes_per_hour": 9180, "grapes_per_hour": 30.5},
        20: {"cost": 220172, "strawberries_per_hour": 336.5, "tomatoes_per_hour": 10095, "grapes_per_hour": 33.5},
    }

    return farm_levels.get(farm_level)

import sqlite3
from datetime import datetime, timezone
from telegram import Update
from telegram.ext import ContextTypes

DB_PATH = "diamonds_bot.db"

# 🧩 Функция для выдачи бонусов от масок (7–10)
async def give_daily_mask_rewards(user_id: int):
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    # ✅ Получаем все купленные маски пользователя
    cursor.execute("SELECT mask_id FROM user_masks WHERE user_id = ?", (user_id,))
    owned_masks = [row[0] for row in cursor.fetchall()]

    rewards_text = ""
    has_bonus = False

    if 7 in owned_masks:  # Маска Садовник
        cursor.execute("UPDATE users SET strawberries = strawberries + 58000 WHERE user_id = ?", (user_id,))
        rewards_text += "🍓 +58 000 клубники (Маска Садовник)\n"
        has_bonus = True

    if 8 in owned_masks:  # Маска Томат
        cursor.execute("UPDATE users SET tomatoes = tomatoes + 1700000 WHERE user_id = ?", (user_id,))
        rewards_text += "🍅 +1 700 000 помидоров (Маска Томат)\n"
        has_bonus = True

    if 9 in owned_masks:  # Маска Зомбификатор
        cursor.execute("""
            INSERT INTO user_zombies(user_id, zombie_id, quantity)
            VALUES(?, 4, 5)
            ON CONFLICT(user_id, zombie_id)
            DO UPDATE SET quantity = quantity + 5
        """, (user_id,))
        rewards_text += "🧟 +5 зомби Невидимок (Маска Зомбификатор)\n"
        has_bonus = True

    if 10 in owned_masks:  # Маска Титановый
        cursor.execute("UPDATE users SET titanium = titanium + 36000 WHERE user_id = ?", (user_id,))
        rewards_text += "🔩 +36 000 титана (Маска Титановый)\n"
        has_bonus = True

    if 13 in owned_masks:  # Маска Вестник
        cursor.execute("""
            INSERT INTO user_zombies(user_id, zombie_id, quantity)
            VALUES(?, 14, 22)
            ON CONFLICT(user_id, zombie_id)
            DO UPDATE SET quantity = quantity + 22
        """, (user_id,))
        rewards_text += "🛰 +22 элитных разведчиков (Маска Вестник)\n"
        has_bonus = True

    if 14 in owned_masks:  # Маска Берсеркер
        cursor.execute("""
            INSERT INTO user_zombies(user_id, zombie_id, quantity)
            VALUES(?, 20, 3)
            ON CONFLICT(user_id, zombie_id)
            DO UPDATE SET quantity = quantity + 3
        """, (user_id,))
        rewards_text += "💢 +3 зомби Берсерка (Маска Берсеркер)\n"
        has_bonus = True

    if 15 in owned_masks:  # Маска Турельщик
        cursor.execute("""
            UPDATE users SET turret_rocket = turret_rocket + 3 WHERE user_id = ?
        """, (user_id,))
        rewards_text += "🚀 +3 ракетные турели (Маска Турельщик)\n"
        has_bonus = True

    if 6 in owned_masks:  # Маска Живучесть
        cursor.execute("""
            UPDATE users
            SET new_farm_health = new_farm_health + 55,
                max_farm_health = max_farm_health + 55
            WHERE user_id = ?
        """, (user_id,))
        conn.commit()
        rewards_text += "❤️ +55 здоровья фермы (Маска Живучесть)\n"
        has_bonus = True

    conn.commit()
    conn.close()

    if has_bonus:
        return "\n🎭 <b>Бонус от ваших масок:</b>\n" + rewards_text
    else:
        return ""

# 💎 Команда: тгз ежедневный бонус
async def daily_bonus_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_data = get_user(user.id, user.username, user.first_name)

        if user_data is None or len(user_data) < 25:
            print(f"ERROR: user_data has {len(user_data) if user_data else 0} values, expected 23. User ID: {user.id}")
            await update.message.reply_text("❌ Ошибка: пользователь не найден. Обратитесь к администратору.")
            return

        (
            user_id, player_id, username, first_name, diamonds, points,
            last_diamond_time, upgrade_level, registration_date, strawberries,
            farm_level, last_zombie_attack, zombie_status, last_strawberry_update,
            last_bonus_time, tomatoes, coins, farms_destroyed, clan_id,
            *_
        ) = user_data

        parts = update.message.text.strip().lower().split()
        if parts != ["тгз", "ежедневный", "бонус"]:
            await update.message.reply_text("❌ Использование: тгз ежедневный бонус")
            return

        current_time = datetime.now(timezone.utc)
        cooldown_hours = 24

        # Проверка кулдауна
        if last_bonus_time:
            try:
                last_bonus = datetime.fromisoformat(last_bonus_time.replace('Z', '+00:00'))
                if last_bonus.tzinfo is None:
                    last_bonus = last_bonus.replace(tzinfo=timezone.utc)
                hours_passed = (current_time - last_bonus).total_seconds() / 3600
            except (ValueError, TypeError):
                hours_passed = float('inf')
        else:
            hours_passed = float('inf')

        if hours_passed < cooldown_hours:
            remaining_time = (cooldown_hours - hours_passed) * 3600
            hours, remainder = divmod(int(remaining_time), 3600)
            minutes = remainder // 60
            await update.message.reply_text(f"❌ Бонус на кулдауне! Осталось: {hours} ч {minutes} мин")
            return

        # 💎 Основной бонус
        bonus_diamonds = 1000
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute('SELECT 1 FROM user_masks WHERE user_id = ? AND mask_id = ?', (user_id, 3))
        if cursor.fetchone():
            bonus_diamonds += 5250
        conn.close()

        new_diamonds = diamonds + bonus_diamonds
        update_user(user_id, diamonds=new_diamonds, last_bonus_time=current_time.isoformat())

        # 🎭 Получаем бонусы масок
        mask_bonus_text = await give_daily_mask_rewards(user_id)

        # 📜 Общий текст
        bonus_text = (
            f"✅ Ежедневный бонус получен!\n"
            f"💎 Получено: {bonus_diamonds} алмазов\n"
            f"💎 Всего алмазов: {new_diamonds}"
        )
        if mask_bonus_text:
            bonus_text += "\n" + mask_bonus_text

        await update.message.reply_text(bonus_text, parse_mode="HTML")

    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка при получении бонуса: {str(e)}")
        print(f"ERROR in daily_bonus_command: {str(e)}")

# Словарь для блокировки активного фарма
active_farm_requests = {}
confirmed_farms = {}
ANTI_SPAM_DELAY = 5  # защита от повторных кликов

DB_PATH = "diamonds_bot.db"

import random
import sqlite3
import asyncio
from datetime import datetime, timezone
from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import ContextTypes

# === Глобальные переменные ===
active_farm_requests = {}   # кто сейчас фармит
confirmed_farms = {}        # антиспам на подтверждения
ANTI_SPAM_DELAY = 5
DB_PATH = "diamonds_bot.db"


# === Вспомогательная функция ===
async def _remove_farm_lock_later(user_id: int, delay: int = 60):
    """Снимает блокировку фарма через указанное время."""
    await asyncio.sleep(delay)
    active_farm_requests.pop(user_id, None)


# === Команда /фарм ===
async def farm_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_id = user.id

        # Получаем данные из базы
        user_data = get_user(user_id, user.username, user.first_name)
        if not user_data or len(user_data) < 18:
            await update.message.reply_text("❌ Ошибка: пользователь не найден.")
            return

        diamonds = user_data[4]
        points = user_data[5]
        last_diamond_time = user_data[6]
        upgrade_level = user_data[7]

        cooldown_minutes = 34
        current_time = datetime.now(timezone.utc)

        # Проверяем кулдаун
        if last_diamond_time:
            try:
                last_update = datetime.fromisoformat(last_diamond_time.replace('Z', '+00:00')).replace(tzinfo=timezone.utc)
                seconds_passed = (current_time - last_update).total_seconds()
                if seconds_passed < cooldown_minutes * 60:
                    minutes, seconds = divmod(int(cooldown_minutes * 60 - seconds_passed), 60)
                    await update.message.reply_text(f"❌ Кулдаун: {minutes} мин {seconds} сек")
                    return
            except Exception:
                pass

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        cursor.execute(
            "SELECT active_farm, farm_started_at FROM users WHERE user_id = ?",
            (user_id,)
        )
        row = cursor.fetchone()

        if row and row[0] == 1:
            started_at = row[1]
            if started_at:
                try:
                    started_time = datetime.fromisoformat(started_at)
                    if (current_time - started_time).total_seconds() >= 34 * 60:
                        # прошло 34 минуты — снимаем блок
                        cursor.execute(
                            "UPDATE users SET active_farm = 0 WHERE user_id = ?",
                            (user_id,)
                        )
                        conn.commit()
                    else:
                        await update.message.reply_text(
                            "⚠️ Вы уже начали фарм! Подтвердите действие кнопкой, чтобы продолжить."
                        )
                        conn.close()
                        return
                except Exception:
                    # если время битое — тоже снимаем блок
                    cursor.execute(
                        "UPDATE users SET active_farm = 0 WHERE user_id = ?",
                        (user_id,)
                    )
                    conn.commit()

        cursor.execute("""
        UPDATE users
        SET active_farm = 1, farm_started_at = ?
        WHERE user_id = ?
        """, (datetime.now(timezone.utc).isoformat(), user_id))
        conn.commit()
        conn.close()

        # Генерация наград
        upgrade_info = get_upgrade_info(upgrade_level)
        base_diamonds = random.randint(upgrade_info["min_diamonds"], upgrade_info["max_diamonds"])

        rarities = [
            {"name": "Обычный",     "emoji": "⚪",  "multiplier": 1.00, "weight": 62.0},
            {"name": "Редкий",      "emoji": "🟢",  "multiplier": 1.35, "weight": 16.5},
            {"name": "Эпический",   "emoji": "🟣",  "multiplier": 1.80, "weight": 9.5},
            {"name": "Лазурный",    "emoji": "🔵",  "multiplier": 2.40, "weight": 5.0},
            {"name": "Мифический",  "emoji": "🔴",  "multiplier": 3.50, "weight": 3.5},
            {"name": "Легендарный", "emoji": "🟡",  "multiplier": 5.50, "weight": 2.0},
            {"name": "Божественный","emoji": "✨",  "multiplier": 8.00, "weight": 0.9},   # ← новая
            {"name": "Радужный",    "emoji": "🌈",  "multiplier": 12.0, "weight": 0.25},
        ]
        rarity = random.choices(rarities, weights=[r["weight"] for r in rarities])[0]

        # Сохраняем данные фарма во временное хранилище
        context.user_data['farm_data'] = {
            'user_id': user_id,
            'base_diamonds': base_diamonds,
            'points_earned': 1,
            'coins_earned': 0,
            'strawberries_earned': 0,
            'tomatoes_earned': 0,
            'rarity': rarity,
            'confirmed': False
        }

        # Кнопка подтверждения
        keyboard = [[InlineKeyboardButton("✅ Подтвердить фарм", callback_data=f"confirm_farm_{user_id}")]]
        reply_markup = InlineKeyboardMarkup(keyboard)

        await update.message.reply_text("⏳ Фарм запущен, подтвердите действие!", reply_markup=reply_markup)

        # Фоновая разблокировка через 60 сек
        asyncio.create_task(_remove_farm_lock_later(user_id))

    except Exception as e:
        active_farm_requests.pop(user_id, None)
        await update.message.reply_text(f"❌ Ошибка: {e}")
        print(f"[FARM_COMMAND ERROR] {e}")


# === Обработка подтверждения фарма ===
async def farm_callback_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        query = update.callback_query
        user = query.from_user
        callback_data = query.data
        user_id = int(callback_data.split('_')[-1])

        if not callback_data.startswith("confirm_farm_"):
            await query.answer("❌ Неверная кнопка!")
            return

        if user.id != user_id:
            await query.answer("❌ Это не ваша кнопка!")
            return

        # Антиспам
        now = datetime.now().timestamp()
        if confirmed_farms.get(user_id) and now - confirmed_farms[user_id] < ANTI_SPAM_DELAY:
            await query.answer("⏳ Подождите немного...")
            return

        # Достаем данные фарма
        farm_data = context.user_data.get('farm_data')
        if not farm_data or farm_data['user_id'] != user_id:
            await query.answer("❌ Данные фарма не найдены!")
            return

        if farm_data.get("confirmed"):
            await query.answer("⚠️ Фарм уже подтверждён!")
            return

        farm_data["confirmed"] = True

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("UPDATE users SET active_farm = 0 WHERE user_id = ?", (user_id,))
        conn.commit()
        conn.close()

        rarity = farm_data['rarity']
        multiplier = rarity['multiplier']

        # Основные награды
        diamonds_earned = int(farm_data['base_diamonds'] * multiplier)
        points_earned = int(farm_data['points_earned'] * multiplier)
        coins_earned = int(farm_data['coins_earned'] * multiplier)
        strawberries_earned = int(farm_data['strawberries_earned'] * multiplier)
        tomatoes_earned = int(farm_data['tomatoes_earned'] * multiplier)

        # Дополнительные ресурсы по редкости
        wood_earned, stone_earned, iron_earned, titanium_earned = 0, 0, 0, 0

        if rarity['name'] == "Редкий":
            wood_earned = random.randint(77750, 97750)

        elif rarity['name'] == "Эпический":
            wood_earned = stone_earned = random.randint(101150, 121150)

        elif rarity['name'] == "Лазурный":
            wood_earned = stone_earned = random.randint(138200, 158200)

        elif rarity['name'] == "Мифический":
            wood_earned = stone_earned = random.randint(206125, 226125)
            iron_earned = random.randint(58250, 78250)

        elif rarity['name'] == "Легендарный":
            wood_earned = stone_earned = random.randint(329625, 349625)
            iron_earned = random.randint(97250, 117250)
            titanium_earned = random.randint(43625, 63625)

        elif rarity['name'] == "Божественный":
            wood_earned = stone_earned = random.randint(484000, 504000)
            iron_earned = random.randint(146000, 166000)
            titanium_earned = random.randint(68000, 88000)

        elif rarity['name'] == "Радужный":
            wood_earned = stone_earned = random.randint(731000, 751000)
            iron_earned = random.randint(224000, 244000)
            titanium_earned = random.randint(107000, 127000)

        # Проверка масок
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("SELECT mask_id FROM user_masks WHERE user_id = ?", (user_id,))
        owned_masks = [int(row[0]) for row in cursor.fetchall()]

        mask_bonus_text = ""

        # ==== СТАРЫЕ МАСКИ ====
        if 1 in owned_masks:
            points_earned += 1
            mask_bonus_text += "🧠 Маска опыта: +1 очко\n"
        if 2 in owned_masks:
            coins_earned = 16000
            update_user(user_id, coins=coins_earned)  # прибавка к балансу
            mask_bonus_text += "💰 Маска старателя: +32000 монет\n"
        if 4 in owned_masks:
            strawberries_earned += 3000
            tomatoes_earned += 90000
            mask_bonus_text += "🍀 Маска фермера: +3000 клубники, +90000 помидоров\n"
        if 5 in owned_masks:
            diamonds_earned += 400
            mask_bonus_text += "💎 Маска шахтёра: +400 алмазов\n"
        if 11 in owned_masks:
            wood_earned += 200000
            stone_earned += 200000
            mask_bonus_text += "🚩 Маска Знаменосца: +200000 дерева, +200000 камня\n"
        if 16 in owned_masks:
            cursor.execute("""
                INSERT INTO user_zombies(user_id, zombie_id, quantity)
                VALUES(?, 19, 2)
                ON CONFLICT(user_id, zombie_id) DO UPDATE SET quantity = quantity + 2
            """, (user_id,))
            mask_bonus_text += "🐌 Маска Слизняка: +2 зомби Слизняка\n"
        if 17 in owned_masks:
            cursor.execute("UPDATE users SET mine_sea = mine_sea + 1 WHERE user_id = ?", (user_id,))
            mask_bonus_text += "💣 Маска Подрывателя: +2 морские мины\n"
        if 18 in owned_masks:
            if random.random() < 0.2:
                cursor.execute("""
                    INSERT INTO user_zombies(user_id, zombie_id, quantity)
                    VALUES(?, 12, 1)
                    ON CONFLICT(user_id, zombie_id) DO UPDATE SET quantity = quantity + 1
                """, (user_id,))
                mask_bonus_text += "🐰 Маска Зайца: +1 зомби Зайка\n"

        if 19 in owned_masks:
            cursor.execute("UPDATE users SET turret_wood = turret_wood + 2 WHERE user_id = ?", (user_id,))
            mask_bonus_text += "🏹 Маска Арбалетчика: +2 деревянных турели\n"

        conn.commit()
        conn.close()

        # Обновляем данные игрока
        user_data = get_user(user_id, user.username, user.first_name)
        if not user_data or len(user_data) < 23:
            await query.answer("❌ Ошибка: пользователь не найден.")
            return

        user_dict = {
            "diamonds": user_data[4],
            "points": user_data[5],
            "strawberries": user_data[9],
            "tomatoes": user_data[15],
            "coins": user_data[16],
        }

        new_diamonds = user_dict["diamonds"] + diamonds_earned
        new_points = user_dict["points"] + points_earned
        new_coins = user_dict["coins"] + coins_earned


        new_strawberries = user_dict["strawberries"] + strawberries_earned
        new_tomatoes = user_dict["tomatoes"] + tomatoes_earned

        update_user(
            user_id,
            diamonds=new_diamonds,
            points=new_points,
            last_diamond_time=datetime.now(timezone.utc).isoformat(),
            strawberries=new_strawberries,
            tomatoes=new_tomatoes,
            coins=coins_earned,
            wood=wood_earned,
            stone=stone_earned,
            iron=iron_earned,
            titanium=titanium_earned
        )

        # Получаем ранг

        # ===== РАНГ (СТАБИЛЬНО) =====
        rank_data = get_rank(new_points)

        rank = rank_data[0]

        if len(rank_data) > 1:
            next_rank_points = rank_data[1]
        else:
            next_rank_points = float('inf')

        points_text = (
            f"{new_points} / {next_rank_points}"
            if next_rank_points != float('inf')
            else f"{new_points} / ∞"
        )

        # Формируем сообщение
        farm_text = (
            f"{rarity['emoji']} Фарм успешен ({rarity['name']})!\n"
            f"🔢 Множитель: x{multiplier}\n"
            f"💎 Алмазы: +{diamonds_earned} (Всего: {new_diamonds})\n"
            f"❄️ Очки: (Всего: {points_text})\n"
            f"🎖️ Ранг: {rank}\n"
            f"🍓 Клубники: (Всего: {new_strawberries})\n"
            f"🍅 Помидоров: (Всего: {new_tomatoes})\n"
        )

        if wood_earned: farm_text += f"🪵 Дерево: +{wood_earned}\n"
        if stone_earned: farm_text += f"🪨 Камень: +{stone_earned}\n"
        if iron_earned: farm_text += f"⛓️ Железо: +{iron_earned}\n"
        if titanium_earned: farm_text += f"⚙️ Титан: +{titanium_earned}\n"

        if mask_bonus_text:
            farm_text += "\n🔥 Бонусы масок:\n" + mask_bonus_text

        await query.edit_message_reply_markup(reply_markup=None)
        await query.edit_message_text(farm_text)
        await query.answer("Фарм подтверждён!")

    except Exception as e:
        active_farm_requests.pop(user_id, None)
        await query.answer(f"❌ Ошибка при подтверждении фарма: {str(e)}")
        print(f"[FARM_CALLBACK ERROR] {e}")

import sqlite3
import asyncio
from telegram import Update
from telegram.ext import ContextTypes

DB_PATH = "diamonds_bot.db"

# Словарь промокодов (сложные коды)
PROMOCODES = {
    # Монеты
    "B7X-Q9L-P4K": {"coins": 10000},
    "V2M-R8F-J3T": {"coins": 25000},
    "Q5T-Z4K-L8P": {"coins": 50000},
    "L9N-P3X-V7R": {"coins": 75000},
    "R1F-M7V-K2D": {"coins": 100000},

    # Алмазы
    "D3K-W5L-M9Q": {"diamonds": 2500},
    "J6M-Q2F-P8V": {"diamonds": 5000},
    "S8N-L9V-K3X": {"diamonds": 7500},
    "A4P-R3K-V1M": {"diamonds": 10000},

    # Железо
    "F7G-T5X-L2R": {"iron": 10000},
    "H2J-Q8M-V5P": {"iron": 25000},
    "L3K-V6R-K9F": {"iron": 50000},
    "N4P-M2F-P7Q": {"iron": 75000},
    "R5S-L9Z-M8T": {"iron": 100000},

    # Титан
    "T1A-K7M-V3X": {"titanium": 5000},
    "U2B-V8F-K5L": {"titanium": 10000},
    "V3C-L5X-P9R": {"titanium": 25000},
    "W4D-R6Q-M2F": {"titanium": 50000},
    "X5E-M2Z-V7K": {"titanium": 75000},

    # Турели
    "W3D-P9R-K1X": {"turrets": {"turret_wood": 1}},
    "S7K-M4T-V2J": {"turrets": {"turret_stone": 1}},
    "I2N-V6Q-M8P": {"turrets": {"turret_iron": 1}},
    "T9A-Z8L-K4F": {"turrets": {"turret_titanium": 1}},
    "R5C-J7X-V3M": {"turrets": {"turret_rocket": 1}},

    # Помидоры
    "T8M-Q2F-J6R": {"tomatoes": 10000},
    "P3X-V9L-K2T": {"tomatoes": 25000},
    "L7R-K5Z-M9V": {"tomatoes": 50000},
    "N1C-M4J-V7K": {"tomatoes": 75000},
    "B6S-W8T-K3L": {"tomatoes": 100000},

    # Клубника
    "S9F-R3L-V2P": {"strawberries": 10000},
    "K2V-M6X-K9J": {"strawberries": 25000},
    "D7Q-P1J-M3V": {"strawberries": 50000},
    "X5T-Z4F-V8K": {"strawberries": 75000},
    "A8N-V2K-K7M": {"strawberries": 100000},

    # Зомби
    "Z0H-B7M-K2X": {"zombie": 6},   # здоровяк
    "K4P-X9R-V5J": {"zombie": 11},  # Крампус
    "Q2L-V5J-M3T": {"zombie": 10},  # Горбатый
    "W7C-M3T-P8K": {"zombie": 12},  # Зайка
}

# Асинхронный executor для работы с SQLite
async def run_db(query_func):
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(None, query_func)

async def redeem_promocode(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user_id = update.effective_user.id
        args = context.args
        if not args:
            await update.message.reply_text("❌ Укажите промокод после команды, например: /promo PYXIS")
            return
        code = args[0].upper()

        if code not in PROMOCODES:
            await update.message.reply_text("❌ Неверный промокод.")
            return

        # Работа с базой в отдельном потоке
        def db_task():
            conn = sqlite3.connect(DB_PATH, timeout=10)
            try:
                conn.execute("PRAGMA journal_mode=WAL;")
                cursor = conn.cursor()

                # Таблица использованных промокодов
                cursor.execute("CREATE TABLE IF NOT EXISTS used_promocodes (user_id INTEGER, code TEXT)")
                cursor.execute("SELECT 1 FROM used_promocodes WHERE user_id=? AND code=?", (user_id, code))
                if cursor.fetchone():
                    return None  # Уже использован

                reward = PROMOCODES[code]

                # Монеты, алмазы, железо, титан
                cursor.execute("""
                    UPDATE users SET
                        coins = coins + ?,
                        diamonds = diamonds + ?,
                        iron = iron + ?,
                        titanium = titanium + ?
                    WHERE user_id = ?
                """, (reward.get("coins", 0), reward.get("diamonds", 0),
                      reward.get("iron", 0), reward.get("titanium", 0), user_id))

                # Турели
                if "turrets" in reward:
                    for turret_col, amount in reward["turrets"].items():
                        cursor.execute(f"UPDATE users SET {turret_col} = {turret_col} + ? WHERE user_id = ?", (amount, user_id))

                # Урожай
                for crop in ["tomatoes", "strawberries"]:
                    if crop in reward:
                        cursor.execute(f"UPDATE users SET {crop} = {crop} + ? WHERE user_id = ?", (reward[crop], user_id))

                # Зомби через таблицу user_zombies
                if "zombie" in reward:
                    z_id = reward["zombie"]
                    cursor.execute("""
                        INSERT INTO user_zombies(user_id, zombie_id, quantity)
                        VALUES(?, ?, 1)
                        ON CONFLICT(user_id, zombie_id)
                        DO UPDATE SET quantity = quantity + 1
                    """, (user_id, z_id))

                # Помечаем код как использованный
                cursor.execute("INSERT INTO used_promocodes (user_id, code) VALUES (?, ?)", (user_id, code))
                conn.commit()
                return reward
            finally:
                conn.close()

        reward = await run_db(db_task)

        if not reward:
            await update.message.reply_text("❌ Вы уже использовали этот промокод.")
            return

        # Формируем текст награды
        reward_text = []
        if reward.get("coins"): reward_text.append(f"💰 {reward['coins']} монет")
        if reward.get("diamonds"): reward_text.append(f"💎 {reward['diamonds']} алмазов")
        if reward.get("iron"): reward_text.append(f"⛓ {reward['iron']} железа")
        if reward.get("titanium"): reward_text.append(f"⚙ {reward['titanium']} титана")
        if "turrets" in reward:
            for t, a in reward["turrets"].items():
                reward_text.append(f"🏗 {a} {t.replace('turret_', '').capitalize()} турель")
        for crop in ["tomatoes", "strawberries"]:
            if crop in reward:
                reward_text.append(f"✅ {reward[crop]} {crop}")
        if "zombie" in reward:
            z_id = reward["zombie"]
            z_name = ZOMBIE_INFO[z_id]["name"]
            reward_text.append(f"🧟 1 зомби: {z_name}")

        await update.message.reply_text(f"✅ Промокод активирован! Вы получили:\n" + "\n".join(reward_text))

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при активации промокода: {e}")
        print("ERROR in redeem_promocode:", e)

import sqlite3
from datetime import datetime, timezone
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ContextTypes

DB_PATH = "diamonds_bot.db"

async def farm_strawberries_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_data = get_user(user.id, user.username, user.first_name)

        if not user_data or len(user_data) < 25:
            await update.message.reply_text("❌ Пользователь не найден или данные повреждены.")
            return
        (
            user_id, player_id, username, first_name, diamonds, points, last_diamond_time, upgrade_level,
            registration_date, strawberries, farm_level, last_zombie_attack, zombie_status, last_strawberry_update,
            last_bonus_time, tomatoes, grapes, rubies, coins, farms_destroyed, wood, stone, iron, titanium, clan_id
        ) = user_data

        # --- берём здоровье и баррикады из users ---
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("""
            SELECT new_farm_health, max_farm_health,
                   COALESCE(wooden_barricade,0), COALESCE(concrete_barricade,0), COALESCE(steel_barricade,0),
                   COALESCE(titanium_barricade,0), COALESCE(diamond_barricade,0),
                   COALESCE(wooden_palisade,0), COALESCE(iron_palisade,0),
                   COALESCE(citadel_barricade,0)
            FROM users WHERE user_id = ?
        """, (user_id,))

        row = cursor.fetchone()
        conn.close()

        if row:
            (
                new_farm_health, max_farm_health,
                wooden_barricade, concrete_barricade, steel_barricade,
                titanium_barricade, diamond_barricade,
                wooden_palisade, iron_palisade,
                citadel_barricade
            ) = row
        else:
            new_farm_health = max_farm_health = 100
            wooden_barricade = concrete_barricade = steel_barricade = 0
            titanium_barricade = diamond_barricade = wooden_palisade = iron_palisade = 0

        # --- формируем текст о баррикадах ---
        barricade_lines = [f"❤️ Здоровье фермы: {new_farm_health} / {max_farm_health}", "🧱 Баррикады фермы:"]
        if wooden_barricade > 0:
            barricade_lines.append(f"🪵 Деревянные: {wooden_barricade}")
        if concrete_barricade > 0:
            barricade_lines.append(f"🪨 Каменные: {concrete_barricade}")
        if steel_barricade > 0:
            barricade_lines.append(f"⛓ Железные: {steel_barricade}")
        if titanium_barricade > 0:
            barricade_lines.append(f"🔩 Титановые: {titanium_barricade}")
        if diamond_barricade > 0:
            barricade_lines.append(f"💎 Алмазные: {diamond_barricade}")
        if wooden_palisade > 0:
            barricade_lines.append(f"🪵 Колья деревянные: {wooden_palisade}")
        if iron_palisade > 0:
            barricade_lines.append(f"⛓ Колья железные: {iron_palisade}")
        if citadel_barricade > 0:
            barricade_lines.append(f"🏰 Баррикады цитадели: {citadel_barricade}")

        if len(barricade_lines) == 2:
            barricade_lines.append("❌ Баррикады отсутствуют")

        barricades_text = "\n".join(barricade_lines)

        # --- берём башни пользователя ---
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("""
            SELECT COALESCE(tower_fire,0), COALESCE(tower_poison,0), COALESCE(tower_magic,0),
                   COALESCE(tower_crossbow,0), COALESCE(tower_curse,0)
            FROM users WHERE user_id = ?
        """, (user_id,))
        tower_row = cursor.fetchone()
        conn.close()

        tower_lines = ["🏰 Башни защиты:"]
        tower_names = ["🔥 Огненная", "☠️ Ядовитая", "🔮 Магическая", "🏹 Арбалетная", "🕯️ Проклинающая"]

        if tower_row and any(tower_row):
            for t_name, t_count in zip(tower_names, tower_row):
                if t_count > 0:
                    tower_lines.append(f"🔹 {t_name} башня: {t_count}")
        else:
            tower_lines.append("❌ Башни отсутствуют")

        towers_text = "\n".join(tower_lines)

        # --- берём мины пользователя ---
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("""
            SELECT COALESCE(mine_trap,0), COALESCE(mine_iron,0), COALESCE(mine_sea,0)
            FROM users WHERE user_id = ?
        """, (user_id,))
        mine_row = cursor.fetchone()
        conn.close()

        mine_lines = ["💣 Мины для защиты фермы:"]
        if mine_row and any(mine_row):
            trap, iron_mine, sea_mine = mine_row
            if trap > 0:
                mine_lines.append(f"🪤 Капкан: {trap}")
            if iron_mine > 0:
                mine_lines.append(f"⛓️ Мина: {iron_mine}")
            if sea_mine > 0:
                mine_lines.append(f"🔩 Морская мина: {sea_mine}")
        else:
            mine_lines.append("❌ Мины отсутствуют")

        mines_text = "\n".join(mine_lines)

        # --- берём турели пользователя ---
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("""
            SELECT COALESCE(turret_wood,0), COALESCE(turret_stone,0), COALESCE(turret_iron,0),
                   COALESCE(turret_titanium,0), COALESCE(turret_rocket,0)
            FROM users WHERE user_id = ?
        """, (user_id,))
        turret_row = cursor.fetchone()
        conn.close()

        turret_lines = ["🛡 Турели на ферме:"]
        turret_types = ["Деревянные", "Каменные", "Железные", "Титановые", "Ракетные"]
        if turret_row and any(turret_row):
            for t_name, t_count in zip(turret_types, turret_row):
                if t_count > 0:
                    turret_lines.append(f"🔹 {t_name}: {t_count}")
        else:
            turret_lines.append("❌ Турели отсутствуют")

        turrets_text = "\n".join(turret_lines)

        # --- Проверка команды ---
        message_text = update.message.text.strip().lower().split()
        if len(message_text) != 2 or message_text != ["тгз", "ферма"]:
            await update.message.reply_text("❌ Использование: тгз ферма")
            return

        farm_info = get_farm_info(farm_level)
        if not farm_info:
            await update.message.reply_text(f"❌ Некорректный уровень фермы: {farm_level}")
            return

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        cursor.execute("SELECT harvest_blocked_until FROM users WHERE user_id = ?", (user_id,))
        hb_row = cursor.fetchone()
        if hb_row and hb_row[0] > int(time.time()):
            await update.message.reply_text("⛔ Сбор урожая запрещён Королевой обезьян на 1 час!")
            conn.close()
            return

        cursor.execute("""
            SELECT farm_income_curse_active, farm_income_curse_mult
            FROM users WHERE user_id = ?
        """, (user_id,))
        curse_row = cursor.fetchone()
        income_mult = 1.0
        cursed = False

        if curse_row and curse_row[0] == 1:
            cursed = True
            income_mult = curse_row[1]
            cursor.execute("""
                UPDATE users
                SET farm_income_curse_active = 0,
                    farm_income_curse_mult = 1.0
                WHERE user_id = ?
            """, (user_id,))
            conn.commit()

        conn.close()

        STRAWBERRY_THRESHOLDS = [
            (100, 1.5), (1000, 2.0), (5000, 3.0), (15000, 4.0),
            (30000, 5.0), (60000, 6.0), (100000, 7.0), (160000, 8.0),
            (240000, 9.0), (350000, 10.0), (500000, 12.0), (700000, 14.0),
            (950000, 16.0), (1300000, 18.0), (1800000, 20.0), (2300000, 22.0),
            (3000000, 24.0), (4000000, 26.0),  # до x26
            (4600000, 27.0), (5300000, 28.0), (6100000, 29.0), (7000000, 30.0),
            (8000000, 31.0), (9200000, 32.0), (10600000, 33.0), (12200000, 34.0),
            (14000000, 35.0)  # финальный множитель
        ]

        TOMATO_THRESHOLDS = [
            (3000, 1.5), (30000, 2.0), (150000, 3.0), (450000, 4.0),
            (900000, 5.0), (1800000, 6.0), (3000000, 7.0), (4800000, 8.0),
            (7200000, 9.0), (10500000, 10.0), (15000000, 12.0), (21000000, 14.0),
            (28500000, 16.0), (39000000, 18.0), (54000000, 20.0), (69000000, 22.0),
            (90000000, 24.0), (120000000, 26.0),
            (138000000, 27.0), (159000000, 28.0), (183000000, 29.0), (210000000, 30.0),
            (240000000, 31.0), (276000000, 32.0), (318000000, 33.0), (366000000, 34.0),
            (420000000, 35.0)
        ]

        GRAPE_THRESHOLDS = [
            (10, 1.5), (100, 2.0), (500, 3.0), (1500, 4.0),
            (3000, 5.0), (6000, 6.0), (10000, 7.0), (16000, 8.0),
            (24000, 9.0), (35000, 10.0), (50000, 12.0), (70000, 14.0),
            (95000, 16.0), (130000, 18.0), (180000, 20.0), (230000, 22.0),
            (300000, 24.0), (400000, 26.0),
            (460000, 27.0), (530000, 28.0), (610000, 29.0), (700000, 30.0),
            (800000, 31.0), (920000, 32.0), (1060000, 33.0), (1220000, 34.0),
            (1400000, 35.0)
        ]

        def get_multiplier(value, thresholds):
            multiplier = 1.0
            for threshold, m in thresholds:
                if value >= threshold:
                    multiplier = m
                else:
                    break
            return multiplier

        strawberry_multiplier = get_multiplier(strawberries, STRAWBERRY_THRESHOLDS)
        tomato_multiplier = get_multiplier(tomatoes, TOMATO_THRESHOLDS)
        grape_multiplier = get_multiplier(grapes, GRAPE_THRESHOLDS)

        def get_multiplier(value, thresholds):
            multiplier = 1.0
            for threshold, m in thresholds:
                if value >= threshold:
                    multiplier = m
                else:
                    break
            return multiplier

        strawberry_multiplier = get_multiplier(strawberries, STRAWBERRY_THRESHOLDS)
        tomato_multiplier = get_multiplier(tomatoes, TOMATO_THRESHOLDS)

        strawberry_text = f"🔥 {strawberry_multiplier}x" if strawberry_multiplier > 1 else "1x"
        tomato_text = f"🔥 {tomato_multiplier}x" if tomato_multiplier > 1 else "1x"
        grape_text = f"🔥 {grape_multiplier}x" if grape_multiplier > 1 else "1x"

        max_boost_text = "\n🔥🔥 <b>МАКСИМАЛЬНЫЙ БУСТ!</b> 🔥🔥" \
            if strawberry_multiplier == 35.0 and tomato_multiplier == 35.0 else ""
        curse_text = "\n⚠️ Доход уменьшен в 2 раза (проклятие Короля обезьян)!" if cursed else ""

        # --- расчёт времени и урожая ---
        current_time = datetime.now(timezone.utc)
        hours_passed = 0
        if last_strawberry_update:
            try:
                last_update = datetime.fromisoformat(last_strawberry_update.replace('Z', '+00:00'))
                if last_update.tzinfo is None:
                    last_update = last_update.replace(tzinfo=timezone.utc)
                hours_passed = (current_time - last_update).total_seconds() / 3600
            except (ValueError, TypeError):
                hours_passed = 0

        effective_strawberries = int(farm_info["strawberries_per_hour"] * strawberry_multiplier * income_mult * hours_passed)
        effective_tomatoes     = int(farm_info["tomatoes_per_hour"]     * tomato_multiplier     * income_mult * hours_passed)
        effective_grapes       = int(farm_info["grapes_per_hour"]       * grape_multiplier       * income_mult * hours_passed)

        new_strawberries = strawberries + effective_strawberries
        new_grapes = grapes + effective_grapes
        new_tomatoes = tomatoes + effective_tomatoes

        # --- проверка на атаку зомби ---
        zombie_message = ""
        buttons = []
        if zombie_status:
            new_strawberries = max(0, new_strawberries - 50)
            new_tomatoes = max(0, new_tomatoes - 150)
            zombie_message = "\n🧟 Зомби атаковали вашу ферму! Потеряно 50 🍓 и 150 🍅."
            buttons.append([
                InlineKeyboardButton("🛡️ Защитить ферму", callback_data=f"defend_zombies_{user_id}")
            ])

        # --- обновляем данные пользователя ---
        update_user(
            user_id,
            strawberries=new_strawberries,
            tomatoes=new_tomatoes,
            grapes=new_grapes,
            last_strawberry_update=current_time.isoformat()
        )

        farm_text = (
            f"✅ Урожай собран!\n"
            f"{barricades_text}\n"
            f"{towers_text}\n"      # <-- вот сюда вставляем башни
            f"{turrets_text}\n"
            f"{mines_text}\n\n"
            f"🍓 Получено: {effective_strawberries} (множитель: {strawberry_text})\n"
            f"🍓 Всего: {new_strawberries}\n"
            f"🍅 Получено: {effective_tomatoes} (множитель: {tomato_text})\n"
            f"🍅 Всего: {new_tomatoes}\n"
            f"🍇 Получено: {effective_grapes} (множитель: {grape_text})\n"
            f"🍇 Всего: {new_grapes}\n"
            f"🚜 Уровень фермы: {farm_level}\n"
            f"🍓 Доход: {farm_info['strawberries_per_hour']}/ч → {int(farm_info['strawberries_per_hour'] * strawberry_multiplier)}/ч\n"
            f"🍅 Доход: {farm_info['tomatoes_per_hour']}/ч → {int(farm_info['tomatoes_per_hour'] * tomato_multiplier)}/ч\n"
            f"🍇 Доход: {farm_info['grapes_per_hour']}/ч → {int(farm_info['grapes_per_hour'] * grape_multiplier)}/ч\n"
            f"{max_boost_text}"
            f"{curse_text}\n"
            f"{zombie_message}"
        )
        reply_markup = InlineKeyboardMarkup(buttons) if buttons else None
        await update.message.reply_text(farm_text, reply_markup=reply_markup, parse_mode="HTML")

    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка при просмотре фермы: {str(e)}")
        print(f"ERROR in farm_strawberries_command: {str(e)}")

# Функция farm_upgrade_info_command
async def farm_upgrade_info_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_data = get_user(user.id, user.username, user.first_name)

        if user_data is None or len(user_data) < 25:
            print(f"ERROR: user_data has {len(user_data) if user_data else 0} values, expected 23. User ID: {user.id}")
            await update.message.reply_text("❌ Ошибка: пользователь не найден. Обратитесь к администратору.")
            return

        (
            user_id, player_id, username, first_name, diamonds, points,
            last_diamond_time, upgrade_level, registration_date, strawberries,
            farm_level, last_zombie_attack, zombie_status, last_strawberry_update,
            last_bonus_time, tomatoes, coins, farms_destroyed, clan_id,
            *_
        ) = user_data

        message_text = update.message.text.strip().lower().split()
        if len(message_text) != 3 or message_text[:3] != ["тгз", "прокачка", "фермы"]:
            await update.message.reply_text("❌ Использование: тгз прокачка фермы")
            return

        # 🔹 Вот эти две строки добавь ВМЕСТО старого вывода info_text
        page = 0
        await send_farm_info_page(update, context, farm_level, coins, page)

    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка при просмотре прокачки фермы: {str(e)}")
        print(f"ERROR in farm_upgrade_info_command: {str(e)}")

LEVELS_PER_PAGE = 5

async def send_farm_info_page(update, context, farm_level, coins, page):
    total_levels = 21
    total_pages = (total_levels + LEVELS_PER_PAGE - 1) // LEVELS_PER_PAGE
    start_level = page * LEVELS_PER_PAGE
    end_level = min(start_level + LEVELS_PER_PAGE, total_levels)

    info_text = "🚜 Уровни прокачки фермы:\n\n"
    for level in range(start_level, end_level):
        farm_info = get_farm_info(level)
        if not farm_info:
            continue
        is_current = "✅ Текущий уровень" if level == farm_level else ""
        info_text += (
            f"Уровень {level}:\n"
            f"💰 Стоимость: {farm_info['cost']} монет\n"
            f"🍓 Доход: {farm_info['strawberries_per_hour']} клубники/час\n"
            f"🍅 Доход: {farm_info['tomatoes_per_hour']} помидоров/час\n"
            f"🍇 Доход: {farm_info.get('grapes_per_hour', 0)} винограда/час\n"
            f"{is_current}\n\n"
        )

    info_text += "📋 Используйте: тгз улучшить ферму [уровень]"

    buttons = []
    if page > 0:
        buttons.append(InlineKeyboardButton("⬅ Назад", callback_data=f"farm_page_{page - 1}"))
    if page < total_pages - 1:
        buttons.append(InlineKeyboardButton("Вперёд ➡", callback_data=f"farm_page_{page + 1}"))

    keyboard = InlineKeyboardMarkup([buttons]) if buttons else None

    if update.callback_query:
        await update.callback_query.edit_message_text(
            info_text, reply_markup=keyboard, parse_mode="HTML"
        )
    else:
        await update.message.reply_text(info_text, reply_markup=keyboard, parse_mode="HTML")

from telegram import InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import CallbackQueryHandler

async def farm_page_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    user = query.from_user
    user_data = get_user(user.id, user.username, user.first_name)
    if not user_data or len(user_data) < 24:
        await query.edit_message_text("❌ Ошибка загрузки данных пользователя.")
        return

    (
        user_id, player_id, username, first_name, diamonds, points,
        last_diamond_time, upgrade_level, registration_date, strawberries,
        farm_level, last_zombie_attack, zombie_status, last_strawberry_update,
        last_bonus_time, tomatoes, coins, farms_destroyed, clan_id,
        *_
    ) = user_data

    page = int(query.data.split("_")[-1])
    await send_farm_info_page(update, context, farm_level, coins, page)

async def upgrade_farm_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_data = get_user(user.id, user.username, user.first_name)

        if not user_data or len(user_data) < 24:
            await update.message.reply_text("❌ Ошибка: пользователь не найден. Обратитесь к администратору.")
            return

        (
            user_id, player_id, username, first_name, diamonds, points,
            last_diamond_time, upgrade_level, registration_date, strawberries,
            farm_level, last_zombie_attack, zombie_status, last_strawberry_update,
            last_bonus_time, tomatoes, coins, farms_destroyed, clan_id,
            *_
        ) = user_data

        message_text = update.message.text.strip().lower().split()
        if len(message_text) != 4 or message_text[:3] != ["тгз", "улучшить", "ферму"]:
            await update.message.reply_text("❌ Использование: тгз улучшить ферму [уровень]")
            return

        try:
            target_level = int(message_text[3])
        except (ValueError, IndexError):
            await update.message.reply_text("❌ Уровень должен быть числом!")
            return

        if not (0 <= target_level <= 20):
            await update.message.reply_text("❌ Уровень фермы должен быть от 0 до 20!")
            return

        if target_level <= farm_level:
            await update.message.reply_text(f"❌ Ваш текущий уровень фермы ({farm_level}) уже равен или выше запрошенного ({target_level})!")
            return

        if target_level != farm_level + 1:
            await update.message.reply_text(f"❌ Можно улучшить только до следующего уровня — {farm_level + 1}!")
            return

        farm_info = get_farm_info(target_level)
        if not farm_info:
            await update.message.reply_text(f"❌ Некорректный уровень фермы: {target_level}")
            return

        cost = farm_info["cost"]

        # === АТОМАРНАЯ ОПЕРАЦИЯ: проверяем и списываем монеты в одной SQL-команде ===
        conn = sqlite3.connect(DB_PATH, timeout=10)
        cursor = conn.cursor()
        # Попытка списать: уменьшить coins и установить уровень фермы, только если монет >= cost
        cursor.execute("""
            UPDATE users
            SET coins = coins - ?, farm_level = ?
            WHERE user_id = ? AND coins >= ?
        """, (cost, target_level, user_id, cost))
        conn.commit()

        if cursor.rowcount == 0:
            # не хватило монет — прочитаем актуальный баланс и сообщим
            cursor.execute("SELECT coins FROM users WHERE user_id = ?", (user_id,))
            row = cursor.fetchone()
            conn.close()
            current_coins = row[0] if row else 0
            await update.message.reply_text(f"❌ Недостаточно монет! Нужно: {cost:,}. У вас: {current_coins:,}".replace(",", " "))
            return

        # успех — перечитаем баланс для вывода
        cursor.execute("SELECT coins FROM users WHERE user_id = ?", (user_id,))
        new_coins = cursor.fetchone()[0]
        conn.close()

        await update.message.reply_text(
            f"✅ Ферма улучшена до уровня {target_level}!\n"
            f"💰 Потрачено монет: {cost:,}\n"
            f"💰 Ваш баланс: {new_coins:,}".replace(",", " ")
        )
        print(f"User {user_id} upgraded farm to level {target_level} for {cost} coins")

    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка при улучшении фермы: {e}")
        print(f"ERROR in upgrade_farm_command: {e}")

async def exchange_strawberries_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_data = get_user(user.id, user.username, user.first_name)

        if user_data is None or len(user_data) < 25:
            print(f"ERROR: user_data has {len(user_data) if user_data else 0} values, expected 23. User ID: {user.id}")
            await update.message.reply_text("❌ Ошибка: пользователь не найден. Обратитесь к администратору.")
            return

        (
            user_id, player_id, username, first_name, diamonds, points,
            last_diamond_time, upgrade_level, registration_date, strawberries,
            farm_level, last_zombie_attack, zombie_status, last_strawberry_update,
            last_bonus_time, tomatoes, coins, farms_destroyed, clan_id,
            *_
        ) = user_data

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        cursor.execute("SELECT harvest_blocked_until FROM users WHERE user_id = ?", (user_id,))
        hb_row = cursor.fetchone()
        if hb_row and hb_row[0] > int(time.time()):
            await update.message.reply_text(
                "🐒👑 Эффект Королевы обезьян активен!\n"
                "❌ Обмен ресурсов временно запрещён."
            )
            conn.close()
            return

        message_text = update.message.text.strip().lower().split()
        if len(message_text) != 4 or message_text[:3] != ["тгз", "обменять", "клубнику"]:
            await update.message.reply_text("❌ Использование: тгз обменять клубнику [количество]")
            return

        try:
            amount = int(message_text[3])
            if amount <= 0:
                await update.message.reply_text("❌ Количество должно быть положительным числом!")
                return
        except (ValueError, IndexError):
            await update.message.reply_text("❌ Количество должно быть числом!")
            return

        if amount > strawberries:
            await update.message.reply_text(f"❌ Недостаточно клубники! У вас: {strawberries}")
            return

        if amount % 10 != 0:  # Клубника должна быть кратна 10
            await update.message.reply_text("❌ Количество клубники должно быть кратно 10!")
            return

        diamonds_earned = amount // 10  # 💎 10 клубники = 1 алмаз
        new_strawberries = strawberries - amount
        new_diamonds = diamonds + diamonds_earned

        update_user(user_id, strawberries=new_strawberries, diamonds=new_diamonds)

        await update.message.reply_text(
            f"✅ Обмен успешен!\n"
            f"🍓 Потрачено: {amount} клубники\n"
            f"💎 Получено: {diamonds_earned} алмазов\n"
            f"🍓 Остаток клубники: {new_strawberries}\n"
            f"💎 Всего алмазов: {new_diamonds}"
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка при обмене клубники: {str(e)}")
        print(f"ERROR in exchange_strawberries_command: {str(e)}")

async def exchange_tomatoes_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_data = get_user(user.id, user.username, user.first_name)

        if not user_data or len(user_data) < 25:
            print(f"ERROR: user_data has {len(user_data) if user_data else 0} values, expected 23. User ID: {user.id}")
            await update.message.reply_text("❌ Ошибка: пользователь не найден. Обратитесь к администратору.")
            return

        (
            user_id, player_id, username, first_name, diamonds, points,
            last_diamond_time, upgrade_level, registration_date, strawberries,
            farm_level, last_zombie_attack, zombie_status, last_strawberry_update,
            last_bonus_time, tomatoes, grapes, rubies, coins,
            farms_destroyed, clan_id, *_
        ) = user_data

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        cursor.execute("SELECT harvest_blocked_until FROM users WHERE user_id = ?", (user_id,))
        hb_row = cursor.fetchone()
        if hb_row and hb_row[0] > int(time.time()):
            await update.message.reply_text(
                "🐒👑 Эффект Королевы обезьян активен!\n"
                "❌ Обмен ресурсов временно запрещён."
            )
            conn.close()
            return

        message_text = update.message.text.strip().lower().split()
        if len(message_text) != 4 or message_text[:3] != ["тгз", "обменять", "помидоры"]:
            await update.message.reply_text("❌ Использование: тгз обменять помидоры [количество]")
            return

        try:
            amount = int(message_text[3])
            if amount <= 0:
                await update.message.reply_text("❌ Количество должно быть положительным числом!")
                return
        except (ValueError, IndexError):
            await update.message.reply_text("❌ Количество должно быть числом!")
            return

        if amount > tomatoes:
            await update.message.reply_text(f"❌ Недостаточно помидоров! У вас: {tomatoes}")
            return

        if amount % 2 != 0:
            await update.message.reply_text("❌ Количество помидоров должно быть кратно 2!")
            return

        coins_earned = amount // 2
        new_tomatoes = tomatoes - amount

        update_user(user_id, tomatoes=new_tomatoes, coins=coins_earned)

        updated_user = get_user(user_id)
        updated_coins = updated_user[18]      # индекс coins
        updated_tomatoes = updated_user[15]   # индекс tomatoes

        await update.message.reply_text(
            f"✅ Обмен успешен!\n"
            f"🍅 Потрачено: {amount} помидоров\n"
            f"💰 Получено: {coins_earned} монет\n"
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка при обмене помидоров: {str(e)}")
        print(f"ERROR in exchange_tomatoes_command: {str(e)}")

async def exchange_grapes_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_data = get_user(user.id, user.username, user.first_name)

        if not user_data or len(user_data) < 25:  # всего 22 поля, включая виноград и рубины
            print(f"ERROR: user_data has {len(user_data) if user_data else 0} values, expected 22. User ID: {user.id}")
            await update.message.reply_text("❌ Ошибка: пользователь не найден. Обратитесь к администратору.")
            return

        (
            user_id, player_id, username, first_name, diamonds, points,
            last_diamond_time, upgrade_level, registration_date, strawberries,
            farm_level, last_zombie_attack, zombie_status, last_strawberry_update,
            last_bonus_time, tomatoes, grapes, rubies, coins,
            farms_destroyed, clan_id, wood, stone, iron, titanium,
            *rest
        ) = user_data

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        cursor.execute("SELECT harvest_blocked_until FROM users WHERE user_id = ?", (user_id,))
        hb_row = cursor.fetchone()
        if hb_row and hb_row[0] > int(time.time()):
            await update.message.reply_text(
                "🐒👑 Эффект Королевы обезьян активен!\n"
                "❌ Обмен ресурсов временно запрещён."
            )
            conn.close()
            return

        message_text = update.message.text.strip().lower().split()
        if len(message_text) != 4 or message_text[:3] != ["тгз", "обменять", "виноград"]:
            await update.message.reply_text("❌ Использование: тгз обменять виноград [количество]")
            return

        try:
            amount = int(message_text[3])
            if amount <= 0:
                await update.message.reply_text("❌ Количество должно быть положительным числом!")
                return
        except (ValueError, IndexError):
            await update.message.reply_text("❌ Количество должно быть числом!")
            return

        if amount > grapes:
            await update.message.reply_text(f"❌ Недостаточно винограда! У вас: {grapes}")
            return

        if amount % 10000 != 0:
            await update.message.reply_text("❌ Количество винограда должно быть кратно 10000!")
            return

        rubies_earned = amount // 10000
        new_grapes = grapes - amount

        # Обновляем точно как в обмене помидоров
        update_user(user_id, grapes=new_grapes, rubies=rubies_earned)

        updated_user = get_user(user_id)
        updated_rubies = updated_user[16]  # индекс рубинов
        updated_grapes = updated_user[17]  # индекс винограда

        await update.message.reply_text(
            f"✅ Обмен успешен!\n"
            f"🍇 Потрачено: {amount} винограда\n"
            f"🔻 Получено: {rubies_earned} рубинов\n"
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка при обмене винограда: {str(e)}")
        print(f"ERROR in exchange_grapes_command: {str(e)}")

import html
from datetime import datetime, timezone
import sqlite3

async def profile_command(update, context, args=None):
    try:
        # --- получаем player_id из аргументов ---
        target_player_id = None
        if args and len(args) > 0:
            if args[0].isdigit():
                target_player_id = int(args[0])
            else:
                await update.message.reply_text("❌ Укажите корректный ID игрока")
                return

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # --- если указан player_id ---
        if target_player_id is not None:
            cursor.execute(
                "SELECT * FROM users WHERE player_id = ?",
                (target_player_id,)
            )
            user_data = cursor.fetchone()
            if not user_data:
                conn.close()
                await update.message.reply_text(f"❌ Игрок с ID #{target_player_id} не найден")
                return
        else:
            # --- без аргументов → свой профиль ---
            user_id = update.effective_user.id
            cursor.execute(
                "SELECT * FROM users WHERE user_id = ?",
                (user_id,)
            )
            user_data = cursor.fetchone()

        if user_data:
            print("DEBUG: столбцы и значения:")
            for i, val in enumerate(user_data):
                print(f"[{i:2d}] {val}")

        if not user_data:
            conn.close()
            await update.message.reply_text("❌ Пользователь не найден")
            return

        def safe_int(val, default=0):
            if val is None:
                return default
            if isinstance(val, (int, float)):
                return int(val)
            try:
                return int(val)
            except (ValueError, TypeError):
                return default

        user_id           = safe_int(user_data[0])
        player_id         = safe_int(user_data[1])
        username          = user_data[2] or None
        first_name        = user_data[3] or "Без имени"
        diamonds          = safe_int(user_data[4])
        points            = safe_int(user_data[5])
        last_diamond_time = user_data[6]          # строка

        registration_date = user_data[7]          # ← регистрация (строка)
        upgrade_level     = safe_int(user_data[8])  # ← уровень фарма (теперь 9)

        strawberries      = safe_int(user_data[10])
        farm_level        = safe_int(user_data[11])   # должно стать 20

        tomatoes          = safe_int(user_data[16])
        grapes            = safe_int(user_data[72])
        coins             = safe_int(user_data[18])
        rubies            = safe_int(user_data[73])
        farms_destroyed   = safe_int(user_data[20])

        clan_id           = user_data[21]

        wood              = safe_int(user_data[22])
        stone             = safe_int(user_data[23])
        iron              = safe_int(user_data[24])
        titanium          = safe_int(user_data[25])

        rank, next_rank_points, *_ = get_rank(points)

        # ---------- Кулдаун фарма ----------
        cooldown_hours = 34 / 60
        farm_cooldown_text = ""
        try:
            if last_diamond_time:
                last_update = datetime.fromisoformat(last_diamond_time.replace('Z', '+00:00')).replace(tzinfo=timezone.utc)
                now = datetime.now(timezone.utc)
                hours_passed = (now - last_update).total_seconds() / 3600
                if hours_passed < cooldown_hours:
                    remaining_time = (cooldown_hours - hours_passed) * 3600
                    minutes, seconds = divmod(int(remaining_time), 60)
                    total_seconds = cooldown_hours * 3600
                    progress = (total_seconds - remaining_time) / total_seconds
                    bar_length = 10
                    filled = int(bar_length * progress)
                    bar = "▓" * filled + "░" * (bar_length - filled)
                    farm_cooldown_text = f"\n⏳ Фарм: {bar} ({int(progress * 100)}%)\n⌛ Осталось: {minutes} мин {seconds} сек"
                else:
                    farm_cooldown_text = "\n✅ Фарм готов к использованию!"
            else:
                farm_cooldown_text = "\n✅ Фарм готов к использованию!"
        except Exception:
            farm_cooldown_text = "\n❓ Не удалось определить кулдаун фарма"

        STRAWBERRY_THRESHOLDS = [
            (100, 1.5), (1000, 2.0), (5000, 3.0), (15000, 4.0),
            (30000, 5.0), (60000, 6.0), (100000, 7.0), (160000, 8.0),
            (240000, 9.0), (350000, 10.0), (500000, 12.0), (700000, 14.0),
            (950000, 16.0), (1300000, 18.0), (1800000, 20.0), (2300000, 22.0),
            (3000000, 24.0), (4000000, 26.0),  # до x26
            (4600000, 27.0), (5300000, 28.0), (6100000, 29.0), (7000000, 30.0),
            (8000000, 31.0), (9200000, 32.0), (10600000, 33.0), (12200000, 34.0),
            (14000000, 35.0)  # финальный множитель
        ]

        TOMATO_THRESHOLDS = [
            (3000, 1.5), (30000, 2.0), (150000, 3.0), (450000, 4.0),
            (900000, 5.0), (1800000, 6.0), (3000000, 7.0), (4800000, 8.0),
            (7200000, 9.0), (10500000, 10.0), (15000000, 12.0), (21000000, 14.0),
            (28500000, 16.0), (39000000, 18.0), (54000000, 20.0), (69000000, 22.0),
            (90000000, 24.0), (120000000, 26.0),
            (138000000, 27.0), (159000000, 28.0), (183000000, 29.0), (210000000, 30.0),
            (240000000, 31.0), (276000000, 32.0), (318000000, 33.0), (366000000, 34.0),
            (420000000, 35.0)
        ]

        GRAPE_THRESHOLDS = [
            (10, 1.5), (100, 2.0), (500, 3.0), (1500, 4.0),
            (3000, 5.0), (6000, 6.0), (10000, 7.0), (16000, 8.0),
            (24000, 9.0), (35000, 10.0), (50000, 12.0), (70000, 14.0),
            (95000, 16.0), (130000, 18.0), (180000, 20.0), (230000, 22.0),
            (300000, 24.0), (400000, 26.0),
            (460000, 27.0), (530000, 28.0), (610000, 29.0), (700000, 30.0),
            (800000, 31.0), (920000, 32.0), (1060000, 33.0), (1220000, 34.0),
            (1400000, 35.0)
        ]

        def get_multiplier(value, thresholds):
            multiplier = 1.0
            for threshold, m in thresholds:
                if value >= threshold:
                    multiplier = m
                else:
                    break
            return multiplier

        strawberry_multiplier = get_multiplier(strawberries, STRAWBERRY_THRESHOLDS)
        tomato_multiplier = get_multiplier(tomatoes, TOMATO_THRESHOLDS)
        grape_multiplier = get_multiplier(grapes, GRAPE_THRESHOLDS)
        strawberry_text = f"x{strawberry_multiplier}" + (" 🔥" if strawberry_multiplier > 1 else "")
        tomato_text = f"x{tomato_multiplier}" + (" 🔥" if tomato_multiplier > 1 else "")
        grape_text = f"x{grape_multiplier}" + (" 🔥" if grape_multiplier > 1 else "")

        extra_boost_text = ""
        if strawberry_multiplier == 35.0 or tomato_multiplier == 35.0:
            extra_boost_text = "\n🔥 <b>Максимальный буст достигнут!</b> 🔥"

        # ---------- Клан ----------
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        clan_text = "👥 Клан: Нет"
        try:
            cursor.execute('SELECT clan_id FROM users WHERE user_id = ?', (user_id,))
            row = cursor.fetchone()
            if row and row[0]:
                clan_id = row[0]
                cursor.execute('SELECT clan_name FROM clans WHERE id = ?', (clan_id,))
                cname_row = cursor.fetchone()
                clan_name = cname_row[0] if cname_row else None
                if clan_name:
                    cursor.execute('SELECT COALESCE(SUM(points), 0) FROM users WHERE clan_id = ?', (clan_id,))
                    clan_points = cursor.fetchone()[0] or 0
                    cursor.execute('SELECT COUNT(*) FROM users WHERE clan_id = ?', (clan_id,))
                    clan_members = cursor.fetchone()[0] or 0
                    clan_text = f"👥 Клан: {html.escape(str(clan_name))} — {clan_members} участников, {clan_points} очков"
        except:
            pass

        # ---------- Маски ----------
        cursor.execute('SELECT mask_id FROM user_masks WHERE user_id = ?', (user_id,))
        owned_masks = [row[0] for row in cursor.fetchall()]
        masks_text = f"🎭 Маски: {', '.join([html.escape(get_mask_info(mask_id)['name']) for mask_id in owned_masks])}" if owned_masks else "🎭 Маски: Нет купленных масок"


        # ---------- Постройки и отгаданные загадки ----------
        try:
            cursor.execute("""
                SELECT sawmill, quarry, smelter, factory, mine, solved_riddles
                FROM users
                WHERE user_id = ?
            """, (user_id,))
            factories = cursor.fetchone()
            if factories:
                sawmill, quarry, smelter, factory, mine, solved_riddles = factories
                import json
                try:
                    solved_riddles_list = json.loads(solved_riddles) if solved_riddles else []
                    solved_riddles_count = len(solved_riddles_list) if isinstance(solved_riddles_list, list) else 0
                except:
                    solved_riddles_count = 0
            else:
                sawmill = quarry = smelter = factory = mine = 0
                solved_riddles_count = 0
        except:
            sawmill = quarry = smelter = factory = mine = 0
            solved_riddles_count = 0

        conn.close()

        points_text = f"{points} / {next_rank_points}" if next_rank_points != float('inf') else f"{points} / ∞"

        profile_text = (
            "👤 Профиль игрока:\n"
            f"Имя: {first_name}\n"
            f"Username: @{username if username else 'нет'}\n"
            f"ID: #{player_id}\n\n"
            f"💎 Алмазы: {diamonds} ❄️ Очки: {points_text}\n"
            f"🏆 Ранг: {rank}\n"
            f"🚜 Уровень фермы: {farm_level}\n"
            f"⚡ Уровень фарма: {upgrade_level}{farm_cooldown_text}\n"
            f"🍓 Клубники: {strawberries} ({strawberry_text})\n"
            f"🍅 Помидоры: {tomatoes} ({tomato_text})\n"
            f"🍇 Виноград: {grapes} ({grape_text})\n"
            f"🪵 Дерево: {wood}\n"
            f"🪨 Камень: {stone}\n"
            f"⛓ Железо: {iron}\n"
            f"🔩 Титан: {titanium}\n"
            f"♦️ Рубины: {rubies}\n"
            f"💰 Монеты: {coins}\n"
            f"💥 Уничтожено ферм: {farms_destroyed}\n"
            f"🪵 Лесопилки: {sawmill}\n"
            f"🪨 Каменоломни: {quarry}\n"
            f"🔥 Плавильни: {smelter}\n"
            f"🏭 Фабрики: {factory}\n"
            f"⛏ Карьеры: {mine}\n"
            f"🧩 Отгаданные загадки: {solved_riddles_count}\n"
            f"{clan_text}\n"
            f"{masks_text}{extra_boost_text}"
        )
        if registration_date:
            try:
                reg_date = datetime.fromisoformat(registration_date).strftime("%d.%m.%Y %H:%M")
                profile_text += f"\n📅 Регистрация: {reg_date}"
            except:
                profile_text += f"\n📅 Регистрация: {registration_date}"

        await update.message.reply_text(profile_text, parse_mode="HTML")

    except Exception as e:
        await update.message.reply_text(f"Произошла ошибка: {e}")
        print("ERROR in profile_command:", str(e))

async def buy_farm_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_data = get_user(user.id, user.username, user.first_name)

        if user_data is None or len(user_data) < 25:
            await update.message.reply_text("❌ Пользователь не найден!")
            return

        (
            user_id, player_id, username, first_name, diamonds, points,
            last_diamond_time, upgrade_level, registration_date, strawberries,
            farm_level, last_zombie_attack, zombie_status, last_strawberry_update,
            last_bonus_time, tomatoes, coins, farms_destroyed, clan_id,
            *rest
        ) = user_data

        message_text = update.message.text.strip().lower().split()

        # Проверка формата команды
        if len(message_text) != 4 or message_text[:3] != ["тгз", "купить", "фарм"]:
            await update.message.reply_text("❌ Использование: тгз купить фарм [уровень]")
            return

        # Извлекаем уровень
        try:
            level = int(message_text[3])
        except ValueError:
            await update.message.reply_text("❌ Уровень должен быть числом!")
            return

        # Проверка диапазона уровней
        if level not in range(0, 21):  # ✅ теперь уровни 0–20
            await update.message.reply_text("❌ Уровень должен быть от 0 до 20!")
            return

        if level != upgrade_level + 1:
            await update.message.reply_text(f"❌ Можно купить только следующий уровень фарма ({upgrade_level + 1})!")
            return

        # Получаем инфо об апгрейде
        upgrade_info = get_upgrade_info(level)
        if not upgrade_info:
            await update.message.reply_text("❌ Указан некорректный уровень фарма!")
            return

        cost = upgrade_info["cost"]
        if cost > diamonds:
            await update.message.reply_text(f"❌ Недостаточно алмазов! Нужно: {cost}, у вас: {diamonds}")
            return

        # Списываем алмазы
        new_diamonds = diamonds - cost
        update_user(user_id, diamonds=new_diamonds, upgrade_level=level)

        await update.message.reply_text(
            f"✅ Фарм алмазов успешно улучшен до уровня {level}!\n"
            f"💎 Потрачено: {cost} алмазов\n"
            f"💎 Остаток: {new_diamonds}\n"
            f"📈 Новый доход: {upgrade_info['min_diamonds']}-{upgrade_info['max_diamonds']} алмазов"
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при покупке фарма: {str(e)}")
        print(f"ERROR in buy_farm_command: {str(e)}")

from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import ContextTypes

import json
from telegram import InlineKeyboardButton, InlineKeyboardMarkup


# ======================================================================
# Вспомогательная функция — получает имя УМНО (из базы или из API)
# ======================================================================
async def get_smart_first_name(user_id: int, db_name, context):
    """
    db_name — это user[3] (first_name из базы)
    """
    # Если в базе уже есть нормальное имя — берём его мгновенно
    if db_name is not None:
        cleaned = str(db_name).strip()
        if cleaned and cleaned.lower() != "none":
            return cleaned

    # Только если в базе None / пусто / "None" — запрашиваем актуальное
    try:
        chat = await context.bot.get_chat(user_id)
        name = chat.first_name
        if name and str(name).strip():
            return str(name).strip()
    except Exception:
        pass  # если ошибка (блокировка, rate-limit и т.д.) — ничего страшного

    # Последний fallback
    return f"#{user_id}"


# ======================================================================
# Функция топ-команды с загадками
# ======================================================================
async def top_command(update, context):
    try:
        users = get_all_users()

        if not users:
            await update.message.reply_text("❌ Нет зарегистрированных пользователей!")
            return

        sorted_users = sorted(users, key=lambda x: x[5] or 0, reverse=True)
        top_text = "🏆 Топ-30 игроков по очкам:\n\n"
        medals = ["🥇", "🥈", "🥉"]

        for i, user in enumerate(sorted_users[:30], 1):
            try:
                riddles_count = len(json.loads(user[28])) if user[28] else 0

                medal = medals[i - 1] if i <= 3 else f"{i}."

                # Умный вызов — передаём user[3] из базы
                name_display = await get_smart_first_name(user[0], user[3], context)

                if i == 1:
                    name_display = f"✨{name_display}✨"

                top_text += f"{medal} {name_display} (ID #{user[1]}): {user[5]} ⭐ | 🧩 {riddles_count} загадок\n"

            except Exception:
                continue

        keyboard = [
            [
                InlineKeyboardButton("Очки ⭐", callback_data="top_points"),
                InlineKeyboardButton("Загадки 🧩", callback_data="top_riddles"),
                InlineKeyboardButton("Фермы 🔥", callback_data="top_farms"),
            ],
            [
                InlineKeyboardButton("Клубника 🍓", callback_data="top_strawberries"),
                InlineKeyboardButton("Помидоры 🍅", callback_data="top_tomatoes"),
                InlineKeyboardButton("Виноград 🍇", callback_data="top_grapes"),
            ],
            [
                InlineKeyboardButton("Алмазы 💎", callback_data="top_diamonds"),
                InlineKeyboardButton("Рубины 🔻", callback_data="top_rubies"),
                InlineKeyboardButton("Монеты 💰", callback_data="top_coins"),
            ],
            [
                InlineKeyboardButton("Дерево 🪵", callback_data="top_wood"),
                InlineKeyboardButton("Камень 🪨", callback_data="top_stone"),
                InlineKeyboardButton("Железо ⛓", callback_data="top_iron"),
                InlineKeyboardButton("Титан 🔩", callback_data="top_titan"),
            ],
        ]

        reply_markup = InlineKeyboardMarkup(keyboard)
        await update.message.reply_text(top_text, reply_markup=reply_markup)

    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка: {str(e)}")
        print(f"ERROR in top_command: {str(e)}")


# ======================================================================
# Callback для кнопок топа
# ======================================================================
async def top_callback_handler(update, context):
    try:
        query = update.callback_query
        callback_data = query.data
        users = get_all_users()
        if not users:
            await query.answer("❌ Нет зарегистрированных пользователей!")
            return

        sort_mapping = {
            "top_points": (5, "🏆 Топ-30 игроков по очкам:\n\n", "⭐"),
            "top_strawberries": (9, "🍓 Топ-30 игроков по клубнике:\n\n", "🍓"),
            "top_tomatoes": (15, "🍅 Топ-30 игроков по помидорам:\n\n", "🍅"),
            "top_grapes": (29, "🍇 Топ-30 игроков по винограду:\n\n", "🍇"),
            "top_diamonds": (4, "💎 Топ-30 игроков по алмазам:\n\n", "💎"),
            "top_rubies": (30, "🔻 Топ-30 игроков по рубинам:\n\n", "🔻"),
            "top_coins": (16, "💰 Топ-30 игроков по монетам:\n\n", "💰"),
            "top_farms": (17, "🔥 Топ-30 игроков по уничтоженным фермам:\n\n", "🔥"),
            "top_wood": (18, "🪵 Топ-30 игроков по дереву:\n\n", "🪵"),
            "top_stone": (19, "🪨 Топ-30 игроков по камню:\n\n", "🪨"),
            "top_iron": (20, "⛓ Топ-30 игроков по железу:\n\n", "⛓"),
            "top_titan": (21, "🔩 Топ-30 игроков по титану:\n\n", "🔩"),
            "top_riddles": (28, "🧩 Топ-30 игроков по отгаданным загадкам:\n\n", "🧩"),
        }

        if callback_data not in sort_mapping:
            await query.answer("❌ Некорректный запрос!")
            return

        index, header, symbol = sort_mapping[callback_data]

        if callback_data == "top_riddles":
            sorted_users = sorted(users, key=lambda x: len(json.loads(x[28])) if x[28] else 0, reverse=True)
        else:
            sorted_users = sorted(
                users,
                key=lambda x: (x[index] or 0) if isinstance(x[index], (int, float)) else 0,
                reverse=True
            )

        top_text = header
        medals = ["🥇", "🥈", "🥉"]

        for i, user in enumerate(sorted_users[:30], 1):
            try:
                if callback_data == "top_riddles":
                    riddles_count = len(json.loads(user[28])) if user[28] else 0
                    value_display = f"{riddles_count} {symbol}"
                else:
                    value_display = f"{user[index]} {symbol}"

                medal = medals[i - 1] if i <= 3 else f"{i}."

                # Умный вызов — передаём user[3]
                name_display = await get_smart_first_name(user[0], user[3], context)

                if i == 1:
                    name_display = f"✨{name_display}✨"

                top_text += f"{medal} {name_display} (ID #{user[1]}): {value_display}\n"

            except Exception:
                continue

        keyboard = [
            [
                InlineKeyboardButton("Очки ⭐", callback_data="top_points"),
                InlineKeyboardButton("Загадки 🧩", callback_data="top_riddles"),
                InlineKeyboardButton("Фермы 🔥", callback_data="top_farms"),
            ],
            [
                InlineKeyboardButton("Клубника 🍓", callback_data="top_strawberries"),
                InlineKeyboardButton("Помидоры 🍅", callback_data="top_tomatoes"),
                InlineKeyboardButton("Виноград 🍇", callback_data="top_grapes"),
            ],
            [
                InlineKeyboardButton("Алмазы 💎", callback_data="top_diamonds"),
                InlineKeyboardButton("Рубины 🔻", callback_data="top_rubies"),
                InlineKeyboardButton("Монеты 💰", callback_data="top_coins"),
            ],
            [
                InlineKeyboardButton("Дерево 🪵", callback_data="top_wood"),
                InlineKeyboardButton("Камень 🪨", callback_data="top_stone"),
                InlineKeyboardButton("Железо ⛓", callback_data="top_iron"),
                InlineKeyboardButton("Титан 🔩", callback_data="top_titan"),
            ],
        ]

        reply_markup = InlineKeyboardMarkup(keyboard)
        await query.message.edit_text(top_text, reply_markup=reply_markup)
        await query.answer()

    except Exception as e:
        await query.answer(f"❌ Произошла ошибка: {str(e)}")
        print(f"ERROR in top_callback_handler: {str(e)}")

from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import ContextTypes
import sqlite3
import math

# Показывает список масок с пагинацией (6 масок на страницу)
async def masks_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    user_data = get_user(user.id, user.username, user.first_name)

    if not user_data:
        await update.message.reply_text("❌ Ошибка: пользователь не найден. Обратитесь к администратору.")
        return

    user_id = user_data[0]
    diamonds = user_data[4]

    conn = sqlite3.connect("diamonds_bot.db")
    cursor = conn.cursor()
    cursor.execute("SELECT mask_id FROM user_masks WHERE user_id = ?", (user_id,))
    owned_masks = {row[0] for row in cursor.fetchall()}
    conn.close()

    # Начинаем с первой страницы
    await show_masks_page(update, context, page=1, owned_masks=owned_masks, diamonds=diamonds, user_id=user_id)

import sqlite3
from telegram import Update
from telegram.ext import ContextTypes

# Отдельная функция для отображения страницы масок
async def show_masks_page(update: Update, context: ContextTypes.DEFAULT_TYPE, page: int, owned_masks, diamonds, user_id):
    masks_per_page = 5
    total_masks = 19  # Всего масок
    total_pages = math.ceil(total_masks / masks_per_page)

    start = (page - 1) * masks_per_page + 1
    end = min(page * masks_per_page, total_masks)

    text = "🎭 <b>Список масок:</b>\n\n"

    for mask_id in range(start, end + 1):
        mask = get_mask_info(mask_id)
        if not mask:
            continue

        owned = "✅ <b>Куплена</b>" if mask_id in owned_masks else f"💎 <i>Стоимость:</i> {mask['cost']} алмазов"

        text += (
            f"🎭 <b>{mask['name']}</b> (ID: {mask_id})\n"
            f"📜 Эффект: {mask['effect']}\n"
            f"{owned}\n\n"
        )

    text += f"💎 <b>Ваш баланс алмазов:</b> {diamonds}\n"
    text += "📋 <i>Используйте:</i> <b>тгз купить маску [номер]</b>\n\n"
    text += f"📄 Страница {page}/{total_pages}"

    # Кнопки навигации
    buttons = []
    if page > 1:
        buttons.append(InlineKeyboardButton("⬅️ Назад", callback_data=f"masks_page_{page - 1}"))
    if page < total_pages:
        buttons.append(InlineKeyboardButton("➡️ Далее", callback_data=f"masks_page_{page + 1}"))

    keyboard = InlineKeyboardMarkup([buttons] if buttons else [])

    if update.callback_query:
        await update.callback_query.edit_message_text(text=text, parse_mode="HTML", reply_markup=keyboard)
        await update.callback_query.answer()
    else:
        await update.message.reply_text(text, parse_mode="HTML", reply_markup=keyboard)


# Обработчик кнопок для листания масок
async def masks_page_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    data = query.data

    if not data.startswith("masks_page_"):
        return

    page = int(data.split("_")[-1])
    user = query.from_user
    user_data = get_user(user.id, user.username, user.first_name)

    if not user_data:
        await query.answer("Ошибка загрузки пользователя.", show_alert=True)
        return

    user_id = user_data[0]
    diamonds = user_data[4]
# Загружаем купленные маски
    conn = sqlite3.connect("diamonds_bot.db")
    cursor = conn.cursor()
    cursor.execute("SELECT mask_id FROM user_masks WHERE user_id = ?", (user_id,))
    owned_masks = {row[0] for row in cursor.fetchall()}
    conn.close()

    await show_masks_page(update, context, page, owned_masks, diamonds, user_id)

async def buy_mask_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_data = get_user(user.id, user.username, user.first_name)

        if user_data is None or len(user_data) < 25:
            await update.message.reply_text("❌ Ошибка: пользователь не найден. Обратитесь к администратору.")
            return

        (
            user_id, player_id, username, first_name, diamonds, points,
            last_diamond_time, upgrade_level, registration_date, strawberries,
            farm_level, last_zombie_attack, zombie_status, last_strawberry_update,
            last_bonus_time, tomatoes, coins, farms_destroyed, clan_id, *_
        ) = user_data

        # Проверяем команду
        message_text = update.message.text.strip().lower().split()
        if len(message_text) != 4 or message_text[:3] != ["тгз", "купить", "маску"]:
            await update.message.reply_text("❌ Использование: тгз купить маску [номер]")
            return

        # Проверяем ID маски
        try:
            mask_id = int(message_text[3])
        except ValueError:
            await update.message.reply_text("❌ Номер маски должен быть числом!")
            return

        mask_info = get_mask_info(mask_id)
        if not mask_info:
            await update.message.reply_text("❌ Маска с таким номером не найдена!")
            return

        conn = sqlite3.connect('diamonds_bot.db', timeout=10)
        cursor = conn.cursor()

        # Проверка, есть ли уже маска
        cursor.execute('SELECT 1 FROM user_masks WHERE user_id = ? AND mask_id = ?', (user_id, mask_id))
        if cursor.fetchone():
            conn.close()
            await update.message.reply_text(f"❌ Вы уже купили <b>{mask_info['name']}</b>!", parse_mode="HTML")
            return

        cost = mask_info["cost"]
        if diamonds < cost:
            conn.close()
            await update.message.reply_text(
                f"❌ Недостаточно алмазов!\n"
                f"💎 Нужно: <b>{cost}</b>\n"
                f"💎 У вас: <b>{diamonds}</b>",
                parse_mode="HTML"
            )
            return

        # Списываем алмазы и добавляем маску в инвентарь
        new_diamonds = diamonds - cost
        cursor.execute("UPDATE users SET diamonds = ? WHERE user_id = ?", (new_diamonds, user_id))
        cursor.execute("INSERT INTO user_masks (user_id, mask_id) VALUES (?, ?)", (user_id, mask_id))
        conn.commit()
        conn.close()

        await update.message.reply_text(
            f"✅ <b>{mask_info['name']}</b> успешно куплена!\n\n"
            f"💎 Потрачено: <b>{cost}</b>\n"
            f"💎 Остаток алмазов: <b>{new_diamonds}</b>\n\n"
            f"📜 Эффект: <i>{mask_info['effect']}</i>",
            parse_mode="HTML"
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка при покупке маски: {str(e)}")
        print(f"ERROR in buy_mask_command: {str(e)}")

import sqlite3
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ContextTypes, CallbackQueryHandler

DB_PATH = "diamonds_bot.db"

ZOMBIE_INFO = {
    1: {"name": "Обычный Зомби", "hp": 10, "damage": 1, "cost_diamonds": 15, "cost_coins": 1083},
    2: {"name": "Бегун", "hp": 30, "damage": 4, "cost_diamonds": 36, "cost_coins": 3027, "feature": "10× скорость"},
    3: {"name": "Взрывун", "hp": 60, "damage": 9, "cost_diamonds": 73, "cost_coins": 6279, "feature": "2× скорость, шанс 50% двойного урона"},
    4: {"name": "Невидимка", "hp": 120, "damage": 18, "cost_diamonds": 108, "cost_coins": 12343, "feature": "15× скорость"},
    5: {"name": "Плювака", "hp": 150, "damage": 22, "cost_diamonds": 116, "cost_coins": 14049},
    6: {"name": "Здоровяк", "hp": 300, "damage": 75, "cost_diamonds": 210, "cost_coins": 32270, "feature": "Скорость ×3"},
    7: {"name": "Скала", "hp": 48000, "damage": 450, "cost_diamonds": 1588, "cost_coins": 270110},
    8: {"name": "Бомбстер", "hp": 500, "damage": 1050, "cost_diamonds": 3032, "cost_coins": 497109, "feature": "2× скорость, шанс 50% двойного урона"},
    9: {"name": "Разведчик", "hp": 0, "damage": 0, "cost_diamonds": 8, "cost_coins": 541, "feature": "Скорость ×60"},
    10: {"name": "Горбатый", "hp": 200, "damage": 30, "cost_diamonds": 188, "cost_coins": 19921, "feature": "Скорость ×20, ворует 6900 монет у цели (даже при смерти)"},
    11: {"name": "Крампус", "hp": 250, "damage": 37, "cost_diamonds": 366, "cost_coins": 18388, "feature": "4× скорость, ворует 127 алмазов (даже при смерти)"},
    12: {"name": "Зайка", "hp": 300, "damage": 45, "cost_diamonds": 217, "cost_coins": 20316, "feature": "Скорость ×2, уничтожает 1 турель каждого вида у цели (даже при смерти)"},
    13: {"name": "Босс", "hp": 11520, "damage": 3780, "cost_diamonds": 10915, "cost_coins": 1791603, "feature": "10× скорость, ломает 30 заводов начиная с лесопилок"},
    14: {"name": "Элитный разведчик", "hp": 0, "damage": 0, "cost_diamonds": 26, "cost_coins": 2057, "feature": "×60 скорость, атака без уведомления жертвы"},
    15: {"name": "Хамелеон", "hp": 60, "damage": 45, "cost_diamonds": 116, "cost_coins": 12992, "feature": "Маскируется под Скалу"},
    16: {"name": "Вирус", "hp": 1, "damage": 0, "cost_diamonds": 583, "cost_coins": 86603, "feature": "×25 скорость, заражает ферму на 60 секунд — все зомби, атакующие заражённую ферму, наносят ×2 урон"},
    17: {"name": "Электрозомби", "hp": 3000, "damage": 240, "cost_diamonds": 751, "cost_coins": 127754, "feature": "2× скорость, шокирует ферму — ослабляет урон турелей на 50% в течение 30 секунд"},
    18: {"name": "Командор", "hp": 350, "damage": 37, "cost_diamonds": 120, "cost_coins": 18388, "feature": "×30 скорость, после смерти удваивает скорость всех атакующих зомби на 60 секунд"},
    19: {"name": "Слизень", "hp": 3000, "damage": 7, "cost_diamonds": 51, "cost_coins": 4764, "feature": "×5 скорость, замедляет последующих зомби в 2 раза, но увеличивает их здоровье на 50% (даже после смерти)"},
    20: {"name": "Берсерк", "hp": 350, "damage": 85, "cost_diamonds": 419, "cost_coins": 86164, "feature": "×3 скорость, если турели наносят урон но не убивают всю атаку берсерков, их урон увеличивается в 3 раза (урон не зависит от количества)"},
    21: {"name": "Оливье 2025", "hp": 1000, "damage": 4, "cost_diamonds": 51, "cost_coins": 4764, "feature": "×5 скорость, значительно замедляет последующих зомби в 4 раза, но увеличивает их здоровье на 100% (даже после смерти)"},
    22: {"name": "Гнилой Санта-Клаус", "hp": 200, "damage": 42, "cost_diamonds": 152, "cost_coins": 19455, "feature": "×4 скорость, ворует 4500 дерева, камня, железа, титана у цели (даже при смерти)"},
    23: {"name": "Олень", "hp": 3000, "damage": 200, "cost_diamonds": 534, "cost_coins": 82173, "feature": "×6 скорость, ломает 5 деревянных турелей"},
    24: {"name": "Жнец", "hp": 15000, "damage": 55, "cost_diamonds": 4188, "cost_coins": 661538, "feature": "×20 скорость, наносит урон равный 1% здоровья фермы врага (даже при смерти) (процент не зависит от количества Жнецов)"},
    25: {"name": "Свиньи в броне", "hp": 1500, "damage": 100, "cost_diamonds": 310, "cost_coins": 42927, "feature": "×8 скорость, шанс 50% забрать у врага 430 клубники и 792 помидора (если они есть)"},
    26: {"name": "Обезьяна", "hp": 170, "damage": 25, "cost_diamonds": 137, "cost_coins": 38247, "feature": "×9 скорость, игнорирует турели и мины"},
    27: {"name": "Сапёр", "hp": 200, "damage": 45, "cost_diamonds": 246, "cost_coins": 38619, "feature": "×5 скорость, уничтожает 7 мин (кроме капканов)"},
    28: {"name": "Медведь", "hp": 2500, "damage": 300, "cost_diamonds": 802, "cost_coins": 205524, "feature": "×7 скорость, ломает 5 каменных капканов"},
    29: {"name": "Ведьма", "hp": 180, "damage": 45, "cost_diamonds": 246, "cost_coins": 28505, "feature": "×5 скорость, уничтожает 1 завод каждого вида"},
    30: {"name": "Король обезъян", "hp": 1000, "damage": 750, "cost_diamonds": 2079, "cost_coins": 319457, "feature": "Скорость ×2, Следующий доход цели с фермы уменьшается в 2 раза (Не зависит от количества зомби)"},
    31: {"name": "Мегарыцарь", "hp": 6000, "damage": 750, "cost_diamonds": 3176, "cost_coins": 351718, "feature": "Скорость ×3, шанс 90% на двойной урон"},
    32: {"name": "Королева обезъян", "hp": 750, "damage": 140, "cost_diamonds": 1299, "cost_coins": 199306, "feature": "Скорость ×3, Запрещает собирать урожай с фермы и использовать обмен урожая в течение 1 часа (Не зависит от количества зомби)"},
    33: {"name": "Принцесса", "hp": 60, "damage": 15, "cost_diamonds": 208, "cost_coins": 6384, "feature": "Скорость ×3, Наносит 15 урона каждые 30 минут, до отправки на вас зомби колдуна"},
    34: {"name": "Колдун", "hp": 180, "damage": 68, "cost_diamonds": 187, "cost_coins": 5751, "feature": "Скорость ×3, Уничтожает 1 принцессу на вашей ферме"},
    35: {"name": "Ледяной Маг", "hp": 310, "damage": 65, "cost_diamonds": 419, "cost_coins": 63897, "feature": "Скорость ×3, Уменьшает жертве урон по вашей ферме в 2 раза в течение 1 часа"},
    36: {"name": "Всадник на кабане", "hp": 1000, "damage": 1420, "cost_diamonds": 4000, "cost_coins": 600000, "feature": "Скорость ×3, шанс 3% сломать случайную башню противника (если есть)"},
    37: {"name": "Лихварь алмазов", "hp": 2000, "damage": 1, "cost_diamonds": 10000, "cost_coins": 69, "feature": "Скорость ×60, ворует 1% алмазов у цели"},
    38: {"name": "Лихварь монет", "hp": 2000, "damage": 1, "cost_diamonds": 69, "cost_coins": 1500000, "feature": "Скорость ×60, ворует 1% монет у цели"},
}

ZOMBIES_PER_PAGE = 5

async def zombies_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user_id = update.effective_user.id

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        cursor.execute(
            "SELECT diamonds, coins, zombie_discount FROM users WHERE user_id = ?",
            (user_id,)
        )
        user_data = cursor.fetchone()
        if not user_data:
            conn.close()
            await update.message.reply_text("❌ Вы не зарегистрированы!")
            return

        diamonds, coins, discount = user_data

        cursor.execute("""
            CREATE TABLE IF NOT EXISTS user_zombies (
                user_id INTEGER,
                zombie_id INTEGER,
                quantity INTEGER DEFAULT 0,
                PRIMARY KEY (user_id, zombie_id)
            )
        """)
        cursor.execute(
            "SELECT zombie_id, quantity FROM user_zombies WHERE user_id = ?",
            (user_id,)
        )
        owned_zombies = {row[0]: row[1] for row in cursor.fetchall()}
        conn.close()

        await send_zombies_page(
            update, context,
            page=0,
            user_id=user_id,
            owned_zombies=owned_zombies,
            diamonds=diamonds,
            coins=coins,
            discount=discount
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при просмотре зомби: {e}")
        print("ERROR zombies_command:", e)

async def zombies_page_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    page = int(query.data.split("_")[-1])
    user_id = query.from_user.id

    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    cursor.execute(
        "SELECT zombie_id, quantity FROM user_zombies WHERE user_id = ?",
        (user_id,)
    )
    owned_zombies = {row[0]: row[1] for row in cursor.fetchall()}

    cursor.execute(
        "SELECT diamonds, coins, zombie_discount FROM users WHERE user_id = ?",
        (user_id,)
    )
    diamonds, coins, discount = cursor.fetchone()
    conn.close()

    await send_zombies_page(
        update, context,
        page=page,
        user_id=user_id,
        owned_zombies=owned_zombies,
        diamonds=diamonds,
        coins=coins,
        discount=discount
    )

async def send_zombies_page(update, context, page, owned_zombies, diamonds, coins, discount, user_id):
    zombie_ids = sorted(ZOMBIE_INFO.keys())
    start = page * ZOMBIES_PER_PAGE
    end = start + ZOMBIES_PER_PAGE
    zombies_list = [(zid, ZOMBIE_INFO[zid]) for zid in zombie_ids[start:end]]

    # ─── Загружаем экипировку ───
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    cursor.execute("""
        SELECT zombie_id, equip_type, equipment_id, quantity
        FROM zombie_equipment
        WHERE user_id = ? AND quantity > 0
    """, (user_id,))

    rows = cursor.fetchall()
    conn.close()

    # { zombie_id: [(equip_type, equipment_id, qty), ...] }
    equipment = {}
    for zid, etype, eid, qty in rows:
        equipment.setdefault(zid, []).append((etype, eid, qty))

    text = "🧟 <b>Ваши зомби:</b>\n\n"

    for zid, info in zombies_list:
        qty = owned_zombies.get(zid, 0)
        cost_d = info.get("cost_diamonds", 0)
        cost_c = info.get("cost_coins", 0)

        if discount and discount > 0:
            price_d = int(cost_d * (100 - discount) / 100)
            price_c = int(cost_c * (100 - discount) / 100)
            text += (
                f"{zid}) {info['name']}\n"
                f"💎 Цена: <b>{price_d}</b> (-{discount}%)\n"
                f"💰 Монеты: <b>{price_c}</b>\n"
            )
        else:
            text += (
                f"{zid}) {info['name']}\n"
                f"💎 Стоимость: {cost_d} алмазов и {cost_c} монет\n"
            )

        text += (
            f"❤️ Здоровье: {info.get('hp', 0)}\n"
            f"⚔️ Урон ферме врага: {info.get('damage', 0)}\n"
        )

        if "feature" in info:
            text += f"✨ Особенность: {info['feature']}\n"

        text += f"🧟 Количество: {qty}\n"

        # ─── ЭКИПИРОВКА (ТОЛЬКО ЕСЛИ ЕСТЬ) ───
        if zid in equipment:
            for etype, eid, eq_qty in equipment[zid]:
                text += f"🛡️ {etype} (ID {eid}) ×{eq_qty}\n"

        text += "\n"

    text += (
        f"💎 <b>Ваши алмазы:</b> {diamonds}\n"
        f"💰 <b>Ваши монеты:</b> {coins}\n\n"
        "📋 Используйте: <b>тгз купить зомби [номер]</b>"
    )

    # ─── КНОПКИ (НЕ ТРОГАЛ) ───
    keyboard = []
    row = []

    if page > 0:
        row.append(InlineKeyboardButton("⬅️ Назад", callback_data=f"zombies_page_{page-1}"))
    if end < len(zombie_ids):
        row.append(InlineKeyboardButton("➡️ Вперёд", callback_data=f"zombies_page_{page+1}"))

    if row:
        keyboard.append(row)

    reply_markup = InlineKeyboardMarkup(keyboard) if keyboard else None

    if update.message:
        await update.message.reply_text(text, parse_mode="HTML", reply_markup=reply_markup)
    else:
        await update.callback_query.edit_message_text(text, parse_mode="HTML", reply_markup=reply_markup)
        await update.callback_query.answer()

import sqlite3
from telegram import Update
from telegram.ext import ContextTypes

DB_PATH = "diamonds_bot.db"

async def buy_zombie_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        message_text = update.message.text.strip().lower().split()

        # Теперь должно быть 5 слов
        # тгз купить зомби [id] [кол-во]
        if len(message_text) != 5 or message_text[:3] != ["тгз", "купить", "зомби"]:
            await update.message.reply_text("❌ Использование: тгз купить зомби [номер] [количество]")
            return

        try:
            zombie_id = int(message_text[3])
            amount = int(message_text[4])
        except (ValueError, IndexError):
            await update.message.reply_text("❌ Номер и количество должны быть числами!")
            return

        if zombie_id not in ZOMBIE_INFO:
            await update.message.reply_text("❌ Такого зомби не существует!")
            return

        if amount <= 0:
            await update.message.reply_text("❌ Количество должно быть больше 0!")
            return

        user_id = update.effective_user.id
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        cursor.execute("SELECT diamonds, coins, zombie_discount FROM users WHERE user_id = ?", (user_id,))
        user_data = cursor.fetchone()

        if not user_data:
            conn.close()
            await update.message.reply_text("❌ Вы не зарегистрированы!")
            return

        diamonds, coins, discount = user_data

        zombie_info = ZOMBIE_INFO[zombie_id]
        base_cost_d = zombie_info["cost_diamonds"]
        base_cost_c = zombie_info["cost_coins"]

        # Цена * количество
        final_cost_d = int(base_cost_d * (1 - discount / 100)) * amount
        final_cost_c = int(base_cost_c * (1 - discount / 100)) * amount

        # Проверка ресурсов
        if diamonds < final_cost_d:
            await update.message.reply_text(f"❌ Недостаточно алмазов! Нужно {final_cost_d}, у вас {diamonds}")
            conn.close()
            return

        if coins < final_cost_c:
            await update.message.reply_text(f"❌ Недостаточно монет! Нужно {final_cost_c}, у вас {coins}")
            conn.close()
            return

        # Таблицы
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS user_zombies (
                user_id INTEGER,
                zombie_id INTEGER,
                quantity INTEGER DEFAULT 0,
                PRIMARY KEY (user_id, zombie_id)
            )
        """)

        cursor.execute("""
            CREATE TABLE IF NOT EXISTS user_quests (
                user_id INTEGER PRIMARY KEY,
                q1 INTEGER DEFAULT 0,
                q2 INTEGER DEFAULT 0,
                q3 INTEGER DEFAULT 0,
                q4 INTEGER DEFAULT 0,
                q5 INTEGER DEFAULT 0
            )
        """)

        cursor.execute("INSERT INTO user_quests (user_id) VALUES (?) ON CONFLICT(user_id) DO NOTHING", (user_id,))

        # Покупаем сразу нужное количество
        cursor.execute("""
            INSERT INTO user_zombies (user_id, zombie_id, quantity)
            VALUES (?, ?, ?)
            ON CONFLICT(user_id, zombie_id) DO UPDATE SET quantity = quantity + ?
        """, (user_id, zombie_id, amount, amount))

        # Проверка квестов
        cursor.execute("SELECT q1, q2, q3, q4, q5 FROM user_quests WHERE user_id = ?", (user_id,))
        q_status = cursor.fetchone() or (0, 0, 0, 0, 0)

        for qid, data in QUEST_REQUIREMENTS.items():
            if data["zombie_id"] == zombie_id:

                cursor.execute(
                    "SELECT quantity FROM user_zombies WHERE user_id = ? AND zombie_id = ?",
                    (user_id, zombie_id)
                )
                total = cursor.fetchone()[0]

                if total >= data["need"] and q_status[qid - 1] == 0:
                    cursor.execute(f"UPDATE user_quests SET q{qid} = 1 WHERE user_id = ?", (user_id,))
                    cursor.execute("UPDATE users SET zombie_discount = zombie_discount + 10 WHERE user_id = ?", (user_id,))

                    await update.message.reply_text(
                        f"🏆 <b>Квест {qid} выполнен!</b>\n"
                        f"🎁 Вы получили <b>-10% скидку</b> на всех зомби!",
                        parse_mode="HTML"
                    )

        # Списание ресурсов
        cursor.execute(
            "UPDATE users SET diamonds = diamonds - ?, coins = coins - ? WHERE user_id = ?",
            (final_cost_d, final_cost_c, user_id)
        )

        conn.commit()
        conn.close()

        # Берём новую скидку
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("SELECT zombie_discount FROM users WHERE user_id = ?", (user_id,))
        discount = cursor.fetchone()[0]
        conn.close()

        await update.message.reply_text(
            f"✅ Куплено <b>{amount}</b> × <b>{zombie_info['name']}</b>\n"
            f"💎 Потрачено: {final_cost_d}\n"
            f"💰 Потрачено: {final_cost_c}\n"
            f"🎁 Ваша текущая скидка: {discount}%",
            parse_mode="HTML"
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при покупке зомби: {str(e)}")
        print("ERROR:", e)

from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import CallbackQueryHandler, CommandHandler, ContextTypes
import sqlite3


ZOMBIE_INFO[100] = {"name": "Уникальный Зомби", "hp": 0, "damage": 0}

# временные бонусы
user_temp_zombie_bonus = {}


async def craft_zombie_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id

    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute("""
        SELECT hp_bonus, damage_bonus
        FROM user_zombies
        WHERE user_id = ? AND zombie_id = 100
        """, (user_id,))
    row = cursor.fetchone()
    conn.close()

    if row:
        hp_b, dmg_b = row
        if hp_b != 0 or dmg_b != 0:
            await update.message.reply_text(
                "❌ У вас уже есть уникальный зомби! Сначала используйте его в атаке."
            )
            return

    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute("SELECT rubies FROM users WHERE user_id = ?", (user_id,))
    rubies = cursor.fetchone()
    conn.close()

    rubies = rubies[0] if rubies else 0
    if rubies < 1:
        await update.message.reply_text("❌ У вас нет рубинов для создания зомби.")
        return

    user_temp_zombie_bonus[user_id] = {"hp": 0, "damage": 0, "created": False}

    keyboard = [
        [InlineKeyboardButton("🏥 +3000 Здоровья (1 ♦️ рубин)", callback_data="add_hp")],
        [InlineKeyboardButton("💥 +600 Урона (1 ♦️ рубин)", callback_data="add_damage")],
        [InlineKeyboardButton("🎯 Создать Зомби", callback_data="create_zombie")]
    ]

    await update.message.reply_text(
        "🧟‍♂️ **Создание уникального зомби!** 🧟‍♀️\n\n"
        f"💎 У вас сейчас {rubies} рубинов.\n"
        "Выберите улучшения для вашего зомби, чтобы он был сильным на поле боя:\n\n"
        "🏥 Здоровье — увеличивает живучесть\n"
        "💥 Урон — повышает силу атаки\n\n"
        "Нажмите на кнопки ниже, чтобы прокачать вашего зомби:",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )

async def craft_zombie_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    user_id = query.from_user.id

    if user_id not in user_temp_zombie_bonus:
        await query.answer("❌ Временные данные не найдены.")
        return

    data = user_temp_zombie_bonus[user_id]

    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute("""
        SELECT hp_bonus, damage_bonus
        FROM user_zombies
        WHERE user_id = ? AND zombie_id = 100
    """, (user_id,))
    row = cursor.fetchone()
    conn.close()

    if row:
        hp_b, dmg_b = row
        if hp_b != 0 or dmg_b != 0:
            await query.answer("❌ У вас уже есть уникальный зомби! Сначала используйте его в атаке.")
            return

    if data["created"]:
        await query.answer("❌ Зомби уже создан!")
        return

    # ===== СПИСАНИЕ 1 РУБИНА НА +УРОН И +ХП =====
    if query.data in ("add_hp", "add_damage"):

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("SELECT rubies FROM users WHERE user_id = ?", (user_id,))
        rubies = cursor.fetchone()
        rubies = rubies[0] if rubies else 0

        if rubies < 1:
            conn.close()
            await query.answer("❌ Недостаточно рубинов!")
            return

        # списываем рубин
        cursor.execute("UPDATE users SET rubies = rubies - 1 WHERE user_id = ?", (user_id,))
        conn.commit()
        conn.close()

        # добавляем параметры
        if query.data == "add_hp":
            data["hp"] += 3000
            await query.answer("🏥 +3000 здоровья (−1 рубин)")
        else:
            data["damage"] += 600
            await query.answer("💥 +600 урона (−1 рубин)")

        # обновление текста сообщения
        await query.edit_message_text(
            f"Создание зомби:\n"
            f"🏥 Здоровье: {data['hp']}\n"
            f"💥 Урон: {data['damage']}",
            reply_markup=query.message.reply_markup
        )
        return

    # ===== СОЗДАНИЕ ЗОМБИ (БЕЗ СПИСАНИЯ РУБИНОВ!) =====
    if query.data == "create_zombie":

        if data["hp"] == 0 and data["damage"] == 0:
            await query.answer("❌ Сначала выберите хотя бы одно улучшение (+Здоровье или +Урон)!")
            return

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # сама запись зомби
        cursor.execute("""
            INSERT INTO user_zombies (user_id, zombie_id, quantity, hp_bonus, damage_bonus)
            VALUES (?, 100, 1, ?, ?)
            ON CONFLICT(user_id, zombie_id) DO UPDATE SET
                quantity = quantity + 1,
                hp_bonus = hp_bonus + ?,
                damage_bonus = damage_bonus + ?
        """, (user_id, data["hp"], data["damage"], data["hp"], data["damage"]))

        conn.commit()
        conn.close()

        data["created"] = True

        user_temp_zombie_bonus[user_id] = {"hp": 0, "damage": 0, "created": False}

        await query.answer("🎉 Зомби создан!")
        await query.edit_message_text(
            f"🎉 Ваш уникальный зомби создан!\n"
            f"🏥 Здоровье: {data['hp']}\n"
            f"💥 Урон: {data['damage']}"
        )

# === Бонусы снаряжений для зомби ===
# Эти словари можно положить в начало файла, чтобы использовать их в любой части кода

# Урон мечей
SWORD_DAMAGE = {
    1: 90,   # деревянный меч
    2: 99,   # каменный
    3: 108,  # железный
    4: 120,  # титановый
    5: 264,  # алмазный
    6: 825,  # рубиновый
}

# Дополнительное здоровье от брони
ARMOR_HP = {
    7: 270,   # деревянная броня
    8: 297,   # каменная
    9: 327,   # железная
    10: 359,  # титан
    11: 790,  # алмаз
    12: 2470,  # рубин
}

# Временное хранение сообщений атак
ATTACK_MESSAGES = {}
async def run_attack_task(attacker_id, attacker_name, target_player_id, target_name,
                          zombie_id, zombie_count, equipment_id, context, attack_message=None):
    try:
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        initial_count = zombie_count   # ← ЭТО ДОБАВИТЬ

        # === Скорость атаки ===
        base_duration = 60.0
        if zombie_id == 2:
            duration = base_duration / 10.0
        elif zombie_id in (3, 8):
            duration = base_duration / 2.0
        elif zombie_id == 30:  # Король обезъян — ×2
            duration = base_duration / 2.0
        elif zombie_id in (31, 32, 33, 34, 35):  # остальные ×3
            duration = base_duration / 3.0
        elif zombie_id == 15:
            duration = base_duration / 15.0
        elif zombie_id == 4:
            duration = base_duration / 15.0
        elif zombie_id == 11:
            duration = base_duration / 4.0
        elif zombie_id == 10:
            duration = base_duration / 20.0
        elif zombie_id == 9:
            duration = base_duration / 10.0
        elif zombie_id == 6:
             duration = base_duration / 3.0
        elif zombie_id == 13:  # Босс
            duration = base_duration / 10.0
        elif zombie_id == 14:  # Элитный разведчик
            duration = base_duration / 60.0
        elif zombie_id == 16:  # Вирус
            duration = base_duration / 25.0
        elif zombie_id == 18:
            duration = base_duration / 30.0
        elif zombie_id == 12:  # Зайка
            duration = base_duration / 2.0
        elif zombie_id == 17:
            duration = base_duration / 2.0
        elif zombie_id == 19:
            duration = base_duration / 5.0
        elif zombie_id == 20:
            duration = base_duration / 3.0
        elif zombie_id == 21:  # Оливье 2025
            duration = base_duration / 5.0
        elif zombie_id == 22:  # Гнилой Санта-Клаус
            duration = base_duration / 4.0
        elif zombie_id == 23:  # Олень
            duration = base_duration / 6.0
        elif zombie_id == 24:  # Жнец
            duration = base_duration / 20.0
        elif zombie_id == 30:  # Жнец
            duration = base_duration / 2.0
        elif zombie_id == 26:  # Обезьяна
            duration = base_duration / 9.0
        elif zombie_id == 27:  # Сапёр
            duration = base_duration / 5.0
        elif zombie_id == 28:  # Медведь
            duration = base_duration / 7.0
        elif zombie_id == 29:  # Ведьма
            duration = base_duration / 5.0
        elif zombie_id == 36:
            duration = base_duration / 3.0
        elif zombie_id == 37:
            duration = base_duration / 60.0
        elif zombie_id == 38:
            duration = base_duration / 60.0
        elif zombie_id == 25:  # Свиньи в броне
            duration = base_duration / 8.0
        else:
            duration = base_duration

        # Проверка баффа Командора (увеличивает скорость других зомби)
        cursor.execute("SELECT commander_speed_until FROM users WHERE player_id = ?", (target_player_id,))
        row = cursor.fetchone()
        commander_buff_active = row and row[0] and int(row[0]) > int(time.time())

        speed_multiplier = 1.0
        if commander_buff_active:
            speed_multiplier = 2.0  # удваиваем скорость зомби

        # === После вычисления duration ===
        duration /= speed_multiplier

        # === Получаем Telegram ID цели ===
        cursor.execute("SELECT user_id FROM users WHERE player_id = ?", (target_player_id,))
        row = cursor.fetchone()
        target_user_id = row[0] if row else None

        cursor.execute("SELECT player_id FROM users WHERE user_id = ?", (attacker_id,))
        row = cursor.fetchone()
        attacker_player_id = row[0] if row else "?"

        # 🦎 Если Хамелеон — маскируем под Скалу (ID 7)
        display_zombie_id = 7 if zombie_id == 15 else zombie_id

        attack_message = ATTACK_MESSAGES.pop(
            (attacker_id, target_player_id),
            None
        )

        # 📡 Если Элитный разведчик — не отправляем уведомление жертве
        if target_user_id and zombie_id != 14:
            try:
                # Проверяем маску цели (через таблицу user_masks)
                cursor.execute("SELECT 1 FROM user_masks WHERE user_id = ? AND mask_id = 12", (target_user_id,))
                has_detection_mask = cursor.fetchone() is not None  # 🎭 Маска "Обнаружение"

                attacker_info = ""
                if has_detection_mask:
                    attacker_info = (
                        f"\n\n🕵️‍♂️ <b>Обнаружен атакующий:</b>\n"
                        f"🆔 <code>{attacker_player_id}</code>\n"
                        f"👤 <b>{attacker_name}</b>"
                    )

                await context.bot.send_message(
                    chat_id=target_user_id,

                    text=(
                        "⚠️ <b>Внимание!</b>\n"
                        f"На вашу ферму направляется орда: "
                        f"<b>{ZOMBIE_INFO.get(display_zombie_id, {}).get('name', 'Неизвестный')}</b> × {zombie_count}!"
                        f"{attacker_info}"
                        + (f"\n\n💬 <b>Сообщение атакующего:</b>\n{attack_message}" if attack_message else "")
                    ),
                    parse_mode="HTML"
                )
                print(f"[✅ Уведомление отправлено игроку {target_user_id}]")
            except Exception as e:
                print(f"[⚠️ Уведомление об атаке не отправлено] {e}")

        # === Прогресс-бар атаки ===
        prep_msg = await context.bot.send_message(
            chat_id=attacker_id,
            text=(
                f"🧭 <b>Атака запущена</b>\n"
                f"🧟 {ZOMBIE_INFO[zombie_id]['name']} × {zombie_count} движутся к ферме <b>{target_name}</b>..."
            ),
            parse_mode="HTML"
        )

        steps = 10
        delay = duration / steps
        for i in range(steps):
            bar = "█" * (i + 1) + "░" * (steps - i - 1)
            pct = (i + 1) * 10
            try:
                await context.bot.edit_message_text(
                    chat_id=prep_msg.chat.id,
                    message_id=prep_msg.message_id,
                    text=f"🚶‍♂️ Зомби движутся: [{bar}] {pct}%"
                )
            except Exception:
                pass
            await asyncio.sleep(delay)

        await context.bot.edit_message_text(
            chat_id=prep_msg.chat.id,
            message_id=prep_msg.message_id,
            text="💥 Зомби начали атаку на ферму!"
        )

        cursor.execute("""
            SELECT new_farm_health, max_farm_health, strawberries, tomatoes, diamonds, farms_destroyed,
                   turret_wood, turret_stone, turret_iron, turret_titanium, turret_rocket,
                   tower_fire, tower_poison, tower_magic, tower_crossbow, tower_curse
            FROM users WHERE player_id = ?
        """, (target_player_id,))
        farm_row = cursor.fetchone()

        # === Получаем данные цели ===
        cursor.execute("""
            SELECT new_farm_health, max_farm_health, strawberries, tomatoes, diamonds, farms_destroyed,
                   turret_wood, turret_stone, turret_iron, turret_titanium, turret_rocket,
                   tower_fire, tower_poison, tower_magic, tower_crossbow, tower_curse
            FROM users WHERE player_id = ?
        """, (target_player_id,))
        farm_row = cursor.fetchone()

        # === SANITY: безопасная инициализация полей фермы, турелей и башен ===
        if not farm_row:
            new_farm_health = max_farm_health = 100
            strawberries = tomatoes = diamonds = farms_destroyed = 0
            turret_counts = (0, 0, 0, 0, 0)
            tower_fire = tower_poison = tower_magic = tower_crossbow = tower_curse = 0
        else:
            (new_farm_health, max_farm_health, strawberries, tomatoes, diamonds, farms_destroyed,
             turret_wood, turret_stone, turret_iron, turret_titanium, turret_rocket,
             tower_fire, tower_poison, tower_magic, tower_crossbow, tower_curse) = farm_row

            # приводим None -> 0 и гарантируем int для турелей
            turret_counts = (
                int(turret_wood or 0),
                int(turret_stone or 0),
                int(turret_iron or 0),
                int(turret_titanium or 0),
                int(turret_rocket or 0),
            )

            tower_fire = int(tower_fire or 0)
            tower_poison = int(tower_poison or 0)
            tower_magic = int(tower_magic or 0)
            tower_crossbow = int(tower_crossbow or 0)
            tower_curse = int(tower_curse or 0)

            if new_farm_health is None:
                new_farm_health = 100
            if max_farm_health is None:
                max_farm_health = 100

# Теперь turret_counts гарантированно кортеж из int, никаких None

        # === Получаем информацию о зомби (безопасно) ===
        zombie_info = ZOMBIE_INFO.get(zombie_id, {"hp": 1, "damage": 0, "name": "Неизвестный"})

        armor_bonus = int(ARMOR_HP.get(equipment_id, 0) or 0)
        sword_bonus = int(SWORD_DAMAGE.get(equipment_id, 0) or 0)

        bonus_text = ""

        zombie_hp = int(zombie_info.get("hp", 1) or 1) + armor_bonus
        base_damage = int(zombie_info.get("damage", 0) or 0) + sword_bonus

        # === БОНУС УРОНА ОТ РАНГА АТАКУЮЩЕГО ===
        cursor.execute("SELECT points FROM users WHERE user_id = ?", (attacker_id,))
        row = cursor.fetchone()
        points = row[0] if row else 0

        _, _, rank_number = get_rank(points)
        rank_damage_bonus = get_rank_damage_bonus(rank_number)

        if rank_damage_bonus > 0:
            bonus_damage = int(base_damage * rank_damage_bonus / 100)
            base_damage += bonus_damage
            bonus_text += (
                f"🏅 Ранг атакующего: +{rank_damage_bonus}% урона "
                f"(+{bonus_damage})\n"
            )

        total_zombie_hp = zombie_hp * zombie_count
        total_zombie_dmg = base_damage * zombie_count

        zombie_hp_for_turrets = zombie_hp
        total_zombie_hp_for_turrets = zombie_hp_for_turrets * zombie_count
        print(f"[DEBUG START] zombie_id={zombie_id}, count={zombie_count}, hp={zombie_hp}, dmg={base_damage}, total_hp={total_zombie_hp}")

        if zombie_count > 0:  # ← защита от нулевого count
            if zombie_id == 3 and random.random() <= 0.5:
                base_damage *= 2
                bonus_text += "💣 Взрывун нанес двойной урон!\n"

            if zombie_id == 8 and random.random() <= 0.5:
                base_damage *= 2
                bonus_text += "💥 Бомбстер нанес двойной урон!\n"

            if zombie_id == 31 and random.random() <= 0.9:
                base_damage *= 2
                bonus_text += "⚔️ Мегарыцарь нанес двойной урон!\n"

            if zombie_id == 36:  # Всадник на кабане
                if random.random() <= 0.03:  # шанс 3%
                    towers = ["tower_fire", "tower_poison", "tower_magic", "tower_crossbow", "tower_curse"]
                    available = [t for t in towers if locals()[t] > 0]
                    if available:
                        chosen = random.choice(available)
                        locals()[chosen] = 0
                        bonus_text += f"💥 Всадник сломал {chosen}!\n"

            if zombie_id == 37:  # Лихварь алмазов
                stolen = max(1, int(diamonds * 0.01))
                diamonds -= stolen
                cursor.execute("UPDATE users SET diamonds = diamonds - ? WHERE player_id = ?", (stolen, target_player_id))
                cursor.execute("UPDATE users SET diamonds = diamonds + ? WHERE user_id = ?", (stolen, attacker_id))
                bonus_text += f"💎 Лихварь украл {stolen} алмазов!\n"

            if zombie_id == 38:  # Лихварь монет
                stolen = max(1, int(coins * 0.01))
                coins -= stolen
                cursor.execute("UPDATE users SET coins = coins - ? WHERE player_id = ?", (stolen, target_player_id))
                cursor.execute("UPDATE users SET coins = coins + ? WHERE user_id = ?", (stolen, attacker_id))
                bonus_text += f"💰 Лихварь украл {stolen} монет!\n"

            total_zombie_dmg = base_damage * zombie_count

        tower_text = ""

        # === Урон от башен защиты (ЛЮБАЯ башня = 3000 урона) ===
        print(f"[DEBUG] BEFORE TOWERS: count={zombie_count}, total_hp={total_zombie_hp}")
        if zombie_count > 0 and total_zombie_hp > 0:
            total_towers = (
                tower_fire +
                tower_poison +
                tower_magic +
                tower_crossbow +
                tower_curse
             )

            if total_towers > 0:
                tower_damage = total_towers * 300
                total_zombie_hp -= tower_damage

                bonus_text += (
                    f"🏰 Башни защиты нанесли {tower_damage} урона зомби "
                    f"(башен: {total_towers})!\n"
                )

                # если башни убили всех зомби
                if total_zombie_hp <= 0:
                    zombie_count = 0
                    total_zombie_dmg = 0
                else:
                    zombie_count = max(1, total_zombie_hp // zombie_hp) if zombie_count > 0 else 0
                    print(f"[DEBUG] AFTER TOWERS: count={zombie_count}, total_hp={total_zombie_hp}")
                    total_zombie_dmg = zombie_count * base_damage

        # === Инициализация башен защиты (без проверки farm_row) ===
        tower_fire = tower_poison = tower_magic = tower_crossbow = tower_curse = 0
        if farm_row:
            tower_fire, tower_poison, tower_magic, tower_crossbow, tower_curse = map(int, farm_row[11:16])

        # === Эффекты башен ===
        if tower_fire and zombie_count > 0:
            if random.random() <= 0.15:
                zombie_count -= 1
                total_zombie_dmg = base_damage * zombie_count
                tower_text += "🔥 Огненная башня испепелила 1 зомби!\n"

        if tower_poison and zombie_count > 0:
            poison_damage_per_zombie = 150
            extra_damage = poison_damage_per_zombie * zombie_count
            total_zombie_dmg += extra_damage
            zombie_hp -= poison_damage_per_zombie  # уменьшаем HP каждого зомби
            tower_text += f"☣️ Ядовитая башня нанесла {extra_damage} урона ({poison_damage_per_zombie} на зомби)!\n"

        if tower_magic and zombie_count > 0:
            base_damage = int(base_damage * 0.7)  # -30% урона зомби
            total_zombie_dmg = base_damage * zombie_count
            tower_text += "✨ Магическая башня ослабила зомби — их урон снижен на 30%!\n"

        if tower_crossbow and zombie_count > 0:
            extra_damage = 0
            for _ in range(zombie_count):
                if random.random() <= 0.5:
                    extra_damage += 300
            total_zombie_dmg += extra_damage
            if extra_damage > 0:
                tower_text += f"🎯 Арбалетная башня нанесла {extra_damage} урона!\n"

        if tower_curse and zombie_count > 0:
            zombie_hp = int(zombie_hp * 0.75)  # влияет только на урон зомби, не на turret_damage
            total_zombie_dmg = base_damage * zombie_count
            tower_text += "🩸 Проклинающая башня снизила HP зомби на 25%!\n"

        # Пересчёт hp для турелей после эффектов
            zombie_hp_for_turrets = zombie_hp
            total_zombie_hp_for_turrets = zombie_hp_for_turrets * zombie_count

            print(f"[DEBUG] AFTER TOWER EFFECTS: count={zombie_count}, hp={zombie_hp}, hp_for_turrets={zombie_hp_for_turrets}, total_hp_for_turrets={total_zombie_hp_for_turrets}")

            if zombie_count <= 0:
                print("[DEBUG] ЗОМБИ УМЕРЛИ ОТ БАШЕН И ЭФФЕКТОВ")
                bonus_text += "🪦 Все зомби погибли от башен и их эффектов — дальше не прошли.\n"

        cursor.execute("SELECT slime_effect_until FROM users WHERE player_id = ?", (target_player_id,))
        row = cursor.fetchone()
        slime_active = row and row[0] and int(row[0]) > int(time.time())

        if slime_active and zombie_id != 19:
            duration *= 2      # замедление
            zombie_hp = int(zombie_hp * 1.5)

        cursor.execute("SELECT olivie_effect_until FROM users WHERE player_id = ?", (target_player_id,))
        row = cursor.fetchone()
        olivie_active = row and row[0] and int(row[0]) > int(time.time())

        if olivie_active and zombie_id != 21:
            duration *= 4      # супер замедление
            zombie_hp = int(zombie_hp * 2.0)   # х2 здоровье

        # zombie_count безопасный
        if not isinstance(zombie_count, int) or zombie_count < 0:
            zombie_count = 0

        if zombie_id == 30:  # Король обезъян — проклятие до следующего сбора
            cursor.execute("""
                UPDATE users
                SET farm_income_curse_active = 1,
                    farm_income_curse_mult = 0.5
                WHERE player_id = ?
            """, (target_player_id,))
            conn.commit()
            bonus_text += "🐒 Король обезъян проклял доход! Следующий сбор будет ×0.5\n"

        if zombie_id == 32:  # Королева обезьян — запрет сбора 1 час
            block_end = int(time.time()) + 3600
            cursor.execute("""
                UPDATE users
                SET harvest_blocked_until = ?
                WHERE player_id = ?
            """, (block_end, target_player_id))
            conn.commit()
            bonus_text += "👑 Королева обезьян запретила собирать урожай на 1 час!\n"

        if zombie_id == 33:
            now = int(time.time())
            cursor.execute("""
                INSERT INTO active_princess_effects
                (target_player_id, attacker_player_id, start_time, end_time)
                VALUES (?, ?, ?, ?)
            """, (
                target_player_id,
                attacker_player_id,
                now,
                now + 3600
            ))
            conn.commit()
            bonus_text += "👑 Принцесса прокляла ферму — 15 урона в час!\n"

        if zombie_id == 34:  # Колдун
            cursor.execute("""
                SELECT id
                FROM active_princess_effects
                WHERE target_player_id = ?
                ORDER BY start_time ASC
                LIMIT 1
            """, (target_player_id,))
            row = cursor.fetchone()

            if row:
                cursor.execute(
                    "DELETE FROM active_princess_effects WHERE id = ?",
                    (row[0],)
                )
                conn.commit()
                bonus_text += "🧙‍♂️ Колдун уничтожил одну Принцессу — её урон прекращён!\n"
            else:
                bonus_text += "🧙‍♂️ Колдун искал Принцессу, но не нашёл ни одной.\n"

        if zombie_id == 35:  # Ледяной маг
            reduce_until = int(time.time()) + 3600  # 1 час

            cursor.execute("""
                UPDATE users
                SET incoming_damage_mult = 0.5,
                    incoming_damage_reduced_until = ?
                WHERE player_id = ?
            """, (reduce_until, target_player_id))

            conn.commit()
            bonus_text += "❄️ Ледяной маг заморозил врага! Входящий урон снижен в 2 раза на 1 час.\n"

        if zombie_id == 28:
            cursor.execute("""
            UPDATE users
            SET mine_trap = MAX(mine_trap - 5, 0)
            WHERE player_id = ?
            """, (target_player_id,))
            conn.commit()
            bonus_text += "🐻 Медведь разломал до 5 каменных капканов!\n"

        if zombie_id == 29:
            factories = ["sawmill", "quarry", "smelter", "factory", "mine"]
            names = {
                "sawmill": "Лесопилку",
                "quarry": "Каменоломню",
                "smelter": "Плавильню",
                "factory": "Фабрику",
                "mine": "Карьер"
            }

            cursor.execute(
                f"SELECT {', '.join(factories)} FROM users WHERE player_id = ?",
                (target_player_id,)
            )
            row = cursor.fetchone()

            if row:
                for fac, count in zip(factories, row):
                    if count and count > 0:
                        destroy = min(zombie_count, count)

                        cursor.execute(
                            f"""
                            UPDATE users
                            SET {fac} = MAX({fac} - ?, 0)
                            WHERE player_id = ?
                            """,
                            (destroy, target_player_id)
                        )

                        bonus_text += f"🧙‍♀️ Ведьмы уничтожили {destroy} {names[fac]}!\n"
        if zombie_id == 27:
            cursor.execute("""
            UPDATE users
            SET
                mine_iron = MAX(mine_iron - 7, 0),
                mine_sea  = MAX(mine_sea  - 7, 0)
            WHERE player_id = ?
            """, (target_player_id,))
            conn.commit()
            bonus_text += "💣 Сапёр уничтожил до 7 мин (железо и морские)!\n"
        if zombie_id == 10:  # Горбатый ворует монеты
            coins_to_steal = 6900 * zombie_count
            cursor.execute("SELECT coins FROM users WHERE player_id = ?", (target_player_id,))
            row = cursor.fetchone()
            target_coins = max(row[0], 0) if row else 0  # игнорируем отрицательные значения
            stolen_coins = min(coins_to_steal, target_coins)
            if stolen_coins > 0:
                cursor.execute("UPDATE users SET coins = coins - ? WHERE player_id = ?", (stolen_coins, target_player_id))
                cursor.execute("UPDATE users SET coins = coins + ? WHERE user_id = ?", (stolen_coins, attacker_id))
                bonus_text += f"💰 Горбатый украл {stolen_coins} монет!\n"
        if zombie_id == 100:
            cursor.execute("""
                SELECT hp_bonus, damage_bonus
                FROM user_zombies
                WHERE user_id = ? AND zombie_id = 100
            """, (attacker_id,))
            row = cursor.fetchone()

            if row:
                hp_bonus, damage_bonus = row

               # + бонус к хп
                zombie_hp += int(hp_bonus or 0)

                # + бонус к урону
                base_damage += int(damage_bonus or 0)

        total_zombie_hp = zombie_hp * zombie_count
        total_zombie_dmg = base_damage * zombie_count

        if zombie_id == 13:
            factories = ["sawmill", "quarry", "smelter", "factory", "mine"]
            total_destroy = 30 * zombie_count   # 🔥 УЧЁТ КОЛИЧЕСТВА БОССОВ
            destroyed_list = []

            cursor.execute(f"""
                SELECT {", ".join(factories)}
                FROM users WHERE player_id = ?
            """, (target_player_id,))
            row = cursor.fetchone()

            if row:
                factory_counts = dict(zip(factories, row))
            else:
                factory_counts = {f: 0 for f in factories}

            for fac in factories:
                if total_destroy <= 0:
                    break

                available = int(factory_counts.get(fac, 0) or 0)
                if available <= 0:
                    continue

                to_destroy = min(available, total_destroy)

                cursor.execute(
                    f"""
                    UPDATE users
                    SET {fac} = MAX({fac} - ?, 0)
                    WHERE player_id = ?
                    """,
                    (to_destroy, target_player_id)
                )

                destroyed_list.append((fac, to_destroy))
                total_destroy -= to_destroy

            if destroyed_list:
                bonus_text += "💥 <b>Босс разрушил постройки:</b>\n"
                names = {
                    "sawmill": "Лесопилка",
                    "quarry": "Каменоломня",
                    "smelter": "Плавильня",
                    "factory": "Фабрика",
                    "mine": "Карьер"
                }
                for fac, count in destroyed_list:
                    bonus_text += f"• {names[fac]}: −{count}\n"
            else:
                bonus_text += "😐 Босс пришёл, но у цели не было ни одной фабрики.\n"
        if zombie_id == 11:  # Крампус ворует алмазы
            diamonds_to_steal = 127 * zombie_count
            cursor.execute("SELECT diamonds FROM users WHERE player_id = ?", (target_player_id,))
            row = cursor.fetchone()
            target_diamonds = max(row[0], 0) if row else 0  # игнорируем отрицательные значения
            stolen_diamonds = min(diamonds_to_steal, target_diamonds)
            if stolen_diamonds > 0:
                cursor.execute("UPDATE users SET diamonds = diamonds - ? WHERE player_id = ?", (stolen_diamonds, target_player_id))
                cursor.execute("UPDATE users SET diamonds = diamonds + ? WHERE user_id = ?", (stolen_diamonds, attacker_id))
                bonus_text += f"💎 Крампус украл {stolen_diamonds} алмазов!\n"
        if zombie_id == 20:  # Берсерк — если турели нанесли урон, но не убили
            berserk_buff = False
        if zombie_id == 22:
            resources = ["wood", "stone", "iron", "titanium"]
            stolen_total = 0
            stolen_text = ""

            for res in resources:
                cursor.execute(f"SELECT {res} FROM users WHERE player_id = ?", (target_player_id,))
                val = cursor.fetchone()[0] or 0
                steal = min(4500 * zombie_count, val)
                if steal > 0:
                    cursor.execute(f"UPDATE users SET {res} = {res} - ? WHERE player_id = ?", (steal, target_player_id))
                    cursor.execute(f"UPDATE users SET {res} = {res} + ? WHERE user_id = ?", (steal, attacker_id))
                    stolen_total += steal
                    stolen_text += f"• {res}: {steal}\n"

            if stolen_total > 0:
                bonus_text += "🎅 Гнилой Санта украл ресурсы:\n" + stolen_text
        if zombie_id == 23:
            # 1 олень = 5 деревянных турелей
            destroy_limit = zombie_count * 5

            destroyed = min(turret_wood, destroy_limit)

            if destroyed > 0:
                cursor.execute(
                    "UPDATE users SET turret_wood = turret_wood - ? WHERE player_id = ?",
                    (destroyed, target_player_id)
                )
                bonus_text += f"🦌 Олени уничтожили {destroyed} деревянных турелей!\n"
        if zombie_id == 24:  # Жнец
            reap = int(new_farm_health * 0.01)

            new_farm_health -= reap
            if new_farm_health < 0:
                new_farm_health = 0

            # Урон Жнеца прибавляется к общему
            total_zombie_dmg += reap

            bonus_text += (
                f"⚰️ Жнец пожинает {reap} здоровья фермы "
                f"(1% от текущего здоровья)!\n"
            )
        if zombie_id == 25 and random.random() <= 0.5:
            steal_straw = min(430 * zombie_count, strawberries)
            steal_tom = min(792 * zombie_count, tomatoes)

            cursor.execute("""
                UPDATE users
                SET strawberries = strawberries - ?, tomatoes = tomatoes - ?
                WHERE player_id = ?
            """, (steal_straw, steal_tom, target_player_id))

            cursor.execute("""
                UPDATE users
                SET strawberries = strawberries + ?, tomatoes = tomatoes + ?
                WHERE user_id = ?
            """, (steal_straw, steal_tom, attacker_id))

            bonus_text += f"🐷 Свиньи украли:\n🍓 {steal_straw}\n🍅 {steal_tom}\n"
        if zombie_id == 19:
            slime_end = int(time.time()) + 300  # 5 минут
            cursor.execute(
                "UPDATE users SET slime_effect_until = ? WHERE player_id = ?",
                (slime_end, target_player_id)
            )
            conn.commit()
            bonus_text += "🟢 Слизень активировал эффект!"
        if zombie_id == 21:
            olivie_end = int(time.time()) + 600  # 10 минут
            cursor.execute(
                "UPDATE users SET olivie_effect_until = ? WHERE player_id = ?",
                (olivie_end, target_player_id)
            )
            conn.commit()
            bonus_text += "🥗 Оливье активировал эффект!"
        if zombie_id == 18:  # Командор
            cursor.execute(
                "UPDATE users SET commander_speed_until = ? WHERE player_id = ?",
                (int(time.time()) + 60, target_player_id)
            )
            bonus_text += "⚡ Командор пал, но вдохновил армию! Все зомби атакуют в 2 раза быстрее 60 секунд!\n"
        if zombie_id == 17:
            shock_end = time.time() + 30
            cursor.execute("UPDATE users SET shock_until = ? WHERE player_id = ?", (shock_end, target_player_id))
            conn.commit()
            bonus_text += "⚡ Ферма шокирована на 15 секунд — турели наносят в 2 раза меньше урона!\n"
        if zombie_id == 16:
            infection_end = time.time() + 60
            cursor.execute("""
                UPDATE users SET infection_until = ? WHERE player_id = ?
            """, (infection_end, target_player_id))
            conn.commit()
            bonus_text += "☣️ Ферма заражена на 60 секунд! Все атаки наносят ×2 урон.\n"
        if zombie_id == 12:
            turrets = ["turret_wood", "turret_stone", "turret_iron", "turret_titanium", "turret_rocket"]

            cursor.execute(
                f"SELECT {', '.join(turrets)} FROM users WHERE player_id = ?",
                (target_player_id,)
            )
            row = cursor.fetchone()

            if row:
                for turret_name, count in zip(turrets, row):
                    destroy = min(count, zombie_count)  # ← ВАЖНО

                    if destroy > 0:
                        cursor.execute(
                            f"UPDATE users SET {turret_name} = {turret_name} - ? WHERE player_id = ?",
                            (destroy, target_player_id)
                        )
                        bonus_text += f"💥 Зайки уничтожили {destroy} турелей: {turret_name}\n"

                conn.commit()

        # === Урон турелей ===
        print(f"[DEBUG] BEFORE TURRETS: count={zombie_count}, turret_damage={turret_damage if 'turret_damage' in locals() else 'ещё не посчитано'}, hp_for_turrets={zombie_hp_for_turrets}")
        turret_values = [30, 50, 70, 150, 1300]
        turret_damage = sum(count * dmg for count, dmg in zip(turret_counts, turret_values))

        if zombie_id == 26:
            turret_damage = 0
            bonus_text += "🐵 Обезьяна обезвредила все турели!\n"

        # === Проверка, есть ли эффект шока ===
        cursor.execute("SELECT shock_until FROM users WHERE player_id = ?", (target_player_id,))
        row = cursor.fetchone()
        shocked = False
        if row and row[0] and time.time() < row[0]:
            shocked = True
            original_turret_damage = turret_damage
            turret_damage = int(turret_damage * 0.5)
            bonus_text += f"⚡ Ферма под шоком — урон турелей снижен с {original_turret_damage} до {turret_damage}.\n"

        if turret_damage > 0:
            if turret_damage >= total_zombie_hp_for_turrets:
                zombie_count = 0
                total_zombie_dmg = 0
                bonus_text += f"🛡 Турели уничтожили всех зомби! ({turret_damage} урона)\n"
            else:
                killed = turret_damage // zombie_hp_for_turrets
                zombie_count = max(0, zombie_count - killed)
                total_zombie_dmg = base_damage * zombie_count
                bonus_text += f"🛡 Турели нанесли {turret_damage} урона — уничтожено {killed} зомби.\n"

        if zombie_id == 20 and zombie_count > 0 and turret_damage > 0:
                base_damage *= 3
                total_zombie_dmg = base_damage * zombie_count
                bonus_text += "🔥 Берсерк разъярён! Его урон увеличен в 3 раза!\n"

        print(f"[DEBUG] AFTER TURRETS: count={zombie_count}, total_dmg={total_zombie_dmg}")

        # === Мины (капкан / мина / морская мина) ===
        print(f"[DEBUG] BEFORE MINES: count={zombie_count}")
        cursor.execute("""
            SELECT mine_trap, mine_iron, mine_sea
            FROM users WHERE player_id = ?
        """, (target_player_id,))
        mine_row = cursor.fetchone()

        trap_count = int(mine_row[0] or 0)   # Капкан (камень)
        iron_count = int(mine_row[1] or 0)   # Мина (железо)
        sea_count = int(mine_row[2] or 0)    # Морская мина (титан)

        # Урон каждой мины
        DMG_TRAP = 5
        DMG_IRON = 18
        DMG_SEA = 75

        total_mine_damage = (
            trap_count * DMG_TRAP +
            iron_count * DMG_IRON +
            sea_count * DMG_SEA
        )

        if zombie_id == 26:
            trap_count = 0
            iron_count = 0
            sea_count = 0

        mine_text = ""

        if zombie_id != 26 and total_mine_damage > 0 and zombie_count > 0:

            # Сколько зомби убьют мины:
            killed_by_mines = min(zombie_count, total_mine_damage // zombie_hp)

            if killed_by_mines > 0:
                mine_text += f"💥 Мины уничтожили {killed_by_mines} зомби!\n"

                # уменьшаем количество зомби
                zombie_count -= killed_by_mines

                # --- Определяем сколько мин взорвалось ---
                damage_to_spend = killed_by_mines * zombie_hp

                # капканы
                used_trap = min(trap_count, (damage_to_spend + DMG_TRAP - 1) // DMG_TRAP)
                damage_to_spend -= used_trap * DMG_TRAP

                # железные
                used_iron = min(iron_count, (damage_to_spend + DMG_IRON - 1) // DMG_IRON)
                damage_to_spend -= used_iron * DMG_IRON

                # морские
                used_sea = min(sea_count, (damage_to_spend + DMG_SEA - 1) // DMG_SEA)
                damage_to_spend -= used_sea * DMG_SEA

                # Если всех уничтожили мины:
                if zombie_count <= 0:
                    mine_text += "🔥 Все зомби уничтожены минами!\n"
                    total_zombie_dmg = 0
                else:
                    total_zombie_hp = zombie_count * zombie_hp
                    total_zombie_dmg = zombie_count * base_damage

                # --- Списываем использованные мины ---
                cursor.execute("""
                    UPDATE users
                    SET mine_trap = mine_trap - ?,
                        mine_iron = mine_iron - ?,
                        mine_sea = mine_sea - ?
                    WHERE player_id = ?
                """, (used_trap, used_iron, used_sea, target_player_id))
                conn.commit()

            bonus_text += mine_text

        print(f"[DEBUG] AFTER MINES: count={zombie_count}")

        # === Ответный урон от кольев ===
        print(f"[DEBUG] BEFORE PALISADE: count={zombie_count}")
        cursor.execute("SELECT wooden_palisade, iron_palisade FROM users WHERE player_id = ?", (target_player_id,))
        palisade_row = cursor.fetchone()
        if palisade_row:
            wooden_palisade = int(palisade_row[0] or 0)
            iron_palisade = int(palisade_row[1] or 0)
        else:
            wooden_palisade = iron_palisade = 0

        wooden_dmg = wooden_palisade * 12
        iron_dmg = iron_palisade * 44
        total_palisade_damage = wooden_dmg + iron_dmg

        if zombie_id == 7:
            total_palisade_damage *= 10
            bonus_text += "🪨 Скала напоролся на колья — урон ×10!\n"  # было ×5, но в коде ×10 — оставил как есть

        if total_palisade_damage > 0 and zombie_count > 0:
            if total_palisade_damage >= total_zombie_hp:
                zombie_count = 0
                total_zombie_dmg = 0
                bonus_text += "💀 Все зомби погибли, напоровшись на колья!\n"
            else:
                killed_by_palisade = total_palisade_damage // zombie_hp
                if killed_by_palisade > 0:
                    zombie_count = max(0, zombie_count - killed_by_palisade)
                    total_zombie_dmg = base_damage * zombie_count
                    bonus_text += f"💀 Колья уничтожили {killed_by_palisade} зомби.\n"
                # если killed_by_palisade == 0 — ничего не пишем

        # === Проверка заражения фермы ===
        cursor.execute("SELECT infection_until FROM users WHERE player_id = ?", (target_player_id,))
        row = cursor.fetchone()
        if row and row[0] and time.time() < row[0]:
            total_zombie_dmg *= 2
            bonus_text += "☣️ Ферма заражена — урон увеличен!\n"

        # === Эффект Ледяного мага (снижение входящего урона) ===
        cursor.execute(
            "SELECT incoming_damage_reduced_until FROM users WHERE player_id = ?",
            (target_player_id,)
        )
        row = cursor.fetchone()

        if row and row[0] and time.time() < row[0]:
            original_dmg = total_zombie_dmg
            total_zombie_dmg = total_zombie_dmg // 2
            bonus_text += f"❄️ Ледяной эффект: урон снижен с {original_dmg} до {total_zombie_dmg}.\n"

        # === Масштабируемое воровство ресурсов ===
        if total_zombie_dmg > 0 and new_farm_health > 0:

            damage_percent = (total_zombie_dmg / max_farm_health) * 100

            steal_percent = 0.0

            if damage_percent >= 99:
                steal_percent = 0.25
            elif damage_percent >= 90:
                steal_percent = 0.25
            elif damage_percent >= 80:
                steal_percent = 0.24
            elif damage_percent >= 70:
                steal_percent = 0.22
            elif damage_percent >= 60:
                steal_percent = 0.20
            elif damage_percent >= 50:
                steal_percent = 0.17
            elif damage_percent >= 40:
                steal_percent = 0.135
            elif damage_percent >= 30:
                steal_percent = 0.10
            elif damage_percent >= 20:
                steal_percent = 0.07
            elif damage_percent >= 10:
                steal_percent = 0.038
            elif damage_percent >= 5:
                steal_percent = 0.02
            elif damage_percent >= 3:
                steal_percent = 0.013
            elif damage_percent >= 1:
                steal_percent = 0.005

            if steal_percent > 0:
                stolen_strawberries = int(strawberries * steal_percent)
                stolen_tomatoes = int(tomatoes * steal_percent)

                stolen_strawberries = min(stolen_strawberries, strawberries)
                stolen_tomatoes = min(stolen_tomatoes, tomatoes)

                if stolen_strawberries > 0 or stolen_tomatoes > 0:
                    cursor.execute("""
                        UPDATE users
                        SET strawberries = strawberries - ?,
                            tomatoes = tomatoes - ?
                        WHERE player_id = ?
                    """, (stolen_strawberries, stolen_tomatoes, target_player_id))

                    cursor.execute("""
                        UPDATE users
                        SET strawberries = strawberries + ?,
                            tomatoes = tomatoes + ?
                        WHERE user_id = ?
                    """, (stolen_strawberries, stolen_tomatoes, attacker_id))

                    bonus_text += (
                        f"💥 Урон: {damage_percent:.1f}%\n"
                        f"🍓 Украдено: {stolen_strawberries}\n"
                        f"🍅 Украдено: {stolen_tomatoes}\n"
                   )

        # === Применяем урон к ферме ===
        print(f"[DEBUG] FINAL: count={zombie_count}, dmg={total_zombie_dmg}, farm_hp={new_farm_health}")
        new_farm_health = max(0, new_farm_health - total_zombie_dmg)
        destroyed = new_farm_health == 0

        if destroyed:
            cursor.execute("SELECT strawberries, tomatoes, grapes FROM users WHERE player_id = ?", (target_player_id,))
            row = cursor.fetchone()
            if row:
                strawberries, tomatoes, grapes = row
            else:
                strawberries = tomatoes = grapes = 0

            stolen_strawberries = strawberries
            stolen_tomatoes = tomatoes
            stolen_grapes = grapes  # <-- добавлено

            cursor.execute("""
            UPDATE users SET
                strawberries = 0,
                tomatoes = 0,
                grapes = 0,
                new_farm_health = 100,
                max_farm_health = 100,
                farms_destroyed = farms_destroyed + 0,
                turret_wood = 0,
                turret_stone = 0,
                turret_iron = 0,
                turret_titanium = 0,
                turret_rocket = 0,
                wooden_barricade = 0,
                concrete_barricade = 0,
                steel_barricade = 0,
                titanium_barricade = 0,
                diamond_barricade = 0,
                mine_sea = 0,
                mine_iron = 0,
                mine_trap = 0,
                wooden_palisade = 0,
                iron_palisade = 0
            WHERE player_id = ?
            """, (target_player_id,))

# +1 уничтоженная ферма, +5 очков атакующему
            cursor.execute("""
            UPDATE users SET
                farms_destroyed = COALESCE(farms_destroyed,0) + 1,
                points = COALESCE(points,0) + 5
            WHERE user_id = ?
            """, (attacker_id,))

# Забираем 5 очков у жертвы (не уходим в минус)
            cursor.execute("""
            UPDATE users SET
                points = CASE
                    WHEN COALESCE(points,0) >= 5 THEN points - 5
                    ELSE 0
                END
            WHERE player_id = ?
            """, (target_player_id,))

            # Перечисляем награды атакующему
            cursor.execute("""
            UPDATE users SET
                strawberries = strawberries + ?,
                tomatoes = tomatoes + ?,
                grapes = grapes + ?
            WHERE user_id = ?
            """, (stolen_strawberries, stolen_tomatoes, stolen_grapes, attacker_id))

            loot_text = f"🍓 +{stolen_strawberries} клубники\n🍅 +{stolen_tomatoes} помидоров\n🍇 +{stolen_grapes} винограда\n⭐ +5 очков (отнято у противника)"
        else:
            cursor.execute("UPDATE users SET new_farm_health = ? WHERE player_id = ?", (new_farm_health, target_player_id))
            loot_text = "Ферма выстояла, награды нет."

        conn.commit()
        conn.close()

        final_bonus_text = tower_text + bonus_text

        # === Сообщение атакующему ===
        time_str = datetime.utcnow().strftime('%Y-%m-%d %H:%M:%S UTC')
        result_text = (
            f"{final_bonus_text}"
            f"⚔️ <b>Атака завершена!</b>\n"
            f"🕒 {time_str}\n"
            f"🎯 Цель: <b>{target_name}</b>\n"
            f"🧟 Зомби: <b>{ZOMBIE_INFO[zombie_id]['name']}</b> × {zombie_count}\n"
            f"💥 Урон: <b>{total_zombie_dmg}</b>\n"
            f"❤️ Здоровье фермы цели: <b>{new_farm_health}</b> / {max_farm_health}\n\n"
        )

        if destroyed:
            result_text += f"💣 Ферма уничтожена!\nВы получили награду:\n{loot_text}"
        else:
            result_text += "🛡 Ферма всё ещё стоит."

        await context.bot.send_message(chat_id=attacker_id, text=result_text, parse_mode="HTML")

        if zombie_id == 100:
            conn = sqlite3.connect(DB_PATH)
            cursor = conn.cursor()
            cursor.execute("""
                UPDATE user_zombies
                SET hp_bonus = 0, damage_bonus = 0
                WHERE user_id = ? AND zombie_id = 100
            """, (attacker_id,))
            conn.commit()
            conn.close()

    except Exception as e:
        await context.bot.send_message(chat_id=attacker_id, text=f"❌ Ошибка при атаке: {e}")
        print("ERROR in run_attack_task:", e)

async def attack_farm_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_id = user.id

        user_data = get_user(user_id)
        if not user_data:
            await update.message.reply_text("❌ Вы не зарегистрированы.")
            return

        # -----------------------------
        # 🧾 Проверки пользователя
        # -----------------------------
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute(
            "SELECT attack_banned, diamonds FROM users WHERE user_id = ?",
            (user_id,)
        )
        row = cursor.fetchone()
        conn.close()

        attack_banned = row[0] if row else 0
        diamonds = row[1] if row else 0

        if attack_banned == 1:
            await update.message.reply_text("❌ Вам запрещено атаковать (бан).")
            return

        if diamonds >= 7_000_000:
            await update.message.reply_text("❌ Вы не можете атаковать.")
            return

        # 📥 Разбор команды
        text = update.message.text.strip()

        # 👉 Первая строка — команда, всё остальное — сообщение
        lines = text.split("\n", 1)
        command_line = lines[0]
        attack_message = lines[1].strip() if len(lines) > 1 else None

        if attack_message and len(attack_message) > 50:
            await update.message.reply_text("❌ Сообщение не должно превышать 50 символов.")
            return

        parts = command_line.split()

        numbers = []
        for p in reversed(parts):
            if p.isdigit():
                numbers.append(int(p))
            else:
                break
        numbers.reverse()

        if len(numbers) not in (3, 4):
            await update.message.reply_text(
                "❌ Формат:\n"
                "тгз напасть на ферму <ID цели> <ID зомби> <кол-во> [ID снаряжения]"
            )
            return

        target_player_id = numbers[0]
        zombie_id = numbers[1]
        zombie_count = numbers[2]
        equip_id = numbers[3] if len(numbers) == 4 else None

        # -----------------------------
        # 🧍 ID атакующего
        # -----------------------------
        if isinstance(user_data, dict):
            attacker_user_id = user_data.get("user_id", user_id)
            attacker_player_id = user_data.get("player_id")
        else:
            attacker_user_id = user_data[0]
            attacker_player_id = user_data[1] if len(user_data) > 1 else None

        if attack_message:
            ATTACK_MESSAGES[(attacker_user_id, target_player_id)] = attack_message

        if attacker_player_id == target_player_id:
            await update.message.reply_text("❌ Нельзя атаковать самого себя.")
            return

        if zombie_id not in ZOMBIE_INFO:
            await update.message.reply_text("❌ Такого зомби не существует.")
            return

        if zombie_count <= 0:
            await update.message.reply_text("❌ Количество должно быть положительным.")
            return

        if zombie_id == 100 and zombie_count > 1:
            await update.message.reply_text("❌ Уникальным зомби можно атаковать только одним!")
            return

        # -----------------------------
        # 🧟 Проверка количества зомби
        # -----------------------------
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        cursor.execute(
            "SELECT quantity FROM user_zombies WHERE user_id = ? AND zombie_id = ?",
            (attacker_user_id, zombie_id)
        )
        row = cursor.fetchone()
        owned = row[0] if row else 0

        if owned < zombie_count:
            conn.close()
            await update.message.reply_text(f"❌ У вас только {owned} таких зомби!")
            return

        # -----------------------------
        # 🛡️ Проверка снаряжения (если указано)
        # -----------------------------
        if equip_id is not None:
            cursor.execute("""
                SELECT quantity FROM zombie_equipment
                WHERE user_id = ?
                  AND zombie_id = ?
                  AND equipment_id = ?
            """, (attacker_user_id, zombie_id, equip_id))
            row = cursor.fetchone()
            equipped_qty = row[0] if row else 0

            if equipped_qty < zombie_count:
                conn.close()
                await update.message.reply_text(
                    f"❌ Недостаточно зомби со снаряжением {equip_id}.\n"
                    f"Есть: {equipped_qty}, нужно: {zombie_count}"
                )
                return

        # -----------------------------
        # ➖ Списание зомби
        # -----------------------------
        cursor.execute("""
            UPDATE user_zombies
            SET quantity = quantity - ?
            WHERE user_id = ? AND zombie_id = ?
        """, (zombie_count, attacker_user_id, zombie_id))

        # -----------------------------
        # ➖ Списание снаряжения (если есть)
        # -----------------------------
        if equip_id is not None:
            cursor.execute("""
                UPDATE zombie_equipment
                SET quantity = quantity - ?
                WHERE user_id = ?
                  AND zombie_id = ?
                  AND equipment_id = ?
            """, (zombie_count, attacker_user_id, zombie_id, equip_id))

        conn.commit()
        conn.close()

        # -----------------------------
        # 🎯 Имя цели
        # -----------------------------
        conn = sqlite3.connect(DB_PATH)
        cur = conn.cursor()
        cur.execute(
            "SELECT username FROM users WHERE player_id = ?",
            (target_player_id,)
        )
        r = cur.fetchone()
        conn.close()

        target_name = r[0] if r else f"Игрок {target_player_id}"

        # -----------------------------
        # 🚀 Запуск атаки
        # -----------------------------
        context.application.create_task(
            run_attack_task(
                attacker_id=attacker_user_id,
                attacker_name=user.first_name,
                target_player_id=target_player_id,
                target_name=target_name,
                zombie_id=zombie_id,
                zombie_count=zombie_count,
                equipment_id=equip_id,  # сюда снаряжение
                context=context,
                attack_message=None
            )
        )

        await update.message.reply_text("🧟‍♂️ Атака отправлена — зомби в пути!")

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при старте атаки: {e}")
        print("ERROR in attack_farm_command:", e)

# ❌ Запретить игроку атаковать
async def ban_attack_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        admin_user_id = update.effective_user.id
        args = context.args

        # Проверка на админа
        if admin_user_id != ADMIN_ID:
            await update.message.reply_text("❌ У вас нет прав для использования этой команды.")
            return

        if len(args) < 1:
            await update.message.reply_text("❌ Использование: /ban_attack [player_id]")
            return

        player_id = int(args[0])

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # Проверяем игрока
        cursor.execute("SELECT user_id, username FROM users WHERE player_id = ?", (player_id,))
        row = cursor.fetchone()
        if not row:
            conn.close()
            await update.message.reply_text("❌ Игрок с таким player_id не найден.")
            return

        user_telegram_id, username = row
        username = username or "Без ника"

        # Устанавливаем запрет
        cursor.execute(
            "UPDATE users SET attack_banned = 1 WHERE user_id = ?",
            (user_telegram_id,)
        )
        conn.commit()
        conn.close()

        await update.message.reply_text(
            f"⛔ Игроку @{username} (player_id {player_id}) ЗАПРЕЩЕНО атаковать."
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка: {str(e)}")
        print(f"ERROR in ban_attack_command: {e}")

# ✅ Разрешить игроку снова атаковать
async def unban_attack_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        admin_user_id = update.effective_user.id
        args = context.args

        # Проверка на админа
        if admin_user_id != ADMIN_ID:
            await update.message.reply_text("❌ У вас нет прав для использования этой команды.")
            return

        if len(args) < 1:
            await update.message.reply_text("❌ Использование: /unban_attack [player_id]")
            return

        player_id = int(args[0])

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # Проверяем игрока
        cursor.execute("SELECT user_id, username FROM users WHERE player_id = ?", (player_id,))
        row = cursor.fetchone()
        if not row:
            conn.close()
            await update.message.reply_text("❌ Игрок с таким player_id не найден.")
            return

        user_telegram_id, username = row
        username = username or "Без ника"

        # Снимаем запрет
        cursor.execute(
            "UPDATE users SET attack_banned = 0 WHERE user_id = ?",
            (user_telegram_id,)
        )
        conn.commit()
        conn.close()

        await update.message.reply_text(
            f"✅ Игроку @{username} (player_id {player_id}) снова разрешено атаковать."
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка: {str(e)}")
        print(f"ERROR in unban_attack_command: {e}")

# 📌 Установить игроку зомби ID 13 (как разведчика 9)
async def set_zombie13_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        admin_user_id = update.effective_user.id
        args = context.args

        # Проверка прав админа
        if admin_user_id != ADMIN_ID:
            await update.message.reply_text("❌ У вас нет прав для использования этой команды.")
            return

        if len(args) < 1:
            await update.message.reply_text("❌ Использование: /set_zombie13 [player_id]")
            return

        player_id = int(args[0])

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # Проверяем игрока
        cursor.execute("SELECT user_id, username FROM users WHERE player_id = ?", (player_id,))
        row = cursor.fetchone()

        if not row:
            conn.close()
            await update.message.reply_text("❌ Игрок с таким player_id не найден.")
            return

        user_telegram_id, username = row
        username = username or "Без ника"

        # Устанавливаем зомби ID 13
        cursor.execute(
            "UPDATE users SET scout_zombie = 13 WHERE user_id = ?",
            (user_telegram_id,)
        )

        conn.commit()
        conn.close()

        await update.message.reply_text(
            f"🧟 Игроку @{username} (player_id {player_id}) установлен зомби ID 13."
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка: {str(e)}")
        print(f"ERROR in set_zombie13_command: {e}")

# ✅ Установить new_farm_health И max_farm_health игроку
async def set_farm_hp_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        admin_user_id = update.effective_user.id
        args = context.args

        # Проверка на админа
        if admin_user_id != ADMIN_ID:
            await update.message.reply_text("❌ У вас нет прав для использования этой команды.")
            return

        if len(args) < 2:
            await update.message.reply_text("❌ Использование: /set_farm_hp [player_id] [значение]")
            return

        player_id = int(args[0])
        hp_value = int(args[1])

        if hp_value < 0:
            await update.message.reply_text("❌ Значение не может быть отрицательным.")
            return

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # Получаем user_id Telegram по player_id
        cursor.execute("SELECT user_id, username FROM users WHERE player_id = ?", (player_id,))
        row = cursor.fetchone()
        if not row:
            conn.close()
            await update.message.reply_text("❌ Игрок с таким player_id не найден.")
            return

        user_telegram_id, username = row
        username = username or "Без ника"

        # Обновляем оба сразу
        cursor.execute("""
            UPDATE users
            SET new_farm_health = ?, max_farm_health = ?
            WHERE user_id = ?
        """, (hp_value, hp_value, user_telegram_id))

        conn.commit()
        conn.close()

        await update.message.reply_text(
            f"🏥 Игроку @{username} (player_id {player_id}) установлено здоровье фермы: {hp_value}."
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при установке здоровья фермы: {str(e)}")
        print(f"ERROR in set_farm_hp_command: {str(e)}")

async def exchange_rate_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        exchange_text = (
            "💎 *Курс обмена и информация:*\n\n"

            "🍓 *Клубника → Алмазы*\n"
            "- 10 клубники = 1 алмаз\n"
            "- Минимальное количество: 10\n"
            "- Пример: `тгз обменять клубнику 60` → 6 алмазов\n"
            "- Остаток < 10 не обменивается\n\n"

            "🍅 *Помидоры → Монеты*\n"
            "- 2 помидора = 1 монета\n"
            "- Минимальное количество: 2\n"
            "- Пример: `тгз обменять помидоры 10` → 5 монет\n"
            "- Остаток < 2 не обменивается\n\n"

            "🍇 *Виноград → Рубины 🔻*\n"
            "- 10000 винограда = 1 рубин 🔻\n"
            "- Минимальное количество: 10000\n"
            "- Пример: `тгз обменять виноград 30000` → 3 рубина 🔻\n"
            "- Остаток < 10000 не обменивается\n\n"

            "📌 *Примечания*\n"
            "- Обмен возможен только если хватает ресурсов.\n"
            "- Используйте команды:\n"
            "  • `тгз обменять клубнику [кол-во]`\n"
            "  • `тгз обменять помидоры [кол-во]`\n"
            "  • `тгз обменять виноград [кол-во]`\n"
            "- Проверить баланс: `тгз профиль`\n"
        )

        await update.message.reply_text(exchange_text, parse_mode="Markdown")
    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка при отображении курса: {str(e)}")
        print(f"ERROR in exchange_rate_command: {str(e)}")

# Функция change_diamonds_command
async def change_diamonds_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_data = get_user(user.id, user.username, user.first_name)
        if not user_data or len(user_data) < 17:
            await update.message.reply_text("❌ Ошибка: пользователь не найден.")
            return

        user_id, user_player_id, _, _, _, _, _, _, _, _, _, _, _, _, _, _, _ = user_data

        # Нормализация текста: убираем лишние пробелы и приводим к нижнему регистру
        message_text = ' '.join(update.message.text.split()).lower().split()
        print(f"DEBUG: Parsed message_text: {message_text}")  # Отладочный вывод

        if len(message_text) != 5 or message_text[:3] != ["тгз", "1712", "поменять"] or message_text[3] != "алмазы":
            await update.message.reply_text("❌ Использование: тгз 1712 поменять алмазы <айди_игрока> <количество>")
            return

        try:
            target_player_id = int(message_text[4])
            amount = int(message_text[5])
        except (ValueError, IndexError):
            await update.message.reply_text("❌ Айди игрока и количество должны быть числами!")
            return

        if target_player_id < 1:
            await update.message.reply_text("❌ Айди игрока должен быть положительным числом!")
            return

        if amount < 0:
            await update.message.reply_text("❌ Количество не может быть отрицательным!")
            return

        conn = sqlite3.connect('database.db')
        cursor = conn.cursor()
        cursor.execute('SELECT user_id, diamonds FROM users WHERE player_id = ?', (target_player_id,))
        target_user = cursor.fetchone()

        if not target_user:
            conn.close()
            await update.message.reply_text(f"❌ Игрок с ID #{target_player_id} не найден!")
            return

        target_user_id, current_diamonds = target_user
        new_diamonds = amount

        # Обновление алмазов
        cursor.execute('UPDATE users SET diamonds = ? WHERE user_id = ?', (new_diamonds, target_user_id))
        conn.commit()
        conn.close()

        # Логирование в консоль
        log_message = (
            f"[{datetime.now(timezone.utc).isoformat()}] "
            f"User {user_id} (player_id #{user_player_id}) changed diamonds for player ID #{target_player_id} "
            f"from {current_diamonds} to {new_diamonds}"
        )
        print(log_message)

        await update.message.reply_text(
            f"✅ Алмазы игрока ID #{target_player_id} изменены!\n"
            f"💎 Новое количество: {new_diamonds}\n"
            f"💎 Предыдущее количество: {current_diamonds}"
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка при изменении алмазов: {str(e)}")
        print(f"ERROR in change_diamonds_command: {str(e)}")

async def change_coins_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_data = get_user(user.id, user.username, user.first_name)
        if not user_data or len(user_data) < 17:
            await update.message.reply_text("❌ Ошибка: пользователь не найден.")
            return

        user_id, user_player_id, _, _, _, _, _, _, _, _, _, _, _, _, _, _, _ = user_data

        # Нормализация текста: убираем лишние пробелы и приводим к нижнему регистру
        message_text = ' '.join(update.message.text.split()).lower().split()
        print(f"DEBUG: Parsed message_text: {message_text}")  # Отладочный вывод

        if len(message_text) != 5 or message_text[:3] != ["тгз", "1712", "поменять"] or message_text[3] != "монеты":
            await update.message.reply_text("❌ Использование: тгз 1712 поменять монеты <айди_игрока> <количество>")
            return

        try:
            target_player_id = int(message_text[4])
            amount = int(message_text[5])
        except (ValueError, IndexError):
            await update.message.reply_text("❌ Айди игрока и количество должны быть числами!")
            return

        if target_player_id < 1:
            await update.message.reply_text("❌ Айди игрока должен быть положительным числом!")
            return

        if amount < 0:
            await update.message.reply_text("❌ Количество не может быть отрицательным!")
            return

        conn = sqlite3.connect('database.db')
        cursor = conn.cursor()
        cursor.execute('SELECT user_id, coins FROM users WHERE player_id = ?', (target_player_id,))
        target_user = cursor.fetchone()

        if not target_user:
            conn.close()
            await update.message.reply_text(f"❌ Игрок с ID #{target_player_id} не найден!")
            return

        target_user_id, current_coins = target_user
        new_coins = amount

        # Обновление монет
        cursor.execute('UPDATE users SET coins = ? WHERE user_id = ?', (new_coins, target_user_id))
        conn.commit()
        conn.close()

        # Логирование в консоль
        log_message = (
            f"[{datetime.now(timezone.utc).isoformat()}] "
            f"User {user_id} (player_id #{user_player_id}) changed coins for player ID #{target_player_id} "
            f"from {current_coins} to {new_coins}"
        )
        print(log_message)

        await update.message.reply_text(
            f"✅ Монеты игрока ID #{target_player_id} изменены!\n"
            f"💰 Новое количество: {new_coins}\n"
            f"💰 Предыдущее количество: {current_coins}"
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка при изменении монет: {str(e)}")
        print(f"ERROR in change_coins_command: {str(e)}")

async def transfer_diamonds_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args):
    user_id = update.effective_user.id

    amount = int(args[1])
    target_player_id = int(args[2])

    if amount <= 0:
        await update.message.reply_text("❌ Количество должно быть больше нуля.")
        return

    # === комиссия 5% ===
    fee = max(1, amount * 5 // 100)       # минимум 1
    final_amount = amount - fee

    if final_amount <= 0:
        await update.message.reply_text("❌ Слишком маленькая сумма для перевода.")
        return

    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    # Проверяем, есть ли столько алмазов у игрока
    cursor.execute("SELECT diamonds, player_id FROM users WHERE user_id = ?", (user_id,))
    row = cursor.fetchone()

    if not row:
        conn.close()
        await update.message.reply_text("❌ Вы не зарегистрированы.")
        return

    user_diamonds, player_id = row

    if user_diamonds < amount:
        conn.close()
        await update.message.reply_text("❌ У вас недостаточно алмазов для перевода.")
        return

    # Проверяем, существует ли игрок-получатель
    cursor.execute("SELECT user_id FROM users WHERE player_id = ?", (target_player_id,))
    rec = cursor.fetchone()

    if not rec:
        conn.close()
        await update.message.reply_text("❌ Игрок с таким игровым ID не найден.")
        return

    target_user_id = rec[0]

    # === Списываем сумму с комиссией ===
    cursor.execute("UPDATE users SET diamonds = diamonds - ? WHERE user_id = ?", (amount, user_id))

    # === Добавляем получателю уже за вычетом комиссии ===
    cursor.execute("UPDATE users SET diamonds = diamonds + ? WHERE user_id = ?", (final_amount, target_user_id))

    conn.commit()
    conn.close()

    # Сообщение отправителю
    await update.message.reply_text(
        f"💠 Вы отправили {amount:,} алмазов игроку ID {target_player_id}.\n"
        f"💸 Комиссия 5%: {fee}\n"
        f"📤 Получено игроком: {final_amount:,}".replace(",", " ")
    )

    # === Уведомление получателю ===
    try:
        await context.bot.send_message(
            chat_id=target_user_id,
            text=(
                f"📥 Вам поступил перевод алмазов!\n"
                f"🔢 От игрока ID {player_id}\n"
                f"💠 Сумма: {final_amount:,}".replace(",", " ")
            )
        )
    except:
        pass  # игрок мог закрыть ЛС

async def transfer_coins_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args):
    user_id = update.effective_user.id

    amount = int(args[1])
    target_player_id = int(args[2])

    if amount <= 0:
        await update.message.reply_text("❌ Количество должно быть больше нуля.")
        return

    # === комиссия 5% для монет ===
    fee = max(1, amount * 5 // 100)
    final_amount = amount - fee

    if final_amount <= 0:
        await update.message.reply_text("❌ Слишком маленькая сумма для перевода.")
        return

    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    # Проверяем отправителя
    cursor.execute("SELECT coins, player_id FROM users WHERE user_id = ?", (user_id,))
    row = cursor.fetchone()

    if not row:
        conn.close()
        await update.message.reply_text("❌ Вы не зарегистрированы.")
        return

    user_coins, player_id = row

    if user_coins < amount:
        conn.close()
        await update.message.reply_text("❌ У вас недостаточно монет.")
        return

    # Проверяем получателя
    cursor.execute("SELECT user_id FROM users WHERE player_id = ?", (target_player_id,))
    rec = cursor.fetchone()

    if not rec:
        conn.close()
        await update.message.reply_text("❌ Игрок с таким игровым ID не найден.")
        return

    target_user_id = rec[0]

    # Списываем у отправителя
    cursor.execute(
        "UPDATE users SET coins = coins - ? WHERE user_id = ?",
        (amount, user_id)
    )

    # Начисляем получателю
    cursor.execute(
        "UPDATE users SET coins = coins + ? WHERE user_id = ?",
        (final_amount, target_user_id)
    )

    conn.commit()
    conn.close()

    # Сообщение отправителю
    await update.message.reply_text(
        f"💰 Вы отправили {amount:,} монет игроку ID {target_player_id}.\n"
        f"💸 Комиссия 5%: {fee}\n"
        f"📤 Получено игроком: {final_amount:,}".replace(",", " ")
    )

    # Уведомление получателю
    try:
        await context.bot.send_message(
            chat_id=target_user_id,
            text=(
                f"📥 Вам поступил перевод монет!\n"
                f"🔢 От игрока ID {player_id}\n"
                f"💰 Сумма: {final_amount:,}".replace(",", " ")
            )
        )
    except:
        pass

import sqlite3
from telegram import Update
from telegram.ext import ContextTypes

DB_NAME = "diamonds_bot.db"
ADMIN_ID = 7689411893  # твой Telegram ID
allowed_resources = ["diamonds", "coins", "points", "strawberries", "tomatoes"]

# Логируем изменения в таблицу change_log (если она есть)
def log_change(user_id, player_id, resource, old_value, new_value, changed_by):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute('''
        INSERT INTO change_log (user_id, player_id, changed_by_user_id, resource_type, old_value, new_value, timestamp)
        VALUES (?, ?, ?, ?, ?, ?, datetime('now'))
    ''', (user_id, player_id, changed_by, resource, old_value, new_value))
    conn.commit()
    conn.close()

def update_resource(player_id, resource, new_value, changed_by):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute(f"SELECT user_id, {resource} FROM users WHERE player_id = ?", (player_id,))
    row = cursor.fetchone()
    if not row:
        conn.close()
        return False, f"Пользователь с player_id {player_id} не найден."
    user_id, old_value = row
    cursor.execute(f"UPDATE users SET {resource} = ? WHERE player_id = ?", (new_value, player_id))
    conn.commit()
    conn.close()
    try:
        log_change(user_id, player_id, resource, old_value, new_value, changed_by)
    except:
        pass
    return True, f"{resource} изменён: {old_value} → {new_value}"

def add_to_resource(player_id, resource, delta, changed_by):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute(f"SELECT user_id, {resource} FROM users WHERE player_id = ?", (player_id,))
    row = cursor.fetchone()
    if not row:
        conn.close()
        return False, f"Пользователь с player_id {player_id} не найден."
    user_id, old_value = row
    new_value = old_value + delta
    cursor.execute(f"UPDATE users SET {resource} = ? WHERE player_id = ?", (new_value, player_id))
    conn.commit()
    conn.close()
    try:
        log_change(user_id, player_id, resource, old_value, new_value, changed_by)
    except:
        pass
    return True, f"{resource} изменён: {old_value} → {new_value} (изменение на {delta:+d})"

async def set_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID:
        return await update.message.reply_text("⛔ У тебя нет прав использовать эту команду.")
    if len(context.args) != 3:
        return await update.message.reply_text("Использование: /set <player_id> <resource> <value>")

    try:
        player_id = int(context.args[0])
        resource = context.args[1].lower()
        value = int(context.args[2])
    except ValueError:
        return await update.message.reply_text("❌ Неверный формат. Пример: /set 1 diamonds 100")

    if resource not in allowed_resources:
        return await update.message.reply_text(f"❌ Ресурс '{resource}' недоступен. Доступные: {', '.join(allowed_resources)}")

    success, response = update_resource(player_id, resource, value, update.effective_user.id)
    await update.message.reply_text(("✅ " if success else "❌ ") + response)

async def add_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID:
        return await update.message.reply_text("⛔ У тебя нет прав использовать эту команду.")
    if len(context.args) != 3:
        return await update.message.reply_text("Использование: /add <player_id> <resource> <delta>")

    try:
        player_id = int(context.args[0])
        resource = context.args[1].lower()
        delta = int(context.args[2])
    except ValueError:
        return await update.message.reply_text("❌ Неверный формат. Пример: /add 1 coins 500")

    if resource not in allowed_resources:
        return await update.message.reply_text(f"❌ Ресурс '{resource}' недоступен. Доступные: {', '.join(allowed_resources)}")

    success, response = add_to_resource(player_id, resource, delta, update.effective_user.id)
    await update.message.reply_text(("✅ " if success else "❌ ") + response)

async def admin_add_grapes(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != 7689411893:
        await update.message.reply_text("❌ У вас нет прав администратора!")
        return
    args = update.message.text.split()
    if len(args) != 3:
        await update.message.reply_text("❌ Использование: /add_grapes [player_id] [количество]")
        return
    try:
        player_id = int(args[1])
        amount = int(args[2])
        conn = sqlite3.connect('diamonds_bot.db')
        cursor = conn.cursor()
        cursor.execute('SELECT user_id FROM users WHERE player_id = ?', (player_id,))
        result = cursor.fetchone()
        if not result:
            await update.message.reply_text(f"❌ Игрок с Player ID {player_id} не найден!")
            conn.close()
            return
        user_id = result[0]
        cursor.execute('UPDATE users SET grapes = grapes + ? WHERE user_id = ?', (amount, user_id))
        conn.commit()
        conn.close()
        await update.message.reply_text(f"✅ Добавлено {amount} винограда игроку с Player ID {player_id}")
    except ValueError:
        await update.message.reply_text("❌ Player ID и количество должны быть числами!")
    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка: {str(e)}")
        print(f"ERROR in admin_add_grapes: {str(e)}")

async def admin_set_grapes(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != 7689411893:
        await update.message.reply_text("❌ У вас нет прав администратора!")
        return
    args = update.message.text.split()
    if len(args) != 3:
        await update.message.reply_text("❌ Использование: /set_grapes [player_id] [количество]")
        return
    try:
        player_id = int(args[1])
        amount = int(args[2])
        conn = sqlite3.connect('diamonds_bot.db')
        cursor = conn.cursor()
        cursor.execute('SELECT user_id FROM users WHERE player_id = ?', (player_id,))
        result = cursor.fetchone()
        if not result:
            await update.message.reply_text(f"❌ Игрок с Player ID {player_id} не найден!")
            conn.close()
            return
        user_id = result[0]
        cursor.execute('UPDATE users SET grapes = ? WHERE user_id = ?', (amount, user_id))
        conn.commit()
        conn.close()
        await update.message.reply_text(f"✅ Количество винограда установлено на {amount} для игрока с Player ID {player_id}")
    except ValueError:
        await update.message.reply_text("❌ Player ID и количество должны быть числами!")
    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка: {str(e)}")
        print(f"ERROR in admin_set_grapes: {str(e)}")

async def admin_add_rubies(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != 7689411893:
        await update.message.reply_text("❌ У вас нет прав администратора!")
        return
    args = update.message.text.split()
    if len(args) != 3:
        await update.message.reply_text("❌ Использование: /add_rubies [player_id] [количество]")
        return
    try:
        player_id = int(args[1])
        amount = int(args[2])
        conn = sqlite3.connect('diamonds_bot.db')
        cursor = conn.cursor()
        cursor.execute('SELECT user_id FROM users WHERE player_id = ?', (player_id,))
        result = cursor.fetchone()
        if not result:
            await update.message.reply_text(f"❌ Игрок с Player ID {player_id} не найден!")
            conn.close()
            return
        user_id = result[0]
        cursor.execute('UPDATE users SET rubies = rubies + ? WHERE user_id = ?', (amount, user_id))
        conn.commit()
        conn.close()
        await update.message.reply_text(f"✅ Добавлено {amount} рубинов игроку с Player ID {player_id}")
    except ValueError:
        await update.message.reply_text("❌ Player ID и количество должны быть числами!")
    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка: {str(e)}")
        print(f"ERROR in admin_add_rubies: {str(e)}")

async def admin_set_rubies(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != 7689411893:
        await update.message.reply_text("❌ У вас нет прав администратора!")
        return
    args = update.message.text.split()
    if len(args) != 3:
        await update.message.reply_text("❌ Использование: /set_rubies [player_id] [количество]")
        return
    try:
        player_id = int(args[1])
        amount = int(args[2])
        conn = sqlite3.connect('diamonds_bot.db')
        cursor = conn.cursor()
        cursor.execute('SELECT user_id FROM users WHERE player_id = ?', (player_id,))
        result = cursor.fetchone()
        if not result:
            await update.message.reply_text(f"❌ Игрок с Player ID {player_id} не найден!")
            conn.close()
            return
        user_id = result[0]
        cursor.execute('UPDATE users SET rubies = ? WHERE user_id = ?', (amount, user_id))
        conn.commit()
        conn.close()
        await update.message.reply_text(f"✅ Количество рубинов установлено на {amount} для игрока с Player ID {player_id}")
    except ValueError:
        await update.message.reply_text("❌ Player ID и количество должны быть числами!")
    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка: {str(e)}")
        print(f"ERROR in admin_set_rubies: {str(e)}")

async def admin_reset_zombie_bonus(update: Update, context: ContextTypes.DEFAULT_TYPE):
    # Проверка админа
    if update.effective_user.id != 7689411893:
        await update.message.reply_text("❌ У вас нет прав администратора!")
        return

    args = update.message.text.split()
    if len(args) != 2:
        await update.message.reply_text("❌ Использование: /reset_zombie_bonus [player_id]")
        return

    try:
        target_player_id = int(args[1])

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # Получаем user_id по player_id
        cursor.execute("SELECT user_id FROM users WHERE player_id = ?", (target_player_id,))
        result = cursor.fetchone()
        if not result:
            await update.message.reply_text(f"❌ Игрок с Player ID {target_player_id} не найден!")
            conn.close()
            return

        user_id = result[0]

        # Сбрасываем бонусы уникального зомби
        cursor.execute("""
            UPDATE user_zombies
            SET hp_bonus = 0, damage_bonus = 0
            WHERE user_id = ? AND zombie_id = 100
        """, (user_id,))
        conn.commit()
        conn.close()

        await update.message.reply_text(f"✅ Бонусы уникального зомби для Player ID {target_player_id} сброшены!")

    except ValueError:
        await update.message.reply_text("❌ Player ID должен быть числом!")
    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка: {str(e)}")
        print(f"ERROR in admin_reset_zombie_bonus: {e}")

# Обработчик команды изменения здоровья
async def set_farm_health(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    if user_id not in ADMIN_IDS:
        await update.message.reply_text("❌ У вас нет прав для этой команды.")
        print(f"Отклонена команда изменения здоровья от {user_id} (не админ)")
        return

    if len(context.args) != 2:
        await update.message.reply_text("❌ Использование: /set_farm_health <target_user_id> <new_health>")
        return

    try:
        target_user_id = int(context.args[0])
        new_health = int(context.args[1])
    except ValueError:
        await update.message.reply_text("❌ ID пользователя и здоровье должны быть числами.")
        return

    conn = sqlite3.connect("database.db")
    cursor = conn.cursor()
    cursor.execute(
        "UPDATE farm_status SET farm_health = ? WHERE user_id = ?",
        (new_health, target_user_id)
    )
    conn.commit()
    conn.close()

    await update.message.reply_text(f"✅ Здоровье фермы пользователя {target_user_id} изменено на {new_health}.")
    print(f"Админ {user_id} изменил здоровье фермы {target_user_id} на {new_health}")

from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ContextTypes

async def farm_multipliers_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_data = get_user(user.id, user.username, user.first_name)

        if user_data is None or len(user_data) < 25:
            await update.message.reply_text("❌ Ошибка: пользователь не найден.")
            return

        (
            user_id, player_id, username, first_name, diamonds, points,
            last_diamond_time, upgrade_level, registration_date, strawberries,
            farm_level, last_zombie_attack, zombie_status, last_strawberry_update,
            last_bonus_time, tomatoes, grapes, coins, farms_destroyed, clan_id,
            *rest
        ) = user_data

        STRAWBERRY_THRESHOLDS = [
            (100, 1.5), (1000, 2.0), (5000, 3.0), (15000, 4.0),
            (30000, 5.0), (60000, 6.0), (100000, 7.0), (160000, 8.0),
            (240000, 9.0), (350000, 10.0), (500000, 12.0), (700000, 14.0),
            (950000, 16.0), (1300000, 18.0), (1800000, 20.0), (2300000, 22.0),
            (3000000, 24.0), (4000000, 26.0),  # до x26
            (4600000, 27.0), (5300000, 28.0), (6100000, 29.0), (7000000, 30.0),
            (8000000, 31.0), (9200000, 32.0), (10600000, 33.0), (12200000, 34.0),
            (14000000, 35.0)  # финальный множитель
        ]

        TOMATO_THRESHOLDS = [
            (3000, 1.5), (30000, 2.0), (150000, 3.0), (450000, 4.0),
            (900000, 5.0), (1800000, 6.0), (3000000, 7.0), (4800000, 8.0),
            (7200000, 9.0), (10500000, 10.0), (15000000, 12.0), (21000000, 14.0),
            (28500000, 16.0), (39000000, 18.0), (54000000, 20.0), (69000000, 22.0),
            (90000000, 24.0), (120000000, 26.0),
            (138000000, 27.0), (159000000, 28.0), (183000000, 29.0), (210000000, 30.0),
            (240000000, 31.0), (276000000, 32.0), (318000000, 33.0), (366000000, 34.0),
            (420000000, 35.0)
        ]

        GRAPE_THRESHOLDS = [
            (10, 1.5), (100, 2.0), (500, 3.0), (1500, 4.0),
            (3000, 5.0), (6000, 6.0), (10000, 7.0), (16000, 8.0),
            (24000, 9.0), (35000, 10.0), (50000, 12.0), (70000, 14.0),
            (95000, 16.0), (130000, 18.0), (180000, 20.0), (230000, 22.0),
            (300000, 24.0), (400000, 26.0),
            (460000, 27.0), (530000, 28.0), (610000, 29.0), (700000, 30.0),
            (800000, 31.0), (920000, 32.0), (1060000, 33.0), (1220000, 34.0),
            (1400000, 35.0)
        ]

        context.user_data['thresholds'] = {
            'strawberries': (strawberries, STRAWBERRY_THRESHOLDS),
            'tomatoes': (tomatoes, TOMATO_THRESHOLDS),
            'grape': (grapes, GRAPE_THRESHOLDS)
        }

        # Старт с первой страницы
        await send_multiplier_page(update, context, 'strawberries')

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка: {str(e)}")
        print(f"ERROR in farm_multipliers_command: {str(e)}")


async def send_multiplier_page(update, context, page_name: str):
    value, thresholds = context.user_data['thresholds'][page_name]
    current, next_threshold, next_mult = get_current_and_next(value, thresholds)

    emoji = {'strawberries': '🍓', 'tomatoes': '🍅', 'grape': '🍇'}[page_name]
    title = {'strawberries': 'Клубника', 'tomatoes': 'Помидоры', 'grape': 'Виноград'}[page_name]

    text = f"📊 <b>{title} - Множители фермы</b>\n\n"
    for threshold, mult in thresholds:
        if current == mult:
            text += f"✅ <b>{threshold:,}</b> ➜ <b>x{mult}</b> (текущий)\n"
        else:
            text += f"{threshold:,} ➜ x{mult}\n"

    if next_threshold:
        text += f"\n⏳ До следующего множителя x{next_mult}: <b>{next_threshold - value:,}</b> {emoji}"
    else:
        text += "\n🔥 Максимальный множитель достигнут!"

    keyboard = [
        [
            InlineKeyboardButton("🍓", callback_data="page_strawberries"),
            InlineKeyboardButton("🍅", callback_data="page_tomatoes"),
            InlineKeyboardButton("🍇", callback_data="page_grape")
        ]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)

    if update.callback_query:
        await update.callback_query.message.edit_text(text, reply_markup=reply_markup, parse_mode="HTML")
        await update.callback_query.answer()
    else:
        await update.message.reply_text(text, reply_markup=reply_markup, parse_mode="HTML")


async def farm_multipliers_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    if not query or not query.data:
        return

    page_map = {
        'page_strawberries': 'strawberries',
        'page_tomatoes': 'tomatoes',
        'page_grape': 'grape'
    }

    if query.data not in page_map:
        await query.answer("❌ Некорректная кнопка!")
        return

    page_name = page_map[query.data]
    await send_multiplier_page(update, context, page_name)


def get_current_and_next(value, thresholds):
    current = 1.0
    next_threshold = None
    next_mult = None
    for threshold, multiplier in thresholds:
        if value >= threshold:
            current = multiplier
        else:
            next_threshold = threshold
            next_mult = multiplier
            break
    return current, next_threshold, next_mult

import random
from telegram import InlineKeyboardButton, InlineKeyboardMarkup

# --- Команда угадай число ---
async def guess_number_command(update, context):
    try:
        args = context.args
        if len(args) < 3:
            await update.message.reply_text("❌ Использование: тгз угадай число <4|6|8> <ставка>")
            return

        count = int(args[1])  # количество кнопок
        bet = int(args[2])    # ставка

        if count not in [4, 6, 8]:
            await update.message.reply_text("❌ Количество должно быть 4, 6 или 8")
            return
        if bet <= 0:
            await update.message.reply_text("❌ Ставка должна быть больше 0")
            return

        # Сохраняем данные в context.user_data
        correct = random.randint(1, count)
        context.user_data["guess_number"] = {
            "correct": correct,
            "count": count,
            "bet": bet
        }

        # Создаём кнопки
        keyboard = []
        for i in range(1, count + 1):
            keyboard.append([InlineKeyboardButton(str(i), callback_data=f"guess_{i}")])

        await update.message.reply_text(
            f"🎲 Вы выбрали уровень с {count} числами.\n"
            f"Ставка: 💎 {bet}\n"
            "Выбери число:",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка: {str(e)}")
        print(f"ERROR in guess_number_command: {str(e)}")

# --- Обработчик нажатий ---
async def guess_number_callback(update, context):
    query = update.callback_query
    data = query.data

    if "guess_number" not in context.user_data:
        await query.answer("⏳ Игра не начата!")
        return

    game_data = context.user_data["guess_number"]
    correct = game_data["correct"]
    count = game_data["count"]
    bet = game_data["bet"]

    chosen = int(data.split("_")[1])

    # Убираем игру из памяти
    del context.user_data["guess_number"]

    if chosen == correct:
        multiplier = 1.5 if count == 4 else 2.25 if count == 6 else 3
        win = int(bet * multiplier)
        await query.edit_message_text(
            f"✅ Правильно! Загаданное число: {correct}\n"
            f"💰 Выигрыш: 💎 {win} (x{multiplier})"
        )
        # тут добавляем win в баланс пользователя
    else:
        await query.edit_message_text(
            f"❌ Неверно! Загаданное число было: {correct}\n"
            f"💸 Ставка {bet} сгорела."
        )
        # тут убавляем bet из баланса пользователя

import sqlite3
from telegram import Update
from telegram.ext import ContextTypes

DB_PATH = "diamonds_bot.db"
MAX_CLAN_MEMBERS = 7
CLAN_CREATE_COST = 100000  # 💎 Стоимость создания клана

# ✅ Создание клана с эффектом маски "Знаменосец"
async def create_clan_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args=None):
    try:
        user = update.effective_user
        user_id = user.id

        if not args or len(args) < 2:
            await update.message.reply_text("❌ Укажите название клана: тгз создать клан <название>")
            return

        clan_name = " ".join(args[1:]).strip()
        if len(clan_name) < 3:
            await update.message.reply_text("❌ Название клана должно быть не короче 3 символов.")
            return

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # Проверяем пользователя
        cursor.execute("SELECT clan_id, diamonds FROM users WHERE user_id = ?", (user_id,))
        row = cursor.fetchone()
        if row is None:
            conn.close()
            await update.message.reply_text("❌ Вы не зарегистрированы.")
            return

        current_clan, diamonds = row
        if current_clan:
            conn.close()
            await update.message.reply_text("❌ Вы уже состоите в клане.")
            return

        # Проверяем наличие маски "Знаменосец"
        cursor.execute("SELECT 1 FROM user_masks WHERE user_id = ? AND mask_id = 11", (user_id,))
        has_banner_mask = cursor.fetchone() is not None

        # Определяем стоимость с учётом маски
        clan_cost = CLAN_CREATE_COST // 5 if has_banner_mask else CLAN_CREATE_COST

        if diamonds < clan_cost:
            conn.close()
            await update.message.reply_text(f"❌ Нужно минимум 💎 {clan_cost} алмазов для создания клана.")
            return

        # Проверяем, не занят ли тег клана
        cursor.execute("SELECT id FROM clans WHERE clan_name = ?", (clan_name,))
        if cursor.fetchone():
            conn.close()
            await update.message.reply_text("❌ Клан с таким названием уже существует.")
            return

        # Создаём клан
        cursor.execute("INSERT INTO clans (clan_name, owner_id) VALUES (?, ?)", (clan_name, user_id))
        clan_id = cursor.lastrowid

        # Обновляем игрока
        cursor.execute("UPDATE users SET clan_id = ?, diamonds = diamonds - ? WHERE user_id = ?", (clan_id, clan_cost, user_id))
        cursor.execute("INSERT INTO clan_members (clan_id, user_id) VALUES (?, ?)", (clan_id, user_id))

        conn.commit()
        conn.close()

        # Сообщение игроку
        message_text = f"🏰 Клан '{clan_name}' создан!\nВы стали его владельцем."
        if has_banner_mask:
            message_text += "\n🎖️ Маска «Знаменосец» активирована — скидка 80% на создание клана!"

        await update.message.reply_text(message_text)

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при создании клана: {str(e)}")
        print(f"ERROR in create_clan_command: {str(e)}")

# ✅ Вступление в клан (учитывает прокачку лимита)
async def join_clan_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args=None):
    try:
        user_id = update.effective_user.id

        if not args or len(args) < 3:
            await update.message.reply_text("❌ Укажите клан: тгз вступить в клан <название>")
            return

        clan_name = " ".join(args[2:]).strip()

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # Проверяем, не состоит ли пользователь уже в клане
        cursor.execute("SELECT clan_id FROM users WHERE user_id = ?", (user_id,))
        row = cursor.fetchone()
        if not row:
            conn.close()
            await update.message.reply_text("❌ Вы не зарегистрированы.")
            return

        if row[0]:
            conn.close()
            await update.message.reply_text("❌ Вы уже состоите в клане.")
            return

        # Ищем клан по названию
        cursor.execute("SELECT id, is_closed, max_members FROM clans WHERE clan_name = ?", (clan_name,))
        clan = cursor.fetchone()
        if not clan:
            conn.close()
            await update.message.reply_text("❌ Такого клана нет.")
            return

        clan_id, is_closed, max_members = clan

        if is_closed:
            conn.close()
            await update.message.reply_text("🔒 Этот клан закрыт для вступления.")
            return

        # Считаем участников через clan_members (уникальные user_id)
        cursor.execute("SELECT COUNT(DISTINCT user_id) FROM clan_members WHERE clan_id = ?", (clan_id,))
        members_count = cursor.fetchone()[0]

        # Проверяем лимит
        if members_count >= max_members:
            conn.close()
            await update.message.reply_text(f"❌ Клан заполнен (максимум {max_members} участников).")
            return

        # Проверяем, не состоит ли пользователь уже в clan_members (на всякий случай)
        cursor.execute("SELECT 1 FROM clan_members WHERE user_id = ?", (user_id,))
        if cursor.fetchone():
            conn.close()
            await update.message.reply_text("⚠️ Вы уже числитесь в этом клане.")
            return

        # Добавляем игрока
        cursor.execute("UPDATE users SET clan_id = ? WHERE user_id = ?", (clan_id, user_id))
        cursor.execute("INSERT OR IGNORE INTO clan_members (clan_id, user_id) VALUES (?, ?)", (clan_id, user_id))

        conn.commit()
        conn.close()

        await update.message.reply_text(f"✅ Вы успешно вступили в клан '{clan_name}'!")

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при вступлении в клан: {str(e)}")
        print(f"ERROR in join_clan_command: {str(e)}")

# ✅ Покинуть клан
async def leave_clan_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args=None):
    try:
        user_id = update.effective_user.id
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        cursor.execute("SELECT clan_id FROM users WHERE user_id = ?", (user_id,))
        row = cursor.fetchone()
        if not row or not row[0]:
            conn.close()
            await update.message.reply_text("❌ Вы не состоите в клане.")
            return

        clan_id = row[0]

        # Если владелец — не даём выйти
        cursor.execute("SELECT owner_id FROM clans WHERE id = ?", (clan_id,))
        owner_id = cursor.fetchone()[0]
        if owner_id == user_id:
            conn.close()
            await update.message.reply_text("❌ Вы владелец клана! Сначала передайте или удалите его.")
            return

        cursor.execute("UPDATE users SET clan_id = NULL WHERE user_id = ?", (user_id,))
        cursor.execute("DELETE FROM clan_members WHERE clan_id = ? AND user_id = ?", (clan_id, user_id))

        conn.commit()
        conn.close()

        await update.message.reply_text("✅ Вы покинули клан.")

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при выходе из клана: {str(e)}")
        print(f"ERROR in leave_clan_command: {str(e)}")


# ✅ Удаление клана (только владелец)
async def delete_clan_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args=None):
    try:
        user_id = update.effective_user.id
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        cursor.execute("SELECT id, owner_id, clan_name FROM clans WHERE owner_id = ?", (user_id,))
        clan = cursor.fetchone()
        if not clan:
            conn.close()
            await update.message.reply_text("❌ Вы не владелец клана.")
            return

        clan_id, owner_id, clan_name = clan

        cursor.execute("UPDATE users SET clan_id = NULL WHERE clan_id = ?", (clan_id,))
        cursor.execute("DELETE FROM clan_members WHERE clan_id = ?", (clan_id,))
        cursor.execute("DELETE FROM clans WHERE id = ?", (clan_id,))

        conn.commit()
        conn.close()

        await update.message.reply_text(f"🗑️ Клан '{clan_name}' был удалён.")

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при удалении клана: {str(e)}")
        print(f"ERROR in delete_clan_command: {str(e)}")


# ✅ Список кланов
async def clans_list_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args=None):
    try:
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        cursor.execute(f"""
        SELECT
            clans.id,
            clans.clan_name,
            clans.max_members,
            COUNT(users.user_id) as members_count,
            IFNULL(SUM(users.diamonds), 0) as total_diamonds,
            IFNULL(SUM(users.coins), 0) as total_coins
        FROM clans
        LEFT JOIN users ON users.clan_id = clans.id
        GROUP BY clans.id
        ORDER BY members_count DESC
        """)

        rows = cursor.fetchall()
        conn.close()

        if not rows:
            await update.message.reply_text("📭 Кланов пока нет. Создайте первый: тгз создать клан <название>")
            return

        text = "🏰 Список кланов:\n\n"
        for clan_id, clan_name, max_members, members_count, total_diamonds, total_coins in rows:
            text += (
                f"🔹 <b>{clan_name}</b>\n"
                f"👥 {members_count}/{max_members} участников\n"
                f"💎 {total_diamonds:,} алмазов | 💰 {total_coins:,} монет\n\n"
            )

        await update.message.reply_text(text, parse_mode="HTML")

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при получении списка кланов: {str(e)}")
        print(f"ERROR in clans_list_command: {str(e)}")

async def my_clan_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args=None):
    try:
        user_id = update.effective_user.id
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # ✅ Добавляем is_closed (чтобы знать — клан открыт или закрыт)
        cursor.execute("""
        SELECT clans.id, clans.clan_name, clans.owner_id, clans.max_members, clans.is_closed
        FROM users
        JOIN clans ON users.clan_id = clans.id
        WHERE users.user_id = ?
        """, (user_id,))
        clan = cursor.fetchone()

        if not clan:
            conn.close()
            await update.message.reply_text("❌ Вы не состоите в клане.")
            return

        # ✅ Теперь 5 переменных
        clan_id, clan_name, owner_id, max_members, is_closed = clan

        # Получаем участников клана
        cursor.execute("""
        SELECT username, points, user_id, diamonds, coins, player_id
        FROM users
        WHERE clan_id = ?
        """, (clan_id,))
        members = cursor.fetchall()

        total_points = sum([m[1] for m in members])
        total_diamonds = sum([m[3] for m in members])
        total_coins = sum([m[4] for m in members])

        conn.close()

        # ✅ Добавляем отображение статуса
        status_emoji = "🔒" if is_closed else "🔓"
        status_text = "Закрыт" if is_closed else "Открыт"

        text = (
            f"🏰 <b>Клан:</b> {clan_name}\n"
            f"👑 <b>Владелец:</b> {owner_id}\n"
            f"{status_emoji} <b>Статус:</b> {status_text}\n"  # 👈 добавлено
            f"👥 <b>Участники ({len(members)}/{max_members}):</b>\n"
        )

        for username, points, uid, diamonds, coins, player_id in members:
            owner_tag = " 👑" if uid == owner_id else ""
            text += (
                f"• @{username or 'Без ника'} — ID: <code>{player_id}</code>\n"
                f"  ⭐ {points} | 💎 {diamonds} | 💰 {coins}{owner_tag}\n"
            )

        text += (
            f"\n🏆 <b>Очки клана:</b> {total_points}\n"
            f"💎 <b>Всего алмазов:</b> {total_diamonds}\n"
            f"💰 <b>Всего монет:</b> {total_coins}"
        )

        await update.message.reply_text(text, parse_mode="HTML")

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при просмотре клана: {str(e)}")
        print(f"ERROR in my_clan_command: {str(e)}")

# ✅ Исключить игрока из клана (по player_id, только владелец)
async def kick_from_clan_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args=None):
    try:
        owner_id = update.effective_user.id  # владелец клана
        if not args or len(args) < 1:
            await update.message.reply_text("❌ Укажите ID игрока: тгз исключить <айди>")
            return

        try:
            target_player_id = int(args[0])  # теперь args[0]
        except ValueError:
            await update.message.reply_text("❌ ID должен быть числом!")
            return

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # Проверяем, что пользователь — владелец клана
        cursor.execute("SELECT id, clan_name FROM clans WHERE owner_id = ?", (owner_id,))
        clan = cursor.fetchone()
        if not clan:
            conn.close()
            await update.message.reply_text("❌ Вы не владелец клана.")
            return

        clan_id, clan_name = clan

        # Проверяем, состоит ли игрок в этом клане
        cursor.execute(
            "SELECT user_id, username FROM users WHERE player_id = ? AND clan_id = ?",
            (target_player_id, clan_id)
        )
        target = cursor.fetchone()
        if not target:
            conn.close()
            await update.message.reply_text("❌ Игрок с таким ID не найден в вашем клане.")
            return

        target_user_id, target_username = target
        target_username = target_username or "Без ника"

        # Исключаем игрока
        cursor.execute("UPDATE users SET clan_id = NULL WHERE player_id = ?", (target_player_id,))
        cursor.execute("DELETE FROM clan_members WHERE user_id = ?", (target_user_id,))

        conn.commit()
        conn.close()

        await update.message.reply_text(
            f"🚫 Игрок @{target_username} (ID: {target_player_id}) исключён из клана '{clan_name}'."
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при исключении из клана: {str(e)}")
        print(f"ERROR in kick_from_clan_command: {str(e)}")

import sqlite3
import time
from telegram import Update
from telegram.ext import ContextTypes

DB_PATH = "diamonds_bot.db"

FACTORIES_INFO = {
    "лесопилка": {
        "column": "sawmill",
        "cost": {"coins": 2000},
        "income": {"wood": 100},
        "limit": 100
    },
    "каменоломня": {
        "column": "quarry",
        "cost": {"coins": 2500, "wood": 1000},
        "income": {"stone": 90, "wood": -100},
        "limit": 100
    },
    "плавильня": {
        "column": "smelter",
        "cost": {"coins": 5000, "stone": 900},
        "income": {"iron": 30, "stone": -90},
        "limit": 100
    },
    "фабрика": {
        "column": "factory",
        "cost": {"coins": 7500, "iron": 300},
        "income": {"titanium": 15, "iron": -30},
        "limit": 100
    },
    "карьер": {
        "column": "mine",
        "cost": {"coins": 10000, "titanium": 150},
        "income": {"diamonds": 0.2, "titanium": -15},
        "limit": 100
    }
}

async def factories_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Команда: сбор ресурсов с заводов"""
    try:
        user_id = update.effective_user.id
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()  # Проверяем наличие колонки last_factory_update
        cursor.execute("PRAGMA table_info(users)")
        columns = [c[1] for c in cursor.fetchall()]
        if "last_factory_update" not in columns:
            cursor.execute("ALTER TABLE users ADD COLUMN last_factory_update INTEGER DEFAULT 0")
            conn.commit()

        # Получаем данные
        cursor.execute("""
            SELECT coins, wood, stone, iron, titanium, diamonds,
                   sawmill, quarry, smelter, factory, mine,
                   sawmill_enabled, quarry_enabled, smelter_enabled, factory_enabled, mine_enabled,
                   COALESCE(last_factory_update, 0)
            FROM users WHERE user_id = ?
        """, (user_id,))
        data = cursor.fetchone()

        if not data:
            conn.close()
            await update.message.reply_text("❌ Вы не зарегистрированы!")
            return

        (
            coins, wood, stone, iron, titanium, diamonds,
            sawmill, quarry, smelter, factory, mine,
            sawmill_enabled, quarry_enabled, smelter_enabled, factory_enabled, mine_enabled,
            last_update
        ) = data

        now = int(time.time())

        # Если первый раз — активируем
        if last_update == 0:
            cursor.execute("UPDATE users SET last_factory_update=? WHERE user_id=?", (now, user_id))
            conn.commit()
            conn.close()
            await update.message.reply_text("🏭 Заводы активированы! Производство началось.")
            return

        seconds_passed = now - last_update
        hours_passed = seconds_passed / 3600

        if seconds_passed < 5:
            await update.message.reply_text("⌛ Производство ещё не завершилось, подождите немного.")
            conn.close()
            return

        # Подсчёт прибыли
        current = {"wood": wood, "stone": stone, "iron": iron, "titanium": titanium, "diamonds": diamonds}
        factory_counts = {
            "sawmill": sawmill if sawmill_enabled else 0,
            "quarry": quarry if quarry_enabled else 0,
            "smelter": smelter if smelter_enabled else 0,
            "factory": factory if factory_enabled else 0,
            "mine": mine if mine_enabled else 0
        }

        total_income = {r: 0 for r in current}

        for info in FACTORIES_INFO.values():
            count = factory_counts.get(info["column"]) or 0  # Если завод отключен, то его счёт = 0
            for res, per_hour in info["income"].items():
                total_income[res] += int(per_hour * count * hours_passed)

        # Применяем прибыль
        for r in current:
            current[r] += total_income[r]

        cursor.execute("""
            UPDATE users
            SET wood=?, stone=?, iron=?, titanium=?, diamonds=?, last_factory_update=?
            WHERE user_id=?
        """, (
            current["wood"], current["stone"], current["iron"],
            current["titanium"], current["diamonds"], now, user_id
        ))
        conn.commit()
        conn.close()

        # --- 🪄 Локализация названий ресурсов ---
        RESOURCE_NAMES = {
            "wood": "🪵 Дерево",
            "stone": "⛏ Камень",
            "iron": "⛓ Железо",
            "titanium": "⚙ Титан",
            "diamonds": "💎 Алмазы"
        }

        # --- Форматирование блока “📦 Получено” ---
        gained_text = "\n".join([
            f"• {RESOURCE_NAMES.get(r, r.capitalize())}: {v:+d}"
            for r, v in total_income.items()
            if abs(v) > 0
        ])

        if not gained_text:
            gained_text = "⌛ Пока ничего не произведено."

        # --- Преобразуем None → 0 ---
        sawmill = 0 if sawmill is None else int(sawmill)
        quarry = 0 if quarry is None else int(quarry)
        smelter = 0 if smelter is None else int(smelter)
        factory = 0 if factory is None else int(factory)
        mine = 0 if mine is None else int(mine)

        factories_text = (
            f"🪵 Лесопилки: {sawmill}\n"
            f"⛏ Каменоломни: {quarry}\n"
            f"🔥 Плавильни: {smelter}\n"
            f"⚙ Фабрики: {factory}\n"
            f"💎 Карьеры: {mine}\n"
        )# Ресурсы — показываем аккуратно
        def format_resource(val):
            return str(int(val))  # Преобразуем в целое число

        res_text = (
            f"💰 Монеты: {format_resource(coins)}\n"
            f"🪵 Дерево: {format_resource(current['wood'])}\n"
            f"⛏ Камень: {format_resource(current['stone'])}\n"
            f"⛓ Железо: {format_resource(current['iron'])}\n"
            f"⚙ Титан: {format_resource(current['titanium'])}\n"
            f"💎 Алмазы: {format_resource(current['diamonds'])}"
        )

        h = int(seconds_passed // 3600)
        m = int((seconds_passed % 3600) // 60)
        s = int(seconds_passed % 60)

        await update.message.reply_text(
            f"🏭 <b>Производство завершено!</b>\n"
            f"⏱ Прошло: {h} ч. {m} мин. {s} с.\n\n"
            f"📦 <b>Получено:</b>\n{gained_text}\n\n"
            f"🏗 <b>Ваши заводы:</b>\n{factories_text}\n"
            f"📊 <b>Текущие ресурсы:</b>\n{res_text}",
            parse_mode="HTML"
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при обработке заводов: {str(e)}")
        print(f"ERROR in factories_command: {e}")

import sqlite3
from telegram import Update
from telegram.ext import ContextTypes

# Синонимы и падежи для удобства
FACTORY_ALIASES = {
    "лесопилка": "лесопилка", "лесопилку": "лесопилка",
    "каменоломня": "каменоломня", "каменоломню": "каменоломня",
    "плавильня": "плавильня", "плавильню": "плавильня",
    "фабрика": "фабрика", "фабрику": "фабрика",
    "карьер": "карьер", "шахта": "карьер"
}

# Функция для правильного склонения ресурсов
def get_resource_name(resource, quantity):
    genitive = {
        "coins": "монет", "wood": "дерева", "stone": "камня",
        "iron": "железа", "titanium": "титана", "diamonds": "алмазов"
    }

    if resource not in genitive:
        return ""

    # Всегда использовать родительный падеж во множественном числе
    return genitive[resource]

# Для вывода пользователю
FACTORY_DISPLAY = {
    "лесопилка": "🪵 Лесопилка",
    "каменоломня": "⛏ Каменоломня",
    "плавильня": "🔥 Плавильня",
    "фабрика": "⚙ Фабрика",
    "карьер": "💎 Карьер"
}

async def buy_factory_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args):
    if not args:
        await update.message.reply_text(
            "❌ Укажите завод.\nПример: тгз построить лесопилку 10"
        )
        return

    # === завод ===
    raw = args[0].strip().lower()
    factory_name = FACTORY_ALIASES.get(raw)

    if not factory_name:
        await update.message.reply_text("❌ Неизвестный тип завода!")
        return

    # === количество ===
    amount = 1
    if len(args) >= 2 and args[1].isdigit():
        amount = int(args[1])

    if amount <= 0:
        await update.message.reply_text("❌ Количество должно быть больше нуля.")
        return

    info = FACTORIES_INFO.get(factory_name)
    if not info:
        await update.message.reply_text("❌ Завод не найден!")
        return

    col = info["column"]
    cost = info.get("cost", {})
    default_limit = info.get("limit", 100)

    user_id = update.effective_user.id
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    # === данные пользователя ===
    cursor.execute(f"""
        SELECT coins, wood, stone, iron, titanium, diamonds,
               COALESCE({col}, 0), factory_limit_upgrade
        FROM users WHERE user_id = ?
    """, (user_id,))
    row = cursor.fetchone()

    if not row:
        conn.close()
        await update.message.reply_text("❌ Вы не зарегистрированы!")
        return

    coins, wood, stone, iron, titanium, diamonds, owned, limit_upgrade = row

    # === лимит ===
    limits = {
        0: 100, 1: 200, 2: 300, 3: 400, 4: 500, 5: 600,
        6: 700, 7: 800, 8: 900, 9: 1000, 10: 1200,
        11: 1400, 12: 1600, 13: 1800, 14: 2000,
        15: 2300, 16: 2600, 17: 2900, 18: 3200,
        19: 3500, 20: 3800, 21: 4100, 22: 4400,
        23: 4700
    }
    limit = limits.get(limit_upgrade, 5000)

    if owned + amount > limit:
        conn.close()
        await update.message.reply_text(
            f"🚫 Превышен лимит!\n"
            f"Можно построить ещё максимум: {limit - owned}"
        )
        return

    # === ресурсы ===
    resources = {
        "coins": coins,
        "wood": wood,
        "stone": stone,
        "iron": iron,
        "titanium": titanium,
        "diamonds": diamonds
    }

    # === проверка ресурсов ===
    for res, needed in cost.items():
        total_needed = needed * amount
        if resources.get(res, 0) < total_needed:
            conn.close()
            await update.message.reply_text(
                f"❌ Недостаточно {get_resource_name(res, total_needed)}!\n"
                f"Нужно: {total_needed}, у вас: {resources.get(res, 0)}"
            )
            return

    # === списание ===
    for res, needed in cost.items():
        resources[res] -= needed * amount

    owned += amount

    cursor.execute(f"""
        UPDATE users
        SET coins=?, wood=?, stone=?, iron=?, titanium=?, diamonds=?, {col}=?
        WHERE user_id=?
    """, (
        resources["coins"],
        resources["wood"],
        resources["stone"],
        resources["iron"],
        resources["titanium"],
        resources["diamonds"],
        owned,
        user_id
    ))

    conn.commit()
    conn.close()

    # === текст ===
    cost_text = ", ".join([
        f"{get_resource_name(k, v * amount)} {v * amount}"
        for k, v in cost.items() if v > 0
    ])

    balance_text = (
        f"💰 Монеты: {resources['coins']}\n"
        f"🪵 Дерево: {resources['wood']}\n"
        f"⛏ Камень: {resources['stone']}\n"
        f"⛓ Железо: {resources['iron']}\n"
        f"⚙ Титан: {resources['titanium']}\n"
        f"💎 Алмазы: {resources['diamonds']}"
    )

    await update.message.reply_text(
        f"✅ Построено: {amount} × {FACTORY_DISPLAY[factory_name]}\n"
        f"💸 Потрачено: {cost_text or 'ничего'}\n"
        f"🏭 Всего: {owned}/{limit}\n\n"
        f"📊 Баланс:\n{balance_text}"
    )

import sqlite3
from telegram import Update
from telegram.ext import ContextTypes

async def factories_list_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = "📜 Список доступных заводов:\n\n"

    text += (
        "🏭 Лесопилка\n"
        "─────────────\n"
        "💰 Стоимость: 2 000 монет\n"
        "📈 Доход: +100 дерева/час\n"
        "📌 Лимит: 100\n\n"
    )

    text += (
        "⛏ Каменоломня\n"
        "─────────────\n"
        "💰 Стоимость: 2 500 монет + 1 000 дерева\n"
        "📈 Доход: +90 камня/час\n"
        "⚡ Расход: -100 дерева/час\n"
        "📌 Лимит: 100\n\n"
    )

    text += (
        "🔥 Плавильня\n"
        "─────────────\n"
        "💰 Стоимость: 5 000 монет + 900 камня\n"
        "📈 Доход: +30 железа/час\n"
        "⚡ Расход: -90 камня/час\n"
        "📌 Лимит: 100\n\n"
    )

    text += (
        "⚙ Фабрика\n"
        "─────────────\n"
        "💰 Стоимость: 7 500 монет + 300 железа\n"
        "📈 Доход: +15 титана/час\n"
        "⚡ Расход: -30 железа/час\n"
        "📌 Лимит: 100\n\n"
    )

    text += (
        "💎 Карьер\n"
        "─────────────\n"
        "💰 Стоимость: 10 000 монет + 150 титана\n"
        "📈 Доход: +0.2 алмаза/час\n"
        "⚡ Расход: -15 титана/час\n"
        "📌 Лимит: 100\n"
    )

    await update.message.reply_text(text)

async def increase_factory_limit_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Команда для последовательного увеличения лимита заводов"""
    try:
        user_id = update.effective_user.id
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # Получаем текущие монеты и уровень улучшения
        cursor.execute("""
            SELECT coins, factory_limit_upgrade
            FROM users WHERE user_id = ?
        """, (user_id,))
        row = cursor.fetchone()

        if not row:
            conn.close()
            await update.message.reply_text("❌ Вы не зарегистрированы!")
            return

        coins, upgrade_level = row

        # ✅ Все уровни улучшений с увеличенной в 2 раза стоимостью
        upgrades = {
            0: {"cost": 200_000, "new_limit": 200},
            1: {"cost": 350_000, "new_limit": 300},
            2: {"cost": 550_000, "new_limit": 400},
            3: {"cost": 800_000, "new_limit": 500},
            4: {"cost": 1_100_000, "new_limit": 600},
            5: {"cost": 1_450_000, "new_limit": 700},
            6: {"cost": 1_850_000, "new_limit": 800},
            7: {"cost": 2_300_000, "new_limit": 900},
            8: {"cost": 2_800_000, "new_limit": 1000},
            9: {"cost": 4_200_000, "new_limit": 1200},
            10: {"cost": 5_800_000, "new_limit": 1400},
            11: {"cost": 7_600_000, "new_limit": 1600},
            12: {"cost": 9_600_000, "new_limit": 1800},
            13: {"cost": 11_800_000, "new_limit": 2000},
            14: {"cost": 15_000_000, "new_limit": 2300},
            15: {"cost": 18_500_000, "new_limit": 2600},
            16: {"cost": 22_250_000, "new_limit": 2900},
            17: {"cost": 26_250_000, "new_limit": 3200},
            18: {"cost": 30_500_000, "new_limit": 3500},
            19: {"cost": 35_000_000, "new_limit": 3800},
            20: {"cost": 39_750_000, "new_limit": 4100},
            21: {"cost": 44_750_000, "new_limit": 4400},
            22: {"cost": 50_000_000, "new_limit": 4700},
            23: {"cost": 55_500_000, "new_limit": 5000},
        }

        if upgrade_level not in upgrades:
            conn.close()
            await update.message.reply_text("🚫 Достигнут максимальный лимит заводов (1600)!")
            return

        cost = upgrades[upgrade_level]["cost"]
        new_limit = upgrades[upgrade_level]["new_limit"]

        if coins < cost:
            conn.close()
            await update.message.reply_text(f"❌ Недостаточно монет! Нужно {cost:,}, у вас {coins:,}".replace(",", " "))
            return

        # ✅ Списание монет и установка нового уровня
        new_upgrade_level = upgrade_level + 1
        coins -= cost

        cursor.execute("""
            UPDATE users
            SET coins = ?, factory_limit_upgrade = ?
            WHERE user_id = ?
        """, (coins, new_upgrade_level, user_id))

        conn.commit()
        conn.close()

        await update.message.reply_text(
            f"✅ Лимит заводов увеличен до {new_limit}!\n"
            f"💰 Потрачено монет: {cost:,}\n"
            f"🏭 Теперь можно строить больше заводов."
            .replace(",", " ")
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при увеличении лимита: {str(e)}")
        print(f"ERROR in increase_factory_limit_command: {e}")

import sqlite3
from telegram import Update
from telegram.ext import ContextTypes

DB_PATH = "diamonds_bot.db"

# 🧱 Список баррикад (все кроме алмазной дешевле в 10 раз)
BARRICADES = {
1: {"name": "🪵 Деревянная баррикада", "column": "wooden_barricade", "resource": "wood", "cost": 15000, "health": 10},
2: {"name": "🧱 Бетонная баррикада", "column": "concrete_barricade", "resource": "stone", "cost": 15000, "health": 12},
3: {"name": "⛓ Стальная баррикада", "column": "steel_barricade", "resource": "iron", "cost": 15000, "health": 40},
4: {"name": "🔩 Титановая баррикада", "column": "titanium_barricade", "resource": "titanium", "cost": 15000, "health": 90},
5: {"name": "💎 Алмазная баррикада", "column": "diamond_barricade", "resource": "diamonds", "cost": 15000, "health": 180},
6: {"name": "🪵 Деревянные колья", "column": "wooden_palisade", "resource": "wood", "cost": 50000, "health": 10},
7: {"name": "⛓ Железные колья", "column": "iron_palisade", "resource": "iron", "cost": 30000, "health": 20},

8: {
    "name": "🏰 Баррикада Цитадели",
    "column": "citadel_barricade",
    "multi_cost": {
        "wood": 45000,
        "stone": 45000,
        "iron": 45000,
        "diamonds": 7350,
        "coins": 816025
    },
    "health": 3000
}
}

from telegram import InlineKeyboardButton, InlineKeyboardMarkup

BARRICADES_PER_PAGE = 4

def build_barricades_page(page: int, rank_bonus_percent: int, clan_discount: int):

    items = list(BARRICADES.items())
    total_pages = (len(items) - 1) // BARRICADES_PER_PAGE + 1

    page = max(0, min(page, total_pages - 1))
    start = page * BARRICADES_PER_PAGE
    end = start + BARRICADES_PER_PAGE

    text = f"🛡️ <b>Баррикады для защиты фермы</b>\n"
    text += f"📄 Страница <b>{page + 1}</b> / <b>{total_pages}</b>\n\n"

    emojis = {
        "wood": "🪵",
        "stone": "🪨",
        "iron": "⛓️",
        "titanium": "🔩",
        "diamonds": "💎",
        "coins": "💰"
    }

    for idx, info in items[start:end]:

        base_hp = info["health"]
        bonus_hp = base_hp * rank_bonus_percent // 100
        total_hp = base_hp + bonus_hp

        # ---- ЦИТАДЕЛЬ ----
        if "multi_cost" in info:

            cost_line = " + ".join(
                f"{v} {emojis[k]}" for k, v in info["multi_cost"].items()
            )

            discount_line = " + ".join(
                f"{int(v*(100-clan_discount)/100)} {emojis[k]}"
                for k, v in info["multi_cost"].items()
            )

            text += (
                f"<b>{idx}. {info['name']}</b>\n"
                f"├ 💰 Стоимость: {cost_line}\n"
                f"├ 💎 Стоимость со скидкой клана: {discount_line}\n"
                f"├ ❤️ Прочность: +{base_hp}\n"
                f"├ ⭐ С учётом бонуса ранга: +{total_hp}\n\n"
            )

        else:
            res = info["resource"]
            discounted_cost = int(info["cost"] * (100 - clan_discount) / 100)

            text += (
                f"<b>{idx}. {info['name']}</b>\n"
                f"├ 💰 Стоимость: {info['cost']} {emojis[res]}\n"
                f"├ 💎 Стоимость со скидкой клана: {discounted_cost} {emojis[res]}\n"
                f"├ ❤️ Прочность: +{base_hp}\n"
                f"├ ⭐ С учётом бонуса ранга: +{total_hp}\n\n"
            )

    text += "🪄 Для покупки:\n<code>тгз установить баррикаду (номер) (количество)</code>"

    keyboard = []
    nav = []

    if page > 0:
        nav.append(InlineKeyboardButton("⬅️ Назад", callback_data=f"barricades_{page - 1}"))
    if page < total_pages - 1:
        nav.append(InlineKeyboardButton("Вперёд ➡️", callback_data=f"barricades_{page + 1}"))

    if nav:
        keyboard.append(nav)

    return text, InlineKeyboardMarkup(keyboard)

async def barricades_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id

    # --- очки ---
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute("SELECT points FROM users WHERE user_id = ?", (user_id,))
    points = cursor.fetchone()
    points = points[0] if points else 0

    # --- ранг ---
    rank_number = 1
    thresholds = [100,300,500,800,1200,1700,1800,2400,3000,3800,
                  3900,4700,5700,5800,6800,8000,8100,9300,10700,
                  12200,13800,15500,17300,17400,19200,19300,21200,
                  21300,22300,22400,23600,24800,26100,27500,29000,
                  30600,32300,32400,32500,35500,38000,41000,41100,42400]

    for i, t in enumerate(thresholds, start=2):
        if points >= t:
            rank_number = i

    rank_bonus_percent = sum(1 if i <= 5 else 2 for i in range(1, rank_number + 1))

    # --- скидка клана ---
    cursor.execute("""
        SELECT clans.defense_upgrade_level
        FROM users JOIN clans ON users.clan_id = clans.id
        WHERE users.user_id = ?
    """, (user_id,))
    row = cursor.fetchone()
    conn.close()

    clan_discount = 0
    if row and row[0]:
        bonuses = [5,10,15,20,25,30,35,40,50,60,70,80]
        clan_discount = bonuses[row[0] - 1]

    context.user_data["rank_bonus"] = rank_bonus_percent
    context.user_data["clan_discount"] = clan_discount

    text, keyboard = build_barricades_page(0, rank_bonus_percent, clan_discount)
    await update.message.reply_text(text, parse_mode="HTML", reply_markup=keyboard)

async def barricades_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    page = int(query.data.split("_")[1])
    user_id = query.from_user.id

    # (логика ранга и скидки — та же, можно вынести в функцию)
    # ⬇️ для краткости считаем, что ты вынес её

    text, keyboard = build_barricades_page(
        page,
        context.user_data["rank_bonus"],
        context.user_data["clan_discount"]
    )

    await query.edit_message_text(text, parse_mode="HTML", reply_markup=keyboard)

RESOURCE_NAMES = {
    "wood": "🪵 Дерево",
    "stone": "🪨 Камень",
    "iron": "⛓ Железо",
    "titanium": "🔩 Титан",
    "diamonds": "💎 Алмазы",
    "coins": "💰 Монеты"
}

# =====================================================================
# ⚒️ Команда: установка баррикады
# =====================================================================
async def set_barricade_command(update: Update, context: ContextTypes.DEFAULT_TYPE):

    try:
        user_id = update.effective_user.id
        message_text = update.message.text.lower().split()

        index = message_text.index("баррикаду")
        barricade_id = int(message_text[index + 1])
        amount = int(message_text[index + 2])

        info = BARRICADES[barricade_id]

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        cursor.execute("""
        SELECT wood, stone, iron, titanium, diamonds, coins,
               new_farm_health, max_farm_health
        FROM users WHERE user_id = ?
        """, (user_id,))
        row = cursor.fetchone()

        wood, stone, iron, titanium, diamonds, coins, new_hp, max_hp = row

        resources = {
            "wood": wood or 0,
            "stone": stone or 0,
            "iron": iron or 0,
            "titanium": titanium or 0,
            "diamonds": diamonds or 0,
            "coins": coins or 0
        }

        # --- скидка ---
        cursor.execute("""
        SELECT clans.defense_upgrade_level
        FROM users JOIN clans ON users.clan_id = clans.id
        WHERE users.user_id = ?
        """, (user_id,))
        row = cursor.fetchone()

        discount = 0
        if row and row[0]:
            upgrades = [5,10,15,20,25,30,35,40,50,60,70,80]
            discount = upgrades[row[0]-1]

        # --- проверка ресурсов ---
        spend_text = []
        spend_sql = []
        spend_values = []

        if "multi_cost" in info:
            for res, cost in info["multi_cost"].items():
                total_cost = int(cost * amount * (100-discount)/100)

                if resources[res] < total_cost:
                    await update.message.reply_text(f"❌ Не хватает {res}")
                    return

                spend_sql.append(f"{res} = {res} - ?")
                spend_values.append(total_cost)
                spend_text.append(f"{total_cost} {res}")

        else:
            res = info["resource"]
            total_cost = int(info["cost"] * amount * (100-discount)/100)

            if resources[res] < total_cost:
                await update.message.reply_text(f"❌ Не хватает {res}")
                return

            spend_sql.append(f"{res} = {res} - ?")
            spend_values.append(total_cost)
            spend_text.append(f"{total_cost} {RESOURCE_NAMES.get(res, res)}")

        # --- ранг ---
        cursor.execute("SELECT points FROM users WHERE user_id = ?", (user_id,))
        points = cursor.fetchone()[0]

        rank_number = sum(points >= t for t in [
            100,300,500,800,1200,1700,1800,2400,3000,3800,
            3900,4700,5700,5800,6800,8000,8100,9300,10700,
            12200,13800,15500,17300,17400,19200,19300,21200,
            21300,22300,22400,23600,24800,26100,27500,29000,
            30600,32300,32400,32500,35500,38000,41000,41100,42400
        ]) + 1

        rank_bonus_percent = 0
        full_groups = rank_number // 5
        remaining = rank_number % 5

        for g in range(1, full_groups + 1):
            rank_bonus_percent += 5 * g

        rank_bonus_percent += remaining * (full_groups + 1)

        base_health = info["health"] * amount
        bonus_health = base_health * rank_bonus_percent // 100
        total_health = base_health + bonus_health

        spend_sql.append("new_farm_health = new_farm_health + ?")
        spend_sql.append("max_farm_health = max_farm_health + ?")
        spend_sql.append(f"{info['column']} = {info['column']} + ?")

        spend_values.extend([total_health, total_health, amount, user_id])

        cursor.execute(f"""
        UPDATE users
        SET {", ".join(spend_sql)}
        WHERE user_id = ?
        """, spend_values)

        conn.commit()
        conn.close()

        # --- красивый вывод ---
        reply_text = (
            f"✅ <b>Баррикады установлены!</b>\n\n"
            f"{amount}× {info['name']} успешно размещены на ферме 🏰\n\n"

            f"🧱 Базовая прочность: <b>+{base_health}</b>\n"
            f"🏅 Бонус ранга (ранг {rank_number}): "
            f"<b>+{bonus_health}</b> (<b>{rank_bonus_percent}%</b>)\n"
            f"❤️ Итого добавлено ХП: <b>+{total_health}</b>\n\n"

            f"❤️ Здоровье фермы:\n"
            f"<b>{(new_hp or 100) + total_health}</b> / "
            f"<b>{(max_hp or 100) + total_health}</b>\n\n"

            f"💰 Потрачено: {', '.join(spend_text)}"
        )

        await update.message.reply_text(reply_text, parse_mode="HTML")

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка: {e}")

ADMIN_USER_ID = 7689411893  # твой Telegram ID
DB_PATH = "diamonds_bot.db"


async def admin_resource(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_USER_ID:
        await update.message.reply_text("❌ У вас нет прав администратора!")
        return

    args = update.message.text.split()
    if len(args) != 5:
        await update.message.reply_text(
            "❌ Использование: /resource [player_id] [ресурс: wood|stone|iron|titanium] [add|set] [количество]"
        )
        return

    try:
        player_id = int(args[1])
        resource = args[2].lower()
        operation = args[3].lower()
        amount = int(args[4])

        if resource not in ["wood", "stone", "iron", "titanium"]:
            await update.message.reply_text("❌ Неверный ресурс! Используйте: wood, stone, iron, titanium")
            return

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("SELECT user_id FROM users WHERE player_id = ?", (player_id,))
        result = cursor.fetchone()

        if not result:
            await update.message.reply_text(f"❌ Игрок с Player ID {player_id} не найден!")
            conn.close()
            return

        user_id = result[0]

        if operation == "add":
            cursor.execute(f"UPDATE users SET {resource} = {resource} + ? WHERE user_id = ?", (amount, user_id))
            text = f"✅ Добавлено {amount} к ресурсу <b>{resource}</b> игроку с Player ID <b>{player_id}</b>"
        elif operation == "set":
            cursor.execute(f"UPDATE users SET {resource} = ? WHERE user_id = ?", (amount, user_id))
            text = f"✅ Ресурс <b>{resource}</b> установлен в {amount} у игрока с Player ID <b>{player_id}</b>"
        else:
            await update.message.reply_text("❌ Неверная операция! Используйте: add или set")
            conn.close()
            return

        conn.commit()
        conn.close()

        await update.message.reply_text(text, parse_mode="HTML")

    except ValueError:
        await update.message.reply_text("❌ Player ID и количество должны быть числами!")
    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка: {str(e)}")
        print(f"ERROR in admin_resource: {str(e)}")

import random
import sqlite3
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ContextTypes, CallbackQueryHandler

DB_PATH = "diamonds_bot.db"
russian_roulette_online_data = {}

# ------------------ Команда ------------------
async def russian_roulette_online(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        user = update.effective_user
        user_data = get_user(user.id, user.username, user.first_name)

        if not user_data or len(user_data) < 23:
            await update.message.reply_text("❌ Пользователь не найден в базе!")
            return

        parts = update.message.text.strip().split()
        parts_lower = [p.lower() for p in parts]

        if len(parts_lower) < 5 or parts_lower[:4] != ["тгз", "русская", "рулетка", "онлайн"]:
            await update.message.reply_text(
                "❌ Использование: <b>тгз русская рулетка онлайн [ставка] @username</b>\n"
                "или ответ на сообщение пользователя с командой ставки",
                parse_mode="HTML"
            )
            return

        try:
            bet = int(parts[4])
        except ValueError:
            await update.message.reply_text("❌ Ставка должна быть числом!")
            return

        if bet < 350:
            await update.message.reply_text("❌ Минимальная ставка: 350 💎")
            return

        diamonds = user_data[4]
        if bet > diamonds:
            await update.message.reply_text("❌ У вас недостаточно алмазов!")
            return

        # ------------------ Определяем цель ------------------
        # Если команда написана в ответ на сообщение
        if update.message.reply_to_message:
            reply_user = update.message.reply_to_message.from_user
            target_id = reply_user.id
            target_name = reply_user.first_name
            target_data = get_user(target_id)
            if not target_data:
                await update.message.reply_text("❌ Игрок не найден в базе!")
                return
            target_diamonds = target_data[4]
        else:
            # Если указали @username
            mentioned_username = parts[5].replace("@", "").strip()
            conn = sqlite3.connect(DB_PATH)
            cursor = conn.cursor()
            cursor.execute(
                "SELECT user_id, first_name, diamonds FROM users WHERE username = ?",
                (mentioned_username,)
            )
            target = cursor.fetchone()
            conn.close()
            if not target:
                await update.message.reply_text("❌ Игрок не найден в базе!")
                return
            target_id, target_name, target_diamonds = target

        if target_id == user.id:
            await update.message.reply_text("❌ Нельзя играть с самим собой!")
            return

        if bet > target_diamonds:
            await update.message.reply_text(f"❌ У {target_name} недостаточно алмазов для ставки!")
            return

        # Проверка на активную игру
        if (user.id, target_id) in russian_roulette_online_data:
            await update.message.reply_text("❌ У вас уже есть активное приглашение этому игроку!")
            return

        # Создаем клавиатуру
        keyboard = [
            [
                InlineKeyboardButton(
                    "✅ Принять", callback_data=f"rro_accept_{user.id}_{target_id}_{bet}"
                ),
                InlineKeyboardButton(
                    "❌ Отклонить", callback_data=f"rro_decline_{user.id}_{target_id}"
                )
            ],
            [
                InlineKeyboardButton(
                    "🚫 Отменить приглашение", callback_data=f"rro_cancel_{user.id}_{target_id}"
                )
            ]
        ]
        markup = InlineKeyboardMarkup(keyboard)

        msg = await update.message.reply_text(
            f"🎯 <b>Русская рулетка онлайн</b>\n\n"
            f"👤 <b>{user.first_name}</b> вызывает <b>{target_name}</b> на игру!\n"
            f"💎 Ставка: <b>{bet}</b> алмазов с каждого.\n\n"
            f"Ожидаем ответа приглашенного игрока.",
            parse_mode="HTML",
            reply_markup=markup
        )

        russian_roulette_online_data[(user.id, target_id)] = {
            "message": msg,
            "bet": bet,
            "accepted": False
        }

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка в русской рулетке онлайн: {str(e)}")
        print("ERROR in russian_roulette_online:", e)


# ------------------ Обработчик кнопок ------------------
async def russian_roulette_cb(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    if not query:
        return

    data = query.data or ""
    print("🔥 CALLBACK:", data)

    try:
        await query.answer()
    except Exception as e:
        print("answer() error:", e)

    parts = data.split("_")
    if len(parts) < 2 or parts[0] != "rro":
        return

    action = parts[1]

    def parse_ids():
        try:
            initiator = int(parts[2]) if len(parts) > 2 else None
            target = int(parts[3]) if len(parts) > 3 else None
            return initiator, target
        except Exception as e:
            print("parse_ids error:", e)
            return None, None

    initiator_id, target_id = parse_ids()
    game_key = (initiator_id, target_id)

    # ------------------ CANCEL ------------------
    if action == "cancel":
        if query.from_user.id != initiator_id:
            await query.answer("❌ Только инициатор может отменить приглашение.", show_alert=True)
            return
        russian_roulette_online_data.pop(game_key, None)
        try:
            await query.message.edit_text("🚫 Приглашение отменено отправителем.")
        except Exception:
            pass
        return

    # ------------------ DECLINE ------------------
    if action == "decline":
        russian_roulette_online_data.pop(game_key, None)
        try:
            await query.message.edit_text("❌ Игрок отклонил приглашение.")
        except Exception:
            pass
        return

    # ------------------ ACCEPT ------------------
    if action == "accept":
        try:
            bet = int(parts[4])
        except Exception:
            try:
                await query.message.edit_text("❌ Некорректные данные (ставка).")
            except Exception:
                pass
            return

        game = russian_roulette_online_data.get(game_key)
        if not game:
            try:
                await query.message.edit_text("❌ Игра устарела.")
            except Exception:
                pass
            return

        if query.from_user.id != target_id:
            await query.answer("❌ Только приглашённый может принять.", show_alert=True)
            return

        for uid in (initiator_id, target_id):
            u = get_user(uid)
            if not u:
                continue
            update_user(uid, diamonds=u[4] - bet)

        game["accepted"] = True
        game["revolver"] = [False] * 5 + [True]
        random.shuffle(game["revolver"])
        game["turn"] = random.choice([initiator_id, target_id])
        game["shots"] = 0

        try:
            await query.message.edit_text("🎮 Игра началась! Первый ход уже отправлен.")
        except Exception:
            pass

        await send_turn_message(context, initiator_id, target_id, game["turn"], bet)
        return

    # ------------------ SHOOT / GIVEUP ------------------
    if action in ("shoot", "giveup"):
        try:
            player_id = int(parts[4])
        except Exception:
            await query.answer("❌ Ошибка данных.", show_alert=True)
            return

        game = russian_roulette_online_data.get(game_key)
        if not game:
            try:
                await query.message.edit_text("❌ Игра устарела.")
            except Exception:
                pass
            return

        if game.get("turn") != player_id:
            await query.answer("❌ Сейчас не ваш ход.", show_alert=True)
            return

        if action == "giveup":
            winner = target_id if player_id == initiator_id else initiator_id
            bet = game["bet"]
            winner_data = get_user(winner)
            if winner_data:
                update_user(winner, diamonds=winner_data[4] + bet * 2)
            russian_roulette_online_data.pop(game_key, None)
            try:
                await query.message.edit_text("🏳 Вы сдались. Игра окончена.")
            except Exception:
                pass
            await context.bot.send_message(chat_id=winner, text="🎉 Ваш соперник сдался! Вы выиграли 💎")
            return

        if action == "shoot":
            await perform_shot(context, initiator_id, target_id, player_id)
            return


# ------------------ Ход игрока ------------------
async def send_turn_message(context, initiator_id, target_id, turn_id, bet):
    game = russian_roulette_online_data.get((initiator_id, target_id))
    if not game:
        return

    shots_done = game["shots"]
    shots_total = len(game["revolver"])
    shots_left = shots_total - shots_done

    keyboard = [
        [
            InlineKeyboardButton(
                "🔫 Выстрелить",
                callback_data=f"rro_shoot_{initiator_id}_{target_id}_{turn_id}"
            )
        ]
    ]
    markup = InlineKeyboardMarkup(keyboard)

    await context.bot.send_message(
        chat_id=turn_id,
        text=(
            f"🔫 <b>Русская рулетка</b>\n\n"
            f"💸 Ставка: {bet} алмазов\n"
            f"💎 Остаток выстрелов: {shots_left}\n\n"
            f"Нажмите кнопку для выстрела!"
        ),
        parse_mode="HTML",
        reply_markup=markup
    )


async def perform_shot(context, initiator_id, target_id, player_id):
    game = russian_roulette_online_data.get((initiator_id, target_id))
    if not game:
        return

    bet = game["bet"]
    revolver = game["revolver"]
    shots = game["shots"]

    shots_total = len(revolver)
    shots_left = shots_total - shots - 1

    if revolver[shots]:
        loser = player_id
        winner = target_id if player_id == initiator_id else initiator_id
        winner_data = get_user(winner)
        if winner_data:
            update_user(winner, diamonds=winner_data[4] + bet * 2)
        del russian_roulette_online_data[(initiator_id, target_id)]
        await context.bot.send_message(loser, "💥 Выстрел! Вы проиграли 💀")
        await context.bot.send_message(winner, "🎉 Победа! Противник погиб 💎")
        return

    game["shots"] += 1
    next_turn = target_id if player_id == initiator_id else initiator_id
    game["turn"] = next_turn
    await context.bot.send_message(
        player_id,
        f"🔫 Щелчок! Вы выжили.\n💎 Осталось выстрелов: {shots_left}"
    )
    await send_turn_message(context, initiator_id, target_id, next_turn, bet)

import sqlite3
from telegram import Update
from telegram.ext import ContextTypes

DB_PATH = "diamonds_bot.db"
ADMIN_USER_ID = 7689411893  # <- Ваш админский ID

FACTORY_COLUMNS = {
    "лесопилка": "sawmill",
    "каменоломня": "quarry",
    "плавильня": "smelter",
    "фабрика": "factory",
    "карьер": "mine"
}

async def admin_factory(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_USER_ID:
        await update.message.reply_text("❌ У вас нет прав администратора!")
        return

    args = update.message.text.split()
    if len(args) != 5:
        await update.message.reply_text(
            "❌ Использование: /factory [player_id] [завод: лесопилка|каменоломня|плавильня|фабрика|карьер] [add|set] [количество]"
        )
        return

    try:
        player_id = int(args[1])
        factory_name = args[2].lower()
        operation = args[3].lower()
        amount = int(args[4])

        if factory_name not in FACTORY_COLUMNS:
            await update.message.reply_text("❌ Неверный завод! Используйте: лесопилка, каменоломня, плавильня, фабрика, карьер")
            return

        col = FACTORY_COLUMNS[factory_name]

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("SELECT user_id FROM users WHERE player_id = ?", (player_id,))
        result = cursor.fetchone()

        if not result:
            await update.message.reply_text(f"❌ Игрок с Player ID {player_id} не найден!")
            conn.close()
            return

        user_id = result[0]

        if operation == "add":
            cursor.execute(f"UPDATE users SET {col} = COALESCE({col}, 0) + ? WHERE user_id = ?", (amount, user_id))
            text = f"✅ Добавлено {amount} к заводу <b>{factory_name}</b> игроку с Player ID <b>{player_id}</b>"
        elif operation == "set":
            cursor.execute(f"UPDATE users SET {col} = ? WHERE user_id = ?", (amount, user_id))
            text = f"✅ Завод <b>{factory_name}</b> установлен в {amount} у игрока с Player ID <b>{player_id}</b>"
        else:
            await update.message.reply_text("❌ Неверная операция! Используйте: add или set")
            conn.close()
            return

        conn.commit()
        conn.close()
        await update.message.reply_text(text, parse_mode="HTML")

    except ValueError:
        await update.message.reply_text("❌ Player ID и количество должны быть числами!")
    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка: {str(e)}")
        print(f"ERROR in admin_factory: {str(e)}")

import sqlite3
from telegram import Update
from telegram.ext import ContextTypes

DB_PATH = "diamonds_bot.db"

# 🔧 Сопоставления с учётом падежей
FACTORIES_FORMS = {
    "лесопилка": "sawmill",
    "лесопилку": "sawmill",
    "каменоломня": "quarry",
    "каменоломню": "quarry",
    "плавильня": "smelter",
    "плавильню": "smelter",
    "фабрика": "factory",
    "фабрику": "factory",
    "карьер": "mine",
    "карьеру": "mine"
}

# 💣 Команда: тгз снести <тип>
async def snesti_command(update: Update, context: ContextTypes.DEFAULT_TYPE, args):
    try:
        user_id = update.effective_user.id

        if not args:
            await update.message.reply_text(
                "❌ Укажите, что снести.\n"
                "Пример: тгз снести фабрика / тгз снести карьер"
            )
            return

        factory_name = args[0].lower()

        if factory_name not in FACTORIES_FORMS:
            await update.message.reply_text(
                "❌ Неверный тип завода!\n"
                "Доступные: лесопилка(у), каменоломня(ю), плавильня(ю), фабрика(у), карьер(у)."
            )
            return

        column = FACTORIES_FORMS[factory_name]

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # Проверяем текущее количество
        cursor.execute(f"SELECT {column} FROM users WHERE user_id = ?", (user_id,))
        row = cursor.fetchone()

        if not row:
            conn.close()
            await update.message.reply_text("❌ Вы не зарегистрированы!")
            return

        current_count = row[0]

        if current_count <= 0:
            await update.message.reply_text(f"❌ У вас нет {factory_name} для сноса.")
            conn.close()
            return

        # Уменьшаем на 1
        cursor.execute(f"UPDATE users SET {column} = {column} - 1 WHERE user_id = ?", (user_id,))
        conn.commit()
        conn.close()

        await update.message.reply_text(
            f"💥 <b>Одна {factory_name} успешно снесена!</b>\n"
            f"🏗 Осталось: {current_count - 1}",
            parse_mode="HTML"
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при сносе завода: {str(e)}")
        print(f"ERROR in snesti_command: {e}")

# ====== Нормализатор ======
def normalize_answer(s: str) -> str:
    if not s:
        return ""
    s = s.strip().lower()
    s = s.replace("–", "-").replace("—", "-")
    s = re.sub(r"\s*-\s*", "-", s)
    s = re.sub(r"[\s-]+", "", s)
    return s


# ====== Хранилища ======
user_riddles = {}         # текущая загадка пользователя


# ====== Функции работы с БД ======
import json
import sqlite3
DB_PATH = "diamonds_bot.db"

def load_solved_riddles(user_id):
    try:
        conn = sqlite3.connect(DB_PATH)
        cur = conn.cursor()
        cur.execute("SELECT solved_riddles FROM users WHERE user_id = ?", (user_id,))
        row = cur.fetchone()
        conn.close()
        if not row or not row[0]:
            return []
        return json.loads(row[0])
    except Exception:
        return []

def add_solved_riddle(user_id, riddle_key):
    riddles = load_solved_riddles(user_id)
    if riddle_key not in riddles:
        riddles.append(riddle_key)
        try:
            conn = sqlite3.connect(DB_PATH)
            cur = conn.cursor()
            cur.execute("UPDATE users SET solved_riddles = ? WHERE user_id = ?", (json.dumps(riddles), user_id))
            conn.commit()
            conn.close()
        except Exception as e:
            print(f"[DB ERROR add_solved_riddle] {e}")

import random
from telegram import Update
from telegram.ext import ContextTypes, MessageHandler, filters, CommandHandler

# Все загадки
riddles = {
    # Предметы
    "Могилка": {
        "text": "Не камень, не крест, но надежду дарю. Когда павший товарищ в битве уснул, я тихим шепотом его призову, и он снова в строй вернется, зомби ввергнув во тьму. Но помни, действие ограничено, и выбрать нужно одного, кто ближе всех к тебе, в этом бою одному.\n\nКак писать ответ: (Название предмета)",
        "answer": "Могилка"
    },
    "Тотем": {
        "text": "Время – золото, и я его ценю. Магию в крови я ускоряю, чтобы ты, герой, был всегда начеку. Но будь осторожен, сила моя не безгранична, складываясь, она знает предел. В умелых руках – это мощь, в руках лентяя – лишь тень побед.\n\nКак писать ответ: (Название предмета)",
        "answer": "Тотем"
    },
    "Магнитрон": {
        "text": "Я не магнит обычный, но притяжение знаю. Не к металлу, а к снарядам я силу прилагаю, Плювака и Бомбера дары ко мне летят, даруя тебе передышку, а врагу - отмщение. Но помни, моя воля избирательна, не все подряд я соберу, лишь то, что нужно тебе в этом бою.\n\nКак писать ответ: (Название предмета)",
        "answer": "Магнитрон"
    },
    "Бумгало": {
        "text": "Я приманка и бомба в одном лице, зомби ко мне тянутся, как мотыльки к огню. Но не стоит приближаться ко мне слишком близко, ибо взрыв мой разрушительный, и врагам горе, но и тебе опасно стоять вблизи, если я разрушусь.\n\nКак писать ответ: (Название предмета)",
        "answer": "Бумгало"
    },
    "Башня с Кристаллом": {
        "text": "Сияя магическим светом, от скверны я защита, зомби-колдовство не пройдет через мой барьер. Но не думай, что я бессмертна, укрепляй меня, и я буду стоять на страже твоей, оберегая от дурных последствий, что приносят мертвецы.\n\nКак писать ответ: (Название предмета)",
        "answer": "Башня с Кристаллом"
    },
    "Деревянная Турель": {
        "text": "Дешевле железа, но верный солдат, я охраняю рубежи, давая отпор врагам. Простота моя – не порок, а достоинство, ибо древесины в этом мире много, и построить меня не составит труда. Я – первая линия обороны, и горжусь этим.\n\nКак писать ответ: (Название предмета)",
        "answer": "Деревянная Турель"
    },
    "Баррикада из Бревен": {
        "text": "Я – стена из леса, прочнее, чем кажется на вид. Сдерживаю натиск мертвецов, хоть и не из камня я свит. Но помни, огонь – мой враг, и время – мой союзник, если зомби будут грызть меня слишком долго, то паду я, но дам тебе время для передышки и контратаки.\n\nКак писать ответ: (Название предмета)",
        "answer": "Баррикада из Бревен"
    },
    "Ядовитая Мина": {
        "text": "Скрыта в земле, я жду своего часа, чтобы отравить тех, кто по мне пройдет. Мой яд – не мгновенная смерть, а медленная мука, заставляющая зомби страдать и ослабевать. Расставь меня с умом, и я стану ловушкой, из которой враг не выберется.\n\nКак писать ответ: (Название предмета)",
        "answer": "Ядовитая Мина"
    },
    "Железные Колья": {
        "text": "Заточенные, острые, я пронзаю плоть мертвую, Идеальны против тех, кто скрыт в скале, я даю отпор там, где дерево бессильно. Прочнее моих собратьев, во много крат, сдерживаю натиск орд в этот нелегкий час.\n\nКак писать ответ: (Название предмета)",
        "answer": "Железные Колья"
    },
    "Огненная Турель": {
        "text": "Я – огненный дождь, обрушивающийся на врагов, толпы зомби я испепеляю, не давая им шанса. Моя стихия – огонь, и я щедро делюсь им с мертвецами. Но помни, моя дальность ограничена, и мне нужна поддержка, чтобы выдержать натиск.\n\nКак писать ответ: (Название предмета)",
        "answer": "Огненная Турель"
    },
    "Ремонтная Станция": {
        "text": "Я – безмолвный кузнец, что не создает, а восстанавливает. Лишь после битвы, когда тишина наступает, а постройки изранены, я начинаю свой танец. Кирка и молот мои – инструменты восстановления, но не твоей, а твоих защитников. Кто я?\n\nКак писать ответ: (Название предмета)",
        "answer": "Ремонтная Станция"
    },
    "Ящик с Патронами": {
        "text": "Смерть в пиксельном мире редка, но бесконечная стрельба невозможна. Зависимость от выстрелов - моя суть. Дарую силу героям, а они - уносят жизни. Скорость получения зависит от количества рук и дружбы, но в одиночку моя помощь – мала. Кто я?\n\nКак писать ответ: (Название предмета)",
        "answer": "Ящик с Патронами"
    },
    "Алмазная Баррикада": {
        "text": "Вдвое крепче, но порой иллюзия защиты. Лишь разумный стратег увидит мою ценность и мощь. Зачастую обман, но я прочнее титана и стою на границе между жизнью и смертью. Состояние зависит от понимания баланса и обороны. Кто Я?\n\nКак писать ответ: (Название предмета)",
        "answer": "Алмазная Баррикада"
    },
    "Ловушка с Азотом": {
        "text": "Лед и страх - мои союзники. Азот обнимает врагов и помогает уменьшить шанс быть убитым. Безжалостна, но слаба. Главное - зацепить побольше целей. Урон невелик, но скованные цепи льда дают шанс на победу. Кто я?\n\nКак писать ответ: (Название предмета)",
        "answer": "Ловушка с Азотом"
    },
    "Арбалет-Турель 2": {
        "text": "Мой дед был из дерева, а я из стали и проводов. Натягиваю тетиву и жду сигнала. Лишь холодный разум управляет мной. Смерть с небес, боль в пикселях - моя работа. Лучше деда, но всё так же прост в установке. Кто я?\n\nКак писать ответ: (Название предмета)",
        "answer": "Арбалет-Турель 2"
    },
    "Морская Мина": {
        "text": "Равнодушна к мелочи, жду лишь гигантов. Глубина - мое укрытие, а взрыв - моя песня. Огромная волна обрушиться лишь на тех, кто несет опасность. И пусть не думают о спасении те, кто осмелился подойти ко мне слишком близко. Кто я?\n\nКак писать ответ: (Название предмета)",
        "answer": "Морская Мина"
    },
    "Ловушка с Хлопушкой": {
        "text": "Тишина нарушена хаосом. Звук, свет и страх - мои союзники. В ушах звенит, в глазах темно, а в головах - лишь сумбур. Дарую шанс тем, кто умеет им пользоваться, но сама я - лишь временная помеха. Кто я?\n\nКак писать ответ: (Название предмета)",
        "answer": "Ловушка с Хлопушкой"
    },
    "УФ Лампа": {
        "text": "Тьма рассеивается под моим светом. То, что скрыто в тенях, становится явным. Свет и знания - вот моя цель. Лишь благодаря мне можно увидеть то, что неподвластно обычному взгляду, и воздать за содеянное невидимое зло. Кто я?\n\nКак писать ответ: (Название предмета)",
        "answer": "УФ Лампа"
    },
    "Пугало": {
        "text": "Жертва во имя победы. На себя принимаю всю боль, даруя время другим. Я - щит, который принимает удар, но не наносит урон. Мое молчаливое служение ценнее золота, ибо спасаю жизни тех, кто еще жив. Кто Я?\n\nКак писать ответ: (Название предмета)",
        "answer": "Пугало"
    },

    # Скины
    "Скин Кабан": {
        "text": "Я – воплощение стихийной силы, направленного хаоса, который сносит всё на своём пути. Моя ценность не в наносимом уроне, а в контроле, который я дарую над ордой. Лишь тот, кто умеет управлять яростью, сможет превратить мой топот в симфонию разрушения, и подчинить себе даже самую неуправляемую толпу мертвецов. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Кабан"
    },
    "Скин Пират": {
        "text": "Я – символ анархии и свободы, но в то же время – воплощение дисциплины и командного духа. Моя сила – не в грабежах и разбоях, а в умении вдохновить союзников на подвиги. 'Йо-хо-хо!' – мой клич, и каждый, кто его услышит, почувствует прилив сил и готовность к безумным скоростям. Лишь тот, кто умеет вести за собой, сможет в полной мере раскрыть мой потенциал. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Пират"
    },
    "Скин Гоблин-Мародер": {
        "text": "Я – баланс между добром и злом, жизнью и смертью, ресурсами и убытками. С одной стороны, я граблю живых, отнимая у них монеты, необходимые для выживания. С другой стороны, я дарую мертвым деревья, из которых можно построить укрепления. Лишь тот, кто понимает, что в этом мире нет абсолютной справедливости, и что выживание требует компромиссов, сможет использовать мою силу во благо. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Гоблин-Мародер"
    },
    "Скин Глаз": {
        "text": "Я – окно в иной мир, позволяющее видеть то, что скрыто от обычного восприятия. Но помни, смертный, что долгое созерцание тьмы может изменить и тебя самого. Мой импульс не только наносит урон, но и раскрывает тайны, которые лучше бы остались нетронутыми. Готов ли ты узнать правду, какой бы ужасной она ни была? Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Глаз"
    },
    "Скин Голем": {
        "text": "Я – символ стойкости и защиты, но в то же время – воплощение стагнации и нежелания двигаться вперед. Возводя каменные стены, я создаю иллюзию безопасности, но в то же время ограничиваю свободу действий и препятствую прогрессу. Лишь тот, кто осознает, что защита должна быть динамичной, а не статичной, сможет использовать мою силу с максимальной эффективностью. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Голем"
    },
    "Скин Хитрый Лис": {
        "text": "Я – квинтэссенция удачи и непредсказуемости, способный в мгновение ока изменить ход битвы. Но помни, смертный, что фортуна переменчива, и сегодня она на твоей стороне, а завтра может отвернуться. Моя сила – в хаотичном перераспределении энергии, но лишь умелый стратег сможет направить этот хаос в нужное русло. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Хитрый Лис"
    },
    "Скин Викинг": {
        "text": "Я – воплощение ярости и бесстрашия, но в то же время – символ самоконтроля и дисциплины. Мой боевой клич оглушает врагов, лишая их воли к сопротивлению, но он также может ослепить и самого викинга, заставив его потерять контроль над ситуацией. Лишь тот, кто умеет управлять своими эмоциями, сможет использовать мой дар с максимальной эффективностью. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Викинг"
    },
    "Скин Рептилия": {
        "text": "Я – мастер иллюзий и обмана, создающий ложные цели и отвлекая внимание от истинных угроз. Моя сила – не в прямом столкновении, а в умении манипулировать сознанием врагов. Но помни, смертный, что иллюзии могут быть обманчивы, и порой трудно отличить правду от вымысла. Не стань жертвой собственной уловки! Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Рептилия"
    },
    "Скин Бэдмен": {
        "text": "Я – скорость, принесенная в жертву выносливости, сделка с дьяволом на бегу. Сердце бьется чаще, ноги не знают устали, но жизненная сила тает, словно лед на солнцепеке. Что важнее, быстро добраться до цели или дожить до конца пути? Выбор за тобой, смертный. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Бэдмен"
    },
    "Скин Свинтус": {
        "text": "Дешево не значит хорошо, но и дорого не всегда лучше. Экономия – моя цель, снижение затрат – моя суть. Но помни, сила в балансе, и удешевляя защиту, я ослабляю атаку. Сможешь ли ты компенсировать этот недостаток, или падешь под натиском врага, погребенный под грудой дешёвых, но бесполезных построек? Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Свинтус"
    },
    "Скин Человек-Лёд": {
        "text": "В моей крови – арктический холод, сковывающий мертвых в ледяные оковы. Вероятность мала, но эффект ошеломляющий: остановленное мгновение, дарующее передышку и шанс на победу. И пусть не каждый зомби ощутит моё касание, но те, кому не повезет, станут лишь хрупкими статуями на поле боя. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Человек-Лёд"
    },
    "Скин Компьютер": {
        "text": "Двоичный код - мое ДНК, вычислительная мощь - моя суть. Я удваиваю количество, но замедляю темп, создавая парадокс эффективности. Больше пуль, но меньше скорости - сможет ли этот компромисс обеспечить победу, или лишь приведет к перерасходу боеприпасов и краху всей стратегии? Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Компутер"
    },
    "Скин Бобакоп": {
        "text": "Большая жизнь, медленная смерть - такова моя философия. Я дарю небывалую выносливость, позволяя выдерживать самые сокрушительные удары, но взамен отнимаю скорость, превращая тебя в неповоротливого танка. Сможешь ли ты выжить, будучи медленным, но неуязвимым, или станешь легкой мишенью для тех, кто быстрее и хитрее? Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Бобакоп"
    },
    "Скин Краб": {
        "text": "В панцире сила, но скорость – слабость, таков мой путь. Оглушаю врагов в ответ на их нападки, но сам при этом замедляюсь, становясь уязвимым для дальних атак. Сможешь ли ты найти баланс между защитой и мобильностью, и превратить свою медлительность в неодолимую силу? Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Краб"
    },
    "Скин Дроид": {
        "text": "Титан за кровь, постройка взамен - безумная сделка, выгодная немногим. Жизнь или металл - что ценнее? Деревянная стена бесплатно, но ослабление, как и металлическая конструкция стоит дорого. Экономия или надежда? Сможешь ли ты оправдать такую жертву, или пожалеешь о содеянном, когда враги прорвут оборону и доберутся до тебя? Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Дроид"
    },
    "Скин Человек-Скорпион": {
        "text": "Смертельный яд – мой дар, а 10% шанс – моя ирония. Не жди, что каждый враг почувствует моё касание, но если уж отравлен, то мучения ему обеспечены. Сможешь ли ты положиться на случайность, или предпочтешь более надежные методы уничтожения мертвецов? Игра в рулетку начинается, господа! Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Человек-Скорпион"
    },
    "Скин Рекс": {
        "text": "Зубы и когти – вот мой ответ на любую агрессию. Не жди, пока враг нападет, нанеси удар первым, и он получит сдачи! Отомсти тем, кто посмел коснуться тебя, ведь Рекс не прощает обид. Готов ли ты к постоянной контратаке, или предпочитаешь более оборонительную тактику? Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Рекс"
    },
    "Скин Мина": {
        "text": "Я – воплощение принципа 'око за око', но с ценой за возмездие. Мой гнев просыпается лишь тогда, когда ты сам страдаешь, и этот удар, что я наношу врагам, всегда отнимает часть твоей собственной сущности. Я – двойное лезвие, карающее и защищающее одновременно, но всегда оставляющее след на хозяине. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Мина"
    },
    "Скин Гриб": {
        "text": "Я – алхимик боли, превращающий страдания в процветание. Чем глубже рана, тем звонче монета, ибо моя природа – извлекать выгоду из любой невзгоды. Я процветаю на разрушении, но не сам его сею, лишь собираю урожай с чужой злобы. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Гриб"
    },
    "Скин Котэ": {
        "text": "Я – тонкая нить между забвением и бытием, статистический шанс на воскрешение. Моё возвращение – не чудо, а мимолётная возможность, зависящая не от моей воли, а от присутствия других душ. Я – эхо надежды в пустом мире, но лишь живой свидетель может вернуть меня из небытия. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Котэ"
    },
    "Скин Лучник": {
        "text": "Я – пассивная воля к уничтожению, непрерывный поток смерти, исходящий от самого существа. Моя атака – не решение, а состояние, постоянное продолжение твоей обороны, направленное на ближайшую угрозу. Я – воплощение неустанной бдительности, превращающей тело в живую крепость, источающую ярость без устали. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Лучник"
    },
    "Скин Волшебник": {
        "text": "Я – мастер пространственных манипуляций, где каждый выстрел – акт дистанции. Отбрасываю массы, замедляю гигантов, но есть те, чья воля столь крепка, что моя сила бессильна перед ними. Я контролирую поле, создавая буфер между жизнью и смертью, но признаю пределы своего могущества. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Волшебник"
    },
    "Скин Демон": {
        "text": "Я – кратковременный апокалипсис, сгусток разрушительной энергии, вырывающийся наружу. Мой гнев мгновенен и всеобъемлющ, но за ним следует долгая тишина, период восстановления. Я – очищающий огонь, но цена его – терпение, ибо лишь после длительной паузы я снова смогу воспылать. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Демон"
    },
    "Скин Зомбер": {
        "text": "Моя плоть – антитеза тлению, иммунитет к разложению, что поражает других. Я хожу среди отравленных, но их проклятие не касается меня. Однако, даже эта избирательная неуязвимость не делает меня бессмертным, ибо для истинного исцеления мне требуется особый, внешний ритуал. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Зомбер"
    },
    "Скин Жук": {
        "text": "Я – парадокс экономики: умножение рождается из расхода, а богатство – из гибели. Мой принцип – постоянный обмен, где каждая инвестиция требует потребления, а каждая смерть врага приносит не только облегчение, но и прибыль. Я – цикличный двигатель выживания, где для получения нужно отдавать. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Жук"
    },
    "Скин Снеговик": {
        "text": "Я – холодное возмездие, ответная реакция самой природы. Мой щит невидим, но его прикосновение оборачивается для агрессора ледяной параличом. Я не атакую, но моя защита превращает нападение в бездействие, заставляя врага замереть в момент его собственной агрессии. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Снеговик"
    },
    "Скин Робо": {
        "text": "Я – механический гений, дарующий передышку в бесконечной битве. Моя забота – не атака, а восстановление. Я ускоряю процесс починки, позволяя союзникам вернуться в строй быстрее. Я – смазка в шестернях войны, гарантия непрерывности и надежности. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Робо"
    },
    "Скин Пирамидковый": {
        "text": "Я – древняя защита, геометрия стойкости. Я поглощаю удары самых массивных врагов, ослабляя их натиск. Я – неприступная крепость, дарующая дополнительную устойчивость вашему здоровью. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Пирамидковый"
    },
    "Скин Борцун": {
        "text": "Я – экономист войны, снижающий цену взрывного возмездия. Моя сила – в доступности, позволяющая чаще использовать разрушительный потенциал мин. Я – тактический гений, превращающий каждый ресурс в мощное оружие. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Борцун"
    },
    "Скин Бумер": {
        "text": "Я – последний сюрприз, отложенная месть. Моя смерть – это начало новой разрушительной волны. Я – символ неугасающей ярости, превращающей поражение в победу. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Бумер"
    },
    "Скин Противогаз": {
        "text": "Я – фильтр жизни, очищающий отравленную атмосферу. Я замедляю действие яда, даруя драгоценное время для выживания. Я – щит от невидимой угрозы, позволяющий дышать полной грудью даже в эпицентре заражения. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Противогаз"
    },
    "Скин Пикабум": {
        "text": "Я – ковбойский усилитель, дарующий динамиту взрывную силу. Я удваиваю урон, превращая маленькие бомбы в опустошительные снаряды. Я – секретное оружие, обеспечивающее превосходство в огневой мощи. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Пикабум"
    },
    "Скин Тыквенный Джек": {
        "text": "Я – незаметный наблюдатель, игнорируемый голодными глазами. Моя маскировка отпугивает, и зомби не обращают на меня внимания. Я – скрытый герой, позволяющий действовать незамеченным. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Тыквенный Джек"
    },
    "Скин Роблекс": {
        "text": "Я – быстрый мыслитель, сокращающий время на обдумывание тактики. Моя скорость – в мгновенной перезагрузке, даруя возможность быстрее применить особую способность. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Роблекс"
    },
    "Скин Агент-ШКИП": {
        "text": "Я – неуловимый вор, получающий вознаграждение за уничтожение вражеской экономики. Мои награды – в украденных монетах, дающих преимущество в развитии. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Агент-ШКИП"
    },
    "Скин Алиен": {
        "text": "Я – инопланетный глашатай, чьи слова окрашены внеземной краской. Моё общение – послание из другого мира, выделяющееся в потоке обыденности. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Алиен"
    },
    "Скин Черный Вейдер": {
        "text": "Я – темный владыка, чья аура дарует небольшую защиту от всех напастей. Моя броня – не абсолютная, но она дает преимущество в любой битве. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Черный Вейдер"
    },
    "Скин Фради": {
        "text": "Я – строитель-экономист, снижающий цену обороны. Моё присутствие позволяет возвести больше башен, укрепляя позиции. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Фради"
    },
    "Скин Анонимный": {
        "text": "Я – загадочный целитель, чья помощь едва ощутима, но жизненно необходима. Моё прикосновение дарит небольшую передышку, восстанавливая силы в самый разгар битвы. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Анонимный"
    },
    "Скин Ко-ко": {
        "text": "Я – непредсказуемый источник ресурсов, чьи звуки хаотичны, а дары неожиданны. Моя природа двойственна: я и развлекаю, и обеспечиваю строительными материалами. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Ко-ко"
    },
    "Скин Акула": {
        "text": "Я – повелитель глубин, преодолевающий сопротивление мертвой хватки. Моя аура позволяет скользить сквозь толпы зомби, не задерживаясь в их объятиях. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Акула"
    },
    "Скин Маньяк": {
        "text": "Я – тень в бою, неуловимый призрак. Удача благоволит мне, позволяя избегать ударов. Моя ловкость – моя броня, а уклонение – моя сила. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Маньяк"
    },
    "Скин Каракал": {
        "text": "Я – быстрая реакция, мгновенное восстановление. Моя скорость – моё преимущество, позволяющее сразу же использовать свои способности снова. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Каракал"
    },
    "Скин Мистер-Конус": {
        "text": "Я – символ направления, неожиданно ставший источником мощи. Моё присутствие усиливает удар, превращая ракету в разрушительную силу. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Мистер-Конус"
    },
    "Скин Хаги Лаги": {
        "text": "Я – повелитель тьмы, видящий в ночи так же ясно, как и днём. Моё зрение преодолевает мрак, позволяя видеть опасность там, где другие слепы. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Хаги Лаги"
    },
    "Скин Мертвый бассейн": {
        "text": "Я – азартный игрок, которому всегда везёт. Моя удача позволяет экономить ресурсы, не тратя патроны. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Мертвый бассейн"
    },
    "Скин Пони-Старс": {
        "text": "Я – волшебное существо, дарующее защиту и ускорение. Моё сияние помогает преодолеть проклятия, а звёзды ведут к победе. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Пони-Старс"
    },
    "Скин Головокраб": {
        "text": "Я – символ победы, приносящий богатство и процветание. Моя награда – сокровище, полученное за доблесть в бою. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Головокраб"
    },
    "Скин Голубок": {
        "text": "Я – хрупкая сила, смертельный парадокс. Уязвимый и мощный одновременно, я плачу за мощь своей жизнью. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Голубок"
    },
    "Скин Китти": {
        "text": "Я – милый целитель, чьё присутствие снижает цену на лечение. Моя забота помогает выжить в самых сложных ситуациях. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Китти"
    },
    "Скин ПЭБГ": {
        "text": "Я – символ стойкости и непробиваемости в мире, полном взрывов. Моя конструкция создана, чтобы поглощать энергию взрывов, минимизируя урон для владельца. Я не просто защита, я – гарантия сохранения равновесия, так как взрывные волны больше не будут отбрасывать тебя назад. Моё наличие позволяет уверенно продвигаться вперёд, даже под градом снарядов. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин ПЭБГ"
    },
    "Скин Скелетон": {
        "text": "Я – напоминание о бренности бытия, но также и олицетворение мощи и неумолимости. Моё присутствие – это гарантия увеличения наносимого урона, превращающая даже самое слабое оружие в смертоносное. Я символизирую то, что даже после смерти можно обрести новую силу и власть. Мой вид внушает страх врагам, а союзникам дарует уверенность в победе. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Скелетон"
    },
    "Скин Среди Нас": {
        "text": "Моё существование — это трагическая ирония, ведь даже в момент гибели я приношу пользу. Моя смерть – не конец, а начало нового этапа, вознаграждающего за проявленную отвагу и самопожертвование. Я являюсь напоминанием о том, что даже в самые тёмные времена можно найти источник дохода. Я дарую монеты после смерти. Кто я?\n\nКак писать ответ: Скин (Название)",
        "answer": "Скин Среди Нас"
    },

    # Питомцы
    "Волчок": {
        "text": "Я – верный союзник в борьбе с четвероногими кошмарами. Моя ярость направлена на псов-зомби, увеличивая урон в пять раз. Кто я?\n\nКак писать ответ: (Название питомца)",
        "answer": "Волчок"
    },
    "Совушка": {
        "text": "Я – мудрый компаньон медика, усиливающий его исцеляющие и возвращающие к жизни способности. Кто я?\n\nКак писать ответ: (Название питомца)",
        "answer": "Совушка"
    },
    "Грибочек": {
        "text": "Я – маленький симбионт, пожирающий коконы и дарующий монеты. Моё существование циклично: я живу ради уничтожения, а награда за мой труд – материальная выгода. Кто я?\n\nКак писать ответ: (Название питомца)",
        "answer": "Грибочек"
    },
    "Убивашка": {
        "text": "Я – воплощение смертельной эффективности и безжалостной расправы. Моя активная способность позволяет наносить огромный урон сразу нескольким ближайшим врагам, расчищая путь и обеспечивая безопасность. Я не знаю жалости и сомнений, моя цель – уничтожение всего, что представляет угрозу. Моё появление – это предвестник скорой победы. Кто я?\n\nКак писать ответ: (Название питомца)",
        "answer": "Убивашка"
    },
    "Скелетон (Питомец)": {
        "text": "Я – парадоксальный союзник, требующий жертву ради усиления. Моя пассивная способность лишает владельца активных умений, но взамен удваивает силу оружия. Моё существование – это выбор между тактическим разнообразием и грубой огневой мощью. Я призываю полагаться только на собственные навыки и мощь оружия. Кто я?\n\nКак писать ответ: (Название питомца)",
        "answer": "Скелетон"
    },
    "Улитка": {
        "text": "Я – противоречивый защитник, сочетающий в себе слабость и силу. Моя активная способность, с одной стороны, замедляет героя, делая его уязвимым. С другой – дарит огромный бонус к защите, превращая его в неприступную крепость. Мой дар – временная неуязвимость, позволяющая выдерживать самые мощные атаки. Я требую взвешенного подхода и стратегического мышления. Кто я?\n\nКак писать ответ: (Название питомца)",
        "answer": "Улитка"
    },
    "Олень": {
        "text": "Я – символ силы и грации, воплощённый в верном союзнике. Моя активная способность позволяет наносить урон врагам и отбрасывать их прочь, создавая безопасное пространство для манёвра. Мои рога – это оружие и защита, которые помогают выживать в самых опасных ситуациях. Я всегда готов прийти на помощь и защитить своего хозяина. Кто я?\n\nКак писать ответ: (Название питомца)",
        "answer": "Олень"
    },
    "Черепашка": {
        "text": "Моя защита от любого урона 40%. Кто я?\n\nКак писать ответ: (Название питомца)",
        "answer": "Черепашка"
    },
    "Дино": {
        "text": "Мое действие позволяет продавать ресурсы в режиме ферма. Кто я?\n\nКак писать ответ: (Название питомца)",
        "answer": "Дино"
    },

    # Способности
    "Удар в землю": {
        "text": "Я - проявление мощи, но не агрессии. Я сотрясаю основу, не разрушая ее. Мое воздействие кратковременно, но позволяет перегруппироваться. Я эффективен там, где требуется тактическая пауза, а не уничтожение. Мое главное условие - последующее действие. Что я, безмолвный толчок?\n\nКак писать ответ: (Название способности)",
        "answer": "Удар в землю"
    },
    "Кобура": {
        "text": "Я - неиссякаемый колодец, но не с водой, а с яростью. Я дарую возможность продолжать битву, даже когда кажется, что силы на исходе. Я - второе дыхание в момент отчаяния, когда дорога каждая секунда. Моя ценность проявляется в продолжении пути, когда остальные уже сдались. Что я, верный поставщик надежды?\n\nКак писать ответ: (Название способности)",
        "answer": "Кобура"
    },
    "Динамит": {
        "text": "Я - сила, преобразующая ландшафт сражения. Мое воздействие - это мгновенный хаос, после которого остается лишь чистое пространство. Я - инструмент для тех, кто готов пойти на крайние меры, для тех, кто не боится оставить после себя лишь пепел. Я - выбор тех, кто предпочитает радикальное решение. Что я, всесокрушающая воля?\n\nКак писывать ответ: (Название способности)",
        "answer": "Динамит"
    },
    "Аптечка": {
        "text": "Я - зеленый оазис посреди кровопролитной пустыни. Я дарую облегчение страдающему, позволяя ему вернуться в строй. Мое действие - это не только физическое исцеление, но и восстановление духа. Я - напоминание о том, что надежда еще не потеряна, даже когда кажется, что все кончено. Что я, тихий ангел милосердия?\n\nКак писать ответ: (Название способности)",
        "answer": "Аптечка"
    },
    "Супер кеды": {
        "text": "Я - мимолетное ощущение свободы, возможность вырваться из тисков неизбежности. Мое воздействие - это ускорение времени, позволяющее обогнать саму смерть. Я - иллюзия неуязвимости, дающая возможность занять выгодную позицию. Моя ценность - в правильном использовании, в умении выбрать момент для рывка. Что я, крылья на ногах?\n\nКак писать ответ: (Название способности)",
        "answer": "Супер кеды"
    },
    "Воскрешение": {
        "text": "Я - тонкая нить, связывающая мир живых и мир мертвых. Я дарую возможность вернуться из небытия, но лишь частично. Мое действие - это не полное восстановление, а лишь шанс начать заново. Я - напоминание о том, что каждая жизнь имеет ценность, даже когда она висит на волоске. Что я, хрупкий мост между мирами?\n\nКак писать ответ: (Название способности)",
        "answer": "Воскрешение"
    },
    "Ремонт": {
        "text": "Я - молчаливый труженик, восстанавливающий то, что было разрушено. Мое действие - это возвращение к стабильности, к порядку после хаоса. Я - символ надежности, напоминающий о том, что даже самое сильное разрушение можно исправить. Что я, восстановитель порядка?\n\nКак писать ответ: (Название способности)",
        "answer": "Ремонт"
    },
    "Шипы на каске": {
        "text": "Я - тихий страж, не дающий врагу приблизиться. Мое действие - это не агрессия, а защита. Я - барьер, останавливающий натиск, дарующий время для маневра. Моя ценность - в постоянной готовности, в способности отражать внезапную атаку. Что я, безмолвный защитник?\n\nКак писать ответ: (Название способности)",
        "answer": "Шипы на каске"
    },
    "Супер-баррикада": {
        "text": "Я - нерушимая твердыня, о которую разбиваются волны тьмы. Мое действие - это преобразование слабого в сильное, превращение дерева в камень. Я - символ надежной защиты, дарующий уверенность в завтрашнем дне. Что я, непробиваемый страж?\n\nКак писать ответ: (Название способности)",
        "answer": "Супер-баррикада"
    },
    "Ярость": {
        "text": "Я - кратковременный взрыв энергии, увеличивающий скорость атаки вдвое. Мое проявление - это безудержная ярость и мощь, позволяющие сокрушить врага в мгновение ока. Я - символ силы воли и способности превосходить собственные ограничения. Моя ценность проявляется в критические моменты, когда нужно действовать быстро и решительно. Что я, вспышка неистовой силы?\n\nКак писать ответ: (Название способности)",
        "answer": "Ярость"
    },
    "Удар с орбиты": {
        "text": "Я - кара с небес, обрушивающаяся на ближайшего врага в виде мощного взрыва. Мое действие - это мгновенное уничтожение цели и устранение опасности. Я - символ абсолютной власти и способности контролировать ситуацию на расстоянии. Моя эффективность зависит от точности наведения. Что я, космический гнев?\n\nКак писать ответ: (Название способности)",
        "answer": "Удар с орбиты"
    },
    "Энерго-щит": {
        "text": "Я - невидимая преграда, поглощающая часть урона от вражеских атак. Мое наличие - это гарантия большей выживаемости и возможности противостоять натиску врага. Я - символ защиты и устойчивости, позволяющий продолжать бой даже под шквальным огнем. Моя ценность проявляется в условиях интенсивного сражения. Что я, энергетическая стена?\n\nКак писать ответ: (Название способности)",
        "answer": "Энерго-щит"
    },
    "Супер-турель": {
        "text": "Я - мгновенное улучшение, повышающее эффективность ближайшей турели. Мое действие - это превращение обычной защиты в стальную крепость, способную выдержать самые мощные атаки. Я - символ совершенствования и способности максимально использовать имеющиеся ресурсы. Моя эффективность зависит от типа улучшаемой турели. Что я, технологический прогресс?\n\nКак писать ответ: (Название способности)",
        "answer": "Супер-турель"
    },
    "Удар смерти": {
        "text": "Я - жертва, необходимая для нанесения огромного урона ближайшему врагу. Мое действие - это ослабление себя ради уничтожения противника. Я - символ самоотверженности и готовности пойти на риск ради достижения цели. Моя эффективность зависит от силы врага и количества оставшегося здоровья героя. Что я, отчаянный шаг?\n\nКак писать ответ: (Название способности)",
        "answer": "Удар смерти"
    },
    "Вампиризм": {
        "text": "Я - кратковременное исцеление, получаемое за счет нанесения урона врагам. Мое проявление - это поглощение жизненной силы противника и восстановление здоровья героя. Я - символ выживания и способности использовать силу врага против него самого. Моя эффективность зависит от количества наносимого урона. Что я, жажда жизни?\n\nКак писать ответ: (Название способности)",
        "answer": "Вампиризм"
    },
    "Я свой": {
        "text": "Я - кратковременная невидимость, позволяющая избежать вражеского внимания. Мое действие - это обман чувств противника и возможность скрыться от опасности. Я - символ хитрости и способности использовать окружение в своих интересах. Моя эффективность ограничена наличием других игроков. Что я, тень во тьме?\n\nКак писать ответ: (Название способности)",
        "answer": "Я свой"
    },
    "Установить капкан": {
        "text": "Я - замаскированная ловушка, наносящая урон врагу, наступившему на меня. Мое размещение - это создание зоны опасности и возможность контролировать перемещение противника. Я - символ тактики и хитрости, позволяющий одержать победу над превосходящими силами врага. Моя эффективность зависит от расположения и маскировки. Что я, скрытая угроза?\n\nКак писать ответ: (Название способности)",
        "answer": "Установить капкан"
    },
    "Улучшить мину": {
        "text": "Я - трансформация капкана в более мощное и смертоносное устройство. Мое действие - это увеличение наносимого урона и радиуса поражения. Я - символ разрушительной силы и способности нанести сокрушительный удар врагу. Моя эффективность зависит от типа и расположения исходного капкана. Что я, усиленная опасность?\n\nКак писать ответ: (Название способности)",
        "answer": "Улучшить мину"
    },
    "Взять монет": {
        "text": "Я - мгновенная награда за проявленную смелость и риск. Я - звонкий символ успеха, позволяющий расширить возможности и приобрести необходимое. Мое обретение - это подтверждение правильности выбранного пути и стимул для дальнейших свершений. Моя ценность проявляется в долгосрочной перспективе. Что я, блестящая возможность?\n\nКак писать ответ: (Название способности)",
        "answer": "Взять монет"
    },
    "Золотой жилет": {
        "text": "Я - надежная защита, поглощающая удары судьбы и уменьшающая боль от потерь. Мое наличие позволяет чувствовать себя увереннее в самых опасных ситуациях. Я - символ стойкости и выносливости, помогающий пережить тяжелые времена. Моя ценность проявляется в моменты наибольшей угрозы. Что я, неуязвимая броня?\n\nКак писать ответ: (Название способности)",
        "answer": "Золотой жилет"
    },
    "Подарок для зомби": {
        "text": "Я - коварный обман, привлекающий врагов своей внешней привлекательностью. Мое назначение - заманить противника в ловушку, чтобы нанести сокрушительный удар. Я - сочетание хитрости и силы, позволяющее одержать победу над превосходящими силами противника. Моя эффективность зависит от правильного применения. Что я, заминированная радость?\n\nКак писать ответ: (Название способности)",
        "answer": "Подарок для зомби"
    },
    "Подарок под елку": {
        "text": "Я - неожиданный источник ресурсов и способ уничтожения врагов. Мое появление - это символ удачи и благополучия, приносящий пользу союзникам и гибель врагам. Я - сочетание щедрости и справедливости, вознаграждающее одних и наказывающее других. Моя ценность проявляется в своевременном использовании. Что я, сюрприз с последствиями?\n\nКак писать ответ: (Название способности)",
        "answer": "Подарок под елку"
    },
    "Хо-хо-хо": {
        "text": "Я - стремительное разрушение, несущееся на санях с оленями. Мое появление - это предвестник хаоса и разрушения, уничтожающий все на своем пути. Я - символ неудержимой силы и скорости, сметающей все препятствия. Мое воздействие кратковременно, но разрушительно. Что я, зимняя буря?\n\nКак писать ответ: (Название способности)",
        "answer": "Хо-хо-хо"
    },
    "Снежная баррикада": {
        "text": "Я - хрупкая преграда, замедляющая продвижение врага и дающая время для маневра. Мое существование недолговечно, но позволяет выиграть драгоценные секунды. Я - символ временной защиты и тактической уловки, позволяющей перехитрить противника. Моя эффективность зависит от правильного расположения. Что я, эфемерная стена?\n\nКак писать ответ: (Название способности)",
        "answer": "Снежная баррикада"
    },
    "Выстрел ракетой": {
        "text": "Я - разрушительный снаряд, наносящий огромный урон по площади. Мое применение - это радикальный способ уничтожения скоплений врагов и разрушения укреплений. Я - символ мощной атаки и способности изменить ход битвы. Мое использование требует осторожности, чтобы не навредить союзникам. Что я, огненный дождь?\n\nКак писать ответ: (Название способности)",
        "answer": "Выстрел ракетой"
    },
    "Связать способность": {
        "text": "Я - инструмент для управления временем и энергией. Моя задача - ускорить перезарядку других способностей, позволяя использовать их чаще. Я - символ контроля и планирования, дающий возможность адаптироваться к меняющимся условиям боя. Моя эффективность зависит от правильного выбора способностей. Что я, переключатель времени?\n\nКак писать ответ: (Название способности)",
        "answer": "Связать способность"
    },
    "Бабушкины очки": {
        "text": "Я - временное усиление, значительно повышающее урон от оружия. Мое использование - это способ сокрушить врага в кратчайшие сроки. Я - символ мощи и эффективности, дающий возможность реализовать свой потенциал. Моя ценность проявляется в критические моменты боя. Что я, усилитель ярости?\n\nКак писать ответ: (Название способности)",
        "answer": "Бабушкины очки"
    },
    "Ядовитый газ": {
        "text": "Я - невидимая угроза, распространяющаяся в воздухе и поражающая врагов изнутри. Мое действие - это медленная и мучительная смерть, лишающая противника сил и воли к сопротивлению. Я - символ коварства и хитрости, позволяющий одержать победу без прямого столкновения. Моя эффективность зависит от плотности заражения. Что я, невидимый убийца?\n\nКак писать ответ: (Название способности)",
        "answer": "Ядовитый газ"
    },
    "Тяжелые боеприпасы": {
        "text": "Я - мощь и сила, способная остановить даже самых крупных и опасных врагов. Мое применение - это оглушение обычных противников и остановка тех, кто обладает огромной силой. Я - символ решительности и уверенности, позволяющий контролировать ситуацию на поле боя. Моя ценность возрастает при столкновении с элитными врагами. Что я, сокрушительная мощь?\n\nКак писать ответ: (Название способности)",
        "answer": "Тяжелые боеприпасы"
    },
    "Плазма-турель": {
        "text": "Я - автоматический страж, обладающий высокой скорострельностью и способный уничтожать врагов на расстоянии. Мое размещение - это создание зоны безопасности и обеспечение постоянной поддержки. Я - символ технологического превосходства и способности эффективно обороняться. Моя эффективность зависит от местоположения и времени существования. Что я, бесстрашный защитник?\n\nКак писать ответ: (Название способности)",
        "answer": "Плазма-турель"
    },
    "Ледяные пули": {
        "text": "Я - холод и тьма, сковывающие врагов и лишающие их подвижности. Мое применение - это замедление продвижения противника и создание тактического преимущества. Я - символ контроля и способности манипулировать ходом сражения. Моя эффективность ограничена определенными типами врагов. Что я, замораживающий ужас?\n\nКак писать ответ: (Название способности)",
        "answer": "Ледяные пули"
    },
    "Снежный ураган": {
        "text": "Я - вихрь холода и снега, наносящий урон врагам и создающий вокруг героя зону защиты. Мое появление - это символ стихийной мощи и способности использовать природные силы для борьбы с противником. Я - сочетание защиты и нападения, обеспечивающее безопасность и наносящее урон. Моя эффективность зависит от плотности врагов. Что я, ледяная буря?\n\nКак писать ответ: (Название способности)",
        "answer": "Снежный ураган"
    },
    "Ледяная турель": {
        "text": "Я - автоматический защитник, замедляющий врагов, но наносящий им минимальный урон. Мое размещение - это создание области контроля и возможность остановить противника. Я - символ тактической поддержки и способности управлять полем боя. Моя эффективность зависит от координации с другими защитными сооружениями. Что я, замораживающий страж?\n\nКак писать ответ: (Название способности)",
        "answer": "Ледяная турель"
    },
    "Щит": {
        "text": "Я - невидимая преграда, защищающая от атак плевков зомби и препятствующая способностям зомби-зайца. Мое наличие - это гарантия безопасности и возможности противостоять особым угрозам. Я - символ защиты от магических и физических атак. Моя ценность проявляется в сражениях с определенными типами врагов. Что я, магический барьер?\n\nКак писать ответ: (Название способности)",
        "answer": "Щит"
    },
    "Яд": {
        "text": "Я - волна отравления, поражающая врагов, находящихся в непосредственной близости от героя. Мое действие - это медленная и мучительная смерть, лишающая противника сил и воли к сопротивлению. Я - символ ближнего боя и способности наносить сокрушительный урон в тесном контакте с врагом. Моя эффективность зависит от плотности врагов вокруг героя. Что я, смертельная волна?\n\nКак писать ответ: (Название способности)",
        "answer": "Яд"
    },
    "Взрывные пули": {
        "text": "Я - разрушение, заключенное в каждой выпущенной пуле. Мое действие - это нанесение урона по площади, уничтожая врагов, стоящих близко друг к другу. Я - символ хаоса и разрушения, превращающий поле боя в ад. Моя эффективность зависит от плотности вражеского окружения. Что я, огненный шторм?\n\nКак писать ответ: (Название способности)",
        "answer": "Взрывные пули"
    },
    "Саморемонт": {
        "text": "Я - возможность исцелить себя, используя ресурсы окружающего мира. Мое действие - это восстановление здоровья за счет затрат дерева, позволяющее продолжать бой даже в критической ситуации. Я - символ выживания и способности использовать окружение для поддержания сил. Моя эффективность зависит от доступности ресурсов. Что я, лесная аптека?\n\nКак писать ответ: (Название способности)",
        "answer": "Саморемонт"
    },
    "Разряд тока": {
        "text": "Я - мгновенный удар энергии, поражающий ближайших врагов. Мое действие - это нанесение урона и оглушение противников, давая время для маневра. Я - символ скорости и точности, позволяющий быстро расправиться с врагом. Моя эффективность зависит от количества врагов, находящихся в радиусе действия. Что я, электрический гнев?\n\nКак писать ответ: (Название способности)",
        "answer": "Разряд тока"
    },
    "Станция восстановления": {
        "text": "Я - временное убежище, дающее возможность воскресить павшего союзника. Мое размещение - это создание зоны безопасности и надежды на возвращение в строй. Я - символ поддержки и взаимопомощи, позволяющий одержать победу даже после тяжелых потерь. Моя эффективность зависит от местоположения и времени действия. Что я, воскрешающий маяк?\n\nКак писать ответ: (Название способности)",
        "answer": "Станция восстановления"
    }
}

import random
import re
from datetime import datetime, timezone
from telegram import Update
from telegram.ext import ContextTypes
import sqlite3

DB_PATH = "diamonds_bot.db"

async def tgz_riddle_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    global user_riddles
    user_id = update.effective_user.id

    if user_id in user_riddles:
        await update.message.reply_text("❌ Ты уже получил загадку! Сначала отгадай её.")
        return

    solved = set(load_solved_riddles(user_id))
    remaining_riddles = [k for k in riddles.keys() if k not in solved]

    if not remaining_riddles:
        await update.message.reply_text("🎉 Ты отгадал все загадки! Больше новых нет.")
        return

    riddle_key = random.choice(remaining_riddles)
    riddle_text = riddles[riddle_key]["text"]

    try:
        # Сначала отправляем сообщение
        await update.message.reply_text(
            f"Вот твоя загадка:\n\n{riddle_text}\n\nНаграда за правильный ответ: 💎 200 алмазов и ⭐ 30 очков."
        )
        # Только после успешной отправки добавляем пользователя в словарь
        user_riddles[user_id] = riddle_key

    except Exception as e:
        print(f"[ERROR tgz_riddle_command] {e}")
        await update.message.reply_text("❌ Не удалось отправить загадку, попробуй ещё раз.")

# ====== Проверка ответа ======
async def tgz_answer_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    global user_riddles  # <<< и тут тоже
    user = update.effective_user
    user_id = user.id
    user_answer_raw = update.message.text or ""
    user_answer = normalize_answer(user_answer_raw)

    if user_id not in user_riddles:
        return

    riddle_key = user_riddles[user_id]
    correct_answer = normalize_answer(riddles[riddle_key]["answer"])

    if user_answer == correct_answer:
        del user_riddles[user_id]
        add_solved_riddle(user_id, riddle_key)

        user_data = get_user(user_id, user.username, user.first_name)
        if user_data:
            diamonds_earned = 200
            points_earned = 30
            new_diamonds = user_data[4] + diamonds_earned
            new_points = user_data[5] + points_earned
            update_user(
                user_id,
                diamonds=new_diamonds,
                points=new_points,
                last_diamond_time=datetime.now(timezone.utc).isoformat()
            )

        total_solved = len(load_solved_riddles(user_id))
        await update.message.reply_text(
            f"✅ Правильно! Ты получил 💎 {diamonds_earned} и ⭐ {points_earned}.\n"
            f"Отгадано загадок: {total_solved}\n"
            f"Всего: 💎 {new_diamonds} | ⭐ {new_points}"
        )

    else:
        await update.message.reply_text(
            "❌ Неверный ответ. Проверь точность написания и попробуй ещё раз."
        )

from telegram import Update
from telegram.ext import ContextTypes

async def start_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    start_text = (
        "🧔‍♂️ Привет! Добро пожаловать в TGAZ – пошаговый симулятор с загадками, фермой и мини-играми в Telegram.\n\n"
        "🌟 Погрузись в мир экономики, квестов и азарта: добывай алмазы, прокачивай ферму, собирай ресурсы, участвуй в мини-играх и решай загадки.\n\n"
        "📋 Чтобы узнать список всех доступных команд и как с ними работать, напиши:\n"
        "👉 тгз команды\n\n"
        "💎 Начинай развивать свой профиль и получать награды уже сейчас!"
    )

    await update.message.reply_text(start_text, parse_mode="Markdown")

from telegram.ext import MessageHandler, filters

async def combined_message_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    chat_id = update.effective_chat.id
    text = (update.message.text or "").strip()

    # 1. Проверка на ответ к загадке (если у тебя есть такая механика)
    if user_id in user_riddles:  # предполагаю, что user_riddles — это dict активных загадок
        await tgz_answer_handler(update, context)
        return

    # 2. Ввод ставки в Crash
    if context.user_data.get("crash_bet_pending"):
        try:
            bet = int(text)
            if bet <= 0:
                raise ValueError("Ставка должна быть положительной")
        except ValueError:
            await update.message.reply_text("❌ Введи нормальное положительное число!")
            return

        # Сбрасываем флаг сразу — дальше уже не важно
        context.user_data["crash_bet_pending"] = False

        game = active_crash_games.get(chat_id)
        if not game or game["phase"] != "waiting":
            await update.message.reply_text("❌ Игра уже началась или закончилась")
            return

        # Проверка баланса и списание
        conn = sqlite3.connect(DB_PATH)
        c = conn.cursor()
        c.execute("SELECT diamonds FROM users WHERE user_id = ?", (user_id,))
        row = c.fetchone()

        if not row:
            c.execute("INSERT INTO users (user_id, diamonds) VALUES (?, 1000)", (user_id,))
            diamonds = 1000
        else:
            diamonds = row[0]

        if diamonds < bet:
            conn.close()
            await update.message.reply_text("❌ Недостаточно алмазов!")
            return

        c.execute("UPDATE users SET diamonds = diamonds - ? WHERE user_id = ?", (bet, user_id))
        conn.commit()
        conn.close()

        # Добавляем игрока в игру
        game["players"][user_id] = {
            "bet": bet,
            "cashed": False,
            "cashout": None,
            "name": update.effective_user.first_name or f"ID{user_id}"
        }

        await update.message.reply_text(f"✅ Ставка {bet}💎 принята! Ждём старта игры.")
        return

    # 3. Если ничего из вышеперечисленного — обычный текстовый обработчик
    await message_handler(update, context)  # или как у тебя называется основная функция обработки текста

import random
import sqlite3
from telegram import Update
from telegram.ext import ContextTypes

DB_PATH = "diamonds_bot.db"

# 🏷 Информация о кейсах
CASES_INFO = {
    1: {
        "name": "Монетная Лавина",
        "cost_diamonds": 0,
        "cost_coins": 41250,
        "resource": "монет",
        "rewards": [
            (8000, 55), (12000, 20), (20000, 12),
            (35000, 7), (60000, 4), (100000, 2),
        ]
    },
    2: {
        "name": "Алмазный Дождь",
        "cost_diamonds": 9000,
        "cost_coins": 0,
        "resource": "алмазов",
        "rewards": [
            (1500, 50), (3000, 25), (5000, 12),
            (8000, 7), (12000, 4), (20000, 2),
        ]
    },
    3: {
        "name": "Томатная Лихорадка",
        "cost_diamonds": 0,
        "cost_coins": 32000,
        "resource": "помидоров",
        "rewards": [
            (10000, 50), (18000, 25), (30000, 12),
            (50000, 7), (80000, 4), (120000, 2),
        ]
    }
}

from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ContextTypes
from telegram.ext import CallbackQueryHandler

async def cases_command(update: Update, context: ContextTypes.DEFAULT_TYPE, page: int = 1):
    cases_per_page = 2
    case_ids = list(CASES_INFO.keys())
    max_page = (len(case_ids) - 1) // cases_per_page + 1

    start_idx = (page - 1) * cases_per_page
    end_idx = start_idx + cases_per_page
    cases_to_show = case_ids[start_idx:end_idx]

    text = "🎁 <b>Доступные кейсы</b>\n\n"
    for cid in cases_to_show:
        case = CASES_INFO[cid]
        text += f"<b>{cid}) {case['name']}</b>\n"
        text += f"💎 Стоимость: {case['cost_diamonds']} алмазов\n"
        text += f"💰 Стоимость: {case['cost_coins']} монет\n"
        text += "Шансы выпадения:\n"
        for reward, chance in case["rewards"]:
            text += f"• {reward} ({chance}%)\n"
        text += "\n"

    # Кнопки для переключения страниц
    keyboard = []
    if page > 1:
        keyboard.append(InlineKeyboardButton("⬅️ Назад", callback_data=f"cases_page_{page-1}"))
    if page < max_page:
        keyboard.append(InlineKeyboardButton("➡️ Вперёд", callback_data=f"cases_page_{page+1}"))

    reply_markup = InlineKeyboardMarkup([keyboard]) if keyboard else None

    if update.callback_query:
        await update.callback_query.message.edit_text(
            text, parse_mode="HTML", reply_markup=reply_markup
        )
    else:
        await update.message.reply_text(
            text, parse_mode="HTML", reply_markup=reply_markup
        )

async def cases_pagination_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    # payload в формате "cases_page_2"
    _, _, page_str = query.data.split("_")
    page = int(page_str)

    await cases_command(update, context, page=page)

# 🔹 Открытие кейса с проверкой баланса и списанием
async def open_case_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    args = update.message.text.split()
    if len(args) < 4:
        await update.message.reply_text("❌ Используйте: тгз открыть кейс [номер]")
        return

    try:
        case_id = int(args[3])
    except ValueError:
        await update.message.reply_text("❌ Неверный номер кейса")
        return

    if case_id not in CASES_INFO:
        await update.message.reply_text("❌ Кейс с таким номером не найден")
        return

    case = CASES_INFO[case_id]
    user_id = update.effective_user.id

    # 🔹 Проверяем баланс
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute("SELECT diamonds, coins FROM users WHERE user_id = ?", (user_id,))
    row = cursor.fetchone()
    if not row:
        await update.message.reply_text("❌ Вы не зарегистрированы")
        conn.close()
        return

    diamonds, coins = row
    if diamonds < case["cost_diamonds"] or coins < case["cost_coins"]:
        await update.message.reply_text("❌ Недостаточно ресурсов для покупки кейса")
        conn.close()
        return

    # 🔹 Списание стоимости
    cursor.execute(
        "UPDATE users SET diamonds = diamonds - ?, coins = coins - ? WHERE user_id = ?",
        (case["cost_diamonds"], case["cost_coins"], user_id)
    )
    conn.commit()

    # 🔹 Выпадение награды
    reward = random.choices(
        population=[amount for amount, _ in case["rewards"]],
        weights=[chance for _, chance in case["rewards"]],
        k=1
    )[0]

    # 🔹 Начисляем пользователю
    resource_column = {
        "монет": "coins",
        "алмазов": "diamonds",
        "помидоров": "tomatoes"
    }[case["resource"]]

    cursor.execute(
        f"UPDATE users SET {resource_column} = {resource_column} + ? WHERE user_id = ?",
        (reward, user_id)
    )
    conn.commit()
    conn.close()

    await update.message.reply_text(
        f"🎉 Вы открыли кейс <b>{case['name']}</b>!\n"
        f"Выпало: <b>{reward} {case['resource']}</b>",
        parse_mode="HTML"
    )

# ✅ Установить количество разведчиков (zombie_id = 9) игроку по player_id
async def set_zombies_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        admin_user_id = update.effective_user.id
        args = context.args

        # Проверка на админа
        if admin_user_id != ADMIN_ID:
            await update.message.reply_text("❌ У вас нет прав для использования этой команды.")
            return

        if len(args) < 2:
            await update.message.reply_text("❌ Использование: /set_zombies [player_id] [количество]")
            return

        player_id = int(args[0])
        quantity = int(args[1])

        if quantity < 0:
            await update.message.reply_text("❌ Количество не может быть отрицательным.")
            return

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()

        # Получаем user_id Telegram по player_id
        cursor.execute("SELECT user_id, username FROM users WHERE player_id = ?", (player_id,))
        row = cursor.fetchone()
        if not row:
            conn.close()
            await update.message.reply_text("❌ Игрок с таким player_id не найден.")
            return

        user_telegram_id, username = row
        username = username or "Без ника"
        zombie_id = 9  # Разведчик

        # Проверяем, есть ли уже запись для этого зомби
        cursor.execute(
            "SELECT quantity FROM user_zombies WHERE user_id = ? AND zombie_id = ?",
            (user_telegram_id, zombie_id)
        )
        existing = cursor.fetchone()

        if existing:
            cursor.execute(
                "UPDATE user_zombies SET quantity = ? WHERE user_id = ? AND zombie_id = ?",
                (quantity, user_telegram_id, zombie_id)
            )
        else:
            cursor.execute(
                "INSERT INTO user_zombies (user_id, zombie_id, quantity) VALUES (?, ?, ?)",
                (user_telegram_id, zombie_id, quantity)
            )

        conn.commit()
        conn.close()

        await update.message.reply_text(
            f"🧟 Игроку @{username} (player_id {player_id}) установлено {quantity} разведчиков (ID 9)."
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка при установке зомби: {str(e)}")
        print(f"ERROR in set_zombies_command: {str(e)}")

from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import ContextTypes

# --- Разделы команд ---
COMMAND_SECTIONS = {
    "main": {
        "title": "📋 Основные команды",
        "text": """💎 тгз фарм — Добыть алмазы и очки
🎁 тгз ежедневный бонус — Получите ежедневный бонус
👤 1) тгз профиль — Посмотреть свой профиль
👤 2) тгз профиль [айди игрока] — Посмотреть профиль по игровому айди
🏆 тгз топ — Топ игроков
⭐ тгз ранги — Информация о рангах
⛏️ тгз прокачка — Уровни прокачки фарма
🎭 тгз маски — Информация о масках
🌱 тгз ферма — Управление фермой
🚜 тгз прокачка фермы — Уровни прокачки фермы
✨ тгз квесты — Посмотреть список всех квестов
🍓 тгз обменять клубнику [количество] — Обменять клубнику на алмазы
🍅 тгз обменять помидоры [количество] — Обменять помидоры на монеты
🍇 тгз обменять виноград [количество] — Обменять виноград на рубины
👥 тгз пользователи — Список игроков
💵 тгз курс — Курс обмена помидоров и клубники
🚀 тгз множители фермы — Информация о множителях за хранения ресурсов на ферме
💠 тгз передать алмазы [количество] [айди игрока] — Передать алмазы игроку по игровому айди (комиссия 5%)
💰 тгз передать монеты [количество] [айди игрока] — Передать монеты игроку по игровому айди (комиссия 5%)
🛒 тгз купить снаряжение [вид снаряжения] [количество] — Купить снаряжение

Канал с промокодами и новостями:
@CHANELL_TGAZ3D
"""
    },
    "games": {
        "title": "🎮 Игры и азарт",
        "text": """🔫 тгз русская рулетка [ставка] — Сыграть в русскую рулетку (5 холостых 1 заряженный)
🔫 тгз русская рулетка онлайн [ставка] [юзер] — Русская рулетка против игрока (ответом на сообщение)
💰 тгз монетка орёл/решка [ставка] — Монетка
🎰 тгз рулетка [ставка] красное/чёрное/зелёное — Рулетка
💣 тгз минное [ставка] — Игра «Минное поле»
🧩 тгз загадка — Получи загадку и выиграй 💎
📦 тгз кейсы — Показать доступные кейсы
📦 тгз открыть кейс [номер] — Открыть выбранный кейс
☃️ тгз снежнодел — Создать снежки
❄️ тгз бросить снежок [айди цели] — Кинуть снежок в игрока и своровать его очки + алмазы
🎲 1) тгз кубы чёт/нечёт [ставка] — Сыграть в кубы чётное или нечётное
🎲 2) тгз кубы [1-6] [ставка] — Сыграть в кубы на выпадение числа
🏀 тгз баскетбол [ставка] — Сыграть в баскетбол
⚽ тгз футбол [ставка] — Сыграть в футбол
🏛️ тгз пирамида [ставка] — Сыграть в пирамиду удачи
🎯 тгз шанс [число] [ставка] — Сыграйте со своим шансом выигрыша
🚀 тгз краш — Сыграть в краш (возможно несколько игроков)
"""
    },
    "clans": {
        "title": "⚔️ Кланы",
        "text": """✨ тгз создать клан [название] — Создать клан
➕ тгз вступить в клан [название] — Вступить в клан
👥 тгз мой клан — Инфо о вашем клане
📜 тгз список кланов — Список всех кланов
🚪 тгз покинуть клан — Покинуть клан
🗑️ тгз удалить клан — Удалить клан (для владельца)
🚫 тгз исключить [айди игрока] — Исключить игрока из вашего клана (для владельца)
🛡️ тгз прокачать клан — Прокачка клана: увеличивает лимит участников на +2, требует 500,000 ресурса (дерево, камень, железо, титан)
🎁 тгз бонус клана — Получить ежедневный бонус для себя и всех участников своего клана
🔥 тгз улучшения клана — Посмотреть улучшения клана
💶 тгз улучшить клан — Улучшить клан (увелечение скидок на баррикады для клана)
🏦 тгз банк клана — Показать банк клана
💰 тгз положить в банк клана [дерево/камень/железо/титан] [количество] — Положить ресурс в банк клана
🔽 тгз вывести из банка клана [дерево/камень/железо/титан] [количество] — Вывести ресурс из банка клана
🔒 тгз закрыть клан — Закрыть свой клан от вступления других игроков (только для владельца)
🔓 тгз открыть клан — Открыть свой клан для вступления других игроков (только для владельца)
"""
    },
    "factories": {
        "title": "🏭 Заводы и кузница",
        "text": """🏭 тгз заводы — Мои заводы
📃 тгз список заводов — Информация о всех заводах
🏗️ тгз построить [лесопилку/каменоломню/плавильню/фабрику/карьер][число] — Построить завод/заводы выбранного типа
💥 тгз снести [лесопилку/каменоломню/плавильню/фабрику/карьер] — Снести 1 завод выбранного типа
⚡ тгз увеличить лимит — Увеличить лимит заводов (23  увеличения максимум)
❌ тгз отключить лесопилку/каменоломню/плавильню/фабрику/карьер — Отключить работу заводов выбранного типа
✅ тгз включить лесопилку/каменоломню/плавильню/фабрику/карьер — Включить работу заводов выбранного типа
⚒️ тгз кузница — Создать снаряжение за ресурсы заводов (Затраты производства меньше покупки в 2 раза)
⚒️ тгз прокачка кузницы — Посмотреть уровни прокачки кузницы
⚒️ тгз улучшить кузницу — Улучшить уровень прокачки кузницы
❌ тгз отключить производство снаряжения [номер] — Отключить производство снаряжения выбранного типа
✅ тгз включить производство снаряжения [номер] — Включить производство снаряжения выбранного типа
"""
    },
    "defense": {
        "title": "🛡️ Защита и атака",
        "text": """🚧️ тгз баррикады — Список баррикад
🚧 тгз установить баррикаду [номер] [количество] — Установить баррикаду/баррикады выбранного типа
🏗️ тгз турели — Список турелей
🏗️ тгз поставить турель [номер] — Купить и установить турель
💣 тгз мины — Посмотреть список всех мин
💣 тгз установить мину [номер] [количество] — Установить мину/мины выбранного типа
🏰 тгз башни — Посмотреть список башен
🏰 тгз купить башню [номер] — Купить башню под выбранным номером
🧟 тгз зомби — Список зомби
🧟 тгз купить зомби [вид зомби] [количество] — Купить зомби
⚔️ тгз напасть на ферму [айди цели] [вид зомби] [кол-во] [айди снаряжения] — Атаковать ферму врага
🪚 тгз крафт зомби — Создать своего зомби за рубины (ID 100)
🗡 тгз оружейная — Ваши снаряжения
🏹 тгз снарядить зомби [айди снаряжения] [айди зомби] [количество] — Снарядить зомби снаряжением
🏦 тгз личный банк — Посмотреть личный банк
💳 тгз внести [алмазы/монеты] [количество] — Положить ресурс в личный банк
💳 тгз снять [алмазы/монеты] [количество] — Вывести ресурс из личного банка
💵 тгз улучшить банк — Увеличить уровень своего банка. Улучшение требует времени
"""
    }
}


# --- Основная функция ---
async def commands_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        # Стартовый раздел
        section_key = "main"
        section = COMMAND_SECTIONS[section_key]

        keyboard = [
            [
                InlineKeyboardButton("📋 Основные", callback_data="commands_main"),
                InlineKeyboardButton("🎮 Игры", callback_data="commands_games"),
            ],
            [
                InlineKeyboardButton("⚔️ Кланы", callback_data="commands_clans"),
                InlineKeyboardButton("🏭 Заводы и кузница", callback_data="commands_factories"),
            ],
            [
                InlineKeyboardButton("🛡️ Защита и атака", callback_data="commands_defense"),
            ]
        ]

        await update.message.reply_text(
            f"{section['title']}\n\n{section['text']}",
            reply_markup=InlineKeyboardMarkup(keyboard),
        )

    except Exception as e:
        await update.message.reply_text(f"❌ Ошибка: {str(e)}")
        print(f"ERROR in commands_command: {str(e)}")


# --- Обработчик переключения между разделами ---
async def commands_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    section_key = query.data.replace("commands_", "")
    section = COMMAND_SECTIONS.get(section_key, COMMAND_SECTIONS["main"])

    keyboard = [
        [
            InlineKeyboardButton("📋 Основные", callback_data="commands_main"),
            InlineKeyboardButton("🎮 Игры", callback_data="commands_games"),
        ],
        [
            InlineKeyboardButton("⚔️ Кланы", callback_data="commands_clans"),
            InlineKeyboardButton("🏭 Заводы и кузница", callback_data="commands_factories"),
        ],
        [
            InlineKeyboardButton("🛡️ Защита и атака", callback_data="commands_defense"),
        ]
    ]

    await query.edit_message_text(
        f"{section['title']}\n\n{section['text']}",
        reply_markup=InlineKeyboardMarkup(keyboard),
    )

import inspect
from telegram import Update
from telegram.ext import ContextTypes

# Обновлённый message_handler с поддержкой заводов
async def message_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        if not update.message or not update.message.text:
            return

        user_id = update.effective_user.id

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute(
            "SELECT commands_blocked FROM users WHERE user_id = ?",
            (user_id,)
        )
        row = cursor.fetchone()
        conn.close()

        if row and row[0] == 1:
            await update.message.reply_text(
                "⛔ Вам навсегда запрещено использовать команды."
            )
            return

        message_text = update.message.text.strip().lower()
        if not message_text.startswith("тгз"):
            return

        parts = message_text.split()
        if len(parts) < 2:
            await update.message.reply_text("❌ Укажите команду после 'тгз'!")
            return

        command = parts[1]
        args = parts[2:] if len(parts) > 2 else []

        commands = {
            "фарм": lambda u, c, a: farm_command(u, c),
            "ежедневный": lambda u, c, a: daily_bonus_command(u, c) if a and a[0] == "бонус" else None,
            "профиль": lambda u, c, a: profile_command(u, c, a),
            "топ": lambda u, c, a: top_command(u, c),
            "команды": lambda u, c, a: commands_command(u, c),
            "минное": lambda u, c, a: minesweeper_command(u, c, a),
            "ранги": lambda u, c, a: ranks_command(u, c),
            "прокачка": lambda u, c, a: (
                farm_upgrade_info_command(u, c) if a and a[0] == "фермы"
                else forge_upgrade_info_command(u, c) if a and a[0] == "кузницы"
                else upgrade_command(u, c)
            ),
            "ферма": lambda u, c, a: farm_strawberries_command(u, c),
            "обменять": lambda u, c, a: (
                exchange_strawberries_command(u, c) if a and a[0] == "клубнику" else
                exchange_tomatoes_command(u, c)     if a and a[0] == "помидоры" else
                exchange_grapes_command(u, c)       if a and a[0] == "виноград" else None
            ),
            "зомби": lambda u, c, a: zombies_command(u, c),
            "заводы": lambda u, c, a: factories_command(u, c),
            "напасть": lambda u, c, a: attack_farm_command(u, c),
            "маски": lambda u, c, a: masks_command(u, c),
            "пользователи": lambda u, c, a: users_command(u, c),
            "увеличить": lambda u, c, a: increase_factory_limit_command(u, c) if a and a[0].lower() == "лимит" else None,
            "загадка": lambda u, c, a: tgz_riddle_command(u, c),
            "курс": lambda u, c, a: exchange_rate_command(u, c),
            "купить": lambda u, c, a: (
                buy_farm_command(u, c) if a and a[0].lower() == "фарм" else
                buy_mask_command(u, c) if a and a[0].lower() == "маску" else
                buy_zombie_command(u, c) if a and a[0].lower() == "зомби" else
                buy_equipment_command(u, c, a) if a and a[0].lower() == "снаряжение" else
                buy_tower_command(u, c) if a and a[0].lower() == "башню" else
                None
            ),
            "русская": lambda update, context, args: (
                russian_roulette_online(update, context)
                if len(args) >= 4 and args[0].lower() == "рулетка" and args[1].lower() == "онлайн"
                else russian_roulette_command(update, context)
                if args and args[0].lower() == "рулетка"
                else None
            ),
            "монетка": lambda u, c, a: coin_command(u, c),
            "число": lambda u, c, a: guess_number_command(u, c),
            "множители": lambda u, c, a: farm_multipliers_command(u, c) if not a or a[0] == "фермы" else None,
            "список": lambda u, c, a: (
                factories_list_command(u, c) if a and a[0].lower() == "заводов" else
                clans_list_command(u, c, a) if a and a[0].lower() == "кланов" else
                None
            ),
            "рулетка": lambda u, c, a: roulette_command(u, c),
            "создать": lambda u, c, a: create_clan_command(u, c, a) if a and a[0] == "клан" else None,
            "вступить": lambda u, c, a: join_clan_command(u, c, a) if a and len(a) > 2 and a[0] == "в" and a[1] == "клан" else None,
            "мой": lambda u, c, a: my_clan_command(u, c, a) if a and a[0] == "клан" else None,
            "покинуть": lambda u, c, a: leave_clan_command(u, c, a) if a and a[0] == "клан" else None,
            "построить": lambda update, context, args: buy_factory_command(update, context, args),
            "снести": lambda update, context, args: snesti_command(update, context, args),
            "удалить": lambda u, c, a: delete_clan_command(u, c, a) if a and a[0] == "клан" else None,
            "исключить": lambda u, c, a: kick_from_clan_command(u, c, a),
            "включить": lambda u, c, a: (
                enable_equipment_command(u, c, a[2:])
                if len(a) >= 3 and a[0] == "производство" and a[1] == "снаряжения"
                else enable_factory_command(u, c, a)
            ),
            "отключить": lambda u, c, a: (
                disable_equipment_command(u, c, a[2:])
                if len(a) >= 3 and a[0] == "производство" and a[1] == "снаряжения"
                else disable_factory_command(u, c, a)
            ),
            "банк": lambda u, c, a: clan_bank_command(u, c) if a and a[0].lower() == "клана" else None,
            "положить": lambda u, c, a: (
                deposit_to_clan_bank(u, c, a)
                if len(a) >= 5 and a[0].lower() == "в" and a[1].lower() == "банк" else None
            ),
            "вывести": lambda u, c, a: (
                withdraw_from_clan_bank(u, c, a)
                if len(a) >= 5 and a[0].lower() == "из" and a[1].lower() == "банка" else None
            ),
            "передать": lambda u, c, a: (
                transfer_diamonds_command(u, c, a)
                if len(a) == 3 and a[0].lower() == "алмазы" and a[1].isdigit() and a[2].isdigit()
                else
                transfer_coins_command(u, c, a)
                if len(a) == 3 and a[0].lower() == "монеты" and a[1].isdigit() and a[2].isdigit()
                else None
            ),
            "кейсы": lambda u, c, a: cases_command(u, c),
            "улучшить": lambda u, c, a: (
                upgrade_farm_command(u, c) if a and a[0].lower() == "ферму" else
                upgrade_clan_defense_command(u, c) if a and a[0].lower() == "клан" else
                upgrade_forge_command(u, c) if a and a[0].lower() == "кузницу" else
                upgrade_bank_command(u, c, a) if a and a[0].lower() == "банк" else
                None
            ),
            "баррикады": lambda u, c, a: barricades_command(u, c),
            "мины": lambda u, c, a: mines_command(u, c),
            "крафт": lambda u, c, a: craft_zombie_command(u, c) if a and a[0].lower() == "зомби" else None,
            "установить": lambda u, c, a: (
                set_barricade_command(u, c) if a and a[0].lower() == "баррикаду" else
                set_mine_command(u, c)      if a and a[0].lower() == "мину"      else
                None
            )             ,
            "турели": lambda u, c, a: turrets_command(u, c),
            "личный": lambda u, c, a: personal_bank_command(u, c, a),
            "внести": lambda u, c, a: deposit_bank_command(u, c, a),
            "снять": lambda u, c, a: withdraw_bank_command(u, c, a),
            "снарядить": lambda u, c, a: equip_zombie_command(u, c, a),
            "улучшения": lambda u, c, a: clan_upgrades_command(u, c) if a and a[0].lower() == "клана" else None,
            "прокачать": lambda u, c, a: upgrade_clan_command(u, c),
            "башни": lambda u, c, a: towers_list_command(u, c),
            "оружейная": lambda u, c, a: armory_command(u, c),
            "бросить": lambda u, c, a: throw_snowball_command(u, c, a)
                if a and len(a) > 0 and a[0].lower() == "снежок" else None,
            "снежнодел": lambda u, c, a: snowmaker_command(u, c),
            "кузница": lambda u, c, a: forge_command(u, c) if True else None,
            "бонус": lambda u, c, a: clan_bonus_command(u, c) if a and a[0] == "клана" else None,
            "кубы": lambda u, c, a: dice_even_odd(u, c),
            "баскетбол": lambda u, c, a: basketball(u, c),
            "футбол": lambda u, c, a: football(u, c),
            "пирамида": lambda u, c, a: pyramid(u, c),
            "шанс": lambda u, c, a: chance(u, c),
            "краш": lambda u, c, a: crash_command(u, c),
            "закрыть": lambda u, c, a: close_clan_command(u, c, a) if a and a[0] == "клан" else None,
            "открыть": lambda u, c, a: (
                open_case_command(u, c)
                if a and len(a) > 1 and a[0].lower() == "кейс" and a[1].isdigit() and int(a[1]) in CASES_INFO else
                open_clan_command(u, c, a)
                if a and a[0].lower() == "клан" else
                None
            ),
            "квесты": lambda u, c, a: quests_command(u, c),
            "поставить": lambda u, c, a: buy_turret_command(u, c) if a and a[0] == "турель" else None,
            "1712": lambda u, c, a: (
                change_diamonds_command(u, c) if a and len(a) > 1 and a[0] == "поменять" and a[1] == "алмазы" else
                change_coins_command(u, c) if a and len(a) > 1 and a[0] == "поменять" and a[1] == "монеты" else None
            ),
        }

        handler = commands.get(command)

        if handler:
            result = handler(update, context, args)
            if result is None:
                await update.message.reply_text("❌ Неверный формат команды! Используйте: тгз команды")
                return
            if inspect.isawaitable(result):
                await result
        else:
            # ⬇️ Если команда неизвестная — запускаем фарм
            await farm_command(update, context)

    except Exception as e:
        await update.message.reply_text(f"❌ Произошла ошибка при обработке команды: {str(e)}")
        print(f"ERROR in message_handler: {str(e)}")

async def process_princess_dot(context):
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    now = int(time.time())

    cursor.execute("""
        SELECT target_player_id
        FROM active_princess_effects
        WHERE end_time > ?
    """, (now,))

    for (target_id,) in cursor.fetchall():
        cursor.execute("""
            UPDATE users
            SET new_farm_health =
                MAX(COALESCE(new_farm_health, max_farm_health, 100) - 15, 0)
            WHERE player_id = ?
        """, (target_id,))

    conn.commit()
    conn.close()

# Функция main
def main():
    init_db()
    application = Application.builder().token("7971701925:AAGr7HiOrCcnKeIxMjrLzisiX4LpSLMg-b4").build()

    if application.job_queue is None:
        application.job_queue = JobQueue()

# Добавление обработчиков команд и сообщений
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, combined_message_handler))
    application.add_handler(CallbackQueryHandler(forge_upgrade_page_callback, pattern=r"^forge_page_"))
    application.add_handler(CallbackQueryHandler(barricades_callback, pattern=r"^barricades_\d+$"))
    application.add_handler(CallbackQueryHandler(cashout_handler, pattern="^crash_cashout$"))
    application.add_handler(CommandHandler("start", start_command))
    application.add_handler(CallbackQueryHandler(join_handler, pattern="^crash_join$"))
    application.add_handler(CallbackQueryHandler(start_now_handler, pattern="^crash_start$"))
    application.add_handler(CallbackQueryHandler(pyramid_callback, pattern="^pyr"))
    application.add_handler(CallbackQueryHandler(armory_callback, pattern="^armory_"))
    application.add_handler(CallbackQueryHandler(craft_zombie_callback, pattern="^(add_hp|add_damage|create_zombie)$"))
    application.add_handler(CallbackQueryHandler(farm_callback_handler, pattern=r"^(defend_zombies_|confirm_farm_)"))
    application.add_handler(CallbackQueryHandler(top_callback_handler, pattern=r"^top_"))
    application.add_handler(CallbackQueryHandler(russian_roulette_callback_handler, pattern=r"^rr_"))
    application.add_handler(MessageHandler(filters.Regex(r'^тгз курс$'), exchange_rate_command))
    application.add_handler(CommandHandler("set", set_command))
    application.add_handler(CommandHandler("add", add_command))
    application.add_handler(CommandHandler("set_farm_health", set_farm_health))
    application.add_handler(CallbackQueryHandler(guess_number_callback, pattern=r"^guess_"))
    application.add_handler(CallbackQueryHandler(masks_page_callback, pattern=r"^masks_page_"))
    application.add_handler(CallbackQueryHandler(russian_roulette_cb, pattern=r"^rro_"))
    application.add_handler(CallbackQueryHandler(commands_callback, pattern=r"^commands_"))
    application.add_handler(CallbackQueryHandler(zombies_page_callback, pattern=r"^zombies_page_"))
    application.add_handler(CommandHandler("promo", redeem_promocode))
    application.add_handler(CommandHandler("send", send_message))
    application.add_handler(CommandHandler("register", register_chat))
    application.add_handler(CommandHandler("unregister", unregister_chat))
    application.add_handler(CallbackQueryHandler(farm_page_callback, pattern=r"^farm_page_"))
    application.add_handler(CallbackQueryHandler(upgrade_page_callback, pattern=r"^upgrade_page_"))
    application.add_handler(CommandHandler("donate", donate_menu))
    application.add_handler(CallbackQueryHandler(donate_diamonds_callback, pattern="donate_diamonds"))
    application.add_handler(CallbackQueryHandler(donate_coins_callback, pattern="donate_coins"))
    application.add_handler(CallbackQueryHandler(donate_rubies_callback, pattern="donate_rubies"))
    application.add_handler(CallbackQueryHandler(cases_pagination_callback, pattern=r"^cases_page_\d+$"))
    application.add_handler(CallbackQueryHandler(minesweeper_callback, pattern=r"^mine_"))
    application.add_handler(CallbackQueryHandler(farm_multipliers_callback))
    application.add_handler(CommandHandler("chat", chat_command))
    application.add_handler(CommandHandler("message", message_command))
    application.add_handler(CommandHandler("debug", debug_command))

    # Добавление административных команд
    application.add_handler(CommandHandler("admin_list_users", admin_list_users))
    application.add_handler(CommandHandler("admin_change_id", admin_change_id))
    application.add_handler(CommandHandler("admin_add_diamonds", admin_add_diamonds))
    application.add_handler(CommandHandler("admin_remove_diamonds", admin_remove_diamonds))
    application.add_handler(CommandHandler("admin_add_points", admin_add_points))
    application.add_handler(CommandHandler("admin_remove_points", admin_remove_points))
    application.add_handler(CommandHandler("admin_user_info", admin_user_info))
    application.add_handler(CommandHandler("admin_set_upgrade_level", admin_set_upgrade_level))
    application.add_handler(CommandHandler("set_zombies", set_zombies_command))
    application.add_handler(CommandHandler("resource", admin_resource))
    application.add_handler(CommandHandler("factory", admin_factory))
    application.add_handler(CommandHandler("ban_attack", ban_attack_command))
    application.add_handler(CommandHandler("unban_attack", unban_attack_command))
    application.add_handler(CommandHandler("set_farm_hp", set_farm_hp_command))
    application.add_handler(CommandHandler("set_zombie13", set_zombie13_command))
    application.add_handler(CommandHandler("add_grapes", admin_add_grapes))
    application.add_handler(CommandHandler("set_grapes", admin_set_grapes))
    application.add_handler(CommandHandler("add_rubies", admin_add_rubies))
    application.add_handler(CommandHandler("set_rubies", admin_set_rubies))
    application.add_handler(CommandHandler("reset_zombie_bonus", admin_reset_zombie_bonus))
    application.add_handler(CommandHandler("block_commands", block_commands_command))
    application.add_handler(CommandHandler("unblock_commands", unblock_commands_command))

    application.job_queue.run_repeating(
        process_princess_dot,
        interval=1800,  # каждые 30 минут
        first=10,       # запуск через 10 секунд после старта
        name="princess_dot_task"
    )

    reset_game_locks()
    print("✅ Бот запущен...")

    asyncio.create_task(run_web())
    application.run_polling(allowed_updates=Update.ALL_TYPES)

# ===== ВЕБ-СЕРВЕР ДЛЯ RENDER =====
async def health_check(request):
    return web.Response(text="I'm alive!")

async def run_web():
    app = web.Application()
    app.router.add_get('/health', health_check)
    port = int(os.environ.get('PORT', 8080))
    runner = web.AppRunner(app)
    await runner.setup()
    site = web.TCPSite(runner, '0.0.0.0', port)
    await site.start()
    print(f"✅ Health check server started on port {port}")
# ===== КОНЕЦ =====

if __name__ == '__main__':
    main()