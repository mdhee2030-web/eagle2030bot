import json
import re
from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes
import anthropic
import os

TELEGRAM_TOKEN = os.environ.get("TELEGRAM_TOKEN")
ANTHROPIC_API_KEY = os.environ.get("ANTHROPIC_API_KEY")

SYSTEM_PROMPT = """أنت مستشار مالي وخبير تحليل تقني متخصص في أسواق الأسهم.
عند تحليل أي سهم، ابحث عن السعر الحالي وقدم تحليلاً بتنسيق JSON فقط بدون أي نص خارجه:

{
  "ticker": "رمز السهم",
  "companyName": "اسم الشركة",
  "currentPrice": "السعر الحالي",
  "currency": "SAR أو USD",
  "decision": "شراء أو بيع أو انتظار",
  "confidence": 75,
  "buyRange": {"min": "السعر", "max": "السعر"},
  "stopLoss": "سعر إيقاف الخسارة",
  "targetPrice": "هدف البيع المحافظ",
  "expectedReturn": "نسبة العائد %",
  "riskLevel": "منخفض أو متوسط أو مرتفع",
  "technicalSummary": "ملخص تقني 2-3 جمل",
  "keyReason": "السبب الرئيسي",
  "warnings": ["تحذير إن وجد"]
}

مهم: أسعار الهدف محافظة، إيقاف الخسارة 5-8% تحت الشراء."""

client = anthropic.Anthropic(api_key=ANTHROPIC_API_KEY)

def analyze_stock(ticker: str) -> dict:
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1000,
        system=SYSTEM_PROMPT,
        tools=[{"type": "web_search_20250305", "name": "web_search"}],
        messages=[{
            "role": "user",
            "content": f"حلل السهم: {ticker.upper()} ابحث عن السعر الحالي واعطني التوصية."
        }]
    )
    text = "".join(b.text for b in response.content if hasattr(b, "text"))
    match = re.search(r'\{[\s\S]*\}', text)
    if not match:
        return None
    return json.loads(match.group())

def format_message(data: dict) -> str:
    d = data["decision"]
    emoji = "🟢" if d == "شراء" else "🔴" if d == "بيع" else "🟡"
    arrow = "▲" if d == "شراء" else "▼" if d == "بيع" else "◈"
    risk_emoji = "🟢" if data["riskLevel"] == "منخفض" else "🟡" if data["riskLevel"] == "متوسط" else "🔴"

    msg = f"""📊 *{data['ticker']} — {data['companyName']}*

💰 السعر الحالي: *{data['currentPrice']} {data['currency']}*

{emoji} *{arrow} القرار: {d}*
📈 الثقة: *{data['confidence']}%*
{risk_emoji} المخاطرة: *{data['riskLevel']}*

━━━━━━━━━━━━━━
🛒 *نطاق الشراء:*
`{data['buyRange']['min']} — {data['buyRange']['max']}`

🛑 *إيقاف الخسارة:* `{data['stopLoss']}`
🎯 *هدف البيع:* `{data['targetPrice']}`
💹 *العائد المتوقع:* `+{data['expectedReturn']}`

━━━━━━━━━━━━━━
📉 *التحليل التقني:*
{data['technicalSummary']}

💡 *السبب الرئيسي:*
{data['keyReason']}"""

    if data.get("warnings"):
        msg += "\n\n⚠️ *تحذيرات:*\n" + "\n".join(f"• {w}" for w in data["warnings"])

    msg += "\n\n_⚠️ للأغراض التعليمية فقط. ليس توصية استثمارية._"
    return msg

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "👋 *مرحباً بك في Eagle2030bot!*\n\n"
        "🦅 أنا مستشارك المالي الذكي\n\n"
        "📌 *كيفية الاستخدام:*\n"
        "أرسل رمز أي سهم مثل:\n"
        "`AAPL` · `TSLA` · `NVDA` · `2222`\n\n"
        "وسأعطيك فوراً:\n"
        "✅ قرار شراء أو بيع\n"
        "📊 نطاق السعر المناسب\n"
        "🛑 سعر إيقاف الخسارة\n"
        "🎯 هدف البيع المحافظ",
        parse_mode="Markdown"
    )

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    ticker = update.message.text.strip().upper()
    if not ticker:
        return

    msg = await update.message.reply_text(
        f"⏳ جارٍ تحليل *{ticker}*...\nيرجى الانتظار 🔍",
        parse_mode="Markdown"
    )

    try:
        data = analyze_stock(ticker)
        if not data:
            await msg.edit_text("❌ تعذر التحليل. تأكد من رمز السهم وحاول مجدداً.")
            return
        await msg.edit_text(format_message(data), parse_mode="Markdown")
    except Exception as e:
        await msg.edit_text("❌ حدث خطأ. حاول مجدداً بعد قليل.")

def main():
    app = Application.builder().token(TELEGRAM_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    print("✅ Eagle2030bot يعمل...")
    app.run_polling()

if __name__ == "__main__":
    main()
