import asyncio
import logging
import os
from datetime import datetime, timedelta
from dataclasses import dataclass, field
from typing import Dict, List, TypedDict, Literal, Set, Optional
from contextlib import asynccontextmanager

import httpx
from dotenv import load_dotenv
from langdetect import detect, DetectorFactory
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, BotCommand
from telegram.ext import (
    Application, ContextTypes, CommandHandler, MessageHandler, CallbackQueryHandler, filters
)
from telegram.constants import ChatAction, ParseMode
from telegram.error import TelegramError, BadRequest

# Load environment variables
load_dotenv()

# Устанавливаем seed для воспроизводимости результатов langdetect
DetectorFactory.seed = 0

# Type definitions
Language = Literal["ru", "en", "tr", "ar"]
SupportedLanguages = Set[Language]
CallbackDataPrefix = Literal["lang"]

class Message(TypedDict):
    role: Literal["user", "assistant", "system"]
    content: str
    timestamp: datetime

class UserData(TypedDict):
    language: Language
    history: List[Message]
    requests: List[datetime]
    is_premium: bool

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
        ru="⏳ Обрабатываю запрос...", 
        en="⏳ Processing request...", 
        tr="⏳ İstek işleniyor...",
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
        ru="⚠️ Слишком много запросов. Подождите {seconds} секунд.",
        en="⚠️ Too many requests. Please wait {seconds} seconds.",
        tr="⚠️ Çok fazla istek. Lütfen {seconds} saniye bekleyin.",
        ar="⚠️ طلبات كثيرة جداً. يرجى الانتظار {seconds} ثانية."
    ))
    help: LocalizedText = field(default_factory=lambda: LocalizedText(
        ru=(
            "🤖 *Я бот-ассистент для программирования*\n\n"
            "📋 *Доступные команды:*\n"
            "/start - Запустить бота\n"
            "/help - Показать справку\n"
            "/language - Изменить язык\n"
            "/clear - Очистить историю\n"
            "/stats - Статистика использования"
        ),
        en=(
            "🤖 *I'm an AI programming assistant*\n\n"
            "📋 *Available commands:*\n"
            "/start - Start the bot\n"
            "/help - Show help\n"
            "/language - Change language\n"
            "/clear - Clear dialog history\n"
            "/stats - Usage statistics"
        ),
        tr=(
            "🤖 *Programlama asistanı botuyum*\n\n"
            "📋 *Mevcut komutlar:*\n"
            "/start - Botu başlat\n"
            "/help - Yardımı göster\n"
            "/language - Dili değiştir\n"
            "/clear - Geçmişi temizle\n"
            "/stats - Kullanım istatistikleri"
        ),
        ar=(
            "🤖 *أنا روبوت مساعد للبرمجة*\n\n"
            "📋 *الأوامر المتاحة:*\n"
            "/start - بدء تشغيل الروبوت\n"
            "/help - عرض المساعدة\n"
            "/language - تغيير اللغة\n"
            "/clear - مسح سجل المحادثة\n"
            "/stats - إحصائيات الاستخدام"
        )
    ))
    language_changed: LocalizedText = field(default_factory=lambda: LocalizedText(
        ru="✅ Язык изменен на Русский",
        en="✅ Language changed to English",
        tr="✅ Dil Türkçe olarak değiştirildi",
        ar="✅ تم تغيير اللغة إلى العربية"
    ))
    history_cleared: LocalizedText = field(default_factory=lambda: LocalizedText(
        ru="✅ История диалога очищена.",
        en="✅ Dialog history cleared.",
        tr="✅ Diyalog geçmişi temizlendi.",
        ar="✅ تم مسح سجل المحادثة."
    ))
    introduction: LocalizedText = field(default_factory=lambda: LocalizedText(
        ru="🤖 Привет! Я ИИ-ассистент по программированию. Задавайте вопросы о коде, алгоритмах, отладке и разработке!",
        en="🤖 Hello! I'm an AI programming assistant. Ask me about code, algorithms, debugging, and development!",
        tr="🤖 Merhaba! Ben bir programlama AI asistanıyım. Kod, algoritmalar, hata ayıklama ve geliştirme hakkında sorular sorun!",
        ar="🤖 مرحباً! أنا مساعد برمجة ذكي. اسألني عن الكود والخوارزميات وإصلاح الأخطاء والتطوير!"
    ))
    stats: LocalizedText = field(default_factory=lambda: LocalizedText(
        ru="📊 *Ваша статистика:*\n👤 Язык: {language}\n💬 Сообщений: {messages}\n🕐 Последняя активность: {last_activity}",
        en="📊 *Your statistics:*\n👤 Language: {language}\n💬 Messages: {messages}\n🕐 Last activity: {last_activity}",
        tr="📊 *İstatistikleriniz:*\n👤 Dil: {language}\n💬 Mesajlar: {messages}\n🕐 Son aktivite: {last_activity}",
        ar="📊 *إحصائياتك:*\n👤 اللغة: {language}\n💬 الرسائل: {messages}\n🕐 آخر نشاط: {last_activity}"
    ))

class Config:
    """Configuration class for bot settings"""
    BOT_TOKEN: str = os.environ.get("BOT_TOKEN", "")
    OPENROUTER_API_KEY: str = os.environ.get("OPENROUTER_API_KEY", "")
    MODEL_NAME: str = os.environ.get("MODEL_NAME", "gpt-4o-mini")
    CHANNEL_USERNAME: str = os.environ.get("CHANNEL_USERNAME", "@CodeNest88")
    DEFAULT_LANGUAGE: Language = "ru"
    ALLOWED_LANGUAGES: SupportedLanguages = {"ru", "en", "tr", "ar"}
    RATE_LIMIT: int = int(os.environ.get("RATE_LIMIT", "5"))
    RATE_PERIOD: int = int(os.environ.get("RATE_PERIOD", "60"))
    MAX_HISTORY_LENGTH: int = int(os.environ.get("MAX_HISTORY_LENGTH", "20"))
    MAX_MESSAGE_LENGTH: int = 4000
    REQUEST_TIMEOUT: int = 60
    MAX_TOKENS: int = 2000

    @classmethod
    def validate(cls) -> None:
        """Validate configuration"""
        if not cls.BOT_TOKEN:
            raise ValueError("BOT_TOKEN environment variable is required")
        if not cls.OPENROUTER_API_KEY:
            raise ValueError("OPENROUTER_API_KEY environment variable is required")

class TextRepository:
    """Repository for localized texts"""
    texts = TextCollection()

    @classmethod
    def get(cls, text_key: str, lang: Language, **format_args) -> str:
        if not hasattr(cls.texts, text_key):
            raise ValueError(f"Unknown text key: {text_key}")
        text = getattr(cls.texts, text_key).get(lang)
        return text.format(**format_args) if format_args else text

class Logger:
    """Enhanced logging setup"""

    @staticmethod
    def setup():
        # Используем текущую директорию для логов
        current_dir = os.path.dirname(os.path.abspath(__file__))
        logs_dir = os.path.join(current_dir, "logs")

        # Создаём директорию логов
        try:
            os.makedirs(logs_dir, exist_ok=True)
            print(f"Logs directory: {logs_dir}")
        except Exception as e:
            print(f"Error creating logs directory: {e}")
            logs_dir = current_dir

        # Пути к файлам логов
        bot_log_path = os.path.join(logs_dir, "bot.log")
        user_actions_log_path = os.path.join(logs_dir, "user_actions.log")

        # Настройка основного логгера
        main_logger = logging.getLogger("main")
        main_logger.setLevel(logging.INFO)

        # Удаляем все существующие обработчики
        main_logger.handlers = []

        # Форматирование
        formatter = logging.Formatter(
            "%(asctime)s [UTC] - %(name)s - %(levelname)s - %(message)s"
        )

        # Обработчик для консоли
        console_handler = logging.StreamHandler()
        console_handler.setFormatter(formatter)
        main_logger.addHandler(console_handler)

        # Обработчик для файла
        file_handler = logging.FileHandler(bot_log_path, encoding="utf-8", mode="a")
        file_handler.setFormatter(formatter)
        main_logger.addHandler(file_handler)

        # Специальный логгер для пользователей
        user_logger = logging.getLogger("user_actions")
        user_logger.setLevel(logging.INFO)
        user_logger.handlers = []
        user_logger.propagate = False

        # Форматирование для пользовательских действий
        user_formatter = logging.Formatter(
            "%(asctime)s [UTC] - UserID: %(message)s"
        )

        # Файловый обработчик для пользовательских действий
        user_file_handler = logging.FileHandler(
            user_actions_log_path, encoding="utf-8", mode="a"
        )
        user_file_handler.setFormatter(user_formatter)
        user_logger.addHandler(user_file_handler)

        # Тестируем логирование
        main_logger.info("=== LOGGER SETUP COMPLETED ===")
        user_logger.info("system:setup - User logger test")

        print("Logger setup completed successfully")
        return main_logger, user_logger

# Initialize loggers
logger, user_logger = Logger.setup()

# Global user data storage
user_data: Dict[int, UserData] = {}

class UserService:
    """Service for managing user data"""

    @staticmethod
    def get_user_data(user_id: int) -> UserData:
        """Get or create user data"""
        if user_id not in user_data:
            user_data[user_id] = UserService._create_default_user_data()
        return user_data[user_id]

    @staticmethod
    def get_language(user_id: int) -> Language:
        return UserService.get_user_data(user_id)["language"]

    @staticmethod
    def get_message_history(user_id: int) -> List[Message]:
        return UserService.get_user_data(user_id)["history"]

    @staticmethod
    def set_language(user_id: int, language: Language) -> None:
        UserService.get_user_data(user_id)["language"] = language

    @staticmethod
    def clear_history(user_id: int) -> None:
        UserService.get_user_data(user_id)["history"] = []

    @staticmethod
    def add_message(user_id: int, role: Literal["user", "assistant"], content: str) -> None:
        user_data_dict = UserService.get_user_data(user_id)
        message: Message = {
            "role": role, 
            "content": content,
            "timestamp": datetime.utcnow()
        }
        user_data_dict["history"].append(message)

        # Ограничиваем историю
        if len(user_data_dict["history"]) > Config.MAX_HISTORY_LENGTH:
            user_data_dict["history"] = user_data_dict["history"][-Config.MAX_HISTORY_LENGTH:]

    @staticmethod
    def check_rate_limit(user_id: int) -> tuple[bool, int]:
        """Check rate limit and return (is_allowed, seconds_to_wait)"""
        user_data_dict = UserService.get_user_data(user_id)
        now = datetime.utcnow()

        # Очищаем старые запросы
        user_data_dict["requests"] = [
            t for t in user_data_dict["requests"]
            if now - t < timedelta(seconds=Config.RATE_PERIOD)
        ]

        if len(user_data_dict["requests"]) >= Config.RATE_LIMIT:
            # Вычисляем время ожидания
            oldest_request = min(user_data_dict["requests"])
            wait_time = int((oldest_request + timedelta(seconds=Config.RATE_PERIOD) - now).total_seconds())
            return False, max(1, wait_time)

        user_data_dict["requests"].append(now)
        return True, 0

    @staticmethod
    def get_stats(user_id: int) -> dict:
        """Get user statistics"""
        user_data_dict = UserService.get_user_data(user_id)
        message_count = len([msg for msg in user_data_dict["history"] if msg["role"] == "user"])

        # Исправляем получение последней активности
        language = user_data_dict["language"]
        if not user_data_dict["history"]:
            if language == "ru":
                last_activity = "Никогда"
            elif language == "en":
                last_activity = "Never"
            elif language == "tr":
                last_activity = "Hiçbir zaman"
            else:  # ar
                last_activity = "أبداً"
        else:
            last_activity = user_data_dict["history"][-1]["timestamp"].strftime("%d.%m.%Y %H:%M")

        return {
            "language": user_data_dict["language"].upper(),
            "messages": message_count,
            "last_activity": last_activity
        }

    @staticmethod
    def _create_default_user_data() -> UserData:
        return {
            "language": Config.DEFAULT_LANGUAGE, 
            "history": [], 
            "requests": [],
            "is_premium": False
        }

class TelegramUIService:
    """Service for Telegram UI components"""

    @staticmethod
    def get_language_buttons() -> InlineKeyboardMarkup:
        return InlineKeyboardMarkup([
            [
                InlineKeyboardButton("🇷🇺 Русский", callback_data="lang:ru"),
                InlineKeyboardButton("🇬🇧 English", callback_data="lang:en")
            ],
            [
                InlineKeyboardButton("🇹🇷 Türkçe", callback_data="lang:tr"),
                InlineKeyboardButton("🇸🇦 العربية", callback_data="lang:ar")
            ]
        ])

    @staticmethod
    def parse_callback_data(data: str) -> tuple[CallbackDataPrefix, str]:
        if ":" not in data:
            raise ValueError(f"Invalid callback_data format: {data}")
        prefix, value = data.split(":", 1)
        return prefix, value  # type: ignore

    @staticmethod
    def split_long_message(text: str, max_length: int = Config.MAX_MESSAGE_LENGTH) -> List[str]:
        """Split long message into chunks"""
        if len(text) <= max_length:
            return [text]

        chunks = []
        current_chunk = ""

        for line in text.split('\n'):
            if len(current_chunk) + len(line) + 1 <= max_length:
                current_chunk += line + '\n'
            else:
                if current_chunk:
                    chunks.append(current_chunk.rstrip())
                current_chunk = line + '\n'

        if current_chunk:
            chunks.append(current_chunk.rstrip())

        return chunks

class AIService:
    """Service for AI interactions"""

    @staticmethod
    @asynccontextmanager
    async def get_http_client():
        """Context manager for HTTP client"""
        async with httpx.AsyncClient(
            timeout=httpx.Timeout(Config.REQUEST_TIMEOUT),
            limits=httpx.Limits(max_keepalive_connections=5, max_connections=10)
        ) as client:
            yield client

    @staticmethod
    async def send_request(user_text: str, lang: Language, message_history: List[Message]) -> str:
        """Send request to AI service"""
        if lang not in Config.ALLOWED_LANGUAGES:
            lang = Config.DEFAULT_LANGUAGE

        # Подготовка системного промпта
        system_prompts = {
            "ru": "Ты — эксперт по программированию. Отвечай на русском языке. Помогай с кодом, алгоритмами, отладке и разработкой.",
            "en": "You are a programming expert. Answer in English. Help with code, algorithms, debugging, and development.",
            "tr": "Sen bir programlama uzmanısın. Türkçe cevap ver. Kod, algoritmalar, hata ayıklama ve geliştirme konularında yardım et.",
            "ar": "أنت خبير في البرمجة. أجب باللغة العربية. ساعد في الكود والخوارزميات وإصلاح الأخطاء والتطوير."
        }

        system_prompt = system_prompts.get(lang, system_prompts[Config.DEFAULT_LANGUAGE])

        # Подготовка сообщений
        messages = [{"role": "system", "content": system_prompt}]

        # Добавляем последние сообщения из истории
        recent_history = message_history[-10:] if len(message_history) > 10 else message_history
        for msg in recent_history:
            messages.append({"role": msg["role"], "content": msg["content"]})

        messages.append({"role": "user", "content": user_text})

        try:
            async with AIService.get_http_client() as client:
                response = await client.post(
                    "https://openrouter.ai/api/v1/chat/completions",
                    headers={
                        "Authorization": f"Bearer {Config.OPENROUTER_API_KEY}",
                        "Content-Type": "application/json",
                        "HTTP-Referer": "https://github.com/your-repo",  # Добавляем реферер
                        "X-Title": "Programming Assistant Bot"
                    },
                    json={
                        "model": Config.MODEL_NAME,
                        "messages": messages,
                        "temperature": 0.7,
                        "max_tokens": Config.MAX_TOKENS,
                        "top_p": 0.9,
                        "frequency_penalty": 0.1,
                        "presence_penalty": 0.1
                    }
                )
                response.raise_for_status()
                data = response.json()
                return data["choices"][0]["message"]["content"]

        except httpx.TimeoutException:
            logger.error("Request timeout to OpenRouter API")
            return TextRepository.get("error", lang) + " (Timeout)"
        except httpx.HTTPStatusError as e:
            logger.error(f"HTTP error from OpenRouter API: {e.response.status_code}")
            return TextRepository.get("error", lang) + f" (HTTP {e.response.status_code})"
        except Exception as e:
            logger.error(f"Error requesting OpenRouter API: {e}")
            return TextRepository.get("error", lang)

class SubscriptionService:
    """Service for subscription management"""

    @staticmethod
    async def check_subscription(user_id: int, context: ContextTypes.DEFAULT_TYPE) -> bool:
        """Check if user is subscribed to the channel"""
        if not Config.CHANNEL_USERNAME:
            return True

        try:
            chat_member = await context.bot.get_chat_member(
                chat_id=Config.CHANNEL_USERNAME, 
                user_id=user_id
            )
            return chat_member.status in ["member", "administrator", "creator"]
        except TelegramError as e:
            logger.error(f"Error checking subscription: {e}")
            return True  # В случае ошибки считаем подписанным

class LogService:
    """Service for logging user actions"""

    @staticmethod
    async def log_user_action(user_id: int, username: Optional[str], action: str) -> None:
        """Log user action"""
        # Логируем через специальный логгер
        try:
            log_message = f"{user_id}:{username or 'Unknown'} - {action}"
            user_logger.info(log_message)
            logger.info(f"User action logged: {log_message}")
        except Exception as e:
            logger.error(f"Failed to log user action: {e}")

# Определяем состояния для ConversationHandler
LANGUAGE_SELECTION = 1

# Command handlers
async def start_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Handler for /start command"""
    if not update.effective_user or not update.message:
        return

    user_id = update.effective_user.id
    username = update.effective_user.username or update.effective_user.first_name

    await LogService.log_user_action(user_id, username, "/start")

    # Проверка подписки
    if not await SubscriptionService.check_subscription(user_id, context):
        language = UserService.get_language(user_id)
        await update.message.reply_text(
            TextRepository.get("not_subscribed", language, channel=Config.CHANNEL_USERNAME)
        )
        return

    language = UserService.get_language(user_id)
    await update.message.reply_text(
        TextRepository.get("welcome", language),
        reply_markup=TelegramUIService.get_language_buttons()
    )

async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Handler for /help command"""
    if not update.effective_user or not update.message:
        return

    user_id = update.effective_user.id
    username = update.effective_user.username or update.effective_user.first_name
    language = UserService.get_language(user_id)

    await LogService.log_user_action(user_id, username, "/help")

    await update.message.reply_text(
        TextRepository.get("help", language),
        parse_mode=ParseMode.MARKDOWN
    )

async def language_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Handler for /language command"""
    if not update.effective_user or not update.message:
        return

    user_id = update.effective_user.id
    username = update.effective_user.username or update.effective_user.first_name
    language = UserService.get_language(user_id)

    await LogService.log_user_action(user_id, username, "/language")

    await update.message.reply_text(
        TextRepository.get("welcome", language),
        reply_markup=TelegramUIService.get_language_buttons()
    )

async def clear_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Handler for /clear command"""
    if not update.effective_user or not update.message:
        return

    user_id = update.effective_user.id
    username = update.effective_user.username or update.effective_user.first_name
    language = UserService.get_language(user_id)

    await LogService.log_user_action(user_id, username, "/clear")

    UserService.clear_history(user_id)
    await update.message.reply_text(TextRepository.get("history_cleared", language))

async def stats_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Handler for /stats command"""
    if not update.effective_user or not update.message:
        return

    user_id = update.effective_user.id
    username = update.effective_user.username or update.effective_user.first_name
    language = UserService.get_language(user_id)

    await LogService.log_user_action(user_id, username, "/stats")

    stats = UserService.get_stats(user_id)
    await update.message.reply_text(
        TextRepository.get("stats", language, **stats),
        parse_mode=ParseMode.MARKDOWN
    )

async def detect_language(text: str) -> Language:
    """Detect text language"""
    try:
        detected = detect(text)
        language_mapping = {
            "ru": "ru",
            "en": "en", "uk": "en",  # Ukrainian -> English
            "tr": "tr",
            "ar": "ar", "fa": "ar", "ur": "ar"  # Persian, Urdu -> Arabic
        }
        return language_mapping.get(detected, Config.DEFAULT_LANGUAGE)
    except Exception:
        return Config.DEFAULT_LANGUAGE

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Handler for incoming messages"""
    if not update.effective_message or not update.effective_message.text or not update.effective_user:
        return

    user_id = update.effective_user.id
    username = update.effective_user.username or update.effective_user.first_name
    user_text = update.effective_message.text.strip()
    language = UserService.get_language(user_id)

    await LogService.log_user_action(user_id, username, f"Message: {user_text[:50]}...")

    # Проверка подписки
    if not await SubscriptionService.check_subscription(user_id, context):
        await update.effective_message.reply_text(
            TextRepository.get("not_subscribed", language, channel=Config.CHANNEL_USERNAME)
        )
        return

    # Проверка лимита запросов
    is_allowed, wait_time = UserService.check_rate_limit(user_id)
    if not is_allowed:
        await update.effective_message.reply_text(
            TextRepository.get("rate_limit", language, seconds=wait_time)
        )
        return

    # Получаем историю сообщений
    message_history = UserService.get_message_history(user_id)

    # Показываем индикатор печати
    await context.bot.send_chat_action(
        chat_id=update.effective_chat.id, 
        action=ChatAction.TYPING
    )

    # Отправляем промежуточное сообщение
    processing_message = await update.effective_message.reply_text(
        TextRepository.get("loading", language)
    )

    try:
        # Добавляем сообщение пользователя в историю
        UserService.add_message(user_id, "user", user_text)

        # Отправляем запрос к AI
        reply = await AIService.send_request(user_text, language, message_history)

        # Добавляем ответ в историю
        UserService.add_message(user_id, "assistant", reply)

        # Удаляем промежуточное сообщение
        try:
            await processing_message.delete()
        except BadRequest:
            pass  # Сообщение уже удалено или недоступно

        # Разбиваем длинные сообщения на части
        message_chunks = TelegramUIService.split_long_message(reply)

        for i, chunk in enumerate(message_chunks):
            if i > 0:
                await asyncio.sleep(0.5)  # Небольшая задержка между частями
            await update.effective_message.reply_text(chunk)

    except Exception as e:
        logger.error(f"Error handling message: {e}")
        try:
            await processing_message.edit_text(TextRepository.get("error", language))
        except BadRequest:
            await update.effective_message.reply_text(TextRepository.get("error", language))

async def callback_handler(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Handler for callback queries"""
    query = update.callback_query
    if not query or not query.from_user:
        return

    user_id = query.from_user.id
    username = query.from_user.username or query.from_user.first_name

    try:
        await query.answer()  # Подтверждаем получение callback

        if not query.data:
            await query.answer("No callback data", show_alert=True)
            return

        prefix, value = TelegramUIService.parse_callback_data(query.data)

        if prefix == "lang" and value in Config.ALLOWED_LANGUAGES:
            UserService.set_language(user_id, value)  # type: ignore
            language = UserService.get_language(user_id)

            await LogService.log_user_action(user_id, username, f"Language selected: {value}")

            # Обновляем сообщение с выбором языка
            await query.edit_message_text(
                TextRepository.get("language_changed", language)
            )

            # Отправляем приветственное сообщение
            await context.bot.send_message(
                chat_id=user_id,
                text=TextRepository.get("introduction", language)
            )
        else:
            await query.answer("Unknown callback", show_alert=True)

    except Exception as e:
        logger.error(f"Error handling callback: {e}")
        try:
            await query.answer("Error processing callback", show_alert=True)
        except Exception:
            pass

async def error_handler(update: object, context: ContextTypes.DEFAULT_TYPE) -> None:
    """Global error handler"""
    logger.error(f"Exception while handling an update: {context.error}", exc_info=True)

async def setup_commands(application: Application) -> None:
    """Setup bot commands"""
    commands = [
        BotCommand("start", "Старт / Start / Başlat / بدء"),
        BotCommand("help", "Помощь / Help / Yardım / مساعدة"),
        BotCommand("language", "Язык / Language / Dil / لغة"),
        BotCommand("clear", "Очистить / Clear / Temizle / مسح"),
        BotCommand("stats", "Статистика / Stats / İstatistik / إحصائيات")
    ]
    try:
        await application.bot.set_my_commands(commands)
        logger.info("Bot commands setup successfully")
    except Exception as e:
        logger.error(f"Failed to set bot commands: {e}")

def main() -> None:
    """Main function to run the bot"""
    try:
        # Validate configuration
        Config.validate()
        logger.info("=== BOT STARTUP ===")
        logger.info(f"Starting Telegram bot with model: {Config.MODEL_NAME}")
        logger.info(f"Channel requirement: {Config.CHANNEL_USERNAME}")

        # Create application
        application = Application.builder().token(Config.BOT_TOKEN).build()

        # Add handlers
        application.add_handler(CommandHandler("start", start_command))
        application.add_handler(CommandHandler("help", help_command))
        application.add_handler(CommandHandler("language", language_command))
        application.add_handler(CommandHandler("clear", clear_command))
        application.add_handler(CommandHandler("stats", stats_command))
        application.add_handler(CallbackQueryHandler(callback_handler))
        application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
        application.add_error_handler(error_handler)

        # Setup commands before starting polling
        async def post_init(app):
            await setup_commands(app)

        application.post_init = post_init

        # Start polling
        application.run_polling(allowed_updates=Update.ALL_TYPES, 
                              drop_pending_updates=True)

    except Exception as e:
        logger.error(f"Failed to start bot: {e}", exc_info=True)
        raise

if __name__ == "__main__":
    main()
# Telegram.Bot9
