# bot-hosting
import os
import re
import time
import asyncio
import threading
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.common.exceptions import WebDriverException
from dotenv import load_dotenv
from telegram import Update, InputFile
from telegram.ext import Application, CommandHandler, ContextTypes


# Load environment variables
load_dotenv()

IG_LOGIN_USERNAME = os.getenv("IG_LOGIN_USERNAME")
IG_LOGIN_PASSWORD = os.getenv("IG_LOGIN_PASSWORD")
TELEGRAM_BOT_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")
IMG_BANNED = "./banned.jpg"
IMG_BACK = "./back.jpg"

watchlist = {}  # {user_id: {"username": str, "mode": str, "running": bool, "start_time": float}}


def is_valid_instagram_username(username):
    if not (1 <= len(username) <= 30):
        return False
    if not re.match(r"^(?!.*\.\.)(?!.*\.$)[a-zA-Z0-9._]+$", username):
        return False
    return True


def instagram_login(driver):
    try:
        driver.get("https://www.instagram.com/accounts/login/")
        time.sleep(4)
        username_input = driver.find_element(By.NAME, "username")
        username_input.send_keys(IG_LOGIN_USERNAME)
        password_input = driver.find_element(By.NAME, "password")
        password_input.send_keys(IG_LOGIN_PASSWORD)
        password_input.submit()
        time.sleep(5)
    except Exception as e:
        print(f"[Login Error] {e}")


def is_instagram_active(driver, username):
    try:
        url = f"https://instagram.com/{username}"
        driver.get(url)
        time.sleep(3)
        page_source = driver.page_source
        if "Sorry, this page isn't available." in page_source:
            return False
        return True
    except WebDriverException as e:
        print(f"[WebDriver Error] {e}")
        return False


def precheck_instagram_status(username):
    options = Options()
    options.add_argument("--disable-gpu")
    options.add_argument("--no-sandbox")
    options.add_argument("--headless")
    options.add_argument("--disable-dev-shm-usage")
    options.add_argument("--window-size=1920,1080")
    driver = webdriver.Chrome(options=options)
    try:
        return is_instagram_active(driver, username)
    finally:
        driver.quit()


def build_message(username, duration_seconds, status):
    hours, rem = divmod(duration_seconds, 3600)
    minutes, seconds = divmod(rem, 60)
    duration_str = f"{int(hours)} hours, {int(minutes)} minutes, {int(seconds)} seconds"
    
    if status == "unban":
        title = "🎉 UNBAN DONE"
        description = f"User @{username} is back."
        image_file = IMG_BACK
    else:
        title = "🚫 BAN DONE"
        description = f"User @{username} has been vanished successfully."
        image_file = IMG_BACK
    
    message = f"{title}\n\n{description}\n\n⏱ Time taken: {duration_str}"
    return message, image_file


def monitor_with_browser(username, mode, user_id, chat_id, application, watcher_data):
    options = Options()
    options.add_argument("--disable-gpu")
    options.add_argument("--no-sandbox")
    options.add_argument("--headless")
    options.add_argument("--disable-dev-shm-usage")
    options.add_argument("--window-size=1920,1080")
    driver = webdriver.Chrome(options=options)
    try:
        instagram_login(driver)
        prev_active = is_instagram_active(driver, username)
        monitoring_start_time = watcher_data.get("start_time", time.time())
        while watchlist.get(user_id, {}).get("running"):
            cur_active = is_instagram_active(driver, username)
            if mode == "ban" and prev_active and not cur_active:
                duration = int(time.time() - monitoring_start_time)
                message, image_file = build_message(username, duration, "ban")
                
                async def send_notification():
                    try:
                        with open(image_file, 'rb') as photo:
                            await application.bot.send_photo(
                                chat_id=chat_id,
                                photo=InputFile(photo),
                                caption=message
                            )
                    except Exception as e:
                        print(f"Error sending notification: {e}")
                
                # Run the async function in the event loop
                future = asyncio.run_coroutine_threadsafe(send_notification(), application._loop)
                future.result()  # Wait for completion
                
                watchlist[user_id]["running"] = False
                break
            elif mode == "unban" and not prev_active and cur_active:
                duration = int(time.time() - monitoring_start_time)
                message, image_file = build_message(username, duration, "unban")
                
                async def send_notification():
                    try:
                        with open(image_file, 'rb') as photo:
                            await application.bot.send_photo(
                                chat_id=chat_id,
                                photo=InputFile(photo),
                                caption=message
                            )
                    except Exception as e:
                        print(f"Error sending notification: {e}")
                
                future = asyncio.run_coroutine_threadsafe(send_notification(), application._loop)
                future.result()
                
                watchlist[user_id]["running"] = False
                break
            prev_active = cur_active
            time.sleep(2)
    finally:
        driver.quit()


async def start_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "👁 Instagram Monitor Bot\n\n"
        "Available commands:\n"
        "/eyeban <username> - Notify if user gets banned\n"
        "/eyeunban <username> - Notify if user returns\n"
        "/cancel - Stop monitoring\n"
        "/help - Show help"
    )


async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "📖 **Commands:**\n\n"
        "`/eyeban <username>` - Notify if the user gets banned\n"
        "`/eyeunban <username>` - Notify if the user returns\n"
        "`/cancel` - Stop monitoring\n"
        "`/help` - Show this help message"
    )


async def eyeban_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    chat_id = update.effective_chat.id
    
    if not context.args:
        await update.message.reply_text("❗ Usage: /eyeban <instagram_username>")
        return
    
    username = ' '.join(context.args).strip('@ ').lower()
    
    if not is_valid_instagram_username(username):
        await update.message.reply_text("❌ Invalid Instagram username.")
        return
    
    checking_msg = await update.message.reply_text("🔎 Checking username status...")
    is_active = precheck_instagram_status(username)
    
    if not is_active:
        await checking_msg.edit_text(f"⚠️ @{username} is already banned.")
        return
    
    await checking_msg.edit_text(f"👁 Watching @{username} for a ban...")
    
    watchlist[user_id] = {
        "username": username,
        "mode": "ban",
        "running": True,
        "start_time": time.time(),
        "chat_id": chat_id
    }
    
    # Start monitoring in a separate thread
    t = threading.Thread(
        target=monitor_with_browser,
        args=(username, "ban", user_id, chat_id, context.application, watchlist[user_id]),
        daemon=True
    )
    t.start()


async def eyeunban_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    chat_id = update.effective_chat.id
    
    if not context.args:
        await update.message.reply_text("❗ Usage: /eyeunban <instagram_username>")
        return
    
    username = ' '.join(context.args).strip('@ ').lower()
    
    if not is_valid_instagram_username(username):
        await update.message.reply_text("❌ Invalid Instagram username.")
        return
    
    checking_msg = await update.message.reply_text("🔎 Checking username status...")
    is_active = precheck_instagram_status(username)
    
    if is_active:
        await checking_msg.edit_text(f"⚠️ @{username} is already active.")
        return
    
    await checking_msg.edit_text(f"👁 Watching @{username} for a return...")
    
    watchlist[user_id] = {
        "username": username,
        "mode": "unban",
        "running": True,
        "start_time": time.time(),
        "chat_id": chat_id
    }
    
    t = threading.Thread(
        target=monitor_with_browser,
        args=(username, "unban", user_id, chat_id, context.application, watchlist[user_id]),
        daemon=True
    )
    t.start()


async def cancel_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    
    if user_id in watchlist and watchlist[user_id]["running"]:
        watchlist[user_id]["running"] = False
        await update.message.reply_text("❎ Monitoring canceled.")
    else:
        await update.message.reply_text("ℹ️ No active monitoring to cancel.")


def main():
    # Create the Application
    application = Application.builder().token(TELEGRAM_BOT_TOKEN).build()

    # Add handlers
    application.add_handler(CommandHandler("start", start_command))
    application.add_handler(CommandHandler("help", help_command))
    application.add_handler(CommandHandler("eyeban", eyeban_command))
    application.add_handler(CommandHandler("eyeunban", eyeunban_command))
    application.add_handler(CommandHandler("cancel", cancel_command))

    # Start the Bot
    print("✅ Telegram Bot is running...")
    application.run_polling()


if __name__ == "__main__":
    main()
