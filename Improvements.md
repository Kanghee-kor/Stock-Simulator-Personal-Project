# Future Improvement Points Explained
This page explains current problems of the application and suggests future improvement points. 

# 1. Sentiment Analysis
This is the most critical limitation. The current keyword based approach has three fundamental limitations:

---

## Problem 1 - Context blindess:
The system cannot understand the same keyword used in different nouns. 
For example: "Apple faces record low competitios" scores + 1. "Apple reports record loss" scores -1. Cannot tell the difference of the usage "record". 

Improvement: Upgrade to a pre-trained financial NLP model. The best free option is FinBERT. It is a version of BERT tuned specifically for financial text. Or do a LCA so that the keywords are categorised. :
``` python
from transformers import pipeline
finbert = pipeline("sentiment-analysis", model="ProsusAI/finbert")
result = finbert("Apple reports record quarterly loss amid tariff concerns")
# Returns: [{'label': 'negative', 'score': 0.97}]
```
---

## Problem 2 - Headline Quality
NewsAPI free tier returns headlines from any source mentioning the company name. This might include irrelevant results. 

Improvement: Filter by source quality - restrict to financial news sources:
``` python
params = {
    "q": news_query,
    "domains": "reuters.com,bloomberg.com,wsj.com,ft.com,cnbc.com,marketwatch.com",
    "language": "en"
}
```
---

## Problem 3 - Using only 5 headlines:
Initially only used 5 samples because of the limited amount of articles that could be uploaded per day. However, only using 5 headlines means that a single irreevant headline can skew the score. 

Improvements: Increase to 10-15 headlines, or use a weighted scoring system where headlines from higher-quality count more. 

---

# 2. Prediction Evaluation Window
Problem: Currently comparing prediction price vs current price at the moment the Predictions tab is opened. This means that the outcome of the prediction changes everytime the tab is reopened. 

Improvement: Fetch the closing price on the target date rather than the current real time price. This requires storing the outcome price at a specific fixed moment. 

``` python
ticker = yf.Ticker(symbol)
hist = ticker.history(start=target_date, end=target_date + timedelta(days=1))
outcome_price = float(hist["Close"].iloc[0])
```
---

# 3. Backtesting System
Currently predictions are only forward looking. This means that weeks of data is needed before meaningful metrics appear. 

Improvement: Add a backtestng module that tests the sentiment signal against historical data:
``` python
def backtest(stock_name, period="6mo"):
    history = get_price_history(stock_name, period=period)
    # For each week in history, simulate what sentiment would have predicted
    # Compare against actual prices
    # Return simulated accuracy and profit/loss
```
This will provide lots of data points without having to wait weeks and is the standard approach in quantitative finance research. 

# 4. Multi Signal Prediction
The current system only uses news sentiment as its prediction signal. However, real trading systems combine multiple signals:
- Price momentum: If a stock has risen for a few days in a row it may continue

- Volume: Unusual trading volume often precedes to price movement

- Earnings calender: Stocks behave differently before and after earnings report

- Sector correlation: If for example a semiconductor sector is broadly up, individual chip design stocks tend to follow the trend

Improvements: A momentum signal could be added for price momentum analysis
``` python
def momentum_signal(name, days=5)
    history = get_price_history(name, period="1mo")
    recent = history["prices"][-days:]
    change = (recent[-1] - recent[0]) / recent[0]
    if change > 0.03:
        return "Bullish"
    elif change < -0.03:
        return "Bearish"
    return "Neutral"
```
---

