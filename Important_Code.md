# Important Code Explained
In this file, important codes that I wrote in making this website will be explained

---

## 1. Batch Price Fetching With Currency Conversion
This code can be found in [`stocks.py`](stocks.py)

``` python
def get_current_prices():
    global _price_cache, _cache_time
    if _price_cache is not None and (time.time() - _cache_time) < CACHE_DURATION:
        return _price_cache

    symbols = [info["symbol"] for info in TICKERS.values()]
    data = yf.download(symbols, period="2d", group_by="ticker", progress=False)

    for name, info in TICKERS.items():
        symbol = info["symbol"]
        currency = info["currency"]
        closes = data[symbol]["Close"].dropna()
        current = convert_to_usd(float(closes.iloc[-1]), currency, rates)
        previous = convert_to_usd(float(closes.iloc[-2]), currency, rates)
        change_percent = round(((current - previous) / previous) * 100, 2)
```
- Uses `yf.download()` with multiple symbols in a single API call instead of 50 individual calls. This dramatically reduces latency

- `period="2d"` fetches two days of data so the app can calculate data-over-day percentage change. (`closes.iloc[-1] vs closes.iloc[-2]`)

- `dropna()` removes NaN values that Yahoo Finance sometimes returns for the most recent data point

- Server-side caching(`_price_cache`, `_cache_time`, `CACHE_DURATION=60`) means subsequent requests within 60 seconds return instantly without hitting Yahoo Finance again

- Generic currency conversion using live exchange rates, especially for Korean Won to USD. (`KRW=X ticker`) means any non-USD stock is automatically converted

---

## 2. The Sentiment Analysis Engine
This code can be found in [`news.py`](news.py)

``` python
POSITIVE_WORDS = [
    "surge", "gain", "profit", "beat", "growth", "record", "rise", "rally",
    "historic", "milestone", "partnership", "deal", "launch", "innovation",
    "expand", "revenue", "earnings", "raised", "increase", "recovery", "rebound"
]

NEGATIVE_WORDS = [
    "drop", "loss", "miss", "decline", "crash", "cut", "fall", "weak",
    "lawsuit", "fraud", "scandal", "recall", "bankruptcy", "investigation",
    "tariff", "sanction", "ban", "suspend", "deficit", "vulnerable"
]

def analyze_sentiment(headlines):
    for headline in headlines:
        title = headline["title"].lower()
        words = title.split()

        up_matches = len(re.findall(r'up\s+\d+%|\+\d+%|\d+%\s+gain', title))
        down_matches = len(re.findall(r'down\s+\d+%|-\d+%|\d+%\s+loss', title))

        pos_count = sum(1 for word in words if any(pw in word for pw in POSITIVE_WORDS)) + up_matches
        neg_count = sum(1 for word in words if any(nw in word for nw in NEGATIVE_WORDS)) + down_matches

        score = pos_count - neg_count
```
This code is a lexicon-based sentiment analysis system. The simplest form of Natural Language Processing(NLP). This works on two levels:

1. Keyword matching: For each word in a headline, the code checks if any positive or negative financial keyword appears as a substring. Using `any(pw in word ...)` rather than exact matching means that "surges" matches with "surge", "gained" matches with "gain". This allows morpholocial variation handling without a stemmer.

2. Regular expression pattern matching: `re.findall(r'up\s+\d+%|\+\d+%`) catches numerical signals like up to 20% or +5%, which normal keyword matching programmes will miss. This was a critical improvement because financial headlines frequently express direction through numbers rather than sentiment words. Also, numbers tend to be a more direct factor when it comes to prediction.

After analysing the financial headlines, the confidence score normalizes the raw sentiment score to 0-1 scale:
``` python
confidence = round(min(abs(score) / 3.0, 1.0), 2)
```

A score of 3.0 or above maps to 100% confidence. This allows the research system to distinguish between weak signals, such as borderline Bullish and strong signals. 

The threshold of ±0.5 was calibrated through testing: Lower thresholds produced too many false signals, while 0.5 requries at least half a net positive/negative word per headline on average. This allowed to filter out the borderline noise. 


## 3. Duplicate-Free Schedule Predicton System
This code can be found in [`stocks.py`](stocks.py)

``` python
def save_prediction(name, verdict, score, price, horizon="1D"):
    target_date = get_target_date(horizon)
    for existing in predictions[name]:
        if existing["target_date"] == target_date and not existing["evaluated"]:
            return  # Skip — prediction already exists for this horizon/date
```
This deduplication logic ensures that regardless of how many times news it fetched, only one prediction per stock per trading day is made. Without this logic, repeated page and news views would create multiple predictions for the same day, polluting the dataset and making accuracy metrics unreliable. 

The `get_target_date()` function also automatically skips weekends:
``` python
target = today + datetime.timedelta(days=delta)
while target.weekday() >= 5:  # 5=Saturday, 6=Sunday
    target += datetime.timedelta(days=1)
```
A prediction made on Friday with horizon 1D targets the following Monday instead of Saturday. 

## 4. Prediction Evaluation & Performance Metrics
This code can be found in [`stocks.py`](stocks.py)

``` python
def compute_metrics(stock_name=None):
    evaluated = [p for p in all_preds if p["evaluated"] and p["outcome"] != "Neutral"]

    correct = sum(1 for p in evaluated if p["outcome"] == "Correct")
    directional_accuracy = correct / len(evaluated)

    errors = [abs(p["price_change_pct"]) for p in evaluated]
    mae = sum(errors) / len(errors)
    rmse = (sum(e**2 for e in errors) / len(errors)) ** 0.5

    tp = sum(1 for p in evaluated if p["direction"] == "Bullish" and p["outcome"] == "Correct")
    fp = sum(1 for p in evaluated if p["direction"] == "Bullish" and p["outcome"] == "Incorrect")
    fn = sum(1 for p in evaluated if p["direction"] == "Bearish" and p["outcome"] == "Incorrect")
    tn = sum(1 for p in evaluated if p["direction"] == "Bearish" and p["outcome"] == "Correct")

    by_confidence = {
        "high": [p for p in evaluated if p["confidence"] >= 0.7],
        "medium": [p for p in evaluated if 0.4 <= p["confidence"] < 0.7],
        "low": [p for p in evaluated if p["confidence"] < 0.4]
    }
```

This function computes four distinct research metrics:
1. Directional Accuracy: This is the most intuitive metric. It tells percentage of the time did the prediction get the direction right? Neutral predictions are excluded from this calculation since they make no directional claim. A random baseline would achieve 50% so anything significantly over 50% indicates that the model has predicted positive.

2. MAE(Mean Absolute Error): Measures how wrong the predictions are in terms of actual price change magnitude. However, since the app predict direction not magnitude, MAE here measures the average price movement regardless of direction. This is usefull for understanding market volatility during the prediction window.

3. RMSE(Root Mean Square Error): This is similar to MAE, but penalises large errors more heavily (squares the values before averaging). If RMSE >> MAE, this means that the model is making a very large error.

4. Confusion Matrix: This is the most nuanced metric:
   - TP(True Positive): Predicted Bullish, price went up
   - FP(False Positive): Predicted Bullish, price went down
   - FN(False Negative): predicted Bearish, price went up
   - TN(True Negative): Predicted Bearish, price went down

This matrix reveals directional bias: If FP>>FN, the model is over optimistic. If TP>>TN, the model detects upward movements better than downward ones.

5. Accuracy by Confidence Level: This is the research valuable model which is yet to be completed: Does high conidence sentiment (score > 0.7) actually outperform low-confidence sentiment? 
