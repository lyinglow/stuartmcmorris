# ATR Trend & Range Model

A TradingView indicator built to work like the ATR tool at investanswers.io: one line that shows the trend, doubles as your stop loss, and flips into a buy or sell signal when the trend changes. A second, faster line sits alongside it so you can see how calm or wild the market is right now.

## What it shows

- **Trend line**: green when the trend is up, red when it's down. This line is also your stop loss.
- **Cloud**: the gap between the trend line and a faster line. A wide cloud means a strong, clear trend.
- **Buy / Sell labels**: appear the moment the trend flips.
- **Table (top right)**: trend direction, how lively (volatile) the market is, and the exact stop loss price.
- **Yellow background**: shows up when volatility is unusually high.

## How to install

1. Open TradingView and pick any chart.
2. Open **Pine Editor** (bottom of the screen).
3. Delete the placeholder code and paste in everything from `atr_trend_range.pine`.
4. Click **Add to Chart**.
5. Click the gear icon on the indicator to adjust settings if you want.

## Settings

- **ATR Length**: how many bars the volatility measurement looks back over. 14 is the standard.
- **Macro Trend Multiplier**: how far the stop loss sits from price. Bigger number, fewer signals, fewer false alarms.
- **Fast Band Multiplier**: how far the faster line sits from price. Controls how wide the cloud looks.

## Best used on

Daily charts and above (12h, 2D, 3D, weekly). It's built for swing trading and longer holds, not day trading or scalping.

## Alerts

Right click the chart, choose **Add Alert**, and pick one of:
- ATR-TRM Buy
- ATR-TRM Sell
- ATR-TRM Volatility Spike

This way you don't need to watch the chart. TradingView will tell you when something changes.
