import telebot
from telebot import types
import sqlite3
import requests
from datetime import datetime

bot = telebot.TeleBot("8104258718:AAGSc6fUduEs3DYic10PcGK27FwLTfHVCpQ")
ADMIN_ID = 7956920182

# База данных
conn = sqlite3.connect("users.db", check_same_thread=False)
cursor = conn.cursor()
cursor.execute("""CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY,
    username TEXT,
    full_name TEXT,
    premium INTEGER DEFAULT 0,
    banned INTEGER DEFAULT 0,
    is_admin INTEGER DEFAULT 0,
    joined_at TEXT)""")
cursor.execute("""CREATE TABLE IF NOT EXISTS history (
    user_id INTEGER,
    username TEXT,
    changed_at TEXT)""")
conn.commit()

def add_user(user):
    cursor.execute("SELECT username FROM users WHERE id = ?", (user.id,))
    row = cursor.fetchone()
    new_username = user.username
    if row:
        old_username = row[0]
        if old_username != new_username:
            cursor.execute("INSERT INTO history (user_id, username, changed_at) VALUES (?, ?, ?)",
                           (user.id, new_username, datetime.now().isoformat()))
        cursor.execute("UPDATE users SET username = ? WHERE id = ?", (new_username, user.id))
    else:
        cursor.execute("INSERT INTO users (id, username, full_name, joined_at) VALUES (?, ?, ?, ?)",
            (user.id, new_username, f"{user.first_name} {user.last_name or ''}", datetime.now().isoformat()))
    conn.commit()

def is_admin(uid):
    cursor.execute("SELECT is_admin FROM users WHERE id = ?", (uid,))
    result = cursor.fetchone()
    return result and result[0] == 1 or uid == ADMIN_ID

def set_status(uid, key, value):
    cursor.execute(f"UPDATE users SET {key} = ? WHERE id = ?", (value, uid))
    conn.commit()

@bot.message_handler(commands=["start"])
def start(msg):
    add_user(msg.from_user)
    bot.reply_to(msg, "👋 Привет! LifeStan готов к работе.\n/help — команды")

@bot.message_handler(commands=["help"])
def help_msg(msg):
    bot.reply_to(msg, "/ip <ip>\n/username <ник>\n/email <почта>\n/phone <номер>\n/history <id>\n/admin — админка\n/giveadmin <user_id>")

@bot.message_handler(commands=["ip"])
def ip_lookup(msg):
    add_user(msg.from_user)
    args = msg.text.split()
    if len(args) < 2:
        return bot.reply_to(msg, "/ip <адрес>")
    try:
        r = requests.get(f"http://ip-api.com/json/{args[1]}").json()
        if r["status"] == "success":
            cursor.execute("SELECT premium FROM users WHERE id = ?", (msg.from_user.id,))
            prem = cursor.fetchone()
            is_premium = prem and prem[0] == 1
            extra = (f"\nРегион: {r.get('regionName')}\nZIP: {r.get('zip')}\nВремя: {r.get('timezone')}"
                     if is_premium else "\nℹ️ Больше данных — купи Премиум")
            bot.reply_to(msg, f"🌍 IP: {r['query']}\nСтрана: {r['country']}\nГород: {r['city']}\n"
                              f"Провайдер: {r['isp']}\nКоординаты: {r['lat']},{r['lon']}{extra}")
        else:
            bot.reply_to(msg, "❌ Не найдено.")
    except:
        bot.reply_to(msg, "⚠️ Ошибка запроса.")

@bot.message_handler(commands=["username"])
def username_lookup(msg):
    args = msg.text.split()
    if len(args) < 2:
        return bot.reply_to(msg, "/username <имя>")
    name = args[1]
    links = f"https://github.com/{name}\nhttps://instagram.com/{name}\nhttps://tiktok.com/@{name}"
    bot.reply_to(msg, f"🔍 Поиск по нику {name}:\n{links}")

@bot.message_handler(commands=["email"])
def email_lookup(msg):
    args = msg.text.split()
    if len(args) < 2:
        return bot.reply_to(msg, "/email <email>")
    email = args[1]
    try:
        r = requests.get(f"https://haveibeenpwned.com/unifiedsearch/{email}",
                         headers={"User-Agent": "Python"}).status_code
        if r == 200:
            bot.reply_to(msg, f"⚠️ Email {email} найден в утечках.")
        else:
            bot.reply_to(msg, f"✅ Email {email} не найден.")
    except:
        bot.reply_to(msg, "⚠️ Ошибка при проверке email.")

@bot.message_handler(commands=["phone"])
def phone_lookup(msg):
    args = msg.text.split()
    if len(args) < 2:
        return bot.reply_to(msg, "/phone <номер>")
    phone = args[1]
    try:
        info = requests.get(f"https://htmlweb.ru/geo/api.php?json&telcod={phone}").json()
        if "country" in info:
            bot.reply_to(msg, f"📱 Телефон: {phone}\nСтрана: {info['country']['english']}\nРегион: {info['region']['name']}")
        else:
            bot.reply_to(msg, "❌ Не удалось найти данные.")
    except:
        bot.reply_to(msg, "⚠️ Ошибка при запросе.")

@bot.message_handler(commands=["history"])
def name_history(msg):
    args = msg.text.split()
    if len(args) < 2:
        return bot.reply_to(msg, "/history <user_id>")
    uid = args[1]
    cursor.execute("SELECT username, changed_at FROM history WHERE user_id = ? ORDER BY changed_at DESC", (uid,))
    rows = cursor.fetchall()
    if not rows:
        bot.reply_to(msg, "❌ История не найдена.")
    else:
        text = "\n".join([f"{row[0]} | {row[1]}" for row in rows])
        bot.reply_to(msg, f"🕓 История ников:\n{text[:4000]}")

@bot.message_handler(commands=["giveadmin"])
def give_admin(msg):
    if msg.from_user.id != ADMIN_ID:
        return bot.reply_to(msg, "❌ Только главный админ может выдавать права")
    args = msg.text.split()
    if len(args) != 2 or not args[1].isdigit():
        return bot.reply_to(msg, "⚠️ Используй: /giveadmin <user_id>")
    uid = int(args[1])
    set_status(uid, "is_admin", 1)
    bot.reply_to(msg, f"✅ Пользователю {uid} выданы права администратора.")

@bot.message_handler(commands=["admin"])
def admin(msg):
    if not is_admin(msg.from_user.id):
        return bot.reply_to(msg, "❌ Нет доступа")
    kb = types.InlineKeyboardMarkup()
    kb.add(
        types.InlineKeyboardButton("Премиум +", callback_data="give_premium"),
        types.InlineKeyboardButton("Премиум -", callback_data="remove_premium")
    )
    kb.add(
        types.InlineKeyboardButton("Бан", callback_data="ban"),
        types.InlineKeyboardButton("Разбан", callback_data="unban")
    )
    kb.add(types.InlineKeyboardButton("👥 Список", callback_data="list_users"))
    bot.send_message(msg.chat.id, "👑 Админ-панель", reply_markup=kb)

@bot.callback_query_handler(func=lambda call: True)
def actions(call):
    if not is_admin(call.from_user.id): return
    uid = call.message.chat.id
    if call.data == "give_premium":
        set_status(uid, "premium", 1)
        bot.answer_callback_query(call.id, "✅ Премиум выдан")
    elif call.data == "remove_premium":
        set_status(uid, "premium", 0)
        bot.answer_callback_query(call.id, "❌ Премиум снят")
    elif call.data == "ban":
        set_status(uid, "banned", 1)
        bot.answer_callback_query(call.id, "🚫 Забанен")
    elif call.data == "unban":
        set_status(uid, "banned", 0)
        bot.answer_callback_query(call.id, "✅ Разбанен")
    elif call.data == "list_users":
        cursor.execute("SELECT id, username, premium, banned, is_admin FROM users")
        rows = cursor.fetchall()
        msg_txt = "\n".join([
            f"{r[1]} | ID {r[0]} | "
            f"{'⭐' if r[2] else ''}{'🚫' if r[3] else ''}{'👑' if r[4] else ''}"
            for r in rows
        ])
        bot.send_message(call.message.chat.id, "👥 Пользователи:\n" + msg_txt[:4000])
print('PYTHON')
bot.polling()
