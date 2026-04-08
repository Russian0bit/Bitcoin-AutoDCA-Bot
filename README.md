#Cryptobotan Bitcoin AutoDCA Bot

Telegram-бот для автоматической DCA-покупки BTC через FixedFloat.

Работает локально на вашем устройстве (Windows / macOS / Linux).

⚡ Возможности

Покупка BTC по расписанию (день / неделя / месяц)
Работает в 2 режимах:
- Manual — вы отправляете USDT вручную
- Auto — бот отправляет USDT сам

Поддержка сетей: Arbitrum, BSC, Polygon

После перезапуска все планы сохраняются

Для работы 24/7: используйте собственный сервер или настройте автозапуск при включении компьютера.

❗ Облачные сервера использовать не рекомендуется (есть риск компрометации ключей, токенов к ботам, API-ключей )

## 🚀 Быстрый старт

# 1) Скачайте бота
```git clone https://github.com/Russian0bit/Bitcoin-AutoDCA-Bot.git```

Перейдите в папку с ботом

```cd Bitcoin-AutoDCA-Bot```

# Установка Python (если не установлен)
Проверьте, установлен ли Python:

```python3 --version```

Если команда не найдена — установите Python одним из способов ниже:

🖥 macOS

```brew install python```

🐧 Linux (Ubuntu / Debian)

```sudo apt update && sudo apt install -y python3 python3-venv python3-pip```

🪟 Windows

Вариант 1 (простой):

Скачать с официального сайта и установить

```https://www.python.org/downloads/```

Обязательно поставить галочку "Add Python to PATH"

Вариант 2 (через терминал):

```winget install Python.Python.3```

# 2) Виртуальные окружение

macOS / Linux:

```python3 -m venv venv && source venv/bin/activate```

Windows (PowerShell):

```python -m venv venv && venv\Scripts\activate```

# 3) Установка зависимостей

```pip install -r requirements.txt```

Если возникает ошибка — попробуйте:

```pip3 install -r requirements.txt```

# 4) Настройка .env

Создайте текстовый файл .env

DCA_TELEGRAM_BOT_TOKEN=your_telegram_bot_token

ADMIN_USER_ID=123456789

FF_API_KEY=your_fixedfloat_api_key

FF_API_SECRET=your_fixedfloat_api_secret

Где взять:

Bot Token → @BotFather (/newbot)

User ID → @my_id_bot

API ключи → https://ff.io

5) Настройка кошелька (для Auto режима)

Создайте файл Wallet.json с содержимым

{
  "private_key": "0xYOUR_PRIVATE_KEY",
  "password": "YOUR_PASSWORD"
}

# 6) Запуск:

Mac / Linux:
```python3 bot.py```

Windows:
```python bot.py```

# 7) Инициализация

В Telegram:
/setwallet

После этого создастся keystore и пароль сохранится.
УДАЛИТЕ файл wallet.json

Поздравляю ваш DCA-бот для автопокупки BTC настроен. 


# 🧠 Пример стратегии
/setdca USDT-ARB 50 24 bc1q...
50$ каждые 24 часа
сеть Arbitrum
BTC на ваш адрес

# 🔒 Безопасность
приватный ключ шифруется (keystore)
пароль хранится в OS keyring
.env и wallet.json не коммитить

# 🧭 Roadmap / улучшения
улучшение UX (интерфейс как приложение)
гибкие стратегии (время, dip-buy)
уведомления о падении BTC
поддержка ETH / SOL / LightningBTC
больше сетей и бирж
Добавление множества BTC-адресов для получения

## 📄 Лицензия

MIT License

## ⚠️ Отказ от ответственности

Проект предоставляется «как есть». Используйте на свой риск.
