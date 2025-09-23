import os
import requests
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, MessageHandler, filters, CallbackContext, CallbackQueryHandler
from dotenv import load_dotenv

# Загружаем переменные окружения из файла .env
load_dotenv()

# Конфигурация
BOT_TOKEN = os.getenv('BOT_TOKEN')
MYMEMORY_API_URL = "http://api.mymemory.translated.net/get"

# Список поддерживаемых языков
LANGUAGES = {
    '🇷🇺 Русский': 'ru',
    '🇺🇸 Английский': 'en',
    '🇪🇸 Испанский': 'es',
    '🇫🇷 Французский': 'fr',
    '🇩🇪 Немецкий': 'de',
    '🇨🇳 Китайский': 'zh',
    '🇯🇵 Японский': 'ja',
    '🇰🇷 Корейский': 'ko',
    '🇮🇹 Итальянский': 'it',
    '🇵🇹 Португальский': 'pt',
    '🇦🇷 Арабский': 'ar',
    '🇹🇷 Турецкий': 'tr'
}

# Хранилище для выбора языка пользователями (в реальном проекте используйте БД)
user_languages = {}

def create_language_keyboard():
    """Создает клавиатуру для выбора языка"""
    keyboard = []
    languages_list = list(LANGUAGES.items())
    
    # Создаем кнопки по 2 в ряду
    for i in range(0, len(languages_list), 2):
        row = []
        for j in range(2):
            if i + j < len(languages_list):
                lang_name, lang_code = languages_list[i + j]
                row.append(InlineKeyboardButton(lang_name, callback_data=f"lang_{lang_code}"))
        keyboard.append(row)
    
    return InlineKeyboardMarkup(keyboard)

async def start(update: Update, context: CallbackContext):
    """Обработчик команды /start"""
    user = update.effective_user
    welcome_text = f"""
Привет, {user.first_name}! 👋

Я бот-переводчик! Я могу переводить текст между различными языками.

📝 **Как использовать:**
1. Выберите целевой язык кнопками ниже
2. Отправьте мне текст для перевода
3. Получите перевод!

⚡ **Команды:**
/start - показать это сообщение
/lang - выбрать язык перевода
/help - справка

Выберите язык для перевода:
    """
    
    await update.message.reply_text(welcome_text, reply_markup=create_language_keyboard())

async def help_command(update: Update, context: CallbackContext):
    """Обработчик команды /help"""
    help_text = """
🆘 **Справка по использованию бота**

📋 **Основные команды:**
/start - начать работу с ботом
/lang - выбрать язык перевода
/help - показать эту справку

🌐 **Как переводить:**
1. Сначала выберите целевой язык с помощью кнопок
2. Отправьте мне любой текст
3. Я автоматически переведу его на выбранный язык

🔤 **Поддерживаемые языки:**
Русский, Английский, Испанский, Французский, Немецкий, Китайский, Японский, Корейский, Итальянский, Португальский, Арабский, Турецкий

💡 **Совет:** Вы можете быстро сменить язык в любое время с помощью команды /lang
    """
    await update.message.reply_text(help_text)

async def lang_command(update: Update, context: CallbackContext):
    """Обработчик команды /lang для выбора языка"""
    await update.message.reply_text("Выберите язык для перевода:", reply_markup=create_language_keyboard())

async def handle_language_selection(update: Update, context: CallbackContext):
    """Обработчик выбора языка"""
    query = update.callback_query
    await query.answer()
    
    lang_code = query.data.replace('lang_', '')
    user_id = query.from_user.id
    
    # Находим название языка по коду
    lang_name = next((name for name, code in LANGUAGES.items() if code == lang_code), "Неизвестный")
    
    # Сохраняем выбор пользователя
    user_languages[user_id] = lang_code
    
    await query.edit_message_text(f"✅ Язык перевода установлен: {lang_name}\n\nТеперь отправьте мне текст для перевода!")

def translate_text(text, target_lang):
    """Функция для перевода текста с помощью MyMemory API"""
    try:
        params = {
            'q': text,
            'langpair': f'auto|{target_lang}',
            'de': 'your-email@gmail.com'  # Рекомендуется указать email для бесплатного API
        }
        
        response = requests.get(MYMEMORY_API_URL, params=params)
        response.raise_for_status()
        
        data = response.json()
        translated_text = data['responseData']['translatedText']
        
        # Информация о качестве перевода
        match_count = data.get('matches', [{}])[0].get('match', 0)
        
        return translated_text, match_count
        
    except Exception as e:
        return None, 0

async def handle_message(update: Update, context: CallbackContext):
    """Обработчик текстовых сообщений"""
    user_id = update.effective_user.id
    text = update.message.text
    
    # Проверяем, выбран ли язык у пользователя
    if user_id not in user_languages:
        await update.message.reply_text("⚠️ Сначала выберите язык для перевода с помощью команды /lang")
        return
    
    target_lang = user_languages[user_id]
    lang_name = next((name for name, code in LANGUAGES.items() if code == target_lang), "Неизвестный")
    
    # Показываем статус "печатает"
    await context.bot.send_chat_action(chat_id=update.effective_chat.id, action="typing")
    
    # Переводим текст
    translated_text, quality = translate_text(text, target_lang)
    
    if translated_text:
        # Формируем ответ
        response = f"""
🔤 **Исходный текст:**
{text}

🌐 **Перевод ({lang_name}):**
{translated_text}

📊 **Качество перевода:** {quality}%
        """
        await update.message.reply_text(response)
    else:
        await update.message.reply_text("❌ Произошла ошибка при переводе. Попробуйте позже.")

async def error_handler(update: Update, context: CallbackContext):
    """Обработчик ошибок"""
    print(f"Ошибка: {context.error}")
    if update and update.effective_message:
        await update.effective_message.reply_text("❌ Произошла непредвиденная ошибка. Попробуйте еще раз.")

def main():
    """Основная функция для запуска бота"""
    if not BOT_TOKEN:
        print("Ошибка: BOT_TOKEN не найден в переменных окружения!")
        return
    
    # Создаем приложение
    application = Application.builder().token(BOT_TOKEN).build()
    
    # Добавляем обработчики
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("help", help_command))
    application.add_handler(CommandHandler("lang", lang_command))
    application.add_handler(CallbackQueryHandler(handle_language_selection, pattern="^lang_"))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    application.add_error_handler(error_handler)
    
    # Запускаем бота
    print("Бот запущен...")
    application.run_polling()

if __name__ == '__main__':
    main()