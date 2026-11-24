
# ICT AMD + FVG Telegram Analyzer Bot

بوت تليجرام لتحليل صور الشارتات والرسائل النصية وفق استراتيجية **AMD** (التجميع، التلاعب، التوزيع) و**الفجوات السعرية العادلة (FVG)** باستخدام نموذج Gemini.

## الملفات الأساسية
- `bot.py`: يستقبل الصور والنصوص من تليجرام، يرسلها للتحليل، ويحفظ ملخصاً في سجل التحليلات.
- `analyses_log.md`: سجل التحليلات السابقة مرتباً زمنياً.
- `.github/workflows/telegram_bot.yml`: ملف تشغيل البوت ومزامنة السجل على GitHub Actions.

## الإعدادات المطلوبة في GitHub Secrets
- `TELEGRAM_BOT_TOKEN`
- `GEMINI_API_KEY`
