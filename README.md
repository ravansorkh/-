import asyncio
from telegram import Update, InputFile
from telegram.ext import Application, CommandHandler, MessageHandler, ConversationHandler, ContextTypes, filters
import os

# شناسه عددی ادمین
ADMIN_ID = 237056482

# لیست سوالات
questions = [
    "1. نام و نام خانوادگی شما چیست؟",
    "2. نام پدر شما چیست؟",
    "3. تاریخ تولد شما چیست؟ (روز/ماه/سال)",
    "4. شماره ملی شما چیست؟",
    "5. آدرس محل سکونت شما چیست؟",
    "6. میزان تحصیلات شما چیست؟",
    "7. شماره تماس شما چیست؟",
    "8. آیدی شبکه‌های اجتماعی خود را بنویسید (در صورت نداشتن 'ندارم' را ارسال کنید)",
    "9. از چه راهی با بزرگمهر آشنا شده‌اید؟",
    "10. تاریخ پایان بیمه ورزشی شما چیست؟",
    "11. لطفاً تصویر کارت ملی خود را ارسال بفرمایید."
]

ASKING = range(1)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("سلام! لطفاً به سوالات زیر برای عضویت پاسخ دهید.")
    await asyncio.sleep(1)
    context.user_data['answers'] = []
    context.user_data['question_index'] = 0
    await update.message.reply_text(questions[0])
    return ASKING

async def handle_response(update: Update, context: ContextTypes.DEFAULT_TYPE):
    question_index = context.user_data.get('question_index', 0)

    # اگر سوال آخر (تصویر کارت ملی) است
    if question_index == len(questions) - 1:
        if update.message.photo:
            photo_file = await update.message.photo[-1].get_file()
            photo_path = f"{update.effective_user.id}_id_card.jpg"
            await photo_file.download_to_drive(photo_path)

            await update.message.reply_text("متشکرم. اطلاعات شما با موفقیت ثبت شد.")
            await context.bot.send_message(chat_id=ADMIN_ID, text=f"کاربر {update.effective_user.full_name} ثبت‌نام را کامل کرد.")

            # ارسال پاسخ‌ها
            answers = context.user_data['answers']
            result = "\n".join(f"{i+1}. {questions[i]}\nپاسخ: {answers[i]}" for i in range(len(answers)))
            await context.bot.send_message(chat_id=ADMIN_ID, text=result)

            # ارسال عکس کارت ملی
            with open(photo_path, "rb") as img:
                await context.bot.send_photo(chat_id=ADMIN_ID, photo=img, caption="تصویر کارت ملی کاربر")

            os.remove(photo_path)
        else:
            await update.message.reply_text("لطفاً یک تصویر ارسال کنید.")
            return ASKING

        return ConversationHandler.END

    else:
        context.user_data['answers'].append(update.message.text)
        context.user_data['question_index'] += 1
        await update.message.reply_text(questions[context.user_data['question_index']])
        return ASKING

async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("فرآیند عضویت لغو شد.")
    return ConversationHandler.END

if __name__ == '__main__':
    TOKEN = os.getenv("BOT_TOKEN", "توکن_خودت_را_اینجا_قرار_بده")

    app = Application.builder().token(TOKEN).build()

    conv_handler = ConversationHandler(
        entry_points=[CommandHandler("start", start)],
        states={
            ASKING: [
                MessageHandler(filters.TEXT & ~filters.COMMAND, handle_response),
                MessageHandler(filters.PHOTO, handle_response),
            ],
        },
        fallbacks=[CommandHandler("cancel", cancel)],
    )

    app.add_handler(conv_handler)

    print("ربات فعال شد...")
    app.run_polling()
.
