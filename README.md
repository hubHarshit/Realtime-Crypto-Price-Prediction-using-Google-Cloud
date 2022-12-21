
Predicting Crypto Market Tendencies – Volatility as an Advantage 

Icon

Description automatically generated 

 

 

 

 

 

 

 

 

 

 

For this project, the team is working on crypto currency data, where archived data is procured from the CoinMarketCap website, and live streaming data is procured from Coinbase API. The goal for the project is to provide a live prediction of market close volume sold across cryptos and closing exchange rate to generate insights that lead to quick profits in the Crypto Currency market. 

 

The steps followed by the team are as follows: 

 

Setting up the pipeline 

Create a cloud function to connect to the Coinbase parser from the Coinbase API 

Graphical user interface, text, application

Description automatically generated 

 

 

Define a job schedule using the Cloud Scheduler with a trigger frequency of 10 minutes. The scheduler triggers the cloud function to transfer the JSON directly to the pubsub and a single dataflow transfers the data to BigQuery. The configuration is shown below. 

Graphical user interface, text, application

Description automatically generated 

Graphical user interface, application

Description automatically generated 

    

Create a Pubsub 

The messages are pulled every time the job is triggered 

Graphical user interface, text, application, email

Description automatically generated 

 

 

 

 

 

 

 

 

Store data in GCS 

With the given configuration in the scheduler, the streaming records are pushed into the GCS buckets as files. 

Graphical user interface, application, table

Description automatically generated 

 

Create a dataflow: The debug logs depict the data being transferred to Big Query at each trigger 

Graphical user interface, text, application, email

Description automatically generated 

 

 

 

 

 

 

 

 

 

 

Dataflow jobs move data from GCS to BigQuery 

Diagram

Description automatically generated 

 

BigQuery data set with live data for analysis 

Below is a screengrab of the live stream crypto dataset 

Table

Description automatically generated 

 

Procuring archive data 
Archive data for crypto exchange rates was procured from the CoinMarketCap website. The data includes name of the crypto currency, the date for which data is procured, the opening exchange rate on the day, the closing exchange rate on the day, the daily high of the exchange rate, the daily low of the exchange rate, the adjusted closing exchange rate on the day and the volume sold on the day. 

A screengrab of the data is available below. 
Table

Description automatically generated 

 

Callouts 

The exchange rates provided in the datasets are against USD. 

Data for the top 28 crypto currencies, according to coinmarketcap, are available in the dataset. 

The open and close values in archive data are recorded at 00 hr at day change. All our metrics and predictions are based on it. 

 

Metrics: 

Fibonacci Retracement: Fibonacci retracement is a technical analysis tool that uses horizontal lines to indicate areas of support or resistance at the key Fibonacci levels before the price continues in the original direction. These levels are derived from the Fibonacci sequence and are commonly used in conjunction with trend lines to find entry and exit points in the market. We have calculated these levels at 38.5%, 50%, 61.8% and 75% respectively which serve as checkpoint for investing and selling for investors. 

 

Data treatment 

To appropriately run a prediction model, the stream data needed to be pivoted, and additional columns such as open exchange rate on the day, as well as highs and lows needed to be queried into the dataset. The query below accomplishes the same and creates a new table to be used for modeling. 

CREATE OR REPLACE TABLE 

  `bd-final-project.coinbase_data.stream` AS 

WITH time_plus_unpivot AS 

( 

  SELECT currency, exchange_rate, new_time 

  FROM 

  ( 

    SELECT BTC, ETH, USDT, BNB, BUSD, XRP, DOGE, ADA, MATIC, DOT, DAI, LTC, TRX, SHIB, SOL, UNI, AVAX, WBTC, LINK, XMR, ATOM, TON, ETC, XLM, BCH, CRO, 

    ALGO, APE, TIMESTAMP(timestamp) AS new_time 

    FROM `bd-final-project.coinbase_data.streaming_data` 

  ) AS p 

  UNPIVOT 

  (exchange_rate FOR currency IN (BTC, ETH, USDT, BNB, BUSD, XRP, DOGE, ADA, MATIC, DOT, DAI, LTC, TRX, SHIB, SOL, UNI, AVAX, WBTC, LINK, XMR, ATOM, 

  TON, ETC, XLM, BCH, CRO, ALGO, APE) 

  ) AS unpvt 

  ORDER BY currency, new_time DESC 

) 

 

SELECT currency, EXTRACT(DATE FROM new_time) AS date_extract, exchange_rate, MAX(open_temp) OVER(PARTITION BY currency, date_) AS open, high, low, new_time, ((high-low)*0.382+low) AS fibo_level_1, ((high-low)*0.5+low) AS fibo_level_2, ((high-low)*0.618+low) AS fibo_level_3, ((high-low)*0.75+low) AS fibo_level_4 

FROM 

( 

  SELECT currency, exchange_rate, high, low, 

  CASE WHEN open_flag IS NULL THEN exchange_rate 

  ELSE 0 END AS open_temp, 

  date_, new_time 

  FROM 

  ( 

    SELECT currency, exchange_rate, 

    MAX(exchange_rate) OVER(PARTITION BY currency, date_) AS high, 

    MIN(exchange_rate) OVER(PARTITION BY currency, date_) AS low, 

    LAG(currency) OVER(PARTITION BY currency, date_ ORDER BY new_time) AS open_flag, 

    date_, new_time 

    FROM 

    ( 

      SELECT currency, exchange_rate, EXTRACT(DATE FROM new_time) AS date_, new_time 

      FROM time_plus_unpivot 

    ) 

  ) 

); 

 

Modeling 

The goal of the project is to predict the close volume and the close exchange rate for a given crypto currency. 

Linear and boosted tree models were used to run predictions for the required variables. The model evaluation is below. 

 

Linear Model for Volume Prediction 

 

 

 

 

 

Boosted Tree Model for Volume Prediction 

Table

Description automatically generated 

 

Linear Model for Close Prediction 

Table

Description automatically generated 

 

Boosted Tree Model for Close Prediction 

Table

Description automatically generated 

 

The Linear Model is selected for Volume prediction and the Boosted Tree Model is selected for Close prediction. 

Recommendations 

Product Marketing 

Market the product to crypto investors and users of Coinbase as a lower risk alternative to market intelligence for crypto investing. Augment the reliability of the prediction with Fibonacci retracement levels to validate and add value to the product. 

Product Improvement 

Model experimentation – Implement models such as LSTM that are known to work well with real-time predictions to improve the quality and reliability of predictions. 

Data improvements – Develop data collection quality to improve additional variables in the real time stream to enhance the model’s capability. 
 

Visualization 

Looker Studio link: https://datastudio.google.com/reporting/59719bab-af88-4dfb-b5e5-ae6ae6e8a0ac 

 

Presentation 

YouTube link: https://youtu.be/wTFHEya_J6k 

 
