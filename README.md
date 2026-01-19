# ================== IMPORTS ==================
import os, time, re, sqlite3
from telegram import Update, ChatPermissions, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    ApplicationBuilder, CommandHandler, MessageHandler, CallbackQueryHandler,
    ContextTypes, filters
)
import openai
from flask import Flask, render_template_string

# ================== CONFIG ==================
TOKEN = os.getenv("BOT_TOKEN") or "YOUR_BOT_TOKEN"
OPENAI_KEY = os.getenv("OPENAI_KEY") or "YOUR_OPENAI_KEY"
OWNER_ID = 123456789  # Replace with your Telegram ID

openai.api_key = OPENAI_KEY

# ================== DATABASE ==================
db = sqlite3.connect("bot.db", check_same_thread=False)
cur = db.cursor()
cur.execute("""
CREATE TABLE IF NOT EXISTS users(
    user_id INTEGER PRIMARY KEY,
    warns INTEGER DEFAULT 0
)
""")
db.commit()

# ================== SETTINGS ==================
BAD_WORDS = ["spam", "scam", "fake"]
LINK_REGEX = r"(https?://|www\.)"
FLOOD_LIMIT = 5
FLOOD_TIME = 5
flood = {}

# ================== TELEGRAM BOT FUNCTIONS ==================

# Start command
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "👑 Ultimate Boss Bot Online!\nAll features are free for everyone!\nAI + Admin + Multi-Niche active."
    )

# Welcome & CAPTCHA
async def welcome(update: Update, context: ContextTypes.DEFAULT_TYPE):
    for user in update.message.new_chat_members:
        btn = [[InlineKeyboardButton("✅ Verify", callback_data=f"ok_{user.id}")]]
        await context.bot.restrict_chat_member(
            update.effective_chat.id,
            user.id,
            ChatPermissions(can_send_messages=False)
        )
        context.job_queue.run_once(
            captcha_timeout, 60, data=(user.id, update.effective_chat.id)
        )
        await update.message.reply_text(
            f"👋 {user.first_name}, click verify in 60s",
            reply_markup=InlineKeyboardMarkup(btn)
        )

async def captcha_timeout(context):
    uid, chat = context.job.data
    try: await context.bot.ban_chat_member(chat, uid)
    except: pass

async def captcha_check(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    uid = int(q.data.split("_")[1])
    if q.from_user.id == uid:
        await context.bot.restrict_chat_member(
            q.message.chat.id,
            uid,
            ChatPermissions(can_send_messages=True)
        )
        await q.message.edit_text("✅ Verified successfully")

# ================== ADMIN COMMANDS ==================
async def warn(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message.reply_to_message:
        uid = update.message.reply_to_message.from_user.id
        cur.execute("INSERT OR IGNORE INTO users(user_id) VALUES(?)", (uid,))
        cur.execute("UPDATE users SET warns = warns + 1 WHERE user_id=?", (uid,))
        db.commit()
        cur.execute("SELECT warns FROM users WHERE user_id=?", (uid,))
        warns = cur.fetchone()[0]
        if warns >= 3:
            await context.bot.ban_chat_member(update.effective_chat.id, uid)
            await update.message.reply_text("🚫 User banned (3 warnings)")
        else:
            await update.message.reply_text(f"⚠ Warning {warns}/3")

async def mute(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message.reply_to_message:
        uid = update.message.reply_to_message.from_user.id
        await context.bot.restrict_chat_member(
            update.effective_chat.id, uid,
            ChatPermissions(can_send_messages=False)
        )
        await update.message.reply_text("🔇 User muted")

async def unmute(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message.reply_to_message:
        uid = update.message.reply_to_message.from_user.id
        await context.bot.restrict_chat_member(
            update.effective_chat.id, uid,
            ChatPermissions(can_send_messages=True)
        )
        await update.message.reply_text("🔊 User unmuted")

async def ban(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message.reply_to_message:
        uid = update.message.reply_to_message.from_user.id
        await context.bot.ban_chat_member(update.effective_chat.id, uid)
        await update.message.reply_text("🚫 User banned")

# ================== ANTI-SPAM / FILTERS ==================
async def anti_flood(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.message.from_user.id
    now = time.time()
    flood.setdefault(uid, []).append(now)
    flood[uid] = [t for t in flood[uid] if now - t < FLOOD_TIME]
    if len(flood[uid]) > FLOOD_LIMIT:
        await context.bot.restrict_chat_member(
            update.effective_chat.id, uid,
            ChatPermissions(can_send_messages=False)
        )
        await update.message.reply_text("🚨 Flood detected – muted")

async def filters_all(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = update.message.text.lower()
    if re.search(LINK_REGEX, text): await update.message.delete()
    if any(word in text for word in BAD_WORDS): await update.message.delete()

# ================== AI REPLIES ==================
async def ai_reply(update: Update, context: ContextTypes.DEFAULT_TYPE):
    # All users have access
    try:
        user_text = update.message.text
        response = openai.ChatCompletion.create(
            model="gpt-3.5-turbo",
            messages=[
                {"role":"system","content":"You are a helpful Telegram bot assistant."},
                {"role":"user","content":user_text}
            ]
        )
        reply = response["choices"][0]["message"]["content"]
        await update.message.reply_text(reply)
    except:
        await update.message.reply_text("⚠ AI Error, try again later.")

# ================== TELEGRAM BOT MAIN ==================
def start_bot():
    app = ApplicationBuilder().token(TOKEN).build()

    # Commands
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("warn", warn))
    app.add_handler(CommandHandler("mute", mute))
    app.add_handler(CommandHandler("unmute", unmute))
    app.add_handler(CommandHandler("ban", ban))

    # Handlers
    app.add_handler(CallbackQueryHandler(captcha_check))
    app.add_handler(MessageHandler(filters.StatusUpdate.NEW_CHAT_MEMBERS, welcome))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, anti_flood))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, filters_all))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, ai_reply))

    print("👑 ALL-IN-ONE BOSS BOT RUNNING (FREE FOR ALL USERS)")
    app.run_polling()

# ================== DASHBOARD (FLASK) ==================
dashboard = Flask(__name__)
DB_PATH = "bot.db"
HTML_TEMPLATE = """
<!doctype html>
<title>Boss Bot Dashboard</title>
<h1>👑 Ultimate Bot Dashboard</h1>
<table border="1" cellpadding="5">
<tr><th>User ID</th><th>Warnings</th></tr>
{% for user in users %}
<tr>
<td>{{ user[0] }}</td>
<td>{{ user[1] }}</td>
</tr>
{% endfor %}
</table>
"""

@dashboard.route("/", methods=["GET"])
def home():
    db = sqlite3.connect(DB_PATH)
    cur = db.cursor()
    cur.execute("SELECT user_id, warns FROM users")
    users = cur.fetchall()
    db.close()
    return render_template_string(HTML_TEMPLATE, users=users)

# ================== MAIN ==================
if __name__ == "__main__":
    from threading import Thread
    # Run bot in separate thread
    Thread(target=start_bot).start()
    # Run Flask dashboard
    dashboard.run(host="0.0.0.0", port=8080)# ==========================================================
# VeloraXF LEVEL 10 – ULTIMATE ONE-FILE AI BUSINESS BOT
# ==========================================================

import os, time, threading, logging, sqlite3
from datetime import datetime, timedelta
from telegram import Update, ChatPermissions
from telegram.ext import Updater, CommandHandler, MessageHandler, Filters, CallbackContext
import openai, pyttsx3
from langdetect import detect

# ================= CONFIG =================
BOT_TOKEN = os.getenv("BOT_TOKEN") or input("Enter BOT_TOKEN: ")
OPENAI_KEY = os.getenv("OPENAI_KEY") or input("Enter OPENAI_KEY: ")
OWNER_ID = int(os.getenv("OWNER_ID") or input("Enter OWNER_ID: "))
CHANNEL_ID = os.getenv("CHANNEL_ID") or input("Enter CHANNEL_ID or username: ")

openai.api_key = OPENAI_KEY
BOT_NAME = "VeloraXF Assistant"
BUSINESS = "Online Marketing & Digital Opportunities"
FOLLOW_UP_MIN = 60
VOICE_TRIGGERS = ["urgent","problem","confused","help","question","vip"]
DASHBOARD_REFRESH = 5

logging.basicConfig(level=logging.INFO)

# ================= DATABASE =================
db = sqlite3.connect("veloraxf_level10_ultimate.db", check_same_thread=False)
c = db.cursor()

c.execute("""CREATE TABLE IF NOT EXISTS users(
    user_id INTEGER PRIMARY KEY,
    username TEXT,
    language TEXT,
    category TEXT,
    last_seen TEXT
)""")
c.execute("""CREATE TABLE IF NOT EXISTS memory(
    user_id INTEGER,
    key TEXT,
    value TEXT
)""")
c.execute("""CREATE TABLE IF NOT EXISTS leads(
    user_id INTEGER,
    stage TEXT,
    last_contact TEXT,
    vip INTEGER DEFAULT 0
)""")
db.commit()

# ================= VOICE =================
engine = pyttsx3.init()
for v in engine.getProperty("voices"):
    if "female" in v.name.lower():
        engine.setProperty("voice", v.id)
        break
def speak(text):
    engine.say(text)
    engine.runAndWait()

# ================= AI RESPONSE =================
def ai_response(user_text, stage, lang):
    prompt = f"""
You are {BOT_NAME}, AI business representative of VeloraXF.
Business: {BUSINESS}
User stage: {stage}
Tone: Friendly for chatting, Professional for business, Strict when necessary
Goals: Educate, qualify lead, promote VeloraXF ethically
Never give false promises about income
Reply in user's language
"""
    res = openai.ChatCompletion.create(
        model="gpt-3.5-turbo",
        messages=[{"role":"system","content":prompt},{"role":"user","content":user_text}],
        max_tokens=300
    )
    return res.choices[0].message["content"]

# ================= USER CLASSIFICATION =================
def classify_user(text):
    text = text.lower()
    if any(x in text for x in ["partner","invest","serious","vip"]):
        return "VIP"
    if any(x in text for x in ["price","join","how","interested"]):
        return "INTERESTED"
    if any(x in text for x in ["just looking","maybe","curious"]):
        return "CASUAL"
    return "CASUAL"

# ================= OWNER ALERT =================
def notify_owner(bot, user, reason):
    msg = f"🚨 *VeloraXF Alert*\nUser: @{user.username}\nReason: {reason}"
    bot.send_message(OWNER_ID, msg, parse_mode="Markdown")

# ================= FOLLOW-UP LOOP =================
def follow_up_loop(bot):
    while True:
        time.sleep(900)
        c.execute("SELECT user_id FROM leads WHERE stage!='CLOSED'")
        for (uid,) in c.fetchall():
            bot.send_message(uid,
                "👋 VeloraXF checking in — would you like to continue where we left off?")
        db.commit()

# ================= HANDLERS =================
def start(update: Update, context: CallbackContext):
    u = update.effective_user
    c.execute("INSERT OR IGNORE INTO users VALUES(?,?,?,?,?)",
              (u.id, u.username, "en", "NEW", datetime.utcnow().isoformat()))
    db.commit()
    update.message.reply_text(
        f"Hello 👋 I'm {BOT_NAME}, your AI assistant for VeloraXF.\nHow can I help you today?"
    )

def handle_message(update: Update, context: CallbackContext):
    u = update.effective_user
    text = update.message.text
    try:
        lang = detect(text)
    except:
        lang = "en"
    category = classify_user(text)
    stage = "QUALIFY" if category != "CASUAL" else "CHAT"
    c.execute("INSERT OR REPLACE INTO leads VALUES(?,?,?,?)",
              (u.id, stage, datetime.utcnow().isoformat(), 1 if category=="VIP" else 0))
    db.commit()
    reply = ai_response(text, stage, lang)
    update.message.reply_text(reply)
    if category == "VIP":
        notify_owner(context.bot, u, "VIP Lead Detected")
    if any(w in text.lower() for w in VOICE_TRIGGERS):
        threading.Thread(target=speak, args=(reply,), daemon=True).start()

def warn(update: Update, context: CallbackContext):
    if update.message.reply_to_message:
        uid = update.message.reply_to_message.from_user.id
        context.bot.send_message(uid, "⚠️ Warning issued by VeloraXF moderation.")

def mute(update: Update, context: CallbackContext):
    if update.message.reply_to_message:
        minutes = int(context.args[0]) if context.args else 5
        until = datetime.utcnow() + timedelta(minutes=minutes)
        context.bot.restrict_chat_member(
            update.effective_chat.id,
            update.message.reply_to_message.from_user.id,
            ChatPermissions(can_send_messages=False),
            until_date=until
        )

# ================= DASHBOARD =================
def clear_screen():
    os.system("cls" if os.name=="nt" else "clear")
def dashboard_loop():
    while True:
        clear_screen()
        c.execute("SELECT COUNT(*), category FROM users GROUP BY category")
        users = c.fetchall()
        c.execute("SELECT COUNT(*), stage, vip FROM leads GROUP BY stage, vip")
        leads = c.fetchall()
        c.execute("SELECT user_id, last_contact FROM leads WHERE stage != 'CLOSED'")
        followups = c.fetchall()
        c.execute("SELECT user_id FROM leads WHERE vip = 1")
        vip_count = len(c.fetchall())

        print("=== VELOARXF LIVE DASHBOARD ===\n")
        print("Users Summary:")
        for count, category in users:
            print(f" - {category}: {count} users")
        print("\nLeads Summary:")
        for count, stage, vip in leads:
            label = "VIP" if vip else "Normal"
            print(f" - {stage} ({label}): {count} leads")
        print(f"\nPending Follow-Ups: {len(followups)}")
        for uid, last in followups:
            last_time = datetime.fromisoformat(last).strftime("%Y-%m-%d %H:%M")
            print(f" - User ID {uid}, last contacted: {last_time}")
        print(f"\nVIPs currently active: {vip_count}")
        print("\n============================")
        time.sleep(DASHBOARD_REFRESH)

# ================= QUICK COMMANDS =================
def show_stats():
    print("=== VELOARXF QUICK STATS ===\n")
    c.execute("SELECT COUNT(*), category FROM users GROUP BY category")
    users = c.fetchall()
    print("Users Summary:")
    for count, category in users:
        print(f" - {category}: {count} users")
    c.execute("SELECT COUNT(*), stage, vip FROM leads GROUP BY stage, vip")
    leads = c.fetchall()
    print("\nLeads Summary:")
    for count, stage, vip in leads:
        label = "VIP" if vip else "Normal"
        print(f" - {stage} ({label}): {count} leads")
    c.execute("SELECT user_id, last_contact FROM leads WHERE stage != 'CLOSED'")
    followups = c.fetchall()
    print(f"\nPending Follow-Ups: {len(followups)}")
    c.execute("SELECT user_id FROM leads WHERE vip = 1")
    vip_count = len(c.fetchall())
    print(f"\nVIPs currently active: {vip_count}")
    print("\n============================\n")

def alert_vip(user_id, message):
    c.execute("UPDATE leads SET vip=1 WHERE user_id=?", (user_id,))
    db.commit()
    print(f"🚨 VIP ALERT: User ID {user_id}\nMessage: {message}\n")

def send_message(user_id, message):
    print(f"📨 Message to User ID {user_id}:\n{message}\n")

# ================= MAIN =================
def main():
    while True:
        try:
            updater = Updater(BOT_TOKEN, use_context=True)
            dp = updater.dispatcher
            dp.add_handler(CommandHandler("start", start))
            dp.add_handler(CommandHandler("warn", warn))
            dp.add_handler(CommandHandler("mute", mute))
            dp.add_handler(MessageHandler(Filters.text & ~Filters.command, handle_message))
            threading.Thread(target=follow_up_loop, args=(updater.bot,), daemon=True).start()
            threading.Thread(target=dashboard_loop, daemon=True).start()
            print("🚀 VeloraXF Level 10 Ultimate Bot is running.")
            updater.start_polling()
            updater.idle()
        except Exception as e:
            print(f"⚠️ Bot crashed! Restarting in 5s... Error: {e}")
            time.sleep(5)

if __name__ == "__main__":
    main()
