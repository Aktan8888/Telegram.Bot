import logging
from typing import Dict, List, TypedDict, Literal, Set
import os
from datetime import datetime, timedelta
from dataclasses import dataclass, field

import httpx
from langdetect import detect, DetectorFactory
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, BotCommand
from telegram.ext import (
    Application, CommandHandler, CallbackQueryHandler, MessageHandler, ContextTypes, filters,
)
from telegram.constants import ChatAction
from telegram.error import TelegramError

# Устанавливаем seed для воспроизводимости результатов langdetect
DetectorFactory.seed = 0

Language = Literal["ru", "en", "tr", "ar"]
SupportedLanguages = Set[Language]
CallbackDataPrefix = Literal["lang"]

class Message(TypedDict):
    role: Literal["user", "assistant", "system"]
    content: str

class UserData(TypedDict):
    language: Language
    history: List[Message]
    requests: List[datetime]

@dataclass
class LocalizedText:
    ru: str
    en: str
    tr: str
    ar: str

    def get(self, lang: Language) -> str:
        return getattr(self, lang)

@dataclass
class TextCollection:
    loading: LocalizedText = field(default_factory=lambda: LocalizedText(
        ru="⏳ Обрабатываю запрос...", en="⏳ Processing request...", tr="⏳ İstek işleniyor...",
        ar="⏳ جاري معالجة الطلب..."
    ))
    not_subscribed: LocalizedText = field(default_factory=lambda: LocalizedText(
        ru="❌ Пожалуйста, подпишитесь на канал {channel} для использования бота.",
        en="❌ Please subscribe to the channel {channel} to use the bot.",
        tr="❌ Botu kullanmak için lütfen {channel} kanalına abone olun.",
        ar="❌ الرجاء الاشتراك في القناة {channel} لاستخدام الروبوت."
    ))
    welcome: LocalizedText = field(default_factory=lambda: LocalizedText(
        ru="👋 Добро пожаловать! Выберите язык:",
        en="👋 Welcome! Please choose a language:",
        tr="👋 Hoş geldiniz! Lütfen bir dil seçin:",
        ar="👋 مرحباً! الرجاء اختيار اللغة:"
    ))
    error: LocalizedText = field(default_factory=lambda: LocalizedText(
        ru="⚠️ Произошла ошибка. Попробуйте позже.",
        en="⚠️ An error occurred. Please try again later.",
        tr="⚠️ Bir hata oluştu. Lütfen tekrar deneyin.",
        ar="⚠️ حدث خطأ. الرجاء المحاولة مرة أخرى لاحقاً."
    ))
    rate_limit: LocalizedText = field(default_factory=lambda: LocalizedText(
        ru="⚠️ Пожалуйста, подождите немного перед отправкой нового запроса.",
        en="⚠️ Please wait a moment before sending a new request.",
        tr="⚠️ Yeni bir istek göndermeden önce lütfen biraz bekleyin.",
        ar="⚠️ يرجى الانتظار قليلاً قبل إرسال طلب جديد."
    ))
    help: LocalizedText = field(default_factory=lambda: LocalizedText(
        ru=(
            "🤖 Я бот-ассистент...\n"
            "/start - Запустить бота\n/help - Показать справку\n"
            "/language - Изменить язык\n/clear - Очистить историю"
        ),
        en=(
            "🤖 I'm an assistant bot...\n"
            "/start - Start the bot\n/help - Show help\n"
            "/language - Change language\n/clear - Clear dialog history"
        ),
        tr=(
            "🤖 Sorularınızı yanıtlayabilen botum...\n"
            "/start - Botu başlat\n/help - Yardımı göster\n"
            "/language - Dili değiştir\n/clear - Geçmişi temizle"
        ),
        ar=(
            "🤖 أنا روبوت مساعد...\n"
            "/start - بدء تشغيل الروبوت\n/help - عرض المساعدة\n"
            "/language - تغيير اللغة\n/clear - مسح سجل المحادثة"
        )
    ))
    language_changed: LocalizedText = field(default_factory=lambda: LocalizedText(
        ru="✅ Язык изменен на Русский",
        en="✅ Language changed to English",
        tr="✅ Dil Türkçe olarak değiştirildi",
        ar="✅ تم تغيير اللغة إلى العربية"
    ))
    history_cleared: LocalizedText = field(default_factory=lambda: LocalizedText(
        ru="✅ История очищена.",
        en="✅ Dialog history cleared.",
        tr="✅ Geçmiş temizlendi.",
        ar="✅ تم مسح سجل المحادثة."
    ))
    # Новое: Текст представления бота после выбора языка
    introduction: LocalizedText = field(default_factory=lambda: LocalizedText(
        ru="🤖 Привет! Я ИИ ассистент по программированию. Чем могу вам помочь?",
        en="🤖 Hello! I'm an AI programming assistant. How can I help you?",
        tr="🤖 Merhaba! Ben bir programlama AI asistanıyım. Size nasıl yardımcı olabilirim?",
        ar="🤖 مرحباً! أنا مساعد برمجة ذكاء اصطناعي. كيف يمكنني مساعدتك؟"
    ))

class TextRepository:
    texts = TextCollection()

    @classmethod
    def get(cls, text_key: str, lang: Language, **format_args) -> str:
        if not hasattr(cls.texts, text_key):
            raise ValueError(f"Неизвестный ключ текста: {text_key}")
        text = getattr(cls.texts, text_key).get(lang)
        return text.format(**format_args) if format_args else text

# Настройка логирования с пользовательским форматом
class CustomUserLogFilter(logging.Filter):
    def filter(self, record):
        # Проверяем, что запись относится к пользовательскому сообщению
        if hasattr(record, 'user_id') and hasattr(record, 'user_message'):
            return True
        return False

# Создаем форматтер для сообщений пользователей
user_formatter = logging.Formatter('%(asctime)s - User ID: %(user_id)s - Username: %(username)s - Message: %(user_message)s')

# Настройка логирования
logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    level=logging.INFO,
    handlers=[logging.StreamHandler()]
)

# Создаем специальный логгер для действий пользователей
user_logger = logging.getLogger("user_actions")
user_logger.setLevel(logging.INFO)

# Создаем обработчик для файла с логами пользователей
file_handler = logging.FileHandler("bot.log")
file_handler.setLevel(logging.INFO)
file_handler.setFormatter(user_formatter)
file_handler.addFilter(CustomUserLogFilter())
user_logger.addHandler(file_handler)

# Основной логгер для системных сообщений
logger = logging.getLogger(__name__)

BOT_TOKEN = os.environ.get("BOT_TOKEN")
OPENROUTER_API_KEY = os.environ.get("OPENROUTER_API_KEY")
MODEL_NAME = os.environ.get("MODEL_NAME", "gpt-4o-mini")
CHANNEL_USERNAME = os.environ.get("CHANNEL_USERNAME", "@CodeNest88")
DEFAULT_LANGUAGE: Language = "ru"
ALLOWED_LANGUAGES: SupportedLanguages = {"ru", "en", "tr", "ar"}
RATE_LIMIT = 5
RATE_PERIOD = 60

user_data: Dict[int, UserData] = {}

class UserService:
    @staticmethod
    def get_language(user_id: int) -> Language:
        if user_id not in user_data:
            user_data[user_id] = UserService.create_default_user_data()
        return user_data[user_id]["language"]

    @staticmethod
    def get_message_history(user_id: int) -> List[Message]:
        if user_id not in user_data:
            user_data[user_id] = UserService.create_default_user_data()
        return user_data[user_id]["history"]

    @staticmethod
    def set_language(user_id: int, language: Language) -> None:
        if user_id not in user_data:
            user_data[user_id] = UserService.create_default_user_data()
        user_data[user_id]["language"] = language

    @staticmethod
    def clear_history(user_id: int) -> None:
        if user_id not in user_data:
            user_data[user_id] = UserService.create_default_user_data()
        user_data[user_id]["history"] = []

    @staticmethod
    def add_message(user_id: int, role: Literal["user", "assistant"], content: str) -> None:
        if user_id not in user_data:
            user_data[user_id] = UserService.create_default_user_data()
        user_data[user_id]["history"].append({"role": role, "content": content})
        if len(user_data[user_id]["history"]) > 20:
            user_data[user_id]["history"] = user_data[user_id]["history"][-20:]

    @staticmethod
    def update_rate_limit(user_id: int) -> bool:
        if user_id not in user_data:
            user_data[user_id] = UserService.create_default_user_data()
        now = datetime.now()
        user_data[user_id]["requests"] = [
            t for t in user_data[user_id]["requests"]
            if now - t < timedelta(seconds=RATE_PERIOD)
        ]
        if len(user_data[user_id]["requests"]) >= RATE_LIMIT:
            return False
        user_data[user_id]["requests"].append(now)
        return True

    @staticmethod
    def create_default_user_data() -> UserData:
        return {"language": DEFAULT_LANGUAGE, "history": [], "requests": []}

class TelegramUIService:
    @staticmethod
    def get_language_buttons() -> InlineKeyboardMarkup:
        return InlineKeyboardMarkup([
            [InlineKeyboardButton("🇷🇺 Русский", callback_data="lang:ru"),
             InlineKeyboardButton("🇬🇧 English", callback_data="lang:en")],
            [InlineKeyboardButton("🇹🇷 Türkçe", callback_data="lang:tr"),
             InlineKeyboardButton("🇸🇦 العربية", callback_data="lang:ar")]
        ])

    @staticmethod
    def parse_callback_data(data: str) -> tuple[CallbackDataPrefix, str]:
        if ":" not in data:
            raise ValueError(f"Неверный формат callback_data: {data}")
        prefix, value = data.split(":", 1)
        return prefix, value

class AIService:
    @staticmethod
    async def send_request(user_text: str, lang: Language, message_history: List[Message]) -> str:
        if lang not in ALLOWED_LANGUAGES:
            lang = DEFAULT_LANGUAGE
        system_prompt = f"Ты — эксперт по программированию. Отвечай на {lang} языке."
        messages = [{"role": "system", "content": system_prompt}] + message_history[-10:] + [{"role": "user", "content": user_text}]
        try:
            async with httpx.AsyncClient(timeout=60.0) as client:
                response = await client.post(
                    "https://openrouter.ai/api/v1/chat/completions",
                    headers={
                        "Authorization": f"Bearer {OPENROUTER_API_KEY}",
                        "Content-Type": "application/json"
                    },
                    json={"model": MODEL_NAME, "messages": messages, "temperature": 0.7, "max_tokens": 1000},
                )
                response.raise_for_status()
                reply = response.json()["choices"][0]["message"]["content"]
                return reply
        except Exception as e:
            logger.error(f"Ошибка при запросе к OpenRouter: {e}")
            return TextRepository.get("error", lang)

async def check_subscription(user_id: int, context: ContextTypes.DEFAULT_TYPE) -> bool:
    """Проверяет, подписан ли пользователь на канал"""
    try:
        chat_member = await context.bot.get_chat_member(chat_id=CHANNEL_USERNAME, user_id=user_id)
        return chat_member.status in ["member", "administrator", "creator"]
    except TelegramError as e:
        logger.error(f"Ошибка при проверке подписки: {e}")
        return True  # В случае ошибки считаем, что пользователь подписан

async def log_user_action(user_id: int, username: str, message: str) -> None:
    """Логирует действие пользователя"""
    record = logging.LogRecord(
        name="user_actions",
        level=logging.INFO,
        pathname="",
        lineno=0,
        msg="",
        args=(),
        exc_info=None
    )
    record.user_id = user_id
    record.username = username or "Unknown"  # В случае, если имя пользователя недоступно
    record.user_message = message
    user_logger.handle(record)

async def start_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Обработчик команды /start"""
    user_id = update.effective_user.id
    language = UserService.get_language(user_id)
    
    # Логируем действие пользователя
    username = update.effective_user.username or update.effective_user.first_name or "Unknown"
    await log_user_action(user_id, username, "/start")
    
    # Проверка подписки
    is_subscribed = await check_subscription(user_id, context)
    if not is_subscribed:
        await update.message.reply_text(
            TextRepository.get("not_subscribed", language, channel=CHANNEL_USERNAME)
        )
        return
    
    await update.message.reply_text(
        TextRepository.get("welcome", language),
        reply_markup=TelegramUIService.get_language_buttons()
    )

async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Обработчик команды /help"""
    user_id = update.effective_user.id
    language = UserService.get_language(user_id)
    
    # Логируем действие пользователя
    username = update.effective_user.username or update.effective_user.first_name or "Unknown"
    await log_user_action(user_id, username, "/help")
    
    await update.message.reply_text(TextRepository.get("help", language))

async def language_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Обработчик команды /language"""
    user_id = update.effective_user.id
    language = UserService.get_language(user_id)
    
    # Логируем действие пользователя
    username = update.effective_user.username or update.effective_user.first_name or "Unknown"
    await log_user_action(user_id, username, "/language")
    
    await update.message.reply_text(
        TextRepository.get("welcome", language),
        reply_markup=TelegramUIService.get_language_buttons()
    )

async def clear_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Обработчик команды /clear"""
    user_id = update.effective_user.id
    language = UserService.get_language(user_id)
    
    # Логируем действие пользователя
    username = update.effective_user.username or update.effective_user.first_name or "Unknown"
    await log_user_action(user_id, username, "/clear")
    
    UserService.clear_history(user_id)
    await update.message.reply_text(TextRepository.get("history_cleared", language))

async def detect_language(text: str) -> Language:
    """Определяет язык текста"""
    try:
        detected = detect(text)
        if detected == "ru":
            return "ru"
        elif detected in ["en", "uk"]:  # Объединяем английский и украинский
            return "en"
        elif detected == "tr":
            return "tr"
        elif detected in ["ar", "fa", "ur"]:  # Арабский, фарси и урду имеют схожие алфавиты
            return "ar"
        return DEFAULT_LANGUAGE
    except:
        return DEFAULT_LANGUAGE

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Обработчик входящих сообщений"""
    if not update.effective_message or not update.effective_message.text:
        return

    user_id = update.effective_user.id
    user_text = update.effective_message.text.strip()
    language = UserService.get_language(user_id)
    
    # Логируем действие пользователя
    username = update.effective_user.username or update.effective_user.first_name or "Unknown"
    await log_user_action(user_id, username, user_text)
    
    # Проверка подписки
    is_subscribed = await check_subscription(user_id, context)
    if not is_subscribed:
        await update.message.reply_text(
            TextRepository.get("not_subscribed", language, channel=CHANNEL_USERNAME)
        )
        return
    
    # Проверка лимита запросов
    if not UserService.update_rate_limit(user_id):
        await update.message.reply_text(TextRepository.get("rate_limit", language))
        return
    
    message_history = UserService.get_message_history(user_id)
    
    # Отправка "печатает..."
    await context.bot.send_chat_action(chat_id=update.effective_chat.id, action=ChatAction.TYPING)
    
    # Отправка промежуточного сообщения
    processing_message = await update.message.reply_text(TextRepository.get("loading", language))
    
    # Добавление сообщения пользователя в историю
    UserService.add_message(user_id, "user", user_text)
    
    # Отправка запроса к AI
    reply = await AIService.send_request(user_text, language, message_history)
    
    # Добавление ответа в историю
    UserService.add_message(user_id, "assistant", reply)
    
    try:
        # Удаление промежуточного сообщения
        await processing_message.delete()
    except TelegramError as e:
        logger.warning(f"Не удалось удалить промежуточное сообщение: {e}")
    
    # Если ответ слишком длинный, разбиваем его на части
    if len(reply) > 4000:
        chunks = [reply[i:i+4000] for i in range(0, len(reply), 4000)]
        for chunk in chunks:
            await update.message.reply_text(chunk)
    else:
        await update.message.reply_text(reply)

async def callback_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Обработчик callback запросов"""
    query = update.callback_query
    data = query.data
    user_id = query.from_user.id

    try:
        prefix, value = TelegramUIService.parse_callback_data(data)
        
        if prefix == "lang" and value in ALLOWED_LANGUAGES:
            UserService.set_language(user_id, value)
            language = UserService.get_language(user_id)
            
            # Логируем выбор языка
            username = query.from_user.username or query.from_user.first_name or "Unknown"
            await log_user_action(user_id, username, f"Выбрал язык: {value}")
            
            await query.answer(TextRepository.get("language_changed", language))
            # Изменяем сообщение с выбором языка
            await query.edit_message_text(
                TextRepository.get("language_changed", language)
            )
            
            # Отправляем приветственное сообщение на выбранном языке
            await context.bot.send_message(
                chat_id=user_id,
                text=TextRepository.get("introduction", language)
            )
        else:
            await query.answer("Unknown callback")
    except Exception as e:
        logger.error(f"Ошибка обработки callback: {e}")
        await query.answer("Error processing callback")

async def setup_commands(application: Application) -> None:
    """Настраивает команды бота"""
    commands = [
        BotCommand("start", "Старт бота / Start bot / Botu başlat / بدء تشغيل الروبوت"),
        BotCommand("help", "Помощь / Help / Yardım / مساعدة"),
        BotCommand("language", "Изменить язык / Change language / Dili değiştir / تغيير اللغة"),
        BotCommand("clear", "Очистить историю / Clear history / Geçmişi temizle / مسح المحادثة")
    ]
    await application.bot.set_my_commands(commands)

def main() -> None:
    """Основная функция запуска бота"""
    application = Application.builder().token(BOT_TOKEN).build()
    
    # Настройка команд
    application.add_handler(CommandHandler("start", start_command))
    application.add_handler(CommandHandler("help", help_command))
    application.add_handler(CommandHandler("language", language_command))
    application.add_handler(CommandHandler("clear", clear_command))
    
    # Обработчик callback запросов
    application.add_handler(CallbackQueryHandler(callback_handler))
    
    # Обработчик сообщений
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    
    # Настройка команд бота при запуске
    application.post_init = setup_commands
    
    # Запуск бота
    application.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == "__main__":
    main()
