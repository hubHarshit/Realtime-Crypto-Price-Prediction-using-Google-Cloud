
# Predicting Crypto Market Tendencies – Volatility as an Advantage 


 ![image](https://user-images.githubusercontent.com/105987080/208795393-41cae2c2-a534-481e-884d-718ac9a3083b.png)


For this project, our team is working on crypto currency data, where archived data is procured from the CoinMarketCap website, and live streaming data is procured from Coinbase API. The goal for the project is to provide a live prediction of market close volume sold across cryptos and closing exchange rate to generate insights that lead to quick profits in the Crypto Currency market. 

 

The steps followed by the team are as follows: 

 

Setting up the pipeline 

a. Create a cloud function to connect to the Coinbase parser from the Coinbase API 
![image](https://user-images.githubusercontent.com/105987080/208795556-b5c31fa7-f2b9-4969-810a-feae3a30be93.png)


b. Define a job schedule using the Cloud Scheduler with a trigger frequency of 10 minutes. The scheduler triggers the cloud function to transfer the JSON directly to the pubsub and a single dataflow transfers the data to BigQuery. The configuration is shown below. 


![image](https://user-images.githubusercontent.com/105987080/208795611-6606775c-85f7-4e3b-aeb1-a12fccba0c80.png)


![image](https://user-images.githubusercontent.com/105987080/208795665-76dff0e1-7ec7-4077-a46d-6fa4919d4d28.png)



    

c. Create a Pubsub 

The messages are pulled every time the job is triggered 

![image](https://user-images.githubusercontent.com/105987080/208795723-cbda2f46-a5e4-4aef-8200-336eea685c8e.png)


d. Store data in GCS 

With the given configuration in the scheduler, the streaming records are pushed into the GCS buckets as files. 
![image](https://user-images.githubusercontent.com/105987080/208795760-fb150afb-f2fe-4f77-9079-06cd4039f81c.png)

 

e. Create a dataflow: The debug logs depict the data being transferred to Big Query at each trigger 

![image](https://user-images.githubusercontent.com/105987080/208795846-7e209f3b-6c5d-4cc7-b22f-813f2487c2b1.png)


f. Dataflow jobs move data from GCS to BigQuery 

![image](https://user-images.githubusercontent.com/105987080/208795890-a13c6512-555d-4302-b910-fb5acf303f69.png)

 

BigQuery data set with live data for analysis 

Below is a screengrab of the live stream crypto dataset 

![image](https://user-images.githubusercontent.com/105987080/208797755-f79e6fd7-9031-4ac6-be8a-7ff384a72512.png)


 

Procuring archive data 
Archive data for crypto exchange rates was procured from the CoinMarketCap website. The data includes name of the crypto currency, the date for which data is procured, the opening exchange rate on the day, the closing exchange rate on the day, the daily high of the exchange rate, the daily low of the exchange rate, the adjusted closing exchange rate on the day and the volume sold on the day. 

A screengrab of the data is available below. 
![image](https://user-images.githubusercontent.com/105987080/208797796-ec0f98d5-0311-46e8-acc5-539d4bc3bbd0.png)

 

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

    FROM `tablename` 

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


![image](https://user-images.githubusercontent.com/105987080/208798088-6c34f567-9ca9-42e8-bc41-b803c1cf726f.png)

Boosted Tree Model for Volume Prediction 


![image](https://user-images.githubusercontent.com/105987080/208798112-a02387c8-c7b1-4d5e-a1d6-087f62b06a72.png)

Linear Model for Close Prediction 


![image](https://user-images.githubusercontent.com/105987080/208798134-6b4a1519-ee1f-4467-b547-0489d5760f0c.png)

Boosted Tree Model for Close Prediction 


![image](https://user-images.githubusercontent.com/105987080/208798158-78ace564-1d98-4d5a-aa19-ae5f2a1235fc.png) 

 

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

 
