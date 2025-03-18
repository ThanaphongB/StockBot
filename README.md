import logging
import yfinance as yf
from telegram import Update
from telegram.ext import Application, CommandHandler, ContextTypes
from datetime import datetime

# ตั้งค่า API Token ของบอท
TOKEN = "7561057674:AAHah1j41nU6AId9S5fk1f9WZDiE-1-BQ-k"

# เปิดระบบ Log
logging.basicConfig(format="%(asctime)s - %(levelname)s - %(message)s", level=logging.INFO)

# เก็บข้อมูลของราคาหุ้นที่ต้องการแจ้งเตือน
price_alerts = {}

# ฟังก์ชันดึงราคาหุ้น
def get_stock_price(symbol):
    try:
        stock = yf.Ticker(symbol)
        price = stock.history(period="1d")["Close"].iloc[-1]
        return price
    except Exception as e:
        return f"Error: {e}"

# ฟังก์ชัน /start
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text("สวัสดี! อยากทราบหุ้นตัวไหนครับ ส่ง /price ตามด้วยชื่อหุ้น เช่น /price AAPL หรือจะตั้งเตือนราคาหุ้นได้ด้วย /setalert AAPL 150")

# ฟังก์ชัน /price ดูราคาหุ้น
async def stock_price(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if len(context.args) == 0:
        await update.message.reply_text("กรุณาใส่ชื่อหุ้น เช่น /price AAPL")
        return
    symbol = context.args[0].upper()
    price = get_stock_price(symbol)
    await update.message.reply_text(f"ราคาหุ้น {symbol} ล่าสุด: ${price:.2f}")

# ฟังก์ชัน /setalert ตั้งค่าการแจ้งเตือนราคาหุ้น
async def set_alert(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if len(context.args) != 2:
        await update.message.reply_text("กรุณาใส่ชื่อหุ้นและราคาที่ต้องการแจ้งเตือน เช่น /setalert AAPL 150")
        return
    
    symbol = context.args[0].upper()
    try:
        target_price = float(context.args[1])
    except ValueError:
        await update.message.reply_text("กรุณาใส่ราคาหุ้นเป็นตัวเลข เช่น /setalert AAPL 150")
        return

    # เก็บข้อมูลการแจ้งเตือน
    price_alerts[symbol] = {
        'target_price': target_price,
        'user': update.message.chat_id
    }
    
    await update.message.reply_text(f"ตั้งค่าการแจ้งเตือนราคาหุ้น {symbol} เมื่อราคาถึง {target_price} เรียบร้อยแล้ว!")

# ฟังก์ชันสำหรับตรวจสอบและแจ้งเตือนราคา
async def check_alerts(context: ContextTypes.DEFAULT_TYPE) -> None:
    for symbol, alert in price_alerts.items():
        current_price = get_stock_price(symbol)
        
        if isinstance(current_price, float) and current_price >= alert['target_price']:
            # ส่งข้อความแจ้งเตือน
            await context.bot.send_message(alert['user'], f"แจ้งเตือน: ราคาหุ้น {symbol} ถึง {current_price:.2f} ซึ่งสูงกว่าหรือเท่ากับที่คุณตั้งไว้ ({alert['target_price']})")
            # ลบการแจ้งเตือนหลังจากที่ราคาถึงแล้ว
            del price_alerts[symbol]

# ตั้งค่าบอท
def main():
    # ใช้ Application แทน Updater
    application = Application.builder().token(TOKEN).build()

    # เพิ่ม handler สำหรับคำสั่ง /start, /price และ /setalert
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("price", stock_price))
    application.add_handler(CommandHandler("setalert", set_alert))

    # สร้าง JobQueue สำหรับการตรวจสอบราคาหุ้น
    job_queue = application.job_queue
    if job_queue:  # ตรวจสอบว่า job_queue ถูกสร้างหรือไม่
        # ตั้งเวลาให้ตรวจสอบราคาหุ้นทุกๆ 60 วินาที
        job_queue.run_repeating(check_alerts, interval=60, first=0)
    else:
        logging.error("JobQueue is not available.")

    # เริ่มต้นการดึงข้อความและตอบกลับ
    application.run_polling()

if __name__ == "__main__":
    main()
