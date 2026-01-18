# -*- coding: utf-8 -*-

import asyncio
from datetime import timedelta

from aiogram import Bot, Dispatcher, types, F
from aiogram.filters import Command, CommandStart
from aiogram.enums import ChatType, ParseMode

TOKEN = "7665134449:AAGFXXqJSTzioOtbbT9kVLBsvwR0G3wi2BY"

bot = Bot(TOKEN, parse_mode=ParseMode.HTML)
dp = Dispatcher()

# ================= ХРАНЕНИЕ =================

admins = set()
mods = set()
helpers = set()
souchs = set()

rules_text = "Правила ещё не установлены."
warns = {}
logs = []

BAD_WORDS = ["porn", "xxx", "sex", "18+", "порно", "секс", "хуй", "еб"]
BAD_LINKS = ["pornhub", "xvideos", "xnxx", "onlyfans"]

# ================= УТИЛИТЫ =================

async def get_role(message: types.Message):
    member = await bot.get_chat_member(message.chat.id, message.from_user.id)

    if member.status == "creator":
        return "owner"
    if member.status == "administrator":
        return "admin"
    if message.from_user.id in souchs:
        return "souch"
    if message.from_user.id in mods:
        return "mod"
    if message.from_user.id in helpers:
        return "helper"
    return "user"

def role_name(role):
    return {
        "owner": "👑 Владелец",
        "admin": "🛡 Админ",
        "souch": "🤝 Соучредитель",
        "mod": "🔧 Модератор",
        "helper": "🆘 Хелпер",
        "user": "👤 Участник"
    }[role]

async def log(text, chat_id):
    logs.append(text)
    for uid in admins | souchs:
        try:
            await bot.send_message(uid, f"📄 LOG:\n{text}")
        except:
            pass

# ================= АНТИ ПОРНО =================

@dp.message(F.chat.type.in_({ChatType.GROUP, ChatType.SUPERGROUP}))
async def anti_porno(message: types.Message):
    if not message.text:
        return

    t = message.text.lower()

    for w in BAD_WORDS + BAD_LINKS:
        if w in t:
            await message.delete()
            await log(
                f"{message.from_user.full_name} — удалено сообщение (порно)",
                message.chat.id
            )
            return

# ================= START =================

@dp.message(CommandStart())
async def start(message: types.Message):
    await message.answer(
        "👋 Привет!\n"
        "Я бот-модератор для групп.\n\n"
        "📌 /commands — команды"
    )

# ================= COMMANDS =================

@dp.message(Command("commands"))
async def commands(message: types.Message):
    await message.answer(
        "/start\n"
        "/help\n"
        "/rules\n"
        "/report\n"
        "/info\n"
        "/staff\n"
        "/ban\n"
        "/mute\n"
        "/warn\n"
        "/logs\n"
        "/setrules\n"
        "/setadmin /setmod /sethelper /setsouch"
    )

@dp.message(Command("help"))
async def help_cmd(message: types.Message):
    await message.answer("🆘 Используй /report или обратись к администрации")

# ================= RULES =================

@dp.message(Command("rules"))
async def rules(message: types.Message):
    await message.answer(f"📜 <b>Правила:</b>\n\n{rules_text}")

@dp.message(Command("setrules"))
async def set_rules(message: types.Message):
    role = await get_role(message)
    if role not in ("owner", "admin"):
        return await message.answer("❌ Нет прав")

    global rules_text
    rules_text = message.text.replace("/setrules", "").strip()
    await message.answer("✅ Правила обновлены")

# ================= INFO / STAFF =================

@dp.message(Command("info"))
async def info(message: types.Message):
    role = await get_role(message)
    await message.answer(
        f"ℹ️ <b>Информация</b>\n"
        f"👤 {message.from_user.full_name}\n"
        f"🎭 {role_name(role)}"
    )

@dp.message(Command("staff"))
async def staff(message: types.Message):
    text = "👮 <b>Состав:</b>\n\n"
    text += f"🤝 Соучредители: {len(souchs)}\n"
    text += f"🛡 Админы: {len(admins)}\n"
    text += f"🔧 Модераторы: {len(mods)}\n"
    text += f"🆘 Хелперы: {len(helpers)}"
    await message.answer(text)

# ================= REPORT =================

@dp.message(Command("report"))
async def report(message: types.Message):
    if not message.reply_to_message:
        return await message.answer("Ответь на сообщение")

    text = (
        f"🚨 REPORT\n"
        f"От: {message.from_user.full_name}\n"
        f"На: {message.reply_to_message.from_user.full_name}"
    )

    for uid in admins | souchs:
        await bot.send_message(uid, text)

    await message.answer("✅ Репорт отправлен")

# ================= MODERATION =================

@dp.message(Command("warn"))
async def warn(message: types.Message):
    role = await get_role(message)
    if role not in ("owner", "admin", "mod"):
        return

    if not message.reply_to_message:
        return

    uid = message.reply_to_message.from_user.id
    warns[uid] = warns.get(uid, 0) + 1

    await message.answer(
        f"⚠️ Предупреждение ({warns[uid]}/3)"
    )

    if warns[uid] >= 3:
        await bot.ban_chat_member(message.chat.id, uid)
        await message.answer("⛔ Пользователь забанен")

@dp.message(Command("mute"))
async def mute(message: types.Message):
    role = await get_role(message)
    if role not in ("owner", "admin", "mod"):
        return

    if not message.reply_to_message:
        return

    await bot.restrict_chat_member(
        message.chat.id,
        message.reply_to_message.from_user.id,
        permissions=types.ChatPermissions(),
        until_date=timedelta(hours=1)
    )

    await message.answer("🔇 Пользователь замучен на 1 час")

@dp.message(Command("ban"))
async def ban(message: types.Message):
    role = await get_role(message)
    if role not in ("owner", "admin"):
        return

    if not message.reply_to_message:
        return

    await bot.ban_chat_member(
        message.chat.id,
        message.reply_to_message.from_user.id
    )
    await message.answer("⛔ Пользователь забанен")

# ================= SET ROLES =================

@dp.message(Command("setadmin"))
async def set_admin(message: types.Message):
    if await get_role(message) != "owner":
        return

    admins.add(message.reply_to_message.from_user.id)
    await message.answer("✅ Админ назначен")

@dp.message(Command("setmod"))
async def set_mod(message: types.Message):
    mods.add(message.reply_to_message.from_user.id)
    await message.answer("✅ Модератор назначен")

@dp.message(Command("sethelper"))
async def set_helper(message: types.Message):
    helpers.add(message.reply_to_message.from_user.id)
    await message.answer("✅ Хелпер назначен")

@dp.message(Command("setsouch"))
async def set_souch(message: types.Message):
    souchs.add(message.reply_to_message.from_user.id)
    await message.answer("✅ Соучредитель назначен")

# ================= LOGS =================

@dp.message(Command("logs"))
async def show_logs(message: types.Message):
    role = await get_role(message)
    if role not in ("owner", "admin"):
        return

    await message.answer("\n".join(logs[-10:]) or "Логов нет")

# ================= RUN =================

async def main():
    print("BOT STARTED")
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
