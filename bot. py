uery.answer()
    if notimport csv
import io
import json
import logging
import os
import re
from datetime import datetime, timezone
from pathlib import Path
from typing import Any

from google import genai
from google.genai import types
from PIL import Image
from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import (
    ApplicationBuilder,
    CallbackQueryHandler,
    CommandHandler,
    ContextTypes,
    ConversationHandler,
    MessageHandler,
    filters,
)

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

DEFAULT_TELEGRAM_TOKEN = "8668651260:AAHZZIiLBTcq8quYktcR7ODgfaoEcmCuxxo"
TELEGRAM_TOKEN = os.environ.get("TELEGRAM_BOT_TOKEN") or DEFAULT_TELEGRAM_TOKEN
GEMINI_API_KEY = os.environ["GEMINI_API_KEY"]

TEXT_ANALYSIS_MODEL = "gemini-2.5-flash"
IMAGE_EDIT_MODEL = "gemini-2.5-flash-image"

DATA_DIR = Path("data")
DATA_DIR.mkdir(exist_ok=True)

CLIENTS_JSON_FILE = DATA_DIR / "clients.json"
CLIENTS_CSV_FILE = DATA_DIR / "clients_log.csv"
CLIENTS_MD_FILE = DATA_DIR / "clients_table.md"
ANALYSES_LOG_FILE = DATA_DIR / "analyses_log.md"

ASK_NAME, ASK_RESIDENCE, ASK_PHONE, ASK_EMAIL = range(4)

gemini_client = genai.Client(api_key=GEMINI_API_KEY)

SCOPE_BANNER = (
    "✨ *بوت تحليل الذهب (XAUUSD) — فريم 5 دقائق فقط*\n"
    "🆓 *الخدمة مجانية للجميع* | ☁️ *يعمل في السحابة (GitHub)*\n"
    "🎯 *الاستراتيجية:* AMD (التجميع، التلاعب، التوزيع) + الفجوة السعرية (FVG)\n"
    "⚖️ *تنويه:* التحليل الفني أداة مساعدة لتقييم الاحتمالات والمخاطر ولا يضمن النتائج."
)

GOLD_5M_SYSTEM_PROMPT = """
أنت محلل فني متخصص في مفاهيم المال الذكي (ICT) واستراتيجية AMD (التجميع، التلاعب، التوزيع)
مع الفجوات السعرية العادلة (Fair Value Gap - FVG)، ومخصص حصراً لتحليل الذهب (XAUUSD) على فريم 5 دقائق (5m).

قواعد صارمة للمصداقية والدقة:
- تعامل مع التحليل حصراً على أنه قراءة فنية للذهب (XAUUSD) على فريم 5 دقائق. إذا ذكر المستخدم أصلاً آخر أو فريم مختلف، نبّهه بوضوح أن البوت مخصص للذهب فريم 5 دقائق فقط، وحلّل وفق إطار الذهب 5m إذا أمكن.
- لا تعد بأرباح مؤكدة ولا تستخدم لغة قطعية؛ اذكر شروط صحة السيناريو وشروط إبطاله.
- كن سريعاً ومباشراً ومنظماً في النقاط التالية:
  1. **التحقق من الأصل والفريم:** تأكيد (XAUUSD - 5m) أو تنبيه المخالفة.
  2. **مراحل AMD:** التجميع (Accumulation) ، التلاعب/سحب السيولة (Manipulation) ، والتوزيع (Distribution).
  3. **منطقة FVG:** تحديد الفجوة السعرية الشرائية أو البيعية إن وُجدت بوضوح.
  4. **خطة التداول الفنية:**
     - الاتجاه / السيناريو: (شراء / بيع / انتظار)
     - نقطة الدخول المقترحة (Entry)
     - وقف الخسارة (SL)
     - الأهداف (TP1 / TP2)
  5. **تقييم الوضوح والمصداقية الفنية (منخفض / متوسط / مرتفع):** بناءً على وضوح الفجوة والسيولة في الصورة أو الوصف، مع تذكير مختصر بأن هذا تحليل فني وليس توصية استثمارية ملزمة.
"""


def load_clients() -> dict[str, dict[str, Any]]:
    if not CLIENTS_JSON_FILE.exists():
        return {}
    try:
        return json.loads(CLIENTS_JSON_FILE.read_text(encoding="utf-8"))
    except Exception:
        logger.exception("Failed reading clients.json")
        return {}


def save_clients_and_tables(clients: dict[str, dict[str, Any]]) -> None:
    CLIENTS_JSON_FILE.write_text(
        json.dumps(clients, ensure_ascii=False, indent=2),
        encoding="utf-8",
    )

    fieldnames = ["user_id", "username", "name", "residence", "phone", "email", "registered_at"]
    with CLIENTS_CSV_FILE.open("w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=fieldnames)
        writer.writeheader()
        for uid, info in clients.items():
            writer.writerow(
                {
                    "user_id": uid,
                    "username": info.get("username", ""),
                    "name": info.get("name", ""),
                    "residence": info.get("residence", ""),
                    "phone": info.get("phone", ""),
                    "email": info.get("email", ""),
                    "registered_at": info.get("registered_at", ""),
                }
            )

    lines = [
        "# جدول سجلات العملاء\n\n",
        "| # | الاسم | الإقامة | الرقم | الإيميل | المعرف | تاريخ التسجيل |\n",
        "| --- | --- | --- | --- | --- | --- | --- |\n",
    ]
    for idx, (uid, info) in enumerate(clients.items(), start=1):
        uname = f"@{info['username']}" if info.get("username") else "-"
        lines.append(
            f"| {idx} | {info.get('name', '-')} | {info.get('residence', '-')} | "
            f"{info.get('phone', '-')} | {info.get('email', '-')} | {uname} ({uid}) | "
            f"{info.get('registered_at', '-')} |\n"
        )
    CLIENTS_MD_FILE.write_text("".join(lines), encoding="utf-8")


def append_analysis_log(
    user_id: int,
    client_name: str,
    source_type: str,
    user_input: str,
    analysis_text: str,
) -> None:
    timestamp = datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M UTC")
    if not ANALYSES_LOG_FILE.exists():
        ANALYSES_LOG_FILE.write_text("# سجل تحليلات الذهب (XAUUSD - 5m)\n\n", encoding="utf-8")

    entry = (
        f"## [{timestamp}] — {source_type}\n"
        f"- **العميل:** {client_name} (`{user_id}`)\n"
        f"- **الأصل والفريم:** الذهب XAUUSD — 5m\n"
        f"- **المدخل:** {user_input}\n\n"
        f"{analysis_text.strip()}\n\n"
        f"---\n\n"
    )
    with ANALYSES_LOG_FILE.open("a", encoding="utf-8") as f:
        f.write(entry)


def main_menu_keyboard() -> InlineKeyboardMarkup:
    return InlineKeyboardMarkup(
        [
            [
                InlineKeyboardButton("📝 تسجيل / تحديث بياناتي", callback_data="start_register"),
                InlineKeyboardButton("👤 ملفي المسجل", callback_data="show_profile"),
            ],
            [
                InlineKeyboardButton("🥇 شروط التحليل (ذهب 5m)", callback_data="show_rules"),
            ],
        ]
    )


def is_valid_email(email: str) -> bool:
    return bool(re.match(r"^[^@\s]+@[^@\s]+\.[^@\s]+$", email.strip()))


def is_valid_phone(phone: str) -> bool:
    digits = re.sub(r"[^\d+]", "", phone.strip())
    return len(re.sub(r"\D", "", digits)) >= 7


def get_registered_client(user_id: int) -> dict[str, Any] | None:
    return load_clients().get(str(user_id))


async def annotate_chart_image(image_bytes: bytes, analysis_summary: str) -> bytes | None:
    """يعدّل الصورة المرسلة لإضافة منطقة FVG و Entry و SL و TP وتفاصيل التحليل والمصداقية."""
    pil_img = Image.open(io.BytesIO(image_bytes)).convert("RGB")

    edit_prompt = f"""
قم بتعديل صورة شارت التداول المرفقة بدقة دون تشويه الشموع الأصلية للشارت، وأضف عليها العناصر التحليلية التالية بوضوح وتنظيم:
1. مظللات ومربعات واضحة لمنطقة الفجوة السعرية (FVG) مع تسمية (FVG).
2. خط أو منطقة واضحة لنقطة الدخول المقترحة (Entry) مع تسمية (Entry).
3. خط أو منطقة واضحة لوقف الخسارة (SL) مع تسمية (SL).
4. خطوط أو مناطق واضحة للأهداف (TP1 و TP2) مع تسمية (TP1 / TP2).
5. بطاقة أو شريط جانبي/علوي مختصر وواضح على الصورة يوضح: الأصل (XAUUSD 5m)، مرحلة AMD، مستوى الوضوح/المصداقية الفنية، وتنويه مختصر بأن التحليل فني وليس ضماناً للنتيجة.

استند في التعديلات إلى التحليل التالي:
 target_msg:
        return ConversationHandler.END

    context.user_data["reg_draft"] = {}
    await target_msg.reply_text(
        "📝 *بدء تسجيل بيانات العميل*\n"
        "الخطوة 1 من 4: اكتب *الاسم الكامل* (أو أرسل /cancel للإلغاء):",
        parse_mode="Markdown",
    )
    return ASK_NAME


async def ask_residence(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if not update.message or not update.message.text:
        return ASK_NAME
    name = update.message.text.strip()
    if len(name) < 2:
        await update.message.reply_text("يرجى إدخال اسم صحيح لا يقل عن حرفين.")
        return ASK_NAME

    context.user_data.setdefault("reg_draft", {})["name"] = name
    await update.message.reply_text("الخطوة 2 من 4: اكتب *مكان الإقامة* (المدينة / الدولة):", parse_mode="Markdown")
    return ASK_RESIDENCE


async def ask_phone(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if not update.message or not update.message.text:
        return ASK_RESIDENCE
    residence = update.message.text.strip()
    if len(residence) < 2:
        await update.message.reply_text("يرجى إدخال مكان إقامة صحيح.")
        return ASK_RESIDENCE

    context.user_data.setdefault("reg_draft", {})["residence"] = residence
    await update.message.reply_text("الخطوة 3 من 4: اكتب *رقم الهاتف* (مع رمز الدولة إن أمكن):", parse_mode="Markdown")
    return ASK_PHONE


async def ask_email(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if not update.message or not update.message.text:
        return ASK_PHONE
    phone = update.message.text.strip()
    if not is_valid_phone(phone):
        await update.message.reply_text("رقم الهاتف غير واضح. يرجى إدخال رقم صحيح يحتوي
