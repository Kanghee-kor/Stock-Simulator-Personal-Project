# Stock-Simulator-Sentiment-Based-Prediction-System
---

# Project Overview
This project is a full stack stock trading simulator and research platform built from scratch.<br>
It started as a simple Python terminal programme with fake company names and random price generation and it evoled into a professional web application powered by real market data, news sentiment analysis and machine learning-adjacent prediction tracking. 

The core research question driving this project is: 
"Can financial news sentiment predict short term stock price movements?

The platfrom allows users to:
- Track 50+ real stocks acriss US, European and Asian markets with live prices from Yahoo Finance
- Buy and Sell stocks with a simulated $10,000 starting portfolio
- Read real time news headlines for each stock, atomatically analyzed for sentiment
- Receive daily automated predictions (Bullish/Bearish/Neutral) based on news sentiment
- Track prediction accuracy over time with research grade metrics including directional acuracy, MAE, RMSE and a confusion matrix
- Compare prediction performance across different confidence levels

My aim of this project was to mirror real alternative data strategies used by quantitative hedge funds, where news sentiment is used as a signal for systematic trading decisions. 

---

# Languages Used

## Python

Python was chosen as the main backend language because of its rich ecosystem of data science and web libraries. It handles all the heavy computation such as fetching real market data, processing news, analysing sentiment, running schedules jobs and managing persistent storage. 

Key libraries used in this project:
- `yfinance`: fetches real stock price data from Yahoo Finance. Used to fetch current prices, historical price charts and currency exchange rates

- `flask`: lightweight web framework that turns Python functions into API endpoints accessible from the browser. Chose this over Django for the simplicity as each route is a single function

- `pandas`: data manipulation library used internally by yfinance. Used directly for handling the DataFrame returned by yf.download(), including dropna() to remove NaN values from historical data

- `requests`: HTTP library for calling the NewsAPI external API to fetch financial headlines

- `schedule`: lightweight python job scheduler used to run daily prediction generation at 8:00 AM automatically

- `threading`: Python's built-in threading module, used to run the scheduler in a background thread so it doesn't block the Flask web server

- `datetime`: Python's built-in date/time module, used for calculating prediction target dates, skipping weekens and timestamping predictions

- `json` and `os`: built-in modules for reading/writing persisten data files

## JavaScript 
JavaScript was used as it runs entirely within the browser and handles everything the user sees and interacts with. Java was necessary mostly because HTML/CSS alone are static. So Java made the page dynamic by updating prices, drawing charts, fetching news without having to reload the page. 

The key concepts that were used is:
- `fetch() API with asynce/await`: sends HTTP requests to Flask API routes and receives JSON responses without reloading the page

- `Chart.js`: a third-party charting library loaded from CDN. Used to render interactive stock price history line charts

- `DOM manipulation`: `document.getElementById()`, `createElement()`, `innerHTML`, `appendChild()` used to dynamically build the stock list, portfolio summary, news section and predictions table entirely from data

- `setInterval()`: auto refreshes prices and portfolio every 60 seconds

- `Event listeners`: `addEventListener("click", ...)` for stock selection, buy/sell buttons, sector filter buttons and tab switching

## HTML
HTML was used to make the skeleton of the web page. The tabs, sidebar, detail panel and placeholder divs that JavaScript fills with dynamic content. Jinja2 templating is used to generate correct static file URLs.

## CSS
CSS was used to handle the visual design. The overall theme was inspired by Bloomberg Terminal aesthetics. Flexbox layout is used throughout for responsive side-by-side panels.

---

# Files Explaination

## app.py
`app.py` is the entry point. It starts Flask, imports `stocks`, `news` and `scheduler`, registers all URL routes and runs the server. <br>
To see `app.py` click here: [`app.py`](app.py)

## stocks.py
`stocks.py ` is the brain of the application. It contains TICKERS dictionary which has the 50+ stocks with symbol, currency, sector, region and news query. Also it contains all portfolio functions (buy, sell, get portfolio, reset), price fetching with caching, currency conversion and the entire prediction system (save, load, evaluate, compute metrics) <br>
To see `stocks.py` click here: [`stocks.py`](stocks.py)

## news.py
`news.py` handles all external news API communication and sentiment analysis. It is completely independent with the rest of the system and it only needs a company name and it returns headlines and a sentiment verdict. <br>
To see `news.py` click here: [`news.py`](news.py)

## scheduler.py
`scheduler.py` runs inside the Flask server as a background thread. It calls `news.py` and `stocks.py` together to generate predictions for all 50+ stocks at 8 AM every trading day. <br>
To see `scheduler.py` click here: [`scheduler.py`](scheduler.py)

## index.html
`index.html` is a single HTML page used for tab-based layout: [`index.html`](index.html)

## style.css 
`style.css` is a code that styles the whole website: [`style.css`](style.css)

## script.js 
`script.js` is a code with all frontend logic that is responsible with fetching, rendering and charts. <br>
To see `script.js` click here: [`script.js`](script.js)

# Additional Markdown

If you want to read more about the important codes written click here: [Important Code Explained](Important_Code.md)

If you want to know more about the future improvements that can be made refer here: [Future Improvements](Improvements.md)

If you want to read about the new prediction module I have made refer here: [Quantitative Prediction Module](Quantitative.md)
