import telebot
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton
import sqlite3
import datetime
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import threading
import time

TOKEN = "8475108316:AAG-qnC7Z7pHboIyd67FP44wNSx9ZdHMgNo"
ADMIN_ID = 1710492499
DEV_USERNAME = "@V0XYT"

bot = telebot.TeleBot(TOKEN, parse_mode='HTML')

def init_db():
    conn = sqlite3.connect('bot_database.db', check_same_thread=False)
    cursor = conn.cursor()
    cursor.execute("""CREATE TABLE IF NOT EXISTS users (
                        user_id INTEGER PRIMARY KEY,
                        messages_left INTEGER DEFAULT 20,
                        expiry_date TEXT,
                        is_unlimited INTEGER DEFAULT 0
                    )""")
    cursor.execute("""CREATE TABLE IF NOT EXISTS settings (
                        key TEXT PRIMARY KEY,
                        value TEXT
                    )""")
    cursor.execute("""CREATE TABLE IF NOT EXISTS accounts (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        user_id INTEGER,
                        email TEXT,
                        password TEXT
                    )""")
    cursor.execute("""CREATE TABLE IF NOT EXISTS channels (
                        channel_id TEXT PRIMARY KEY
                    )""")
    conn.commit()
    cursor.execute("INSERT OR IGNORE INTO settings (key, value) VALUES ('free_msgs', '20')")
    cursor.execute("INSERT OR IGNORE INTO settings (key, value) VALUES ('welcome_photo', 'https://telegra.ph/file/default.jpg')")
    conn.commit()
    conn.close()

init_db()

def get_setting(key):
    conn = sqlite3.connect('bot_database.db', check_same_thread=False)
    cursor = conn.cursor()
    cursor.execute("SELECT value FROM settings WHERE key=?", (key,))
    row = cursor.fetchone()
    conn.close()
    return row[0] if row else None

def set_setting(key, value):
    conn = sqlite3.connect('bot_database.db', check_same_thread=False)
    cursor = conn.cursor()
    cursor.execute("INSERT OR REPLACE INTO settings (key, value) VALUES (?, ?)", (key, value))
    conn.commit()
    conn.close()

def check_user_access(user_id):
    if user_id == ADMIN_ID:
        return True, "مسؤول"
    
    conn = sqlite3.connect('bot_database.db', check_same_thread=False)
    cursor = conn.cursor()
    cursor.execute("SELECT messages_left, expiry_date, is_unlimited FROM users WHERE user_id=?", (user_id,))
    row = cursor.fetchone()
    
    if not row:
        free_msgs = int(get_setting('free_msgs'))
        cursor.execute("INSERT INTO users (user_id, messages_left) VALUES (?, ?)", (user_id, free_msgs))
        conn.commit()
        conn.close()
        return True, "مستخدم جديد"
    
    msgs, expiry, unlimited = row
    if unlimited == 1:
        conn.close()
        return True, "اشتراك دائم"
    
    if expiry:
        try:
            exp_date = datetime.datetime.strptime(expiry, "%Y-%m-%d")
            if datetime.datetime.now() <= exp_date:
                conn.close()
                return True, "اشتراك ساري"
        except:
            pass
            
    if msgs > 0:
        conn.close()
        return True, "رسائل مجانية متبقية"
        
    conn.close()
    return False, "انتهى الرصيد"

def deduct_message(user_id):
    if user_id == ADMIN_ID:
        return
    conn = sqlite3.connect('bot_database.db', check_same_thread=False)
    cursor = conn.cursor()
    cursor.execute("SELECT messages_left, is_unlimited FROM users WHERE user_id=?", (user_id,))
    row = cursor.fetchone()
    if row and row[1] == 0:
        cursor.execute("UPDATE users SET messages_left = messages_left - 1 WHERE user_id=? AND messages_left > 0", (user_id,))
        conn.commit()
    conn.close()

@bot.message_handler(commands=['start'])
def start_command(message):
    user_id = message.from_user.id
    send_main_menu(message.chat.id, user_id)

def send_main_menu(chat_id, user_id):
    photo = get_setting('welcome_photo')
    caption = f"مرحبا بك في بوت الشد الخارجي\n\nالبوت ملك لـ <a href='https://t.me/{DEV_USERNAME.lstrip("@")}'>𓏺$ ⤹𓏺  𝗨𝗡!𝗖𝗟 𝗔𝗕𝗬𝗦𝗦 . 𓁹 𓏺⤸</a>"
    
    markup = InlineKeyboardMarkup(row_width=2)
    markup.add(InlineKeyboardButton("➕ إضافة حساب", callback_data="add_account"),
               InlineKeyboardButton("📋 حساباتي", callback_data="my_accounts"))
    markup.add(InlineKeyboardButton("📧 إرسال إيميل", callback_data="send_email"),
               InlineKeyboardButton("📝 نموذج", callback_data="template"))
    markup.add(InlineKeyboardButton("🗑️ حذف حساب", callback_data="delete_account"),
               InlineKeyboardButton("📨 سجل الإرسال", callback_data="send_log"))
    markup.add(InlineKeyboardButton("📊 إحصائيات", callback_data="statistics"),
               InlineKeyboardButton("ℹ️ مساعدة", callback_data="help"))
    markup.add(InlineKeyboardButton("🌐 تغيير اللغة", callback_data="change_lang"),
               InlineKeyboardButton("👨‍💻 المطور", callback_data="developer"))
    markup.add(InlineKeyboardButton("📺 الشرح", callback_data="guide"))
    
    if user_id == ADMIN_ID:
        markup.add(InlineKeyboardButton("👑 لوحة تحكم المطور", callback_data="admin_panel"))

    try:
        bot.send_photo(chat_id, photo, caption=caption, reply_markup=markup)
    except:
        bot.send_message(chat_id, caption, reply_markup=markup)

@bot.callback_query_handler(func=lambda call: True)
def callback_handler(call):
    user_id = call.from_user.id
    
    if call.data == "developer":
        bot.answer_callback_query(call.id)
        bot.send_message(call.message.chat.id, f"للتواصل مع المطور وطلب اشتراك أو شحن رصيد:\n{DEV_USERNAME}")
        
    elif call.data == "admin_panel" and user_id == ADMIN_ID:
        bot.answer_callback_query(call.id)
        markup = InlineKeyboardMarkup(row_width=2)
        markup.add(InlineKeyboardButton("➕ إضافة اشتراك لمستخدم", callback_data="admin_add_sub"),
                   InlineKeyboardButton("⚙️ تعديل رسائل الجدد", callback_data="admin_set_msgs"))
        markup.add(InlineKeyboardButton("🖼️ تغيير صورة الترحيب", callback_data="admin_set_photo"),
                   InlineKeyboardButton("🔙 عودة", callback_data="back_home"))
        bot.edit_message_caption(caption="👑 أهلاً بك في لوحة تحكم المطور السحري:", chat_id=call.message.chat.id, message_id=call.message.message_id, reply_markup=markup)

    elif call.data == "back_home":
        try:
            bot.delete_message(call.message.chat.id, call.message.message_id)
        except:
            pass
        send_main_menu(call.message.chat.id, user_id)

    elif call.data == "admin_set_msgs" and user_id == ADMIN_ID:
        msg = bot.send_message(call.message.chat.id, "أرسل عدد الرسائل المجانية الجديدة للمستخدمين الجدد (رقم فقط):")
        bot.register_next_step_handler(msg, save_new_free_msgs)

    elif call.data == "admin_set_photo" and user_id == ADMIN_ID:
        msg = bot.send_message(call.message.chat.id, "أرسل رابط الصورة الجديدة (صورة رابط تليجراف أو رابط مباشر):")
        bot.register_next_step_handler(msg, save_new_photo)

    elif call.data == "admin_add_sub" and user_id == ADMIN_ID:
        msg = bot.send_message(call.message.chat.id, "أرسل بيانات المستخدم بالصيغة التالية:\n`[ID] [عدد الأيام]`\nمثال: `123456789 30` (ولاجعلها دائمة اكتب `unlimited`)")
        bot.register_next_step_handler(msg, process_add_subscription)

    elif call.data == "add_account":
        bot.answer_callback_query(call.id)
        msg = bot.send_message(call.message.chat.id, "أرسل جيميل الحساب الخاص بك:")
        bot.register_next_step_handler(msg, step_save_email)

    elif call.data == "my_accounts":
        bot.answer_callback_query(call.id)
        conn = sqlite3.connect('bot_database.db', check_same_thread=False)
        cursor = conn.cursor()
        cursor.execute("SELECT email FROM accounts WHERE user_id=?", (user_id,))
        accs = cursor.fetchall()
        conn.close()
        if accs:
            text = "📋 حساباتك المضافة:\n" + "\n".join([f"- {acc[0]}" for acc in accs])
            bot.send_message(call.message.chat.id, text)
        else:
            bot.send_message(call.message.chat.id, "❌ ليس لديك أي حسابات مضافة.")

    elif call.data == "send_email":
        allowed, reason = check_user_access(user_id)
        if not allowed:
            bot.answer_callback_query(call.id, f"❌ انتهى رصيدك المجاني. تواصل مع المطور لشحن الحساب: {DEV_USERNAME}", show_alert=True)
            return
        bot.answer_callback_query(call.id)
        msg = bot.send_message(call.message.chat.id, "أدخل البريد المستهدف للإرسال إليه (مثال: abuse@telegram.org):")
        bot.register_next_step_handler(msg, step_target_email)

def save_new_free_msgs(message):
    try:
        val = int(message.text.strip())
        set_setting('free_msgs', str(val))
        bot.send_message(message.chat.id, f"✅ تم تحديث عدد الرسائل المجانية للجدد إلى: {val}")
    except:
        bot.send_message(message.chat.id, "❌ قيمة غير صحيحة، يرجى إرسال رقم.")

def save_new_photo(message):
    url = message.text.strip()
    set_setting('welcome_photo', url)
    bot.send_message(message.chat.id, "✅ تم تحديث صورة الترحيب بنجاح.")

def process_add_subscription(message):
    try:
        parts = message.text.strip().split()
        target_id = int(parts[0])
        duration = parts[1]
        
        conn = sqlite3.connect('bot_database.db', check_same_thread=False)
        cursor = conn.cursor()
        
        if duration.lower() == 'unlimited':
            cursor.execute("UPDATE users SET is_unlimited=1 WHERE user_id=?", (target_id,))
            bot.send_message(message.chat.id, f"✅ تم إعطاء اشتراك دائم للمستخدم: {target_id}")
        else:
            days = int(duration)
            expiry = (datetime.datetime.now() + datetime.timedelta(days=days)).strftime("%Y-%m-%d")
            cursor.execute("UPDATE users SET expiry_date=?, is_unlimited=0 WHERE user_id=?", (expiry, target_id))
            bot.send_message(message.chat.id, f"✅ تم منح اشتراك للمستخدم {target_id} لمدة {days} يوم (ينتهي في {expiry})")
        conn.commit()
        conn.close()
    except Exception as e:
        bot.send_message(message.chat.id, "❌ حدث خطأ في الصياغة. تأكد من إرسال (ID والمدة).")

def step_save_email(message):
    email = message.text.strip()
    msg = bot.send_message(message.chat.id, "أرسل باسورد التطبيق (App Password) الخاص بجوجل:")
    bot.register_next_step_handler(msg, step_save_password, email)

def step_save_password(message, email):
    password = message.text.strip()
    user_id = message.from_user.id
    
    conn = sqlite3.connect('bot_database.db', check_same_thread=False)
    cursor = conn.cursor()
    cursor.execute("INSERT INTO accounts (user_id, email, password) VALUES (?, ?, ?)", (user_id, email, password))
    conn.commit()
    conn.close()
    
    bot.send_message(message.chat.id, "✅ تم إضافة الحساب بنجاح!")

def step_target_email(message):
    target = message.text.strip()
    msg = bot.send_message(message.chat.id, "أدخل موضوع الرسالة (Subject):")
    bot.register_next_step_handler(msg, step_subject, target)

def step_subject(message, target):
    subject = message.text.strip()
    msg = bot.send_message(message.chat.id, "أدخل محتوى الرسالة أو الكليشة (النص/اللينكات):")
    bot.register_next_step_handler(msg, step_body, target, subject)

def step_body(message, target, subject):
    body = message.text.strip()
    msg = bot.send_message(message.chat.id, "أدخل عدد الرسائل المطلوب إرسالها (مثال: 10 أو 50):")
    bot.register_next_step_handler(msg, step_count, target, subject, body)

def step_count(message, target, subject, body):
    try:
        count = int(message.text.strip())
        user_id = message.from_user.id
        
        conn = sqlite3.connect('bot_database.db', check_same_thread=False)
        cursor = conn.cursor()
        cursor.execute("SELECT email, password FROM accounts WHERE user_id=?", (user_id,))
        accs = cursor.fetchall()
        conn.close()
        
        if not accs:
            bot.send_message(message.chat.id, "❌ ليس لديك أي حسابات مسجلة في البوت! قم بإضافة حساب أولاً.")
            return

        bot.send_message(message.chat.id, f"🚀 جاري بدء إرسال {count} رسالة عبر الحسابات المضافة...")
        threading.Thread(target=send_emails_thread, args=(message.chat.id, user_id, target, subject, body, count, accs)).start()
        
    except Exception as e:
        bot.send_message(message.chat.id, "❌ خطأ في إدخال العدد، يرجى المحاولة مرة أخرى.")

def send_emails_thread(chat_id, user_id, target, subject, body, count, accs):
    success = 0
    for i in range(count):
        acc = accs[i % len(accs)]
        email_user = acc[0]
        email_pass = acc[1]
        
        try:
            msg = MIMEMultipart()
            msg['From'] = email_user
            msg['To'] = target
            msg['Subject'] = subject
            msg.attach(MIMEText(body, 'plain'))
            
            server = smtplib.SMTP('smtp.gmail.com', 587)
            server.starttls()
            server.login(email_user, email_pass)
            server.sendmail(email_user, target, msg.as_string())
            server.quit()
            
            success += 1
            deduct_message(user_id)
            time.sleep(1)
        except Exception as e:
            pass
            
    bot.send_message(chat_id, f"✅ انتهت العملية بنجاح!\nتم إرسال {success} رسالة من إجمالي {count} رسالة.")

print("Bot is running...")
bot.infinity_polling()

