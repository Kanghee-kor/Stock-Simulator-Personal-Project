# Building a Quantitative Prediction Engine
The simulator started with one prediction engine: a news-sentiment classifier that scores headlines by counting positive and negative keywords. It works, but it is not outstanding. It does not know what the price is actually been doing. Therefore, I build a second engine that predicts direction purely from historical price data and technical indicators, where no news is involved. Later when more data is gathered, I will combine the first engine with the second engine to get a combined analysis of both sentiment and quantitative. 

---

## Why a second signal at all
Sentiment and price action answer different questions. Sentiment asks "what is being said about this stock right now?" Technical analysis asks "what has the price itself been doing?" These two can agree, disagree or say nothing useful at all. But I wanted to be able to see all that instead of just blending the signals into one number and losing the distinction. 

---

## The indicators and what each one is actually measuring. 
All of the data used are stored in `calculated_time_series_features()`, which pulls a year of daily price history per stock and derives the following:

### Moving Averages (MA5 / MA20 / MA50 / MA200)
This plain rolling means of th closing price over 5, 20, 50 and 200 days smooth out day-to-day noise. This tends to make underlying the trend easier. I specifically used `MA20` as the reference for "is the current price meaningfully above of below its recent trend":

```
price_vs_MA20_pct = (current_price / MA20 - 1) * 100
```
A stock trading more than ~2% above its 20-day average is treated as showing upward momentum, while more than 2% below shows the opposite. 

### RSI (Relative Strength Index)
This is a momentum oscillator between 0 and 100 that measures how fast and how far a price has moved recently:

```
gain = rolling_mean(positive price changes, 14 days)
loss = rolling_mean(negative price changes, 14 days)
RS = gain / loss
RSI = 100 - (100 / (1 + RS))
```
Above 70 is generally considered as "overbought". Below 30 is read as "oversold". <br>
Note: This is a simplifiedd version of RSI not the full Wilder's original method. It is still good enough for its purpose, but acknowledging that it is not identical to what professional firms will use. 

### MACD (Moving Average Convergence Divergence)
A trend-following momentum indicator built from two exponential moving averages:

```
EMA12         = 12-day exponential moving average of price
EMA26         = 26-day exponential moving average of price
MACD line     = EMA12 - EMA26
Signal line   = 9-day EMA of the MACD line
Histogram     = MACD line - Signal line
```

When the MACD line crosses above the signal line (positive histogram), it means a bullish momentum, crossing below is bearish. Because it uses *exponential* averages, it reacts faster to recent price changes than a simple *linear* moving average would. 

### Bollinger Bands
Band drawn a fixed number of standard deviations above and below a moving average, meant to bracket "normal" price movement:

```
Middle band = MA20
Upper band  = MA20 + (2 x std_dev_of_price_over_20_days)
Lower band  = MA20 - (2 x std_dev_of_price_over_20_days)
```

Price near the lower band suggests that the stock is trading unusually low realtive to its recent range (potential bounce). Near the upper band suggests unusually high (potentail pullback). <br>
I track this a `BB_position`, a normalized 0-1 value: `(price - lower_band) / (upper_band - lower_band)`. 

### ATR (Average True Range)
A volatility measure that accounts for overnight gaps, not just the day's high-low range:

```
True Range = max(
    high - low,
    abs(high - previous_close),
    abs(low - previous_close)
)
ATR_14 = rolling_mean(True Range, 14 days)
```
Even though I compute this, I don't actually feed it into the trading signal right now. This is because I don't have enought data to make this line of code meaningful yet. 

### Volume Ratio
Today's volume divided by the 20-day average volume. <br>
A ratio above 1.5 means usually heavy trading - the idea being that a price move on high volume reflects real conviction, while the same move on thin volume is less trustworthy. 

---

## Combining them into one signal 
`generate_trading_signal()` takes all of the mentioned analysis and turns it inot a single weighted score:

| Factor | Condition | Weight |
| --- | --- | --- |
| Price vs MA20 | >2% above / below | ±0.30 |
| MACD | Histogram positive + bullish cross / negative + bearish cross | ±0.25 |
| RSI | <30 (oversold -> bullish) / >70 (overbought -> bearish) | ±0.20 |
| Bolliger position | <0.2 (near lower band -> bullish) / >0.8 (near upper band -> bearish) | ±0.15 |
| Volume | High volume + positive return / high volume + negative return | ±0.10 |

The weights sum to 1.0, Final score maps to a verdict:
```
score >= 0.3  -> Bullish
score <= -0.3 -> Bearish
otherwise     -> Neutral
```
Confidence is `min(abs(score), 1.0)` for a directional call, or `abs(score / 0.3` when it lands in the Neutral zone. Trend gets the heaviest weight (30%) as it is the most persistant of these signals. On the other hand, volume gets the leatst (10%) because on its own it says nothing about direction, only about how much to trust a move that's already happened. 


