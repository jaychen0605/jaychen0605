python
複製程式碼
import ccxt
import pandas as pd
import time

# 创建交易所实例，这里以币安为例
exchange = ccxt.binance({
    'apiKey': 'YOUR_API_KEY',
    'secret': 'YOUR_SECRET',
})

def fetch_data(symbol, timeframe, limit=100):
    """获取历史K线数据"""
    ohlcv = exchange.fetch_ohlcv(symbol, timeframe, limit=limit)
    data = pd.DataFrame(ohlcv, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume'])
    data['timestamp'] = pd.to_datetime(data['timestamp'], unit='ms')
    return data

def strategy(data):
    """简单的移动平均线交叉策略"""
    data['ma_short'] = data['close'].rolling(window=5).mean()
    data['ma_long'] = data['close'].rolling(window=20).mean()

    # 产生信号：短期均线向上穿过长期均线买入，反之卖出
    data['signal'] = 0
    data['signal'][5:] = np.where(data['ma_short'][5:] > data['ma_long'][5:], 1, -1)
    data['position'] = data['signal'].shift()
    return data

def execute_trade(symbol, side, amount):
    """执行交易"""
    try:
        order = exchange.create_market_order(symbol, side, amount)
        print(f"Trade executed: {side} {amount} {symbol}")
    except Exception as e:
        print(f"Trade failed: {e}")

# 设置交易参数
symbol = 'BTC/USDT'
timeframe = '1h'
amount = 0.001  # 每次交易的比特币数量

# 获取初始数据
data = fetch_data(symbol, timeframe)

while True:
    # 获取最新数据
    new_data = fetch_data(symbol, timeframe, limit=1)
    data = data.append(new_data).drop_duplicates(subset='timestamp', keep='last')
    
    # 应用策略
    data = strategy(data)
    
    # 获取最新信号
    latest_signal = data['signal'].iloc[-1]
    latest_position = data['position'].iloc[-1]
    
    # 执行交易
    if latest_signal == 1 and latest_position == -1:
        execute_trade(symbol, 'buy', amount)
    elif latest_signal == -1 and latest_position == 1:
        execute_trade(symbol, 'sell', amount)
    
    # 等待下一次循环
    time.sleep(60 * 60)  # 每小时检查一次
