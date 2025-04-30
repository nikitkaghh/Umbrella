from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ApplicationBuilder, CommandHandler, CallbackQueryHandler, MessageHandler, ContextTypes, filters
import requests

TOKEN = "7236647733:AAGINM4k6YLoIM0KXXywjmw4izdwgneoYOI"
WEATHER_API_KEY = "8407293301d21ff927e624ad6449860d"

user_states = {}

def main_menu_keyboard():
    return [
        [InlineKeyboardButton("Тарифы", callback_data='tariffs')],
        [InlineKeyboardButton("Карта", url='https://yandex.ru/maps/35/krasnodar')],
        [InlineKeyboardButton("Подписка", callback_data='subscribe')],
        [InlineKeyboardButton("Погода", callback_data='weather')],
        [InlineKeyboardButton("Оставить отзыв", callback_data='feedback')],
        [InlineKeyboardButton("Связь с поддержкой", callback_data='support')]
    ]

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Привет! Выбери, что тебя интересует:",
                                    reply_markup=InlineKeyboardMarkup(main_menu_keyboard()))

async def handle_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    data = query.data
    user_id = query.from_user.id

    if data == 'tariffs':
        keyboard = [
            [InlineKeyboardButton("Одиночная — 299 ₽", callback_data='sub_single')],
            [InlineKeyboardButton("Семейная — 499 ₽", callback_data='sub_family')],
            [InlineKeyboardButton("Поминутная — 1 ₽/мин", callback_data='sub_minute')],
            [InlineKeyboardButton("Назад в меню", callback_data='back_to_menu')]
        ]
        await query.message.edit_text("Наши тарифы:", reply_markup=InlineKeyboardMarkup(keyboard))

    elif data == 'subscribe':
        keyboard = [
            [InlineKeyboardButton("Оформить подписку", url='https://t.me/umbrellarental_bot?start=subscribe')],
            [InlineKeyboardButton("Назад в меню", callback_data='back_to_menu')]
        ]
        await query.message.edit_text("Выберите тип подписки:", reply_markup=InlineKeyboardMarkup(keyboard))

    elif data.startswith('sub_'):
        await query.message.edit_text("Спасибо! Информация о подписке будет доступна в будущих версиях.",
                                      reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("Назад в меню", callback_data='back_to_menu')]]))

    elif data == 'weather':
        try:
            url = f"http://api.openweathermap.org/data/2.5/weather?q=Krasnodar&units=metric&lang=ru&appid={WEATHER_API_KEY}"
            response = requests.get(url).json()
            temp = response["main"]["temp"]
            feels = response["main"]["feels_like"]
            desc = response["weather"][0]["description"]
            msg = f"Погода в Краснодаре:\n{temp}°C, ощущается как {feels}°C\n{desc.capitalize()}"
            await query.message.edit_text(msg, reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("Назад в меню", callback_data='back_to_menu')]]))
        except:
            await query.message.edit_text("Извините, не удалось получить данные о погоде.")

    elif data == 'feedback':
        user_states[user_id] = 'feedback'
        await query.message.edit_text("Напиши свой отзыв одним сообщением. Мы обязательно его прочитаем.")

    elif data == 'support':
        user_states[user_id] = 'support'
        await query.message.edit_text("Опиши проблему — и мы постараемся помочь.")

    elif data == 'back_to_menu':
        await query.message.edit_text("Главное меню:", reply_markup=InlineKeyboardMarkup(main_menu_keyboard()))

    await query.answer()

async def handle_messages(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.message.from_user.id
    state = user_states.get(user_id)

    if state == 'feedback':
        await update.message.reply_text("Спасибо за отзыв! Мы его получили.")
        print(f"[ОТЗЫВ от {user_id}]: {update.message.text}")
        user_states.pop(user_id)
    elif state == 'support':
        await update.message.reply_text("Спасибо! Поддержка получила ваше сообщение.")
        print(f"[ПОДДЕРЖКА от {user_id}]: {update.message.text}")
        user_states.pop(user_id)
    else:
        await update.message.reply_text("Напиши /start, чтобы открыть меню.")

app = ApplicationBuilder().token(TOKEN).build()
app.add_handler(CommandHandler("start", start))
app.add_handler(CallbackQueryHandler(handle_callback))
app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_messages))
app.run_polling()
