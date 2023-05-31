# BanterBot
Hello, I'm just having some trouble connecting the python script to the MetaTrader5 application 

import MetaTrader5 as mt5
import pandas as pd
import time
import numpy as np
from datetime import datetime, timedelta
from pytz import timezone
import requests
from bs4 import BeautifulSoup
from itertools import product

#Define your constants at the top
LOGIN = 70047815
PASSWORD = "e7AAESLN"
SERVER = "MetaQuotes-Demo"
RISK_PERCENTAGE = 0.01
ACCOUNT_CURRENCY = "USD"  # Replace with your account currency
POINTS = 0.0001  # Change to 1 for non-forex symbols like SPX500
SYMBOL = "EURUSD"
TIMEFRAME = mt5.TIMEFRAME_M15
NON_TRADE_DAYS = ["FOMC Statement", "Federal Funds Rate",
                  "Non-Farm Employment Change", "Unemployment Rate", "CPI"]



# Connect to MetaTrader 5
def connect(account):
    mt5.initialize()
    authorized =mt5.login(LOGIN , password=PASSWORD, server=SERVER)

    if authorized:
        print("Connected: Connecting to MT5 Client")
        return True
    else:
        print("Failed to connect at account #{}, error code: {}"
              .format(account, mt5.last_error()))
        return False
    

def fetch_data(SYMBOL, timeframe, num_candles):
    print("Fetching data...")
    rates = mt5.copy_rates_from_pos(SYMBOL, timeframe, 0, num_candles)
    if rates is None:
        print("Failed to fetch data. Error:", mt5.last_error())
        return None
    print("Data fetched successfully. Converting to DataFrame...")
    data = pd.DataFrame(rates)
    data['time'] = pd.to_datetime(data['time'], unit='s')
    return data


def is_trading_time():
    server_time_utc = datetime.fromtimestamp(mt5.time_current())
    cst = timezone('America/Chicago')
    server_time_cst = server_time_utc.astimezone(cst)
    hour = server_time_cst.hour

    return (1 <= hour < 4) or (6 <= hour < 12)


def is_weekend_close_time():
    server_time_utc = datetime.fromtimestamp(mt5.symbol_info_tick("EURUSD").time)
    cst = timezone('America/Chicago')
    server_time_cst = server_time_utc.astimezone(cst)
    weekday = server_time_cst.weekday()
    hour = server_time_cst.hour
    minute = server_time_cst.minute

    return weekday == 4 and (hour > 15 or (hour == 15 and minute >= 50))


def close_all_trades(SYMBOL):
    open_positions = mt5.positions_get(SYMBOL=SYMBOL)
    if open_positions == None or len(open_positions) == 0:
        return

    for position in open_positions:
        trade_result = mt5.Close(position.ticket)
        if trade_result.retcode != mt5.TRADE_RETCODE_DONE:
            print(f"Trade close failed, retcode: {trade_result.retcode}")
        else:
            print(f"Trade closed successfully, ticket: {position.ticket}")


def calculate_lot_size(account_balance, stop_loss_pips, risk_percentage, account_currency):
    if account_currency == "USD":
        risk_per_trade = account_balance * risk_percentage
        risk_per_pip = risk_per_trade / stop_loss_pips
        lot_size = risk_per_pip / (10 * POINTS)
    else:
        # Add calculation for other account currencies if needed
        lot_size = 0.01
    return lot_size


def check_non_trade_day():
    url = "https://www.forexfactory.com/calendar"
    response = requests.get(url)
    soup = BeautifulSoup(response.content, "html.parser")

    events_today = soup.find_all(
        "tr", class_="calendar__row calendar_row calendar__row--today")

    for event in events_today:
        event_title = event.find("td", class_="calendar__event").text.strip()
        if event_title in NON_TRADE_DAYS:
            return True
    return False


def market_structure(SYMBOL):
    data = fetch_data(SYMBOL, "D1", 300)
    data['ema_70'] = data['close'].ewm(span=70).mean()
    data['ema_210'] = data['close'].ewm(span=210).mean()

    if data['ema_70'].iloc[-1] > data['ema_210'].iloc[-1]:
        return "BULLISH"
    else:
        return "BEARISH"


def market_shifts(data):
    data['Higher_High'] = (data['high'] > data['high'].shift(1)) & (
        data['high'] > data['high'].shift(2))
    data['Higher_Low'] = (data['low'] > data['low'].shift(1)) & (
        data['low'] > data['low'].shift(2))
    data['Lower_High'] = (data['high'] < data['high'].shift(1)) & (
        data['high'] < data['high'].shift(2))
    data['Lower_Low'] = (data['low'] < data['low'].shift(1)) & (
        data['low'] < data['low'].shift(2))
    return data


def ote_entry(data, market_structure_status):
    last_5_candles = data.iloc[-5:]
    high_5 = last_5_candles['high'].max()
    low_5 = last_5_candles['low'].min()
    fib_levels = [1, 0.79, 0.62, 0, -0.5, -1, -1.5, -2]

    if market_structure_status == "BULLISH":
        entry_levels = [low_5 + (high_5 - low_5) *
                        level for level in fib_levels]
        trade_zone = entry_levels[1:3]
        buy_level = entry_levels[2]
        stop_loss = entry_levels[0]
        return buy_level, stop_loss, entry_levels, trade_zone
    elif market_structure_status == "BEARISH":
        entry_levels = [high_5 - (high_5 - low_5) *
                        level for level in fib_levels]
        trade_zone = entry_levels[1:3]
        sell_level = entry_levels[2]
        stop_loss = entry_levels[0]
        return sell_level, stop_loss, entry_levels, trade_zone
    else:
        return None, None, None, None


def manage_trade(trade, price, take_profit_levels, stop_loss):
    current_profit = trade.profit
    current_sl = trade.sl
    tp1, tp2, tp3, tp4 = take_profit_levels

    if price >= tp1 and current_profit >= 30 and current_sl != stop_loss:
        # Move stop loss to entry
        trade.sl = stop_loss
        print("Stop loss moved to entry")

    if price >= tp2 and current_profit >= 60:
        # Move stop loss to TP1
        trade.sl = tp1
        print("Stop loss moved to TP1")

    if price >= tp3 and current_profit >= 80:
        # Move stop loss to TP2
        trade.sl = tp2
        print("Stop loss moved to TP2")

    if price >= tp4:
        # Move stop loss to TP3 and add trailing stop
        trade.sl = tp3 - 15 * POINTS
        print("Stop loss moved to TP3 and trailing stop added")


def place_order(SYMBOL, order_type, volume, price, stop_loss, take_profit):
    if order_type == mt5.ORDER_TYPE_BUY:
        order = mt5.OrderCheck(
            action=mt5.TRADE_ACTION_DEAL,
            SYMBOL=SYMBOL,
            volume=volume,
            type=order_type,
            price=price,
            sl=stop_loss,
            tp=take_profit,
            deviation=20,
            type_filling=mt5.ORDER_FILLING_IOC,
        )
    elif order_type == mt5.ORDER_TYPE_SELL:
        order = mt5.OrderCheck(
            action=mt5.TRADE_ACTION_DEAL,
            SYMBOL=SYMBOL,
            volume=volume,
            type=order_type,
            price=price,
            sl=stop_loss,
            tp=take_profit,
            deviation=20,
            type_filling=mt5.ORDER_FILLING_IOC,
        )

    result = mt5.order_check(order)

    if result[0] == mt5.TRADE_RETCODE_DONE:
        print(f"Trade placed successfully, ticket: {result.order}")
        return result.order
    else:
        print(f"Trade placement failed, retcode: {result.retcode}")
        return None


def calculate_stop_loss_pips(high, low):
    return abs(high - low) / POINTS

def main():
    #Initialize MT5
    if not connect(LOGIN):
        print("Failed to connect to MT5")
        return
    
    while True:
        if is_weekend_close_time():
            close_all_trades("EURUSD")
            open_trades = {}  # Clear the dictionary when all trades are closed
            break

        print("Checking if it's trading time asnd not a non-trading day...")
        if is_trading_time() and not check_non_trade_day():
            print("Fetching market structure...") 
            market_structure_status = market_structure("EURUSD")

            data_15m = fetch_data("EURUSD", TIMEFRAME, 500)
            data_15m = market_shifts(data_15m)
            last_candle = data_15m.iloc[-1]

            # Check if a new Higher High or Lower Low was created
            if last_candle['Higher_High'] and market_structure_status == "BULLISH":
                stop_loss_pips = calculate_stop_loss_pips(
                    last_candle['low'], last_candle['high'])
                lot_size = calculate_lot_size(
                    mt5.account_info().balance, stop_loss_pips, RISK_PERCENTAGE, ACCOUNT_CURRENCY)
                entry, stop_loss, levels, trade_zone = ote_entry(
                    data_15m, market_structure_status)
                # Set take profit at the top of the trade zone
                take_profit = trade_zone[1]
                ticket = place_order("EURUSD", mt5.ORDER_TYPE_BUY,
                                    lot_size, entry, stop_loss, take_profit)
                open_trades[ticket] = {
                    "entry": entry, "take_profit_levels": levels, "stop_loss": stop_loss}

            elif last_candle['Lower_Low'] and market_structure_status == "BEARISH":
                stop_loss_pips = calculate_stop_loss_pips(
                    last_candle['high'], last_candle['low'])
                lot_size = calculate_lot_size(
                    mt5.account_info().balance, stop_loss_pips, RISK_PERCENTAGE, ACCOUNT_CURRENCY)
                entry, stop_loss, levels, trade_zone = ote_entry(
                    data_15m, market_structure_status)
                # Set take profit at the top of the trade zone
                take_profit = trade_zone[1]
                ticket = place_order("EURUSD", mt5.ORDER_TYPE_SELL,
                                    lot_size, entry, stop_loss, take_profit)
                open_trades[ticket] = {
                    "entry": entry, "take_profit_levels": levels, "stop_loss": stop_loss}

            # Manage open trades
            for ticket, trade_details in open_trades.items():
                trade = mt5.positions_get(ticket=ticket)
                if trade is None or len(trade) == 0:
                    # Trade has been closed, remove it from the dictionary
                    del open_trades[ticket]
                else:
                    manage_trade(trade[0], last_candle['close'],
                                trade_details['take_profit_levels'], trade_details['stop_loss'])

        
    
    mt5.shutdown() 

if __name__ == "__main__":
    main() 

