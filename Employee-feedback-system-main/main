import requests
import datetime
import os

from telegram import Update, ReplyKeyboardMarkup, ReplyKeyboardRemove
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    ConversationHandler,
    PicklePersistence,
    ContextTypes,
    filters
)

from sqlalchemy import (
    create_engine,
    Column,
    Integer,
    String,
    DateTime,
    Text
)
from sqlalchemy.orm import declarative_base, sessionmaker, relationship


################################################################################
# 1. НАСТРОЙКИ
################################################################################

# Токен телеграм-бота
TELEGRAM_BOT_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")

# Настройки YandexGPT
# ссылка, куда отправляются запросы для анализа текста
YANDEX_GPT_API_ENDPOINT = "https://llm.api.cloud.yandex.net/foundationModels/v1/completion"

# OAuth-токен и FolderID YandexGPT
YANDEX_OAUTH_TOKEN = os.getenv("YANDEX_OAUTH_TOKEN")
YANDEX_FOLDER_ID = os.getenv("YANDEX_FOLDER_ID")


def get_iam_token():
    response = requests.post(
        'https://iam.api.cloud.yandex.net/iam/v1/tokens',
        json={'yandexPassportOauthToken': YANDEX_OAUTH_TOKEN}
    )
    response.raise_for_status()
    return response.json()['iamToken']


# Путь к базе данных SQLite
DB_PATH = "sqlite:///my_survey_bot.db"


################################################################################
# 2. МОДЕЛИ БАЗЫ ДАННЫХ (SQLALCHEMY)
################################################################################

Base = declarative_base()


class Ideas(Base):
    __tablename__ = "ideas"

    id = Column(Integer, primary_key=True)
    text_idea = Column(String, nullable=True)
    chat_id = Column(String, nullable=True)
    user_name = Column(String, nullable=True)
    created_at = Column(DateTime, default=datetime.datetime.utcnow)
    status = Column(String, default='На модерации')
    moderator_comment = Column(Text, nullable=True)


class AnalysisLog(Base):
    __tablename__ = "analysis_logs"

    id = Column(Integer, primary_key=True)
    created_at = Column(DateTime, default=datetime.datetime.utcnow)
    # место хранения JSON ответа от YandexGPT
    analysis_result = Column(Text, nullable=True)


class AdminUser(Base):
    __tablename__ = "admin_user"

    id = Column(Integer, primary_key=True)
    user_name = Column(String, nullable=True)
    user_chat_id = Column(Integer, nullable=True)


engine = create_engine(DB_PATH, echo=False)
Base.metadata.create_all(engine)
SessionLocal = sessionmaker(bind=engine)


################################################################################
# 3. ФУНКЦИИ ДЛЯ РАБОТЫ С ДАННЫМИ
################################################################################

def request_yandex_gpt(user_text: str) -> dict:
    # Получение iamToken
    iam_token = get_iam_token()

    headers = {
        "Authorization": f"Bearer {iam_token}",
        "Accept": "application/json",
        "Content-Type": "application/json"
    }

    data = {
        "modelUri": f"gpt://{YANDEX_FOLDER_ID}/yandexgpt",
        "completionOptions": {
            "temperature": 0.8,
            "maxTokens": 1000
        },
        "messages": [
            {
                "role": "system",
                "text": "Ты помощник, который анализирует отзывы сотрудников."
            },
            {
                "role": "user",
                "text": user_text
            }
        ]
    }

    try:
        response = requests.post(
            YANDEX_GPT_API_ENDPOINT,
            headers=headers,
            json=data
        )
        response.raise_for_status()
        result = response.json()
        # Текст ответа ищем в result["result"]["alternatives"][0]["message"]["text"]
        # или возвращаем весь словарь
        return result
    except requests.RequestException as e:
        print(f"[ERROR] request_yandex_gpt: {e}")
        return {}

def analyze_all_ideas_with_yandex_gpt() -> dict:
    session = SessionLocal()
    ideas = session.query(Ideas).all()
    session.close()

    if not ideas:
        return {"error": "Нет инициатив для анализа."}

    idea_payload = [
        {"text": idea.text_idea, "chat_id": int(idea.chat_id)}
        for idea in ideas
    ]

    prompt = (
        "Ты получаешь список инициатив от сотрудников компании"
        "Каждая инициатива содержит текст предложения и chat_id отправителя. Твоя задача:\n\n"
        "1. Проанализируй все инициативы.\n"
        "2. Объедини похожие по смыслу инициативы в группы (кластеры).\n"
        "3. Для каждой группы:\n"
        "◦ Сформулируй объединённую идею, которая обобщает все инициативы внутри группы.\n"
        "◦ Примерная структура:\n"
        " идея 1: ...\n"
        " идея 2: ...\n"
        "и т.д."
        f"А вот данные: ```{idea_payload}```"
    )

    return request_yandex_gpt(prompt)


################################################################################
# 4. СОСТОЯНИЯ ДЛЯ CONVERSATIONHANDLER
################################################################################
(
    MAIN_MENU,                  # Главное меню: "Предложить идею"/"Мои идеи"
    SUBMIT_IDEA_TEXT,           # Ввод текста идеи
    SUBMIT_IDEA_CONFIRMATION,   # Подтверждение перед отправкой
    MY_IDEAS_LIST,              # Просмотр своих идей
    IDEA_DETAILS,               # Детали конкретной идей (статус, комментарии)
    IDEA_DETAILS_SELECT,        # Выбор идеи
    MODERATION_PANEL,           # Панель админа, для управления идеями
    # Добавить администраторов (для догступа к admin-панели)
    ADD_AN_MODERATION,
    MODERATION_LIST,            # Список идей админу
    MODERATION_DECISION,        # Одобрить / отклонить / запросить доработку
    MODERATION_COMMENT,         # Комментарий модератора (если требуется)
    ANALYTICS_VIEW,             # Просмотр аналитики (графики, статистика)
) = range(12)


################################################################################
# 5. ХЕНДЛЕРЫ
################################################################################


def main_menu_markup(chat_id) -> ReplyKeyboardMarkup:

    if check_chat_id(chat_id):
        keyboard = [
            ["Предложить идею"],
            ["Мои идеи"],
            ["Статус идеи"],
            ["Admin"]
        ]

    else:
        keyboard = [
            ["Предложить идею"],
            ["Мои идеи"],
            ["Статус идеи"]
        ]

    return ReplyKeyboardMarkup(keyboard, resize_keyboard=True)


def admin_menu_markup() -> ReplyKeyboardMarkup:
    keyboard = [
        ["Анализ"],
        ["Список идей"],
        ["Главное меню"],
        ["Добавить admin"]
    ]
    return ReplyKeyboardMarkup(keyboard, resize_keyboard=True)


def check_chat_id(chat_id):
    """
    Проверка Chat_id для доступа к панели админа 
    """

    session = SessionLocal()

    answers_query = (
        session.query(AdminUser)
        .all()
    )

    admin_chat_id = []
    for adm in answers_query:
        admin_chat_id.append(
            adm.user_chat_id
        )

    if chat_id in admin_chat_id:
        return True

    else:
        return False


async def start_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """
    /start — Начальное меню.
    """

    await update.message.reply_text(
        "Привет! Выберите действие:",
        reply_markup=main_menu_markup(
            update.message.chat.id)  # вызываем функцию
    )
    return MAIN_MENU


async def handle_role_choice(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """
    Обрабатываем выбор пункта меню: "Предложить идею", "Мои идеи", "Admin".
    """
    text = update.message.text.strip()

    if text == "Предложить идею":
        await update.message.reply_text("Введите вашу идею:")
        return SUBMIT_IDEA_TEXT

    elif text == "Мои идеи":
        return await list_of_my_ideas(update, context)

    elif text == "Admin":
        await update.message.reply_text(
            f"Добро пожаловать в Admin-панель, {update.message.chat.full_name}!",
            reply_markup=admin_menu_markup()
        )
        return MODERATION_PANEL

    elif text == "Статус идеи":
        return await show_idea_choices(update, context)

    else:
        await update.message.reply_text("Неизвестная команда. Пожалуйста, выберите действие снова.")
        return MAIN_MENU


# ------------------------------------------------------------------------------
# 5.1 "Предложить идею"
# ------------------------------------------------------------------------------

async def receive_new_ideas(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """
    Получаем текст идеи и запрашиваем подтверждение.
    """

    text_idea = update.message.text.strip()
    context.user_data["pending_idea"] = text_idea

    await update.message.reply_text(
        f"Вы ввели идею:\n\n\"{text_idea}\"\n\nВсё верно?",
        reply_markup=ReplyKeyboardMarkup(
            [["Да", "Нет"]],
            resize_keyboard=True,
            one_time_keyboard=True
        )
    )
    return SUBMIT_IDEA_CONFIRMATION


async def confirm_idea_submission(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """
    Подтверждение идеи перед сохранением.
    """
    response = update.message.text.strip().lower()

    if response == "да":
        session = SessionLocal()
        chat_id = update.message.chat.id
        user_name = update.message.chat.username
        text_idea = context.user_data.get("pending_idea")

        new_i = Ideas(text_idea=text_idea,
                      chat_id=chat_id, user_name=user_name)
        session.add(new_i)
        session.commit()
        session.close()

        await update.message.reply_text("Ваша идея успешно добавлена! Переход в главное меню.",
                                        reply_markup=main_menu_markup(chat_id))
        context.user_data.pop("pending_idea", None)
        return MAIN_MENU

    elif response == "нет":
        await update.message.reply_text("Введите вашу идею заново:",
                                        reply_markup=ReplyKeyboardRemove())
        return SUBMIT_IDEA_TEXT

    else:
        await update.message.reply_text("Пожалуйста, ответьте 'Да' или 'Нет'.")
        return SUBMIT_IDEA_CONFIRMATION


# ------------------------------------------------------------------------------
# 5.2 "Мои идеи"
# ------------------------------------------------------------------------------


async def list_of_my_ideas(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """
    Списко всех идей, только конкретного пользователя
    """

    session = SessionLocal()
    chat_id = update.message.chat.id

    # Сборка идей
    answers_query = (
        session.query(Ideas)
        .filter(Ideas.chat_id == chat_id)
        .all()
    )

    all_ideas = []
    for ide in answers_query:
        all_ideas.append({
            "text_idea": ide.text_idea
        })

    reviews_str = 'Список всех ваших идей:\n'+"\n".join(f"{r['text_idea']}"
                                                        for r in all_ideas)

    await update.message.reply_text((reviews_str))

    return MAIN_MENU


async def idea_details(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """
    Показывает детали идеи по ID из клавиатуры.
    """
    idea_text = update.message.text.strip()

    try:
        idea_id = int(idea_text.split(":")[0])
    except (IndexError, ValueError):
        await update.message.reply_text("Пожалуйста, выберите идею из списка.")
        return MAIN_MENU

    session = SessionLocal()
    idea = session.query(Ideas).filter(Ideas.id == idea_id).first()
    session.close()

    if not idea:
        await update.message.reply_text("Идея не найдена.")
        return MAIN_MENU

    text = (
        f"*Идея:* {idea.text_idea}\n"
        f"*Дата:* {idea.created_at.strftime('%Y-%m-%d %H:%M')}\n"
        f"*Статус:* {idea.status or '—'}\n"
    )

    if idea.moderator_comment:
        text += f"*Комментарий модератора:* {idea.moderator_comment}"

    await update.message.reply_text(text, parse_mode="Markdown", reply_markup=main_menu_markup(update.message.chat.id))
    return MAIN_MENU


async def show_idea_choices(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """
    Показываем пользователю клавиатуру с его идеями.
    """
    session = SessionLocal()
    chat_id = update.message.chat.id

    user_ideas = session.query(Ideas).filter(Ideas.chat_id == chat_id).all()
    session.close()

    if not user_ideas:
        await update.message.reply_text("У вас пока нет идей.")
        return MAIN_MENU

    # Клавиатура: ID + первые 30 символов текста
    keyboard = [[f"{idea.id}: {idea.text_idea[:30]}"] for idea in user_ideas]

    await update.message.reply_text(
        "Выберите идею для просмотра статуса:",
        reply_markup=ReplyKeyboardMarkup(keyboard, resize_keyboard=True)
    )
    return IDEA_DETAILS_SELECT


# # ------------------------------------------------------------------------------
# # 5.3 "Admin"
# # ------------------------------------------------------------------------------

async def moderation_panel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """
    Панель админа
    """
    text = update.message.text.strip()

    if text == "Управление списком":
        await update.message.reply_text("Введите вашу идею:")
        return SUBMIT_IDEA_TEXT

    elif text == "Анализ":
        return await analytics_view(update, context)

    elif text == "Добавить admin":
        await update.message.reply_text("Введите Chat_id пользователя:")
        return ADD_AN_MODERATION

    elif text == "Главное меню":
        await update.message.reply_text("Возврат в главное меню, выберите действие.", reply_markup=main_menu_markup(update.message.chat.id))
        return MAIN_MENU

    elif text == "Список идей":
        return await moderation_list(update, context)

    elif text == "Оставить комментарий":
        return await moderation_comment(update, context)

    else:
        await update.message.reply_text("Неизвестная команда. Пожалуйста, выберите действие снова.", reply_markup=main_menu_markup(update.message.chat.id))
        return MAIN_MENU


async def add_moderation(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """
    Дать доступ пользователю к admin-панели
    """

    session = SessionLocal()
    chat_id = update.message.text.strip()
    user_name = update.get
    new_ad = AdminUser(user_name=user_name, user_chat_id=chat_id)
    session.add(new_ad)
    session.commit()


async def moderation_list(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """
    Список всех идей
    """
    session = SessionLocal()
    ideas = session.query(Ideas).order_by(Ideas.created_at.desc()).all()

    if not ideas:
        await update.message.reply_text("На данный момент у сотрудников нет предложений.")
        return MODERATION_PANEL

    text_list = "\n\n".join(
        [f"{i.id}. {i.text_idea} (от @{i.user_name})" for i in ideas])
    await update.message.reply_text(f"Список идей:\n{text_list}\n\nВведите номер идеи для модерации:")

    session.close()
    return MODERATION_DECISION


async def moderation_decision(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """
    Модерация идей
    """
    context.user_data["idea_id"] = update.message.text.strip()

    keyboard = [
        ["Одобрить", "Отклонить"],
        ["Отправить комментарий"]
    ]

    await update.message.reply_text(
        "Выберите действие для идеи:",
        reply_markup=ReplyKeyboardMarkup(
            keyboard, resize_keyboard=True, one_time_keyboard=True)
    )
    return MODERATION_COMMENT


async def moderation_comment(update: Update, context: ContextTypes.DEFAULT_TYPE):
    session = SessionLocal()
    text = update.message.text.strip()

    if context.user_data.get("awaiting_comment"):
        idea_id = context.user_data.get("idea_id")
        idea = session.query(Ideas).filter(Ideas.id == idea_id).first()

        if not idea:
            await update.message.reply_text("Идея не найдена.", reply_markup=ReplyKeyboardRemove())
            session.close()
            return MODERATION_PANEL

        idea.moderator_comment = text
        session.commit()
        session.close()

        context.user_data.pop("awaiting_comment", None)
        await update.message.reply_text("Комментарий сохранён.", reply_markup=ReplyKeyboardRemove())
        return MODERATION_PANEL

    idea_id = context.user_data.get("idea_id")
    idea = session.query(Ideas).filter(Ideas.id == idea_id).first()

    if not idea:
        await update.message.reply_text("Идея не найдена.", reply_markup=ReplyKeyboardRemove())
        session.close()
        return MODERATION_PANEL

    if text == "Отправить комментарий":
        context.user_data["awaiting_comment"] = True
        await update.message.reply_text("Введите свой комментарий:", reply_markup=ReplyKeyboardRemove())
        session.close()
        return MODERATION_COMMENT

    elif text in ["Одобрить", "Отклонить"]:
        comment = "Одобрена." if text == "Одобрить" else "Отклонена."

        author_chat_id = idea.chat_id
        idea_text = idea.text_idea

        try:
            await context.bot.send_message(
                chat_id=author_chat_id,
                text=f"Ваша идея:\n\n\"{idea_text}\"\n\nбыла {comment.lower()} модератором."
            )
        except Exception as e:
            print(
                f"[ERROR] Не удалось отправить сообщение пользователю {author_chat_id}: {e}")

        session.delete(idea)
        session.commit()
        session.close()

        await update.message.reply_text(
            f"Идея была {comment.lower()} и автор уведомлён.",
            reply_markup=ReplyKeyboardRemove()
        )
        return MODERATION_PANEL

    else:
        await update.message.reply_text("Пожалуйста, выберите действие из меню.")
        session.close()
        return MODERATION_COMMENT


async def analytics_view(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Пожалуйста, подождите, анализ идей выполняется...")

    result = analyze_all_ideas_with_yandex_gpt()

    if "error" in result:
        await update.message.reply_text(result["error"])
        return MODERATION_PANEL

    try:
        text = result["result"]["alternatives"][0]["message"]["text"]
    except Exception as e:
        await update.message.reply_text(f"Ошибка разбора ответа от YandexGPT: {e}")
        return MODERATION_PANEL

    session = SessionLocal()
    log = AnalysisLog(analysis_result=text)
    session.add(log)
    session.commit()
    session.close()

    if len(text) > 4000:
        for i in range(0, len(text), 4000):
            await update.message.reply_text(text[i:i + 4000])
    else:
        await update.message.reply_text(text)

    return MODERATION_PANEL



# ################################################################################
# # 6. СБОРКА И ЗАПУСК (APPLICATION)
# ################################################################################

def main():
    persistence = PicklePersistence(filepath="bot_data.pkl")
    application = (
        ApplicationBuilder()
        .token(TELEGRAM_BOT_TOKEN)
        .persistence(persistence)
        .build()
    )

    conv_handler = ConversationHandler(
        entry_points=[CommandHandler("start", start_command)],
        states={
            MAIN_MENU: [
                MessageHandler(filters.TEXT & ~filters.COMMAND,
                               handle_role_choice)
            ],

            # 6.1: "Идеи"
            SUBMIT_IDEA_TEXT: [
                MessageHandler(filters.TEXT & ~filters.COMMAND,
                               receive_new_ideas),
            ],

            SUBMIT_IDEA_CONFIRMATION: [
                MessageHandler(filters.TEXT & ~filters.COMMAND,
                               confirm_idea_submission)
            ],

            # 6.2: "Список идей"

            MY_IDEAS_LIST: [
                MessageHandler(filters.TEXT & ~filters.COMMAND,
                               list_of_my_ideas),
            ],
            IDEA_DETAILS: [
                MessageHandler(filters.TEXT & ~filters.COMMAND, idea_details),
            ],
            IDEA_DETAILS_SELECT: [
                MessageHandler(filters.TEXT & ~filters.COMMAND, idea_details)
            ],

            # 6.3: "Admin"

            MODERATION_PANEL: [
                MessageHandler(filters.TEXT & ~filters.COMMAND,
                               moderation_panel),
            ],
            ADD_AN_MODERATION: [
                # здесь проверяем код доступа
                MessageHandler(filters.TEXT & ~filters.COMMAND,
                               add_moderation),
            ],
            MODERATION_LIST: [
                MessageHandler(filters.TEXT & ~filters.COMMAND,
                               moderation_list),
            ],
            MODERATION_DECISION: [
                MessageHandler(filters.TEXT & ~filters.COMMAND,
                               moderation_decision),
            ],
            MODERATION_COMMENT: [
                MessageHandler(filters.TEXT & ~filters.COMMAND,
                               moderation_comment),
            ],
            ANALYTICS_VIEW: [
                MessageHandler(filters.TEXT & ~filters.COMMAND,
                               analytics_view),
            ],
        },

        fallbacks=[CommandHandler("start", start_command)],
        allow_reentry=True,
    )

    application.add_handler(conv_handler)

    # Запуск бота
    application.run_polling()


if __name__ == "__main__":
    main()
