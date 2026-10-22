
import os
import sqlite3
import logging
from datetime import datetime, timezone
from html import escape

from telegram import (
    Update,
    InlineKeyboardButton,
    InlineKeyboardMarkup,
    ReplyKeyboardMarkup,
)
from telegram.ext import (
    Application,
    CommandHandler,
    CallbackQueryHandler,
    MessageHandler,
    ContextTypes,
    filters,
)

# ========== CONFIGURATION ==========

BOT_TOKEN = os.getenv("BOT_TOKEN", "").strip()
ADMIN_IDS = {8475808767}
DB_PATH = os.getenv("DB_PATH", "market_bot.sqlite3")

logging.basicConfig(
    format="%(asctime)s %(levelname)s %(message)s",
    level=logging.INFO,
)
logging.getLogger("httpx").setLevel(logging.WARNING)

# ডেমো পণ্য: নিজের পণ্য ও মূল্য এখানে পরিবর্তন করো।
PRODUCTS = {
    "design": {
        "name": "Logo & Banner Design",
        "price": 250,
        "description": "প্রফেশনাল লোগো ও ব্যানার ডিজাইন সার্ভিস।",
    },
    "promotion": {
        "name": "Social Media Promotion",
        "price": 150,
        "description": "সোশ্যাল মিডিয়া প্রমোশন সার্ভিস।",
    },
    "custom": {
        "name": "Custom Digital Service",
        "price": 100,
        "description": "আপনার প্রয়োজন অনুযায়ী ডিজিটাল সার্ভিস।",
    },
}

MENU = ReplyKeyboardMarkup(
    [
        ["🛍️ পণ্য দেখুন", "📦 আমার অর্ডার"],
        ["💳 পেমেন্ট তথ্য", "🆘 সাপোর্ট"],
    ],
    resize_keyboard=True,
)

# ========== DATABASE ==========

def connect_db():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    return conn


def init_db():
    with connect_db() as conn:
        conn.execute("""
            CREATE TABLE IF NOT EXISTS orders (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL,
                username TEXT,
                product_key TEXT NOT NULL,
                product_name TEXT NOT NULL,
                price INTEGER NOT NULL,
                note TEXT NOT NULL,
                payment_ref TEXT DEFAULT '',
                status TEXT DEFAULT 'Pending',
                created_at TEXT NOT NULL
            )
        """)

        conn.execute("""
            CREATE TABLE IF NOT EXISTS settings (
                key TEXT PRIMARY KEY,
                value TEXT NOT NULL
            )
        """)


def get_setting(key, default=""):
    with connect_db() as conn:
        row = conn.execute(
            "SELECT value FROM settings WHERE key=?",
            (key,),
        ).fetchone()

    return row["value"] if row else default


def set_setting(key, value):
    with connect_db() as conn:
        conn.execute("""
            INSERT INTO settings(key, value)
            VALUES (?, ?)
            ON CONFLICT(key)
            DO UPDATE SET value=excluded.value
        """, (key, value))


def is_admin(user_id):
    return user_id in ADMIN_IDS


# ========== KEYBOARDS ==========

def product_keyboard():
    rows = []

    for key, product in PRODUCTS.items():
        rows.append([
            InlineKeyboardButton(
                f'{product["name"]} - ৳{product["price"]}',
                callback_data=f"product:{key}",
            )
        ])

    return InlineKeyboardMarkup(rows)


def admin_keyboard():
    return InlineKeyboardMarkup([
        [InlineKeyboardButton(
            "📦 সাম্প্রতিক অর্ডার",
            callback_data="admin:orders",
        )],
        [InlineKeyboardButton(
            "💳 bKash নম্বর সেট",
            callback_data="set:bkash",
        )],
        [InlineKeyboardButton(
            "💳 Nagad নম্বর সেট",
            callback_data="set:nagad",
        )],
        [InlineKeyboardButton(
            "🆘 সাপোর্ট সেট",
            callback_data="set:support",
        )],
    ])


def order_status_keyboard(order_id):
    return InlineKeyboardMarkup([
        [
            InlineKeyboardButton(
                "✅ Confirm",
                callback_data=f"status:{order_id}:Confirmed",
            ),
            InlineKeyboardButton(
                "🔄 Processing",
                callback_data=f"status:{order_id}:Processing",
            ),
        ],
        [
            InlineKeyboardButton(
                "❌ Cancel",
                callback_data=f"status:{order_id}:Cancelled",
            )
        ],
    ])


# ========== CUSTOMER COMMANDS ==========

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    name = escape(update.effective_user.first_name or "বন্ধু")

    await update.effective_message.reply_text(
        f"আসসালামু আলাইকুম, {name}! 👋\n\n"
        "<b>স্বাগতম Masud Digital Market-এ!</b>\n\n"
        "🛍️ পণ্য দেখুন\n"
        "📦 অর্ডার করুন\n"
        "💳 পেমেন্ট তথ্য দেখুন\n"
        "🆘 সাপোর্ট নিন\n\n"
        "নিচের মেনু ব্যবহার করুন।",
        parse_mode="HTML",
        reply_markup=MENU,
    )


async def show_products(update, context):
    await update.effective_message.reply_text(
        "🛍️ <b>আমাদের পণ্য ও সার্ভিস</b>\n"
        "পছন্দের একটি সার্ভিস নির্বাচন করুন:",
        parse_mode="HTML",
        reply_markup=product_keyboard(),
    )


async def product_selected(update, context):
    query = update.callback_query
    await query.answer()

    key = query.data.split(":", 1)[1]
    product = PRODUCTS.get(key)

    if not product:
        await query.message.reply_text("পণ্যটি পাওয়া যায়নি।")
        return

    context.user_data["product_key"] = key

    keyboard = InlineKeyboardMarkup([
        [InlineKeyboardButton(
            "🛒 অর্ডার করুন",
            callback_data=f"order:{key}",
        )]
    ])

    await query.message.reply_text(
        f"<b>{escape(product['name'])}</b>\n\n"
        f"{escape(product['description'])}\n\n"
        f"💰 মূল্য: <b>৳{product['price']}</b>",
        parse_mode="HTML",
        reply_markup=keyboard,
    )


async def order_selected(update, context):
    query = update.callback_query
    await query.answer()

    key = query.data.split(":", 1)[1]

    if key not in PRODUCTS:
        await query.message.reply_text("পণ্যটি পাওয়া যায়নি।")
        return

    context.user_data["product_key"] = key
    context.user_data["awaiting_order"] = True

    await query.message.reply_text(
        "📝 অর্ডারের জন্য প্রয়োজনীয় তথ্য লিখুন।\n\n"
        "উদাহরণ: আপনার প্রয়োজন ও অতিরিক্ত নির্দেশনা।\n"
        "ব্যক্তিগত পাসওয়ার্ড, OTP বা PIN দেবেন না।\n\n"
        "বাতিল করতে /cancel লিখুন।"
    )


async def my_orders(update, context):
    user_id = update.effective_user.id

    with connect_db() as conn:
        rows = conn.execute("""
            SELECT id, product_name, price, status
            FROM orders
            WHERE user_id=?
            ORDER BY id DESC LIMIT 10
        """, (user_id,)).fetchall()

    if not rows:
        await update.effective_message.reply_text(
            "আপনার এখনো কোনো অর্ডার নেই।",
            reply_markup=MENU,
        )
        return

    lines = ["📦 <b>আপনার অর্ডারসমূহ</b>\n"]

    for row in rows:
        lines.append(
            f"#{row['id']} | {escape(row['product_name'])}\n"
            f"মূল্য: ৳{row['price']} | "
            f"স্ট্যাটাস: {escape(row['status'])}"
        )

    await update.effective_message.reply_text(
        "\n\n".join(lines),
        parse_mode="HTML",
        reply_markup=MENU,
    )


async def payment_info(update, context):
    bkash = escape(get_setting("bkash", "সেট করা হয়নি"))
    nagad = escape(get_setting("nagad", "সেট করা হয়নি"))

    await update.effective_message.reply_text(
        "💳 <b>পেমেন্ট তথ্য</b>\n\n"
        f"bKash: <code>{bkash}</code>\n"
        f"Nagad: <code>{nagad}</code>\n\n"
        "অর্ডার করার পর অ্যাডমিনের নির্দেশনা অনুসরণ করুন।\n"
        "পেমেন্ট স্বয়ংক্রিয়ভাবে যাচাই হয় না।\n"
        "কখনো PIN বা OTP শেয়ার করবেন না।",
        parse_mode="HTML",
        reply_markup=MENU,
    )


async def support(update, context):
    contact = get_setting("support", "")

    if not contact:
        contact = "অ্যাডমিন এখনো সাপোর্ট তথ্য সেট করেননি।"

    await update.effective_message.reply_text(
        f"🆘 <b>কাস্টমার সাপোর্ট</b>\n\n{escape(contact)}",
        parse_mode="HTML",
        reply_markup=MENU,
    )


# ========== ADMIN PANEL ==========

async def admin(update, context):
    if not is_admin(update.effective_user.id):
        await update.effective_message.reply_text(
            "⛔ এই কমান্ড শুধু অ্যাডমিন ব্যবহার করতে পারবেন।"
        )
        return

    await update.effective_message.reply_text(
        "🛠️ <b>Admin Panel</b>\nএকটি অপশন বেছে নিন:",
        parse_mode="HTML",
        reply_markup=admin_keyboard(),
    )


async def admin_actions(update, context):
    query = update.callback_query
    await query.answer()

    if not is_admin(query.from_user.id):
        await query.message.reply_text("⛔ অনুমতি নেই।")
        return

    data = query.data

    if data.startswith("set:"):
        key = data.split(":", 1)[1]

        if key not in {"bkash", "nagad", "support"}:
            return

        context.user_data["setting_key"] = key
        context.user_data["awaiting_setting"] = True

        await query.message.reply_text(
            f"{key} এর নতুন তথ্য লিখে পাঠান।\n"
            "বাতিল করতে /cancel লিখুন।"
        )
        return

    if data == "admin:orders":
        with connect_db() as conn:
            rows = conn.execute("""
                SELECT id, user_id, product_name, price, status
                FROM orders
                ORDER BY id DESC LIMIT 10
            """).fetchall()

        if not rows:
            await query.message.reply_text("এখনো কোনো অর্ডার নেই।")
            return

        for row in rows:
            await query.message.reply_text(
                f"📦 অর্ডার #{row['id']}\n"
                f"User ID: {row['user_id']}\n"
                f"পণ্য: {escape(row['product_name'])}\n"
                f"মূল্য: ৳{row['price']}\n"
                f"স্ট্যাটাস: {escape(row['status'])}",
                parse_mode="HTML",
                reply_markup=order_status_keyboard(row["id"]),
            )
        return

    if data.startswith("status:"):
        _, raw_id, status = data.split(":", 2)

        if status not in {"Confirmed", "Processing", "Cancelled"}:
            return

        order_id = int(raw_id)

        with connect_db() as conn:
            row = conn.execute(
                "SELECT user_id FROM orders WHERE id=?",
                (order_id,),
            ).fetchone()

            if row:
                conn.execute(
                    "UPDATE orders SET status=? WHERE id=?",
                    (status, order_id),
                )

        if not row:
            await query.message.reply_text("অর্ডারটি পাওয়া যায়নি।")
            return

        await query.message.reply_text(
            f"অর্ডার #{order_id} এর স্ট্যাটাস: {status}"
        )

        try:
            await context.bot.send_message(
                row["user_id"],
                f"📦 আপনার অর্ডার #{order_id} এর নতুন স্ট্যাটাস: {status}",
            )
        except Exception:
            logging.exception("Customer notification failed")


# ========== TEXT HANDLER ==========

async def handle_text(update, context):
    text = (update.effective_message.text or "").strip()
    user = update.effective_user

    if context.user_data.get("awaiting_setting"):
        if not is_admin(user.id):
            return

        key = context.user_data.get("setting_key")
        if key not in {"bkash", "nagad", "support"}:
            context.user_data.clear()
            return

        if not text or len(text) > 100:
            await update.effective_message.reply_text(
                "তথ্যটি ১ থেকে ১০০ অক্ষরের মধ্যে দিন।"
            )
            return

        set_setting(key, text)
        context.user_data.pop("awaiting_setting", None)
        context.user_data.pop("setting_key", None)

        await update.effective_message.reply_text(
            "✅ তথ্য সংরক্ষণ করা হয়েছে।",
            reply_markup=MENU,
        )
        return

    if context.user_data.get("awaiting_order"):
        key = context.user_data.get("product_key")
        product = PRODUCTS.get(key)

        if not product:
            context.user_data.pop("awaiting_order", None)
            await update.effective_message.reply_text(
                "পণ্যটি পাওয়া যায়নি। আবার চেষ্টা করুন।"
            )
            return

        if len(text) < 3 or len(text) > 1000:
            await update.effective_message.reply_text(
                "অর্ডারের বিবরণ ৩ থেকে ১০০০ অক্ষরের মধ্যে লিখুন।"
            )
            return

        with connect_db() as conn:
            cur = conn.execute("""
                INSERT INTO orders (
                    user_id, username, product_key,
                    product_name, price, note, created_at
                ) VALUES (?, ?, ?, ?, ?, ?, ?)
            """, (
                user.id,
                user.username or "",
                key,
                product["name"],
                product["price"],
                text,
                datetime.now(timezone.utc).isoformat(),
            ))
            order_id = cur.lastrowid

        context.user_data.pop("awaiting_order", None)
        context.user_data.pop("product_key", None)

        await update.effective_message.reply_text(
            f"✅ অর্ডার গ্রহণ করা হয়েছে!\n\n"
            f"অর্ডার ID: #{order_id}\n"
            f"পণ্য: {product['name']}\n"
            f"মূল্য: ৳{product['price']}\n"
            "স্ট্যাটাস: Pending",
            reply_markup=MENU,
        )

        for admin_id in ADMIN_IDS:
            try:
                await context.bot.send_message(
                    admin_id,
                    f"🆕 নতুন অর্ডার #{order_id}\n"
                    f"কাস্টমার: {escape(user.full_name)}\n"
                    f"User ID: {user.id}\n"
                    f"পণ্য: {escape(product['name'])}\n"
                    f"মূল্য: ৳{product['price']}\n"
                    f"বিবরণ: {escape(text)}",
                    parse_mode="HTML",
                    reply_markup=order_status_keyboard(order_id),
                )
            except Exception:
                logging.exception("Admin notification failed")

        return

    if text == "🛍️ পণ্য দেখুন":
        await show_products(update, context)
    elif text == "📦 আমার অর্ডার":
        await my_orders(update, context)
    elif text == "💳 পেমেন্ট তথ্য":
        await payment_info(update, context)
    elif text == "🆘 সাপোর্ট":
        await support(update, context)
    else:
        await update.effective_message.reply_text(
            "মেনু থেকে একটি অপশন বেছে নিন অথবা /start লিখুন।",
            reply_markup=MENU,
        )


async def cancel(update, context):
    context.user_data.pop("awaiting_order", None)
    context.user_data.pop("product_key", None)
    context.user_data.pop("awaiting_setting", None)
    context.user_data.pop("setting_key", None)

    await update.effective_message.reply_text(
        "বর্তমান কাজ বাতিল করা হয়েছে।",
        reply_markup=MENU,
    )


# ========== START BOT ==========

def main():
    if not BOT_TOKEN:
        raise SystemExit(
            "BOT_TOKEN সেট করা নেই। Hosting Secrets-এ BOT_TOKEN যোগ করুন।"
        )

    init_db()

    app = Application.builder().token(BOT_TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("admin", admin))
    app.add_handler(CommandHandler("cancel", cancel))
    app.add_handler(CommandHandler("products", show_products))

    app.add_handler(
        CallbackQueryHandler(product_selected, pattern=r"^product:")
    )
    app.add_handler(
        CallbackQueryHandler(order_selected, pattern=r"^order:")
    )
    app.add_handler(
        CallbackQueryHandler(admin_actions, pattern=r"^(set:|admin:orders$|status:)")
    )

    app.add_handler(
        MessageHandler(filters.TEXT & ~filters.COMMAND, handle_text)
    )

    logging.info("Masud Digital Market Bot is starting.")
    app.run_polling(allowed_updates=Update.ALL_TYPES)


if __name__ == "__main__":
    main()
    
