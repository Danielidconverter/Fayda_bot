import os
import io
import sqlite3
import logging
from PIL import Image, ImageOps
import fitz
from telegram import (
    Update,
    ReplyKeyboardMarkup,
    InlineKeyboardButton,
    InlineKeyboardMarkup,
)
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    CallbackQueryHandler,
    ContextTypes,
    filters,
)

BOT_TOKEN = os.getenv("BOT_TOKEN", "YOUR_BOT_TOKEN_HERE")
ADMIN_ID = int(os.getenv("ADMIN_ID", "123456789"))

TELEBIRR_NUMBER = "0926617393"
TELEBIRR_NAME = "Workinesh"
COST_PER_CONVERT = 10
WELCOME_BONUS = 10

logging.basicConfig(level=logging.INFO)

def init_db():
    conn = sqlite3.connect("fayda_bot.db")
    c = conn.cursor()
    c.execute("""
        CREATE TABLE IF NOT EXISTS users (
            user_id INTEGER PRIMARY KEY,
            balance INTEGER DEFAULT 10,
            color_mode TEXT DEFAULT 'color',
            mirror_mode INTEGER DEFAULT 0,
            file_type TEXT DEFAULT 'png',
            template_id INTEGER DEFAULT 3
        )
    """)
    conn.commit()
    conn.close()

def get_user(user_id):
    conn = sqlite3.connect("fayda_bot.db")
    c = conn.cursor()
    c.execute("SELECT * FROM users WHERE user_id=?", (user_id,))
    row = c.fetchone()
    if not row:
        c.execute("INSERT INTO users (user_id, balance) VALUES (?, ?)", (user_id, WELCOME_BONUS))
        conn.commit()
        c.execute("SELECT * FROM users WHERE user_id=?", (user_id,))
        row = c.fetchone()
    conn.close()
    return {
        "user_id": row[0],
        "balance": row[1],
        "color_mode": row[2],
        "mirror_mode": row[3],
        "file_type": row[4],
        "template_id": row[5],
    }

def update_balance(user_id, amount):
    conn = sqlite3.connect("fayda_bot.db")
    c = conn.cursor()
    c.execute("UPDATE users SET balance = balance + ? WHERE user_id=?", (amount, user_id))
    conn.commit()
    conn.close()

def update_setting(user_id, column, value):
    conn = sqlite3.connect("fayda_bot.db")
    c = conn.cursor()
    c.execute(f"UPDATE users SET {column}=? WHERE user_id=?", (value, user_id))
    conn.commit()
    conn.close()

def main_menu():
    keyboard = [
        ["ጀምር", "ጨርስ"],
        ["💳 ክፈል", "💰 ቀሪ ሂሳብ"],
        ["⚙️ ማስተካከያዎች"]
    ]
    return ReplyKeyboardMarkup(keyboard, resize_keyboard=True)

def settings_menu():
    keyboard = [
        [
            InlineKeyboardButton("📋 ቴምፕሌት", callback_data="set_template"),
            InlineKeyboardButton("🪞 አቀማመጥ (Mirror)", callback_data="set_mirror"),
        ],
        [
            InlineKeyboardButton("🎨 ቀለም", callback_data="set_color"),
            InlineKeyboardButton("📄 የፋይል አይነት", callback_data="set_filetype"),
        ]
    ]
    return InlineKeyboardMarkup(keyboard)

def process_fayda_pdf(pdf_bytes, color_mode="color", mirror=False, file_type="png", template_id=3):
    doc = fitz.open(stream=pdf_bytes, filetype="pdf")
    page = doc[0]
    pix = page.get_pixmap(dpi=300)
    full_img = Image.open(io.BytesIO(pix.tobytes("png"))).convert("RGB")
    w, h = full_img.size

    front_box = (int(w * 0.08), int(h * 0.56), int(w * 0.49), int(h * 0.82))
    back_box  = (int(w * 0.51), int(h * 0.56), int(w * 0.92), int(h * 0.82))

    front_card = full_img.crop(front_box)
    back_card = full_img.crop(back_box)

    card_w, card_h = 1016, 642
    front_card = front_card.resize((card_w, card_h), Image.Resampling.LANCZOS)
    back_card = back_card.resize((card_w, card_h), Image.Resampling.LANCZOS)

    if template_id == 5:
        gap = 80
        margin = 60
    else:
        gap = 35
        margin = 40

    canvas_w = (card_w * 2) + gap + (margin * 2)
    canvas_h = card_h + (margin * 2)
    canvas = Image.new("RGB", (canvas_w, canvas_h), "white")

    canvas.paste(front_card, (margin, margin))
    canvas.paste(back_card, (margin + card_w + gap, margin))

    if color_mode == "gray":
        canvas = ImageOps.grayscale(canvas).convert("RGB")

    if mirror:
        canvas = ImageOps.mirror(canvas)

    output = io.BytesIO()
    if file_type == "pdf":
        canvas.save(output, format="PDF", resolution=300.0)
        ext = "pdf"
    else:
        canvas.save(output, format="PNG")
        ext = "png"
    output.seek(0)
    return output, ext

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = get_user(update.effective_user.id)
    text = (
        f"👋 እንኳን ወደ **Fayda ID Converter** በደህና መጡ!\n\n"
        f"🎁 ለሙከራ **{WELCOME_BONUS} ነጥብ** (1 ካርድ) በነጻ ተሰጥቶዎታል።\n"
        f"💰 ያለዎት ቀሪ ሂሳብ፦ **{user['balance']} ነጥብ**\n\n"
        f"📌 **አጠቃቀም፦**\n"
        f"1. የፋይዳ (Fayda) **PDF ፋይል** ይላኩ።\n"
        f"2. ቦቱ ወዲያውኑ ለህትመት ዝግጁ አድርጎ ይልክልዎታል።\n"
        f"3. ቀለም ወይም የፋይል አይነት ለመቀየር **⚙️ ማስተካከያዎች** ይጫኑ።"
    )
    await update.message.reply_text(text, parse_mode="Markdown", reply_markup=main_menu())

async def handle_text(update: Update, context: ContextTypes.DEFAULT_TYPE):
    msg = update.message.text
    user_id = update.effective_user.id
    user = get_user(user_id)

    if msg in ["ጀምር", "/start"]:
        await start(update, context)

    elif msg in ["ጨርስ", "/done"]:
        await update.message.reply_text("✅ ዝግጅቱ ተጠናቋል። አዲስ PDF መላክ ይችላሉ።", reply_markup=main_menu())

    elif msg == "💰 ቀሪ ሂሳብ":
        await update.message.reply_text(f"💰 ያለዎት ቀሪ ሂሳብ፦ **{user['balance']} ነጥብ** ነው።", parse_mode="Markdown")

    elif msg == "⚙️ ማስተካከያዎች":
        await update.message.reply_text("⚙️ **ቅንብሮች**\nምን መቀየር ይፈልጋሉ?", parse_mode="Markdown", reply_markup=settings_menu())

    elif msg == "💳 ክፈል":
        pay_text = (
            f"💳 **1 PDF = {COST_PER_CONVERT} ብር (10 ነጥብ)**\n\n"
            f"💰 **ወደዚህ ይክፈሉ፦**\n"
            f"📱 **ቴሌብር (Telebirr):** `{TELEBIRR_NUMBER}`\n"
            f"👤 **ስም:** {TELEBIRR_NAME}\n\n"
            f"📩 ከከፈሉ በኋላ **Transaction ID** ወይም **የደረሰኝ ስክሪንሾት** እዚህ ይላኩ።"
        )
        context.user_data["awaiting_payment"] = True
        await update.message.reply_text(pay_text, parse_mode="Markdown")

    elif context.user_data.get("awaiting_payment"):
        context.user_data["awaiting_payment"] = False
        admin_kb = InlineKeyboardMarkup([
            [
                InlineKeyboardButton("✅ +10 ነጥብ (10 ብር)", callback_data=f"approve_{user_id}_10"),
                InlineKeyboardButton("✅ +50 ነጥብ (50 ብር)", callback_data=f"approve_{user_id}_50"),
            ],
            [
                InlineKeyboardButton("✅ +100 ነጥብ (100 ብር)", callback_data=f"approve_{user_id}_100"),
                InlineKeyboardButton("❌ ውድቅ አድርግ", callback_data=f"reject_{user_id}"),
            ]
        ])
        await context.bot.send_message(
            chat_id=ADMIN_ID,
            text=f"🔔 **አዲስ የክፍያ ጥያቄ!**\n👤 ተጠቃሚ: `{user_id}` ({update.effective_user.first_name})\n📝 መልዕክት/ID: `{msg}`",
            parse_mode="Markdown",
            reply_markup=admin_kb
        )
        await update.message.reply_text("⏳ የክፍያ መረጃዎ ለአስተዳዳሪ ተልኳል። እንደተረጋገጠ ሂሳብዎ ይገባል!")

async def handle_document(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    user = get_user(user_id)
    doc = update.message.document

    if not doc.file_name.lower().endswith(".pdf"):
        await update.message.reply_text("⚠️ እባክዎ የፋይዳ **PDF ፋይል** ብቻ ይላኩ።")
        return

    if user["balance"] < COST_PER_CONVERT:
        await update.message.reply_text(
            f"❌ ቀሪ ሂሳብዎ በቂ አይደለም (ያለዎት: {user['balance']} ነጥብ)።\nእባክዎ **💳 ክፈል** የሚለውን ተጭነው ሂሳብ ይሙሉ።"
        )
        return

    wait_msg = await update.message.reply_text("⏳ ፋይሉ እየተዘጋጀ ነው፣ እባክዎ ይጠብቁ...")

    try:
        file = await doc.get_file()
        pdf_bytes = await file.download_as_bytearray()

        result_io, ext = process_fayda_pdf(
            pdf_bytes,
            color_mode=user["color_mode"],
            mirror=bool(user["mirror_mode"]),
            file_type=user["file_type"],
            template_id=user["template_id"],
        )

        update_balance(user_id, -COST_PER_CONVERT)
        new_bal = user["balance"] - COST_PER_CONVERT

        caption = f"✅ ተዘጋጅቷል!\n💰 ቀሪ ሂሳብ፦ **{new_bal} ነጥብ**"
        if ext == "pdf":
            await update.message.reply_document(document=result_io, filename=f"Fayda_ID.{ext}", caption=caption, parse_mode="Markdown")
        else:
            await update.message.reply_photo(photo=result_io, caption=caption, parse_mode="Markdown")

        await wait_msg.delete()

    except Exception as e:
        logging.error(e)
        await wait_msg.edit_text("❌ ፋይሉን ማንበብ አልተቻለም። ትክክለኛ የፋይዳ PDF መሆኑን ያረጋግጡ።")

async def handle_photo(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    photo_file_id = update.message.photo[-1].file_id
    admin_kb = InlineKeyboardMarkup([
        [
            InlineKeyboardButton("✅ +10 ነጥብ", callback_data=f"approve_{user_id}_10"),
            InlineKeyboardButton("✅ +50 ነጥብ", callback_data=f"approve_{user_id}_50"),
        ],
        [
            InlineKeyboardButton("✅ +100 ነጥብ", callback_data=f"approve_{user_id}_100"),
            InlineKeyboardButton("❌ ውድቅ አድርግ", callback_data=f"reject_{user_id}"),
        ]
    ])
    await context.bot.send_photo(
        chat_id=ADMIN_ID,
        photo=photo_file_id,
        caption=f"🔔 **የክፍያ ደረሰኝ ፎቶ!**\n👤 ተጠቃሚ: `{user_id}` ({update.effective_user.first_name})",
        parse_mode="Markdown",
        reply_markup=admin_kb
    )
    await update.message.reply_text("⏳ ደረሰኝዎ ለአስተዳዳሪ ተልኳል። እንደተረጋገጠ ሂሳብዎ ይገባል!")

async def button_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    data = query.data
    user_id = query.from_user.id

    if data == "set_color":
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("🎨 ቀለም (Color)", callback_data="save_color_color"),
             InlineKeyboardButton("⬛ ግራጫ (Grayscale)", callback_data="save_color_gray")]
        ])
        await query.edit_message_text("እባክዎ የቀለም ሁኔታ ይምረጡ፦", reply_markup=kb)

