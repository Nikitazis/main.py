import telebot
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton
import threading
import time
import random

# Токен вашего бота
TOKEN = "7585523393:AAGSY2Y2FP6kyxyAD_Gytb7tJ3OxmFjrt2w"

bot = telebot.TeleBot(TOKEN)

bot.remove_webhook()
print("Webhook удален")

# Переменные для хранения данных и состояния
user_data = {}
is_publishing = {}  # Хранит состояние публикации для каждого пользователя (True/False)
last_message_id = {}  # Хранит ID последнего сообщения для каждого канала



# Стартовая команда
@bot.message_handler(commands=['start'])
def start(message):
    chat_id = message.chat.id
    welcome_text = (
        "👋 Добро пожаловать! Вы можете настроить бота для публикации сообщений.\n\n"
        "📥 Укажите канал с шаблонами.\n"
        "📤 Укажите канал для публикации.\n"
        "🔗 Укажите ссылку для замены.\n"
        "⏱ Задайте интервал публикации.\n"
        "⏹ Остановите публикацию."
    )
    markup = InlineKeyboardMarkup(row_width=2)
    markup.add(
        InlineKeyboardButton("📥 Указать источник", callback_data="set_source"),
        InlineKeyboardButton("📤 Указать цель", callback_data="set_target"),
        InlineKeyboardButton("⏹ Остановить", callback_data="stop")
    )
    bot.send_message(chat_id, welcome_text, reply_markup=markup)

@bot.callback_query_handler(func=lambda call: True)
def handle_callback(call):
    chat_id = call.message.chat.id
    if call.data == "set_source":
        bot.send_message(chat_id, "Отправьте ссылку на канал с шаблонами (например, @source_channel).")
        user_data[chat_id] = {"state": "waiting_for_source"}
    elif call.data == "set_target":
        bot.send_message(chat_id, "Отправьте ссылку на канал для публикации (например, @target_channel).")
        user_data[chat_id]["state"] = "waiting_for_target"
    elif call.data == "stop":
        is_publishing[chat_id] = False
        bot.send_message(chat_id, "⏹ Публикация остановлена.")

@bot.message_handler(func=lambda message: True)
def handle_message(message):
    chat_id = message.chat.id
    if chat_id not in user_data:
        bot.send_message(chat_id, "Сначала введите /start для начала.")
        return

    state = user_data[chat_id].get("state")
    if state == "waiting_for_source":
        user_data[chat_id]["source"] = message.text
        user_data[chat_id]["state"] = "waiting_for_target"
        bot.send_message(chat_id, "Источник сохранён! Теперь отправьте ссылку на канал для публикации.")
    elif state == "waiting_for_target":
        user_data[chat_id]["target"] = message.text
        user_data[chat_id]["state"] = "waiting_for_link"
        bot.send_message(chat_id, "Целевой канал сохранён! Теперь отправьте ссылку, которую нужно подставлять.")
    elif state == "waiting_for_link":
        user_data[chat_id]["link"] = message.text
        user_data[chat_id]["state"] = "waiting_for_interval"
        bot.send_message(chat_id, "Ссылка сохранена! Теперь укажите интервал публикации (в секундах).")
    elif state == "waiting_for_interval":
        try:
            interval = int(message.text)
            if interval <= 0:
                raise ValueError
            user_data[chat_id]["interval"] = interval
            is_publishing[chat_id] = True
            bot.send_message(chat_id, "Настройка завершена! Публикация начнётся.")
            threading.Thread(target=publish_messages, args=(chat_id,)).start()
        except ValueError:
            bot.send_message(chat_id, "Интервал должен быть положительным числом.")
    else:
        bot.send_message(chat_id, "Неизвестное состояние. Начните с команды /start.")

# Функция получения шаблонов из канала
def get_templates_from_channel(source_channel):
    # Получение новых сообщений из канала
    updates = bot.get_updates()
    messages = []
    for update in updates:
        if update.message and update.message.chat.username == source_channel.lstrip("@"):
            messages.append(update.message.text)
    return messages

# Функция публикации сообщений
def publish_messages(chat_id):
    while is_publishing.get(chat_id, False):
        try:
            source = user_data[chat_id]["source"]
            target = user_data[chat_id]["target"]
            link = user_data[chat_id]["link"]
            interval = user_data[chat_id]["interval"]

            # Получаем шаблоны из указанного канала
            templates = get_templates_from_channel(source)

            if not templates:
                bot.send_message(chat_id, "❌ Не удалось найти сообщения в канале с шаблонами.")
                break

            # Берём случайный шаблон
            random_template = random.choice(templates)

            # Подставляем ссылку
            updated_message = random_template.replace("{link}", link)

            # Удаляем предыдущее сообщение, если оно было отправлено
            if target in last_message_id:
                try:
                    bot.delete_message(target, last_message_id[target])
                except:
                    pass

            # Отправляем сообщение
            sent_message = bot.send_message(target, updated_message)
            last_message_id[target] = sent_message.message_id

            time.sleep(interval)
        except Exception as e:
            bot.send_message(chat_id, f"Ошибка: {e}")
            break

bot.polling(none_stop=True, interval=1, timeout=20, threaded=False)
