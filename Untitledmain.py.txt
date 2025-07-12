
import os
import time
import requests
from datetime import datetime

# Telegram Setup - Get secrets from environment
TELEGRAM_BOT_TOKEN = os.environ.get("BOT_TOKEN")
TELEGRAM_CHAT_ID = os.environ.get("CHAT_ID")

# Function to send Telegram message
def send_telegram_message(message):
    url = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"
    data = {
        "chat_id": TELEGRAM_CHAT_ID,
        "text": message
    }
    try:
        response = requests.post(url, data=data)
        return response.status_code == 200
    except Exception as e:
        print(f"Error sending Telegram message: {e}")
        return False

# Send confirmation message
try:
    if send_telegram_message("✅ Rsdscanner_bot is online and connected to Telegram."):
        print("Confirmation message sent to Telegram.")
    else:
        print("Failed to send confirmation message.")
except Exception as e:
    print(f"Failed to send confirmation message. Error: {e}")

# Binance Futures API endpoint
BINANCE_FUTURES_ENDPOINT = "https://fapi.binance.com/fapi/v1/klines"


# Get all futures symbols
def get_futures_symbols():
    exchange_info = requests.get(
        "https://fapi.binance.com/fapi/v1/exchangeInfo").json()
    return [
        symbol["symbol"] for symbol in exchange_info["symbols"]
        if symbol["contractType"] == "PERPETUAL"
    ]


# Check if a candle has at least 60% body
def is_valid_candle(candle):
    open_price = float(candle[1])
    high = float(candle[2])
    low = float(candle[3])
    close = float(candle[4])
    body = abs(close - open_price)
    full_range = high - low
    return full_range > 0 and (body / full_range) >= 0.6


# Check for matching pattern
def match_pattern(prev, curr):
    prev_open = float(prev[1])
    prev_close = float(prev[4])
    curr_open = float(curr[1])
    curr_close = float(curr[4])

    # Check for opposite candles and valid body size
    if not is_valid_candle(prev) or not is_valid_candle(curr):
        return False

    prev_green = prev_close > prev_open
    curr_red = curr_close < curr_open
    curr_green = curr_close > curr_open
    prev_red = prev_close < prev_open

    if (prev_green and curr_red) or (prev_red and curr_green):
        # Check for nearly equal body sizes (difference ≤ 15%)
        prev_body = abs(prev_close - prev_open)
        curr_body = abs(curr_close - curr_open)
        return abs(prev_body - curr_body) / max(prev_body, curr_body) <= 0.15

    return False


# Scan symbols
symbols = get_futures_symbols()
print(f"[INFO] Monitoring {len(symbols)} Binance Futures coins...")

while True:
    print(f"[{datetime.utcnow().strftime('%Y-%m-%d %H:%M:%S')}] Scanning...")
    for symbol in symbols:
        try:
            params = {"symbol": symbol, "interval": "3h", "limit": 3}
            response = requests.get(BINANCE_FUTURES_ENDPOINT, params=params)
            data = response.json()

            if len(data) >= 3 and match_pattern(data[-3], data[-2]):
                direction = "🔴⬅️🟢" if float(data[-2][4]) > float(
                    data[-2][1]) else "🟢➡️🔴"
                alert = f"🔔 Pattern match on {symbol}\nTime: {datetime.utcnow().strftime('%Y-%m-%d %H:%M UTC')}\nPattern: {direction}"
                send_telegram_message(alert)

        except Exception as e:
            print(f"[ERROR] {symbol}: {str(e)}")

    time.sleep(600)  # Run every 10 minutes
