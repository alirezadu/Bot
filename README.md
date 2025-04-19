import logging
import json
from datetime import datetime
from aiogram import Bot, Dispatcher, types
from aiogram.types import ReplyKeyboardMarkup, KeyboardButton
from aiogram.enums import ParseMode
import asyncio
from keep_alive import keep_alive

# ---------------- CONFIG ----------------
API_TOKEN = '7796593965:AAHLHcBhTwRCigv6IBsmJHYYM16I_QjsEHA'
MAIN_ADMIN_ID = 928758237  # آیدی عددی خودت
# ---------------------------------------

bot = Bot(token=API_TOKEN, parse_mode=ParseMode.HTML)
dp = Dispatcher()

CONFIG_FILE = 'configs.json'
PURCHASE_FILE = 'purchases.json'
ADMINS_FILE = 'admins.json'

def load_file(filename, default):
    try:
        with open(filename, 'r') as f:
            return json.load(f)
    except:
        return default

def save_file(filename, data):
    with open(filename, 'w') as f:
        json.dump(data, f)

# ---------------- UI ----------------
def user_menu():
    kb = ReplyKeyboardMarkup(resize_keyboard=True)
    kb.add(KeyboardButton("تست رایگان"))
    kb.add(KeyboardButton("خرید سرویس"))
    kb.add(KeyboardButton("ارتباط با پشتیبانی"))
    return kb

# ---------------- START ----------------
@dp.message(commands=["start"])
async def start_cmd(message: types.Message):
    await message.answer("سلام! به ربات فروش VPN خوش اومدی.", reply_markup=user_menu())

# ---------------- TEST CONFIG ----------------
@dp.message(lambda m: m.text == "تست رایگان")
async def test_config(message: types.Message):
    configs = load_file(CONFIG_FILE, {"test": [], "paid": []})
    admins = load_file(ADMINS_FILE, [MAIN_ADMIN_ID])
    if configs['test']:
        cfg = configs['test'].pop(0)
        save_file(CONFIG_FILE, configs)
        await message.answer(f"کانفیگ تست رایگان شما:\n\n{cfg}")
        for admin in admins:
            await bot.send_message(admin, f"کاربر @{message.from_user.username or 'ندارد'} تست رایگان دریافت کرد.")
    else:
        await message.answer("کانفیگ تست موجود نیست.")

# ---------------- BUY ----------------
@dp.message(lambda m: m.text == "خرید سرویس")
async def buy_start(message: types.Message):
    await message.answer("لطفاً رسید پرداخت خود را به صورت عکس یا متن ارسال کنید.")

@dp.message(lambda m: m.content_type in ["photo", "text"])
async def handle_receipt(message: types.Message):
    if message.from_user.id in load_file(ADMINS_FILE, [MAIN_ADMIN_ID]):
        return

    data = {
        "user_id": message.from_user.id,
        "username": message.from_user.username or "ندارد",
        "text": message.text or '',
        "photo": message.photo[-1].file_id if message.photo else '',
        "date": datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    }

    purchases = load_file(PURCHASE_FILE, [])
    purchases.append(data)
    save_file(PURCHASE_FILE, purchases)

    await message.answer("رسید شما دریافت شد، منتظر تایید ادمین باشید.")

    msg = f"سفارش جدید:\n\nUser: @{data['username']}\nID: {data['user_id']}\nDate: {data['date']}"
    if data['text']:
        msg += f"\n\n{data['text']}"
    for admin in load_file(ADMINS_FILE, [MAIN_ADMIN_ID]):
        if data['photo']:
            await bot.send_photo(admin, data['photo'], caption=msg)
        else:
            await bot.send_message(admin, msg)

# ---------------- CONFIRM ----------------
@dp.message(commands=["confirm"])
async def confirm_cmd(message: types.Message):
    if message.from_user.id not in load_file(ADMINS_FILE, [MAIN_ADMIN_ID]):
        return

    configs = load_file(CONFIG_FILE, {"test": [], "paid": []})
    purchases = load_file(PURCHASE_FILE, [])

    if not purchases or not configs['paid']:
        await message.answer("سفارشی موجود نیست یا کانفیگ پولی تموم شده.")
        return

    order = purchases.pop(0)
    config = configs['paid'].pop(0)

    save_file(PURCHASE_FILE, purchases)
    save_file(CONFIG_FILE, configs)

    await bot.send_message(order['user_id'], f"خرید شما تایید شد:\n\n{config}")
    await message.answer(f"ارسال شد برای @{order['username']}")

# ---------------- ADMIN: STOCK ----------------
@dp.message(commands=["stock"])
async def stock_cmd(message: types.Message):
    if message.from_user.id not in load_file(ADMINS_FILE, [MAIN_ADMIN_ID]):
        return

    configs = load_file(CONFIG_FILE, {"test": [], "paid": []})
    await message.answer(f"تعداد کانفیگ تست: {len(configs['test'])}\nتعداد کانفیگ پولی: {len(configs['paid'])}")

# ---------------- ADMIN: ORDERS ----------------
@dp.message(commands=["orders"])
async def orders_cmd(message: types.Message):
    if message.from_user.id not in load_file(ADMINS_FILE, [MAIN_ADMIN_ID]):
        return

    purchases = load_file(PURCHASE_FILE, [])
    if not purchases:
        await message.answer("سفارشی ثبت نشده.")
        return

    msg = "\n\n".join([f"{p['username']} - {p['date']}" for p in purchases])
    await message.answer(f"لیست سفارش‌ها:\n\n{msg}")

# ---------------- ADMIN: ADD ADMIN ----------------
@dp.message(commands=["add_admin"])
async def add_admin_cmd(message: types.Message):
    if message.from_user.id != MAIN_ADMIN_ID:
        return

    parts = message.text.split()
    if len(parts) != 2:
        await message.answer("مثال: /add_admin 123456789")
        return

    new_admin = int(parts[1])
    admins = load_file(ADMINS_FILE, [MAIN_ADMIN_ID])
    if new_admin not in admins:
        admins.append(new_admin)
        save_file(ADMINS_FILE, admins)
        await message.answer("ادمین اضافه شد.")
    else:
        await message.answer("این ادمین قبلاً اضافه شده.")

# ---------------- FILTER BAD WORDS ----------------
bad_words = ['vpnfree', 'proxy', 'spam']

@dp.message(lambda m: m.text and any(word in m.text.lower() for word in bad_words))
async def filter_bad_words(message: types.Message):
    await message.delete()
    await bot.send_message(message.chat.id, "پیام شما مجاز نیست.")

# ---------------- SUPPORT ----------------
@dp.message(lambda m: m.text == "ارتباط با پشتیبانی")
async def contact_support(message: types.Message):
    await message.answer("برای پشتیبانی با آیدی زیر در تماس باشید:\n@YourSupportID")

# ---------------- RUN ----------------
async def main():
    await dp.start_polling(bot)

if __name__ == "__main__":
    keep_alive()  # فعال‌سازی وب‌سرور برای جلوگیری از خواب
    asyncio.run(main())
