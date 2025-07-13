# https-github.com-yourusername-telegram_bot
import os
import json
import asyncio
import random

from telethon.sync import TelegramClient
from telethon.errors import SessionPasswordNeededError
from telethon.tl.functions.channels import CreateChannelRequest

from telegram import Update, ReplyKeyboardMarkup, KeyboardButton
from telegram.ext import (
    ApplicationBuilder, CommandHandler, MessageHandler,
    ContextTypes, ConversationHandler, filters
)

BOT_TOKEN = "8152518280:AAEW16Pcz-zICssTKLnjv6o6V-1OVndm_7A"
SESSIONS_DIR = "sessions"
if not os.path.exists(SESSIONS_DIR):
    os.makedirs(SESSIONS_DIR)

LANG, PHONE, API_ID, API_HASH, CODE = range(5)
SELECT_GROUP_COUNT, SELECT_DELAY, SELECT_ACCOUNTS, ACCOUNT_COUNT = range(5, 9)

user_data = {}
user_lang = {}

def get_lang(user_id):
    return user_lang.get(user_id, 'ar')

def get_main_menu(lang='ar'):
    if lang == 'en':
        keyboard = [
            [KeyboardButton("➕ Add Account"), KeyboardButton("🏗️ Create Groups")],
            [KeyboardButton("🌐 Change Language")]
        ]
    else:
        keyboard = [
            [KeyboardButton("➕ إضافة حساب"), KeyboardButton("🏗️ إنشاء مجموعات")],
            [KeyboardButton("🌐 تغيير اللغة")]
        ]
    return ReplyKeyboardMarkup(keyboard, resize_keyboard=True)

texts = {
    'ar': {
        'start': "اختر من القائمة:",
        'ask_phone': "📱 أرسل رقم الهاتف مع كود الدولة (مثال: +201234567890):",
        'ask_api_id': "🧩 أرسل الـ API ID:",
        'ask_api_hash': "🔐 أرسل الـ API HASH:",
        'code_sent': "✉️ تم إرسال الكود، من فضلك أدخله الآن:",
        'added': "✅ تم تسجيل الحساب وحفظ الجلسة بنجاح.",
        'ask_account_count': "📥 كم عدد الحسابات التي تريد إضافتها؟",
        'all_accounts_added': "✅ تم إضافة كل الحسابات.",
        'select_accounts': "👥 اختر أرقام الحسابات التي تريد استخدامها (مثال: 1 3 4):",
        '2fa_error': "🔒 الحساب عليه تحقق بخطوتين. هذا النوع غير مدعوم حالياً.",
        'error': "❌ حدث خطأ: {error}",
        'group_count': "📦 كم عدد المجموعات التي تريد إنشاءها؟",
        'delay_seconds': "⏱️ كم ثانية بين كل مجموعة؟",
        'done': "✅ تم إنشاء جميع المجموعات.",
        'language_changed': "🌐 تم تغيير اللغة إلى العربية."
    },
    'en': {
        'start': "Please choose from the menu:",
        'ask_phone': "📱 Send the phone number including country code (e.g., +201234567890):",
        'ask_api_id': "🧩 Send the API ID:",
        'ask_api_hash': "🔐 Send the API HASH:",
        'code_sent': "✉️ Code sent. Please enter it now:",
        'added': "✅ Account logged in and session saved.",
        'ask_account_count': "📥 How many accounts do you want to add?",
        'all_accounts_added': "✅ All accounts have been added.",
        'select_accounts': "👥 Select account numbers to use (e.g., 1 3 4):",
        '2fa_error': "🔒 This account uses 2FA and is not supported.",
        'error': "❌ Error: {error}",
        'group_count': "📦 How many groups do you want to create?",
        'delay_seconds': "⏱️ How many seconds between each group?",
        'done': "✅ All groups have been created.",
        'language_changed': "🌐 Language switched to English."
    }
}

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    lang = get_lang(update.effective_user.id)
    await update.message.reply_text(
        texts[lang]['start'],
        reply_markup=get_main_menu(lang)
    )

async def text_menu_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    text = update.message.text.strip()
    lang = get_lang(user_id)
    txt = texts[lang]

    if text in ["🌐 Change Language", "🌐 تغيير اللغة"]:
        user_lang[user_id] = 'en' if lang == 'ar' else 'ar'
        await update.message.reply_text(texts[user_lang[user_id]]['language_changed'], reply_markup=get_main_menu(user_lang[user_id]))
        return ConversationHandler.END

    if text in ["➕ Add Account", "➕ إضافة حساب"]:
        await update.message.reply_text(txt['ask_account_count'])
        return ACCOUNT_COUNT

    if text in ["🏗️ Create Groups", "🏗️ إنشاء مجموعات"]:
        sessions = [f.replace(".session", "") for f in os.listdir(SESSIONS_DIR) if f.endswith(".session")]
        msg = texts[lang]['select_accounts'] + "\n"
        for idx, session in enumerate(sessions):
            msg += f"{idx+1}. {session}\n"
        context.user_data['available_sessions'] = sessions
        await update.message.reply_text(msg)
        return SELECT_ACCOUNTS

    await update.message.reply_text("❓ غير مفهوم. الرجاء استخدام الأزرار.")
    return ConversationHandler.END

async def handle_account_count(update: Update, context: ContextTypes.DEFAULT_TYPE):
    count = int(update.message.text.strip())
    context.user_data['account_total'] = count
    context.user_data['account_index'] = 0
    context.user_data['accounts'] = []
    lang = get_lang(update.effective_user.id)
    await update.message.reply_text(texts[lang]['ask_phone'])
    return PHONE

async def get_phone(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['phone'] = update.message.text.strip()
    lang = get_lang(update.effective_user.id)
    await update.message.reply_text(texts[lang]['ask_api_id'])
    return API_ID

async def get_api_id(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['api_id'] = int(update.message.text.strip())
    lang = get_lang(update.effective_user.id)
    await update.message.reply_text(texts[lang]['ask_api_hash'])
    return API_HASH

async def get_api_hash(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['api_hash'] = update.message.text.strip()
    phone = context.user_data['phone']
    api_id = context.user_data['api_id']
    api_hash = context.user_data['api_hash']
    session_path = os.path.join(SESSIONS_DIR, phone.replace('+', ''))
    client = TelegramClient(session_path, api_id, api_hash)
    await client.connect()
    await client.send_code_request(phone)
    context.bot_data[update.effective_chat.id] = client
    return CODE

async def get_code(update: Update, context: ContextTypes.DEFAULT_TYPE):
    code = update.message.text.strip()
    phone = context.user_data['phone']
    api_id = context.user_data['api_id']
    api_hash = context.user_data['api_hash']
    client: TelegramClient = context.bot_data[update.effective_chat.id]
    lang = get_lang(update.effective_user.id)
    try:
        await client.sign_in(phone, code)
        json_path = os.path.join(SESSIONS_DIR, phone.replace('+', '') + ".json")
        with open(json_path, 'w') as f:
            json.dump({'api_id': api_id, 'api_hash': api_hash}, f)
        await update.message.reply_text(texts[lang]['added'])
    except SessionPasswordNeededError:
        await update.message.reply_text(texts[lang]['2fa_error'])
    except Exception as e:
        await update.message.reply_text(texts[lang]['error'].format(error=str(e)))
    finally:
        await client.disconnect()

    context.user_data['account_index'] += 1
    if context.user_data['account_index'] < context.user_data['account_total']:
        await update.message.reply_text(texts[lang]['ask_phone'])
        return PHONE
    else:
        await update.message.reply_text(texts[lang]['all_accounts_added'])
        return ConversationHandler.END

async def handle_selected_accounts(update: Update, context: ContextTypes.DEFAULT_TYPE):
    selected = list(map(int, update.message.text.strip().split()))
    selected_sessions = []
    for i in selected:
        try:
            selected_sessions.append(context.user_data['available_sessions'][i - 1])
        except IndexError:
            continue
    context.user_data['selected_sessions'] = selected_sessions
    lang = get_lang(update.effective_user.id)
    await update.message.reply_text(texts[lang]['group_count'])
    return SELECT_GROUP_COUNT

async def handle_group_count(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data['group_count'] = int(update.message.text.strip())
    lang = get_lang(update.effective_user.id)
    await update.message.reply_text(texts[lang]['delay_seconds'])
    return SELECT_DELAY

async def handle_group_delay(update: Update, context: ContextTypes.DEFAULT_TYPE):
    delay = int(update.message.text.strip())
    group_count = context.user_data['group_count']
    selected_sessions = context.user_data['selected_sessions']
    result = await create_groups(group_count, delay, selected_sessions)
    await update.message.reply_text(result)
    return ConversationHandler.END

async def create_groups(group_count, delay, sessions):
    health_tips = [
        "Don’t skip your yearly check-up 🩺📅",
        "Your lungs love fresh air – take a deep breath 🌬️🌲",
        "Healing takes time – be patient 🕒❤️‍🩹",
        "Every step counts – move more 🏃‍♂️🚶‍♀️",
        "Say goodbye to energy drinks 👋⚡ Hello sleep 😴",
        "Fruits > Fast food 🥭🍟 Make better choices",
        "Take care of your heart – it does so much for you ❤️🫀",
        "Smile more, stress less 😊🧘‍♀️",
        "Stretch daily to stay limber 🧘‍♂️🦵",
        "Don’t ignore back pain 🧍‍♂️⚠️ It speaks volumes"
    ]

    async def run_for_session(session_name):
        session_output = []
        try:
            json_path = os.path.join(SESSIONS_DIR, session_name + ".json")
            if not os.path.exists(json_path):
                session_output.append(f"[{session_name}] ❌ Missing API credentials.")
                return "\n".join(session_output)

            with open(json_path, 'r') as f:
                creds = json.load(f)

            api_id = creds['api_id']
            api_hash = creds['api_hash']
            client = TelegramClient(os.path.join(SESSIONS_DIR, session_name), api_id, api_hash)
            await client.connect()

            async with client:
                for _ in range(group_count):
                    name = random.choice([
                        "Future Doctors 🩺", "General Medicine Hub 💉", "Surgery Insights 🔪", "Professional Pharmacists 💊",
                        "Nursing and Healthcare 👩‍⚕️", "Drug Guide Central 💊", "Medical Discussions 💬", "Free Medical Courses 📚",
                        "Internship Doctors 👨‍⚕️", "Medical References 📚", "Diagnosis Handbook 🔬", "Step-by-Step Medicine 🩺",
                        "Medical Books PDF 📚", "Doctors Without Borders 🌍", "Rare Medical Cases 🧬", "Medical Question Bank 💡",
                        "Pre-Med Corner 📚", "Clinical Medicine Academy 🩺", "Simplified Medical Science 🔬", "Modern Medicine Updates 💡",
                        "Basic Medicine for Students 📚", "Clinical Pharmacists 💊", "Psychiatry Community 🧠", "Nursing Courses 👩‍⚕️",
                        "Pediatrics in Practice 👶", "Smart Nurse Network 👩‍⚕️", "Pharmacy Today 💊", "Anatomy Secrets 🧠",
                        "Healthcare Practitioners Guide 🩺", "Visual Medicine 👀", "Daily Medical Debates 💬",
                        "Study Group for Med Students 📚", "Preventive Medicine Zone 🛡️", "Women & Child Health 👩‍👧‍👦",
                        "Emergency Medical Cases 🚑", "FDA Alerts & Updates 💡", "Quick Drug Reviews 💊", "Dental Students Lounge 🦷",
                        "Medical Video Bank 🎥", "MBBS Courses Central 📚", "Alternative Medicine Encyclopedia 🌿",
                        "Psychiatry for All 🧠", "Day Surgery Room 🏥", "Hospital ER Group 🚑", "Cardiology Clinic ❤️",
                        "Family & Community Medicine 🏡", "Pathology Review Zone 🔬", "USMLE Q&A Hub 📚",
                        "Simplified Surgery Tips 🔪", "First Year Med Students 📚", "Radiology Club 🩻",
                        "Lab Test Results Explained 🧪", "Free Courses for Doctors 📚", "Complete Medical Library 📚",
                        "Internal Medicine Simplified 🩺"
                    ])
                    result = await client(CreateChannelRequest(title=name, about="Auto Created", megagroup=True))
                    group = result.chats[0]

                    await client.send_message(group.id, "Group created in 2025 ✅✅")
                    selected_tips = random.sample(health_tips, 10)
                    for tip in selected_tips:
                        await asyncio.sleep(1)
                        await client.send_message(group.id, tip)

                    session_output.append(f"[{session_name}] ✅ Created: {name}")
                    await asyncio.sleep(delay)
        except Exception as e:
            session_output.append(f"[{session_name}] ❌ Error: {e}")
        return "\n".join(session_output)

    tasks = [run_for_session(session) for session in sessions]
    results = await asyncio.gather(*tasks)
    return "\n\n".join(results)

def main():
    app = ApplicationBuilder().token(BOT_TOKEN).build()
    conv_handler = ConversationHandler(
        entry_points=[
            CommandHandler("start", start),
            MessageHandler(filters.Regex("^(➕ إضافة حساب|➕ Add Account|🏗️ إنشاء مجموعات|🏗️ Create Groups|🌐 تغيير اللغة|🌐 Change Language)$"), text_menu_handler),
        ],
        states={
            ACCOUNT_COUNT: [MessageHandler(filters.TEXT & ~filters.COMMAND, handle_account_count)],
            PHONE: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_phone)],
            API_ID: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_api_id)],
            API_HASH: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_api_hash)],
            CODE: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_code)],
            SELECT_ACCOUNTS: [MessageHandler(filters.TEXT & ~filters.COMMAND, handle_selected_accounts)],
            SELECT_GROUP_COUNT: [MessageHandler(filters.TEXT & ~filters.COMMAND, handle_group_count)],
            SELECT_DELAY: [MessageHandler(filters.TEXT & ~filters.COMMAND, handle_group_delay)],
        },
        fallbacks=[CommandHandler("start", start)],
        allow_reentry=True,
    )
    app.add_handler(conv_handler)
    app.run_polling()

if __name__ == "__main__":
    main()
