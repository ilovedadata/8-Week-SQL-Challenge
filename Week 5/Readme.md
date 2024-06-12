### 1. Data Cleansing Steps

1. How many unique nodes are there on the Data Bank system?
~~~sql
CREATE TEMP TABLE weekly_sales_cleaned AS
 (SELECT to_date(week_date, 'DD-MM-YY') AS week_date,
 EXTRACT (WEEK FROM to_date(week_date, 'DD-MM-YY')) AS nr_week,
 EXTRACT (MONTH FROM to_date(week_date, 'DD-MM-YY')) AS date_month,
 EXTRACT (YEAR FROM to_date(week_date, 'DD-MM-YY')) AS date_year,
 CASE WHEN segment LIKE '%1%' THEN 'Young Adults'
 WHEN segment LIKE '%2%' THEN 'Middle Aged'
 WHEN segment LIKE '%4%' OR SEGMENT LIKE '%3%' THEN 'Retirees'
 ELSE NULL
 END AS age_band,
 CASE WHEN segment LIKE '%C%' THEN 'Couples'
 WHEN segment LIKE '%F%' THEN 'Families'
 ELSE NULL
 END AS demographic,
 ROUND(1.0*sales/transactions, 2) AS avg_transaction, region, platform, segment, customer_type, transactions, sales
 FROM data_mart.weekly_sales);

SELECT * FROM weekly_sales_cleaned 
LIMIT 10;
~~~

| week_date                | nr_week | date_month | date_year | age_band     | demographic | avg_transaction | region | platform | segment | customer_type | transactions | sales    |
| ------------------------ | ------- | ---------- | --------- | ------------ | ----------- | --------------- | ------ | -------- | ------- | ------------- | ------------ | -------- |
| 2020-08-31T00:00:00.000Z | 36      | 8          | 2020      | Retirees     | Couples     | 30.31           | ASIA   | Retail   | C3      | New           | 120631       | 3656163  |
| 2020-08-31T00:00:00.000Z | 36      | 8          | 2020      | Young Adults | Families    | 31.56           | ASIA   | Retail   | F1      | New           | 31574        | 996575   |
| 2020-08-31T00:00:00.000Z | 36      | 8          | 2020      |              |             | 31.20           | USA    | Retail   | null    | Guest         | 529151       | 16509610 |
| 2020-08-31T00:00:00.000Z | 36      | 8          | 2020      | Young Adults | Couples     | 31.42           | EUROPE | Retail   | C1      | New           | 4517         | 141942   |
| 2020-08-31T00:00:00.000Z | 36      | 8          | 2020      | Middle Aged  | Couples     | 30.29           | AFRICA | Retail   | C2      | New           | 58046        | 1758388  |
| 2020-08-31T00:00:00.000Z | 36      | 8          | 2020      | Middle Aged  | Families    | 182.54          | CANADA | Shopify  | F2      | Existing      | 1336         | 243878   |
| 2020-08-31T00:00:00.000Z | 36      | 8          | 2020      | Retirees     | Families    | 206.64          | AFRICA | Shopify  | F3      | Existing      | 2514         | 519502   |
| 2020-08-31T00:00:00.000Z | 36      | 8          | 2020      | Young Adults | Families    | 172.11          | ASIA   | Shopify  | F1      | Existing      | 2158         | 371417   |
| 2020-08-31T00:00:00.000Z | 36      | 8          | 2020      | Middle Aged  | Families    | 155.84          | AFRICA | Shopify  | F2      | New           | 318          | 49557    |
| 2020-08-31T00:00:00.000Z | 36      | 8          | 2020      | Retirees     | Couples     | 35.02           | AFRICA | Retail   | C3      | New           | 111032       | 3888162  |

### 2. Data Exploration

1. What day of the week is used for each week_date value?
~~~sql
SELECT nr_day, COUNT(nr_day) AS count_of_days
FROM
 (SELECT EXTRACT (DOW FROM week_date) AS nr_day
 FROM weekly_sales_cleaned) nr_day_subq
 GROUP BY nr_day;
~~~
| nr_day | count_of_days |
| ------ | ------------- |
| 1      | 17117         |

2. What range of week numbers are missing from the dataset?
~~~sql
SELECT DISTINCT week_date, nr_week
FROM weekly_sales_cleaned
ORDER BY nr_week;
/* Looking at the output, week numbers from 1 to 12 and from 37 to 52 */
~~~

| week_date                | nr_week |
| ------------------------ | ------- |
| 2020-03-23T00:00:00.000Z | 13      |
| 2018-03-26T00:00:00.000Z | 13      |
| 2019-03-25T00:00:00.000Z | 13      |
| 2020-03-30T00:00:00.000Z | 14      |
| 2019-04-01T00:00:00.000Z | 14      |
| 2018-04-02T00:00:00.000Z | 14      |
| 2020-04-06T00:00:00.000Z | 15      |
| 2019-04-08T00:00:00.000Z | 15      |
| 2018-04-09T00:00:00.000Z | 15      |
| 2018-04-16T00:00:00.000Z | 16      |
| 2020-04-13T00:00:00.000Z | 16      |
| 2019-04-15T00:00:00.000Z | 16      |
| 2020-04-20T00:00:00.000Z | 17      |
| 2019-04-22T00:00:00.000Z | 17      |
| 2018-04-23T00:00:00.000Z | 17      |
| 2019-04-29T00:00:00.000Z | 18      |
| 2018-04-30T00:00:00.000Z | 18      |
| 2020-04-27T00:00:00.000Z | 18      |
| 2020-05-04T00:00:00.000Z | 19      |
| 2019-05-06T00:00:00.000Z | 19      |
| 2018-05-07T00:00:00.000Z | 19      |
| 2019-05-13T00:00:00.000Z | 20      |
| 2020-05-11T00:00:00.000Z | 20      |
| 2018-05-14T00:00:00.000Z | 20      |
| 2020-05-18T00:00:00.000Z | 21      |
| 2018-05-21T00:00:00.000Z | 21      |
| 2019-05-20T00:00:00.000Z | 21      |
| 2018-05-28T00:00:00.000Z | 22      |
| 2020-05-25T00:00:00.000Z | 22      |
| 2019-05-27T00:00:00.000Z | 22      |
| 2020-06-01T00:00:00.000Z | 23      |
| 2019-06-03T00:00:00.000Z | 23      |
| 2018-06-04T00:00:00.000Z | 23      |
| 2018-06-11T00:00:00.000Z | 24      |
| 2020-06-08T00:00:00.000Z | 24      |
| 2019-06-10T00:00:00.000Z | 24      |
| 2020-06-15T00:00:00.000Z | 25      |
| 2019-06-17T00:00:00.000Z | 25      |
| 2018-06-18T00:00:00.000Z | 25      |
| 2019-06-24T00:00:00.000Z | 26      |
| 2018-06-25T00:00:00.000Z | 26      |
| 2020-06-22T00:00:00.000Z | 26      |
| 2018-07-02T00:00:00.000Z | 27      |
| 2020-06-29T00:00:00.000Z | 27      |
| 2019-07-01T00:00:00.000Z | 27      |
| 2020-07-06T00:00:00.000Z | 28      |
| 2018-07-09T00:00:00.000Z | 28      |
| 2019-07-08T00:00:00.000Z | 28      |
| 2019-07-15T00:00:00.000Z | 29      |
| 2018-07-16T00:00:00.000Z | 29      |
| 2020-07-13T00:00:00.000Z | 29      |
| 2020-07-20T00:00:00.000Z | 30      |
| 2018-07-23T00:00:00.000Z | 30      |
| 2019-07-22T00:00:00.000Z | 30      |
| 2018-07-30T00:00:00.000Z | 31      |
| 2020-07-27T00:00:00.000Z | 31      |
| 2019-07-29T00:00:00.000Z | 31      |
| 2018-08-06T00:00:00.000Z | 32      |
| 2020-08-03T00:00:00.000Z | 32      |
| 2019-08-05T00:00:00.000Z | 32      |
| 2018-08-13T00:00:00.000Z | 33      |
| 2020-08-10T00:00:00.000Z | 33      |
| 2019-08-12T00:00:00.000Z | 33      |
| 2019-08-19T00:00:00.000Z | 34      |
| 2020-08-17T00:00:00.000Z | 34      |
| 2018-08-20T00:00:00.000Z | 34      |
| 2019-08-26T00:00:00.000Z | 35      |
| 2018-08-27T00:00:00.000Z | 35      |
| 2020-08-24T00:00:00.000Z | 35      |
| 2020-08-31T00:00:00.000Z | 36      |
| 2019-09-02T00:00:00.000Z | 36      |
| 2018-09-03T00:00:00.000Z | 36      |

3. How many total transactions were there for each year in the dataset?
~~~sql
SELECT date_year, COUNT(transactions) AS nr_transact_year,
SUM(transactions) AS sum_transact_year
FROM weekly_sales_cleaned
GROUP BY date_year
ORDER BY date_year;
~~~

| date_year | nr_transact_year | sum_transact_year |
| --------- | ---------------- | ----------------- |
| 2018      | 5698             | 346406460         |
| 2019      | 5708             | 365639285         |
| 2020      | 5711             | 375813651         |

4. What is the total sales for each region for each month?
~~~sql
SELECT region, date_month, SUM(sales) AS tot_region_sales
FROM weekly_sales_cleaned
GROUP BY region, date_month
ORDER BY region, date_month;
~~~

| region        | date_month | tot_region_sales |
| ------------- | ---------- | ---------------- |
| AFRICA        | 3          | 567767480        |
| AFRICA        | 4          | 1911783504       |
| AFRICA        | 5          | 1647244738       |
| AFRICA        | 6          | 1767559760       |
| AFRICA        | 7          | 1960219710       |
| AFRICA        | 8          | 1809596890       |
| AFRICA        | 9          | 276320987        |
| ASIA          | 3          | 529770793        |
| ASIA          | 4          | 1804628707       |
| ASIA          | 5          | 1526285399       |
| ASIA          | 6          | 1619482889       |
| ASIA          | 7          | 1768844756       |
| ASIA          | 8          | 1663320609       |
| ASIA          | 9          | 252836807        |
| CANADA        | 3          | 144634329        |
| CANADA        | 4          | 484552594        |
| CANADA        | 5          | 412378365        |
| CANADA        | 6          | 443846698        |
| CANADA        | 7          | 477134947        |
| CANADA        | 8          | 447073019        |
| CANADA        | 9          | 69067959         |
| EUROPE        | 3          | 35337093         |
| EUROPE        | 4          | 127334255        |
| EUROPE        | 5          | 109338389        |
| EUROPE        | 6          | 122813826        |
| EUROPE        | 7          | 136757466        |
| EUROPE        | 8          | 122102995        |
| EUROPE        | 9          | 18877433         |
| OCEANIA       | 3          | 783282888        |
| OCEANIA       | 4          | 2599767620       |
| OCEANIA       | 5          | 2215657304       |
| OCEANIA       | 6          | 2371884744       |
| OCEANIA       | 7          | 2563459400       |
| OCEANIA       | 8          | 2432313652       |
| OCEANIA       | 9          | 372465518        |
| SOUTH AMERICA | 3          | 71023109         |
| SOUTH AMERICA | 4          | 238451531        |
| SOUTH AMERICA | 5          | 201391809        |
| SOUTH AMERICA | 6          | 218247455        |
| SOUTH AMERICA | 7          | 235582776        |
| SOUTH AMERICA | 8          | 221166052        |
| SOUTH AMERICA | 9          | 34175583         |
| USA           | 3          | 225353043        |
| USA           | 4          | 759786323        |
| USA           | 5          | 655967121        |
| USA           | 6          | 703878990        |
| USA           | 7          | 760331754        |
| USA           | 8          | 712002790        |
| USA           | 9          | 110532368        |

5. What is the total count of transactions for each platform
~~~sql
SELECT platform, COUNT(transactions) AS count_transact
FROM weekly_sales_cleaned
GROUP BY platform
ORDER BY platform;
~~~

| platform | count_transact |
| -------- | -------------- |
| Retail   | 8568           |
| Shopify  | 8549           |

6. What is the percentage of sales for Retail vs Shopify for each month?
~~~sql
SELECT date_year, date_month, ROUND(100.0*retail_sales/(retail_sales+shopify_sales), 2) AS pct_ret_sales,
ROUND(100.0*shopify_sales/(retail_sales+shopify_sales), 2) AS pct_shop_sales
FROM
 (SELECT date_year, date_month, 
 SUM(CASE WHEN platform = 'Retail' THEN sales ELSE NULL END) AS retail_sales,
 SUM(CASE WHEN platform = 'Shopify' THEN sales ELSE NULL END) AS shopify_sales
 FROM weekly_sales_cleaned
 GROUP BY date_year, date_month) ret_shop_subq
ORDER BY date_year, date_month;
~~~

| date_year | date_month | pct_ret_sales | pct_shop_sales |
| --------- | ---------- | ------------- | -------------- |
| 2018      | 3          | 97.92         | 2.08           |
| 2018      | 4          | 97.93         | 2.07           |
| 2018      | 5          | 97.73         | 2.27           |
| 2018      | 6          | 97.76         | 2.24           |
| 2018      | 7          | 97.75         | 2.25           |
| 2018      | 8          | 97.71         | 2.29           |
| 2018      | 9          | 97.68         | 2.32           |
| 2019      | 3          | 97.71         | 2.29           |
| 2019      | 4          | 97.80         | 2.20           |
| 2019      | 5          | 97.52         | 2.48           |
| 2019      | 6          | 97.42         | 2.58           |
| 2019      | 7          | 97.35         | 2.65           |
| 2019      | 8          | 97.21         | 2.79           |
| 2019      | 9          | 97.09         | 2.91           |
| 2020      | 3          | 97.30         | 2.70           |
| 2020      | 4          | 96.96         | 3.04           |
| 2020      | 5          | 96.71         | 3.29           |
| 2020      | 6          | 96.80         | 3.20           |
| 2020      | 7          | 96.67         | 3.33           |
| 2020      | 8          | 96.51         | 3.49           |

7. What is the percentage of sales by demographic for each year in the dataset?
~~~sql
SELECT date_year, 
ROUND(100.0*coup_sales/(coup_sales+fam_sales+null_sales), 2) AS pct_coup_sales,
ROUND(100.0*fam_sales/(coup_sales+fam_sales+null_sales), 2) AS pct_fam_sales,
ROUND(100.0*null_sales/(coup_sales+fam_sales+null_sales), 2) AS pct_null_sales
FROM
 (SELECT date_year, 
 SUM(CASE WHEN demographic = 'Couples' THEN sales ELSE NULL END) AS coup_sales,
 SUM(CASE WHEN demographic = 'Families' THEN sales ELSE NULL END) AS fam_sales,
 SUM(CASE WHEN demographic IS NULL THEN sales ELSE NULL END) AS null_sales
 FROM weekly_sales_cleaned
 GROUP BY date_year) coup_fam_subq
ORDER BY date_year;
~~~

| date_year | pct_coup_sales | pct_fam_sales | pct_null_sales |
| --------- | -------------- | ------------- | -------------- |
| 2018      | 26.38          | 31.99         | 41.63          |
| 2019      | 27.28          | 32.47         | 40.25          |
| 2020      | 28.72          | 32.73         | 38.55          |

8. Which age_band and demographic values contribute the most to Retail sales?
~~~sql
SELECT age_band, demographic, 
ROUND(100.0*SUM(sales)/(SELECT SUM(sales) FROM weekly_sales_cleaned WHERE platform='Retail')) AS fraction_sales
FROM weekly_sales_cleaned
WHERE platform = 'Retail'
GROUP BY age_band, demographic
ORDER BY fraction_sales DESC;
~~~

| age_band     | demographic | fraction_sales |
| ------------ | ----------- | -------------- |
|              |             | 41             |
| Retirees     | Families    | 17             |
| Retirees     | Couples     | 16             |
| Middle Aged  | Families    | 11             |
| Young Adults | Couples     | 7              |
| Middle Aged  | Couples     | 5              |
| Young Adults | Families    | 4              |

9. Can we use the avg_transaction column to find the average transaction size for each year for Retail vs Shopify? If not - how would you calculate it instead?
~~~sql
/* No, to find the average, sum all the sales and divide by the sum of all the transactions.*/
SELECT date_year, platform, 
ROUND(1.0*SUM(sales)/SUM(transactions), 2)
FROM weekly_sales_cleaned
GROUP BY date_year, platform
ORDER BY date_year, platform;
~~~

| date_year | platform | round  |
| --------- | -------- | ------ |
| 2018      | Retail   | 36.56  |

### 3. Before & After Analysis
Taking the week_date value of 2020-06-15 as the baseline week where the Data Mart sustainable packaging changes came into effect. We would include all week_date values for 2020-06-15 as the start of the period after the change and the previous week_date values would be before
Using this analysis approach - answer the following questions:

1. What is the total sales for the 4 weeks before and after 2020-06-15? What is the growth or reduction rate in actual values and percentage of sales?
~~~sql
SELECT *,
before_sales + after_sales AS tot_sales,
after_sales - before_sales AS growth,
ROUND(ABS((100.0*after_sales/before_sales)), 2) - 100 AS growth_rate
FROM
 (SELECT SUM(CASE WHEN nr_week BETWEEN 
 (SELECT DISTINCT nr_week FROM weekly_sales_cleaned WHERE week_date = '2020-06-15') AND (SELECT DISTINCT nr_week + 3 FROM weekly_sales_cleaned WHERE week_date = '2020-06-15') AND date_year = '2020' THEN sales ELSE NULL END) AS after_sales, SUM(CASE WHEN nr_week BETWEEN (SELECT DISTINCT nr_week - 4 FROM weekly_sales_cleaned WHERE week_date = '2020-06-15') AND (SELECT DISTINCT nr_week - 1 FROM weekly_sales_cleaned WHERE week_date = '2020-06-15') AND date_year = '2020' THEN sales ELSE NULL END) AS before_sales
 FROM weekly_sales_cleaned) bef_aft;
~~~

| after_sales | before_sales | tot_sales  | growth    | growth_rate |
| ----------- | ------------ | ---------- | --------- | ----------- |
| 2318994169  | 2345878357   | 4664872526 | -26884188 | -1.15       |

2. What about the entire 12 weeks before and after?
~~~sql
SELECT *,
before_sales + after_sales AS tot_sales,
after_sales - before_sales AS growth,
ROUND(ABS((100.0*after_sales/before_sales)), 2) - 100 AS growth_rate
FROM
 (SELECT SUM(CASE WHEN nr_week BETWEEN (SELECT DISTINCT nr_week FROM weekly_sales_cleaned WHERE week_date = '2020-06-15') AND (SELECT DISTINCT nr_week + 11 FROM weekly_sales_cleaned WHERE week_date = '2020-06-15') AND date_year = '2020' THEN sales ELSE NULL END) AS after_sales, SUM(CASE WHEN nr_week BETWEEN (SELECT DISTINCT nr_week - 12 FROM weekly_sales_cleaned WHERE week_date = '2020-06-15') AND (SELECT DISTINCT nr_week - 1 FROM weekly_sales_cleaned WHERE week_date = '2020-06-15') AND date_year = '2020' THEN sales ELSE NULL END) AS before_sales 
 FROM weekly_sales_cleaned) bef_aft;
~~~

| after_sales | before_sales | tot_sales   | growth     | growth_rate |
| ----------- | ------------ | ----------- | ---------- | ----------- |
| 6973947753  | 7126273147   | 14100220900 | -152325394 | -2.14       |

3. How do the sale metrics for these 2 periods before and after compare with the previous years in 2018 and 2019?
~~~sql
SELECT *,
before_sales + after_sales AS tot_sales,
after_sales - before_sales AS growth,
ROUND(ABS((100.0*after_sales/before_sales)), 2) - 100 AS growth_rate
FROM
 (SELECT date_year,
 SUM(CASE WHEN nr_week BETWEEN 21 AND 24 THEN sales ELSE NULL END) AS before_sales,
 SUM(CASE WHEN nr_week BETWEEN 25 AND 28 THEN sales ELSE NULL END) AS after_sales
 FROM weekly_sales_cleaned
 GROUP BY date_year) bef_aft;
~~~

| date_year | before_sales | after_sales | tot_sales  | growth    | growth_rate |
| --------- | ------------ | ----------- | ---------- | --------- | ----------- |
| 2019      | 2249989796   | 2252326390  | 4502316186 | 2336594   | 0.10        |
| 2018      | 2125140809   | 2129242914  | 4254383723 | 4102105   | 0.19        |
| 2020      | 2345878357   | 2318994169  | 4664872526 | -26884188 | -1.15       |

~~~sql
SELECT *,
before_sales + after_sales AS tot_sales,
after_sales - before_sales AS growth,
ROUND(ABS((100.0*after_sales/before_sales)), 2) - 100 AS growth_rate
FROM
 (SELECT date_year,
 SUM(CASE WHEN nr_week BETWEEN 13 AND 24 THEN sales ELSE NULL END) AS before_sales,
 SUM(CASE WHEN nr_week BETWEEN 25 AND 37 THEN sales ELSE NULL END) AS after_sales
 FROM weekly_sales_cleaned
 GROUP BY date_year
 ORDER BY date_year) bef_aft;
~~~

| date_year | before_sales | after_sales | tot_sales   | growth     | growth_rate |
| --------- | ------------ | ----------- | ----------- | ---------- | ----------- |
| 2018      | 6396562317   | 6500818510  | 12897380827 | 104256193  | 1.63        |
| 2019      | 6883386397   | 6862646103  | 13746032500 | -20740294  | -0.30       |
| 2020      | 7126273147   | 6973947753  | 14100220900 | -152325394 | -2.14       |

### 4. Bonus Question
Which areas of the business have the highest negative impact in sales metrics performance in 2020 for the 12 week before and after period?
Do you have any further recommendations for Danny’s team at Data Mart or any interesting insights based off this analysis?

1. What day of the week is used for each week_date value?
~~~sql
# Focusing on the 10 biggest losses in absolute value, the critical market seems to be in oceania and asia, where the retail platform is losing the most money. Particular attention should be paid to the Retirees age band and to the Guest customer type.
SELECT * FROM
 (SELECT *,
 before_sales + after_sales AS tot_sales,
 after_sales - before_sales AS growth,
 ROUND(ABS((100.0*after_sales/before_sales)), 2) - 100 AS growth_rate
 FROM
  (SELECT date_year, region, platform, age_band, demographic, customer_type,
  SUM(CASE WHEN nr_week BETWEEN 13 AND 24 THEN sales ELSE NULL END) AS before_sales,
  SUM(CASE WHEN nr_week BETWEEN 25 AND 37 THEN sales ELSE NULL END) AS after_sales
  FROM weekly_sales_cleaned
  GROUP BY date_year, region, platform, age_band, demographic, customer_type) bef_aft
 WHERE date_year = '2020') only_2020_subq
ORDER BY growth, region, platform, age_band, demographic, customer_type
LIMIT 10;
~~~

| date_year | region  | platform | age_band    | demographic | customer_type | before_sales | after_sales | tot_sales  | growth    | growth_rate |
| --------- | ------- | -------- | ----------- | ----------- | ------------- | ------------ | ----------- | ---------- | --------- | ----------- |
| 2020      | OCEANIA | Retail   |             |             | Guest         | 793203251    | 760352031   | 1553555282 | -32851220 | -4.14       |
| 2020      | ASIA    | Retail   |             |             | Guest         | 605059313    | 576625191   | 1181684504 | -28434122 | -4.70       |
| 2020      | OCEANIA | Retail   | Retirees    | Couples     | Existing      | 298429377    | 286558374   | 584987751  | -11871003 | -3.98       |
| 2020      | OCEANIA | Retail   | Retirees    | Families    | Existing      | 365491363    | 355341719   | 720833082  | -10149644 | -2.78       |
| 2020      | OCEANIA | Retail   | Middle Aged | Families    | Existing      | 222107094    | 212140211   | 434247305  | -9966883  | -4.49       |
| 2020      | ASIA    | Retail   | Retirees    | Couples     | Existing      | 200058271    | 190527375   | 390585646  | -9530896  | -4.76       |
| 2020      | AFRICA  | Retail   |             |             | Guest         | 541926177    | 533702201   | 1075628378 | -8223976  | -1.52       |
| 2020      | USA     | Retail   |             |             | Guest         | 204306790    | 197203239   | 401510029  | -7103551  | -3.48       |
| 2020      | ASIA    | Retail   | Retirees    | Families    | Existing      | 244092189    | 237915427   | 482007616  | -6176762  | -2.53       |
| 2020      | ASIA    | Retail   | Middle Aged | Families    | Existing      | 131698799    | 125714950   | 257413749  | -5983849  | -4.54       |
