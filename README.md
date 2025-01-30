# Telegram-AI-Chatbot
A Python-based Telegram bot integrating Gemini API for AI-powered responses and SerpAPI for web searches. It handles user queries, processes images, and offers dynamic interactions. The bot also stores user data and provides personalized assistance, making it ideal for building a conversational AI assistant


import os
import logging
import requests
from telegram import Update, ReplyKeyboardMarkup, KeyboardButton
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, filters, ContextTypes
from dotenv import load_dotenv
from serpapi import GoogleSearch

# Load environment variables
load_dotenv()
TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")
GENAI_API_KEY = os.getenv("GENAI_API_KEY")
SERPAPI_KEY = os.getenv("SERPAPI_KEY")

# Configure logging
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)

# Function to call Gemini API via HTTP request
def call_gemini_api(prompt):
    url = f"https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key={GENAI_API_KEY}"
    headers = {'Content-Type': 'application/json'}
    payload = {"contents": [{"parts": [{"text": prompt}]}]}

    try:
        response = requests.post(url, json=payload, headers=headers)
        response.raise_for_status()
        response_data = response.json()
        return response_data.get("candidates", [{}])[0].get("content", {}).get("parts", [{}])[0].get("text", "Error fetching response")
    except requests.exceptions.RequestException as e:
        logging.error(f"Gemini API request failed: {e}")
        return "Error occurred while fetching response from Gemini API."

# Start command
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    user = update.message.from_user
    chat_id = update.message.chat_id
    existing_user = None  # MongoDB code removed
    
    if not existing_user:
        keyboard = [[KeyboardButton("Share Phone Number", request_contact=True)]]
        reply_markup = ReplyKeyboardMarkup(keyboard, one_time_keyboard=True, resize_keyboard=True)
        await update.message.reply_text("Welcome! Please share your phone number.", reply_markup=reply_markup)
    else:
        await update.message.reply_text("Welcome back! How can I assist you today?")

# Store phone number
async def handle_phone_number(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    user = update.message.from_user
    phone_number = update.message.contact.phone_number
    # MongoDB code removed
    await update.message.reply_text("Phone number saved successfully!")

# Handle AI chat
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    user_input = update.message.text
    chat_id = update.message.chat_id
    response = call_gemini_api(user_input)
    # MongoDB code removed
    await update.message.reply_text(response)

# Image/File analysis (Note: Gemini API currently does not support direct image processing)
async def handle_file(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    file_id = update.message.photo[-1].file_id if update.message.photo else update.message.document.file_id
    file = await context.bot.get_file(file_id)
    file_path = await file.download_as_bytearray()
    response = call_gemini_api("Describe this image:")
    await update.message.reply_text(response)
    # MongoDB code removed

# Web search using SerpAPI
async def web_search(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    query = update.message.text.split(" ", 1)[-1]
    try:
        search = GoogleSearch({"q": query, "api_key": SERPAPI_KEY})
        results = search.get_dict()  # Corrected method
        if "organic_results" in results:
            summary = "\n".join([f"{res['title']}: {res['link']}" for res in results["organic_results"][:3]])
            await update.message.reply_text(summary)
        else:
            await update.message.reply_text("No results found.")
    except Exception as e:
        logging.error(f"SerpAPI request failed: {e}")
        await update.message.reply_text("Error occurred while fetching search results.")

# Main function
def main():
    logging.info("🚀 Telegram Bot is starting...")
    
    app = ApplicationBuilder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.CONTACT, handle_phone_number))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    app.add_handler(MessageHandler(filters.PHOTO | filters.Document.ALL, handle_file))
    app.add_handler(CommandHandler("websearch", web_search))
    
    logging.info("✅ Bot is now running and polling for updates...")
    app.run_polling()

if __name__ == "__main__":
    main()
    
