import os
import time
import requests
import hmac
import hashlib
from datetime import datetime

# ── Config ───────────────────────────────────────────────────────────────────
TELEGRAM_TOKEN   = os.environ.get("TELEGRAM_TOKEN")
CHAT_ID          = os.environ.get("CHAT_ID")
BYBIT_API_KEY    = os.environ.get("BYBIT_API_KEY")
BYBIT_API_SECRET = os.environ.get("BYBIT_API_SECRET")
TESTNET          = os.environ.get("TESTNET", "true").lower() == "true"
LEVERAGE         = int(os.environ.get("LEVERAGE", "2"))

SYMBOL           = "BTCUSDT"
INTERVAL         = "60"
CHECK_INTERVAL   = 300
MIN_SIGNALS      = 4        # confluencia 4/7
RISK_PER_TRADE   = 0.05
TRAILING_PCT     = 0.0015
ATR_MULTIPLIER   = 1.5      # stop loss = ATR x 1.5

BASE_URL = "https://api-testnet.bybit.com" if TESTNET else "https://api.bybit.com"

# ── Telegram ──────────────────────────────────────────────────────────────────
def send_telegram(message):
    url = f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendMessage"
    try:
        requests.post(url, json={"chat_id": CHAT_ID, "text": message, "parse_mode": "HTML"}, timeout=10)
    except Exception as e:
        print(f"Telegram error: {e}")

# ── Bybit helpers ─────────────────────────────────────────────────────────────
def bybit_sign(params):
    ts = str(int(time.time() * 1000))
    params["api_key"]   = BYBIT_API_KEY
    params["timestamp"] = ts
    sorted_params = "&".join(f"{k}={v}" for k, v in sorted(params.items()))
    params["sign"] = hmac.new(BYBIT_API_SECRET.encode(), sorted_params.encode(), hashlib.sha256).hexdigest()
    return params

def get_klines(limit=200):
    url = f"{BASE_URL}/v5/market/kline"
    params = {"category": "linear", "symbol": SYMBOL, "interval": INTERVAL, "limit": limit}
    r = requests.get(url, params=params, timeout=10)
    data = r.json()
    if data["retCode"] != 0:
        return []
    return data["result"]["list"]

def get_balance():
    params = bybit_sign({"accountType": "UNIFIED"})
    url = f"{BASE_URL}/v5/account/wallet-balance"
    r = requests.get(url, params=params, timeout=10)
    data = r.json()
    try:
        return float(data["result"]["list"][0]["totalEquity"])
    except:
        return 0.0

def set_leverage():
    params = bybit_sign({
        "category": "linear",
        "symbol": SYMBOL,
        "buyLeverage": str(LEVERAGE),
        "sellLeverage": str(LEVERAGE),
    })
    url = f"{BASE_URL}/v5/position/set-leverage"
    requests.post(url, json=params, timeout=10)

def place_order(side, qty, stop_loss_price):
    params = bybit_sign({
        "category": "linear",
        "symbol": SYMBOL,
        "side": side,
        "orderType": "Market",
        "qty": str(qty),
        "stopLoss": str(round(stop_loss_price, 1)),
        "slTriggerBy": "MarkPrice",
    })
    url = f"{BASE_URL}/v5/order/create"
    r = requests.post(url, json=params, timeout=10)
    return r.json()

def close_position():
    params = bybit_sign({
        "category": "linear",
        "symbol": SYMBOL,
        "side": "Sell",
        "orderType": "Market",
        "qty": "0",
        "reduceOnly": "true",
    })
    url = f"{BASE_URL}/v5/order/create"
    r = requests.post(url, json=params, timeout=10)
    return r.json()

# ── Indicadores ───────────────────────────────────────────────────────────────
def ema(prices, period):
    k = 2 / (period + 1)
    result = [sum(prices[:period]) / period]
    for p in prices[period:]:
        result.append(p * k + result[-1] * (1 - k))
    return result

def rsi(prices, period=14):
    gains, losses = [], []
    for i in range(1, len(prices)):
        diff = prices[i] - prices[i-1]
        gains.append(max(diff, 0))
        losses.append(max(-diff, 0))
    avg_gain = sum(gains[:period]) / period
    avg_loss = sum(losses[:period]) / period
    for i in range(period, len(gains)):
        avg_gain = (avg_gain * (period-1) + gains[i]) / period
        avg_loss = (avg_loss * (period-1) + losses[i]) / period
    if avg_loss == 0:
        return 100
    return 100 - (100 / (1 + avg_gain / avg_loss))

def macd(prices, fast=12, slow=26, signal=9):
    ema_fast = ema(prices, fast)
    ema_slow = ema(prices, slow)
    min_len = min(len(ema_fast), len(ema_slow))
    macd_line = [ema_fast[-min_len+i] - ema_slow[-min_len+i] for i in range(min_len)]
    signal_line = ema(macd_line, signal)
    return macd_line[-1], signal_line[-1], macd_line[-1] - signal_line[-1]

def adx(highs, lows, closes, period=14):
    tr_list, plus_dm, minus_dm = [], [], []
    for i in range(1, len(closes)):
        tr = max(highs[i]-lows[i], abs(highs[i]-closes[i-1]), abs(lows[i]-closes[i-1]))
        tr_list.append(tr)
        plus_dm.append(max(highs[i]-highs[i-1], 0) if highs[i]-highs[i-1] > lows[i-1]-lows[i] else 0)
        minus_dm.append(max(lows[i-1]-lows[i], 0) if lows[i-1]-lows[i] > highs[i]-highs[i-1] else 0)
    atr = sum(tr_list[:period]) / period
    pdi = sum(plus_dm[:period]) / period
    mdi = sum(minus_dm[:period]) / period
    for i in range(period, len(tr_list)):
        atr = (atr*(period-1) + tr_list[i]) / period
        pdi = (pdi*(period-1) + plus_dm[i]) / period
        mdi = (mdi*(period-1) + minus_dm[i]) / period
    pdi_pct = 100 * pdi / atr if atr else 0
    mdi_pct = 100 * mdi / atr if atr else 0
    dx = 100 * abs(pdi_pct - mdi_pct) / (pdi_pct + mdi_pct) if (pdi_pct + mdi_pct) else 0
    return dx, pdi_pct, mdi_pct

def atr(highs, lows, closes, period=14):
    tr_list = []
    for i in range(1, len(closes)):
        tr = max(highs[i]-lows[i], abs(highs[i]-closes[i-1]), abs(lows[i]-closes[i-1]))
        tr_list.append(tr)
    avg = sum(tr_list[:period]) / period
    for i in range(period, len(tr_list)):
        avg = (avg * (period-1) + tr_list[i]) / period
    return avg

def supertrend(highs, lows, closes, period=10, multiplier=3.0):
    atr_vals = []
    for i in range(1, len(closes)):
        tr = max(highs[i]-lows[i], abs(highs[i]-closes[i-1]), abs(lows[i]-closes[i-1]))
        atr_vals.append(tr)

    # ATR suavizado
    atr_smooth = [sum(atr_vals[:period]) / period]
    for i in range(period, len(atr_vals)):
        atr_smooth.append((atr_smooth[-1] * (period-1) + atr_vals[i]) / period)

    upper_band = []
    lower_band = []
    for i in range(len(atr_smooth)):
        idx = i + period
        hl2 = (highs[idx] + lows[idx]) / 2
        upper_band.append(hl2 + multiplier * atr_smooth[i])
        lower_band.append(hl2 - multiplier * atr_smooth[i])

    supertrend_vals = []
    trend = []
    for i in range(len(upper_band)):
        idx = i + period
        if i == 0:
            supertrend_vals.append(upper_band[i])
            trend.append(-1)
        else:
            prev_st = supertrend_vals[-1]
            prev_trend = trend[-1]
            if prev_trend == 1:
                st = max(lower_band[i], prev_st) if closes[idx] > prev_st else upper_band[i]
                t = 1 if closes[idx] > st else -1
            else:
                st = min(upper_band[i], prev_st) if closes[idx] < prev_st else lower_band[i]
                t = -1 if closes[idx] < st else 1
            supertrend_vals.append(st)
            trend.append(t)

    return trend[-1], supertrend_vals[-1]  # 1=bullish, -1=bearish

# ── Kelly Criterion ───────────────────────────────────────────────────────────
win_history = []

def kelly_size(balance, price, atr_val):
    if len(win_history) < 10:
        fraction = RISK_PER_TRADE
    else:
        w = sum(win_history) / len(win_history)
        b = 1.5
        fraction = max(min((w - (1-w)/b), 0.10), 0.01)
    stop_distance = atr_val * ATR_MULTIPLIER
    capital_risk = balance * fraction * LEVERAGE
    qty = capital_risk / price
    stop_loss = price - stop_distance
    return round(qty, 4), fraction, round(stop_loss, 1)

# ── Señales ───────────────────────────────────────────────────────────────────
def get_signals():
    klines = get_klines(200)
    if not klines:
        return None

    klines = list(reversed(klines))
    closes = [float(k[4]) for k in klines]
    highs  = [float(k[2]) for k in klines]
    lows   = [float(k[3]) for k in klines]
    price  = closes[-1]

    ema9_val   = ema(closes, 9)
    ema21_val  = ema(closes, 21)
    ema200_val = ema(closes, 200)
    rsi_val    = rsi(closes)
    _, _, histogram = macd(closes)
    adx_val, _, _ = adx(highs, lows, closes)
    atr_val    = atr(highs, lows, closes)
    st_trend, st_val = supertrend(highs, lows, closes)

    signals = 0
    details = []

    # 1. EMA 9 > EMA 21
    if ema9_val[-1] > ema21_val[-1]:
        signals += 1
        details.append("✅ EMA9 > EMA21")
    else:
        details.append("❌ EMA9 < EMA21")

    # 2. Precio > EMA 200
    if price > ema200_val[-1]:
        signals += 1
        details.append("✅ Precio > EMA200")
    else:
        details.append("❌ Precio < EMA200")

    # 3. RSI 45-70
    if 45 < rsi_val < 70:
        signals += 1
        details.append(f"✅ RSI {rsi_val:.1f}")
    else:
        details.append(f"❌ RSI {rsi_val:.1f}")

    # 4. MACD positivo
    if histogram > 0:
        signals += 1
        details.append("✅ MACD bullish")
    else:
        details.append("❌ MACD bearish")

    # 5. ADX > 25
    if adx_val > 25:
        signals += 1
        details.append(f"✅ ADX {adx_val:.1f}")
    else:
        details.append(f"❌ ADX {adx_val:.1f}")

    # 6. Supertrend bullish
    if st_trend == 1:
        signals += 1
        details.append(f"✅ Supertrend alcista")
    else:
        details.append(f"❌ Supertrend bajista")

    # 7. ATR info (no cuenta como señal pero ajusta SL)
    details.append(f"📊 ATR: ${atr_val:,.0f}")

    return {
        "price": price,
        "signals": signals,
        "details": details,
        "rsi": rsi_val,
        "adx": adx_val,
        "atr": atr_val,
        "supertrend": st_trend,
    }

# ── Main loop ─────────────────────────────────────────────────────────────────
in_position   = False
entry_price   = 0
highest_price = 0

def main():
    global in_position, entry_price, highest_price

    mode = "🧪 TESTNET" if TESTNET else "🔴 REAL"
    send_telegram(
        f"✅ <b>Bot BTC/USDT iniciado</b> {mode}\n"
        f"⚡ Apalancamiento: x{LEVERAGE}\n"
        f"📊 Timeframe: 1H | Kelly 5%\n"
        f"🎯 Indicadores: EMA9/21/200 + RSI + MACD + ADX + Supertrend\n"
        f"🔢 Confluencia: {MIN_SIGNALS}/6 señales\n"
        f"🛑 Stop Loss: ATR x{ATR_MULTIPLIER} dinámico"
    )
    print("Bot iniciado...")
    set_leverage()

    while True:
        try:
            data = get_signals()
            if not data:
                time.sleep(CHECK_INTERVAL)
                continue

            price   = data["price"]
            signals = data["signals"]
            details = "\n".join(data["details"])
            atr_val = data["atr"]
            now     = datetime.utcnow().strftime("%H:%M:%S")

            print(f"[{now}] BTC ${price:,.0f} | Señales: {signals}/6")

            if in_position:
                highest_price = max(highest_price, price)
                trailing_stop = highest_price * (1 - TRAILING_PCT)

                if price <= trailing_stop:
                    pnl = ((price - entry_price) / entry_price) * 100 * LEVERAGE
                    win_history.append(1 if pnl > 0 else 0)
                    wins = sum(win_history)
                    close_position()
                    send_telegram(
                        f"🔴 <b>POSICIÓN CERRADA</b>\n"
                        f"💵 Entrada: ${entry_price:,.0f}\n"
                        f"💵 Salida: ${price:,.0f}\n"
                        f"⚡ x{LEVERAGE}\n"
                        f"📊 PnL: <b>{pnl:+.2f}%</b>\n"
                        f"🏆 Win rate: {wins}/{len(win_history)} ({wins/len(win_history)*100:.0f}%)\n"
                        f"🕐 {now} UTC"
                    )
                    in_position   = False
                    entry_price   = 0
                    highest_price = 0

            elif signals >= MIN_SIGNALS:
                balance = get_balance()
                qty, fraction, stop_loss = kelly_size(balance, price, atr_val)
                result = place_order("Buy", qty, stop_loss)

                if result.get("retCode") == 0:
                    in_position   = True
                    entry_price   = price
                    highest_price = price
                    send_telegram(
                        f"🟢 <b>ENTRADA LONG BTC</b>\n"
                        f"💵 Precio: ${price:,.0f}\n"
                        f"⚡ Apalancamiento: x{LEVERAGE}\n"
                        f"📊 Señales: {signals}/6\n"
                        f"{details}\n"
                        f"🎯 Kelly: {fraction*100:.1f}% → {qty} BTC\n"
                        f"🛑 Stop Loss: ${stop_loss:,.0f} (ATR dinámico)\n"
                        f"🕐 {now} UTC"
                    )
                else:
                    print(f"Error orden: {result}")

        except Exception as e:
            print(f"Error: {e}")

        time.sleep(CHECK_INTERVAL)

if __name__ == "__main__":
    main()
