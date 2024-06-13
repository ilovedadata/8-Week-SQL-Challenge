### 2. Digital Analysis

1. How many users are there?
~~~sql
SELECT COUNT(DISTINCT user_id)  AS count_of_user
FROM clique_bait.users;
~~~
| count_of_user |
| ------------- |
| 500           |

2. How many cookies does each user have on average?
~~~sql
SELECT ROUND(AVG(nr_cookies), 2) AS avg_cookies
FROM
 (SELECT user_id, COUNT(DISTINCT cookie_id)  AS nr_cookies
 FROM clique_bait.users
 GROUP BY user_id) nr_cookies_subq;
~~~
| avg_cookies |
| ----------- |
| 3.56        |

~~~sql
# Alternative
SELECT  ROUND(COUNT(DISTINCT cookie_id)*1.0/COUNT(DISTINCT user_id), 2) AS avg_cookies
FROM clique_bait.users;
~~~
| avg_cookies |
| ----------- |
| 3.56        |

3. What is the unique number of visits by all users per month?
~~~sql
SELECT event_month, SUM(count_of_visits) AS tot_visits
FROM
 (SELECT user_id, event_month, COUNT(DISTINCT visit_id) AS count_of_visits
 FROM
  (SELECT visit_id, user_id, 
  EXTRACT(MONTH FROM event_time) AS event_month
  FROM
   (SELECT * 
   FROM clique_bait.events e
   JOIN clique_bait.users u
   USING (cookie_id)) merge_subq) month_subq
  GROUP BY user_id, event_month) grouped_subq
GROUP BY event_month
ORDER BY event_month;
~~~
| event_month | tot_visits |
| ----------- | ---------- |
| 1           | 876        |
| 2           | 1488       |
| 3           | 916        |
| 4           | 248        |
| 5           | 36         |

4. What is the number of events for each event type?
~~~sql
SELECT event_name, COUNT(event_name)
FROM clique_bait.events e
JOIN clique_bait.event_identifier ei
USING(event_type)
GROUP BY (event_name)
ORDER BY event_name;
~~~
| event_name    | count |
| ------------- | ----- |
| Ad Click      | 702   |
| Ad Impression | 876   |
| Add to Cart   | 8451  |
| Page View     | 20928 |
| Purchase      | 1777  |

5. What is the percentage of visits which have a purchase event?
~~~sql
SELECT ROUND(100.0*COUNT(DISTINCT visit_id)/(SELECT COUNT(DISTINCT visit_id) FROM clique_bait.events e), 2) AS pct_purchases
FROM
 (SELECT visit_id, event_type, event_name
 FROM clique_bait.events e
 JOIN clique_bait.event_identifier ei
 USING(event_type)
 WHERE event_name = 'Purchase') purchase_subq;
~~~
| pct_purchases |
| ------------- |
| 49.86         |

6. What is the percentage of visits which view the checkout page but do not have a purchase event?
~~~sql
SELECT COUNT (CASE WHEN page_name = 'Checkout' AND event_name IS NULL THEN 1 ELSE NULL END) AS not_purchase, COUNT(CASE WHEN 1=1 THEN 1 ELSE NULL END) AS tot_checkout, ROUND(100.0 * COUNT (CASE WHEN page_name = 'Checkout' AND event_name IS NULL THEN 1 ELSE NULL END) / COUNT (CASE WHEN 1=1 THEN 1 ELSE NULL END)) AS pct_not_purchase
FROM
 (SELECT * FROM
  (SELECT visit_id, page_id, page_name
  FROM clique_bait.events e
  JOIN clique_bait.page_hierarchy ph
  USING(page_id)
  WHERE page_name = 'Checkout') checkout_subq  
 LEFT JOIN   
  (SELECT visit_id, event_type, event_name
  FROM clique_bait.events e
  JOIN clique_bait.event_identifier ei
  USING(event_type)
  WHERE event_name = 'Purchase') purchase_subq
 USING (visit_id)) check_purch_subq;
~~~
| not_purchase | tot_checkout | pct_not_purchase |
| ------------ | ------------ | ---------------- |
| 326          | 2103         | 16               |

7. What are the top 3 pages by number of views?
~~~sql
SELECT *
FROM
 (SELECT page_id, COUNT(DISTINCT visit_id) AS count_of_visits
 FROM clique_bait.events 
 WHERE event_type = 1
 GROUP BY page_id) count_of_visits_subq
JOIN
 (SELECT page_id, page_name FROM clique_bait.page_hierarchy) page_subq  
 USING(page_id)
ORDER BY count_of_visits DESC
LIMIT 3;
~~~
| page_id | count_of_visits | page_name    |
| ------- | --------------- | ------------ |
| 2       | 3174            | All Products |
| 12      | 2103            | Checkout     |
| 1       | 1782            | Home Page    |

8. What is the number of views and cart adds for each product category?
~~~sql
SELECT product_category, event_type, SUM(count_of_visits) AS tot_visits
FROM
 (SELECT *
 FROM
  (SELECT page_id, event_type, COUNT(DISTINCT visit_id) AS count_of_visits
  FROM clique_bait.events 
  WHERE event_type = 1 OR event_type = 2
  GROUP BY page_id, event_type) count_of_visits_subq
 JOIN
  (SELECT page_id, product_category 
  FROM clique_bait.page_hierarchy
  WHERE product_category IS NOT NULL) product_subq  
 USING(page_id)) page_subq
    
GROUP BY product_category, event_type
ORDER BY product_category, event_type;
~~~
| product_category | event_type | tot_visits |
| ---------------- | ---------- | ---------- |
| Fish             | 1          | 4633       |
| Fish             | 2          | 2789       |
| Luxury           | 1          | 3032       |
| Luxury           | 2          | 1870       |
| Shellfish        | 1          | 6204       |
| Shellfish        | 2          | 3792       |

9. What are the top 3 products by purchases?
~~~sql
SELECT page_name, purchases_cnt    
FROM
 (SELECT page_id, COUNT(page_id) AS purchases_cnt
 FROM 
  (SELECT *
  FROM clique_bait.events e
  WHERE event_type = 2) cart_adds    
 JOIN 
  (SELECT visit_id
  FROM clique_bait.events e
  WHERE event_type = 3) purchases
 USING(visit_id)
 GROUP BY page_id
 ORDER BY purchases_cnt DESC
 LIMIT 3) top_3
JOIN
clique_bait.page_hierarchy 
USING(page_id);
~~~
| page_name | purchases_cnt |
| --------- | ------------- |
| Lobster   | 754           |
| Crab      | 719           |
| Oyster    | 726           |

### 3. Product Funnel Analysis
Using a single SQL query - create a new output table which has the following details:
-How many times was each product viewed?
-How many times was each product added to cart?
-How many times was each product added to a cart but not purchased (abandoned)?
-How many times was each product purchased?
Additionally, create another table which further aggregates the data for the above points but this time for each product category instead of individual products.

~~~sql
CREATE TEMP TABLE all_things_products AS
 (SELECT * FROM
  (SELECT page_id, 
  COUNT(CASE WHEN event_type = 1 THEN 1 ELSE NULL END) AS times_viewed,
  COUNT(CASE WHEN event_type = 2 THEN 1 ELSE NULL END) AS times_added_to_cart,
  COUNT(CASE WHEN event_type = 2 AND visit_id NOT IN (SELECT visit_id FROM clique_bait.events e WHERE event_type = 3) THEN 1 ELSE NULL END) AS cartadd_not_purchases, COUNT(CASE WHEN event_type = 2 AND visit_id IN (SELECT visit_id FROM clique_bait.events e WHERE event_type = 3) THEN 1 ELSE NULL END) AS cartadd_purchases
  FROM clique_bait.events e
  GROUP BY page_id) stats_subq
 JOIN clique_bait.page_hierarchy ph
 USING(page_id)
 WHERE product_category IS NOT NULL
 ORDER BY page_id);
~~~
~~~sql
SELECT * FROM all_things_products;
~~~
| page_id | times_viewed | times_added_to_cart | cartadd_not_purchases | cartadd_purchases | page_name      | product_category | product_id |
| ------- | ------------ | ------------------- | --------------------- | ----------------- | -------------- | ---------------- | ---------- |
| 3       | 1559         | 938                 | 227                   | 711               | Salmon         | Fish             | 1          |
| 4       | 1559         | 920                 | 213                   | 707               | Kingfish       | Fish             | 2          |
| 5       | 1515         | 931                 | 234                   | 697               | Tuna           | Fish             | 3          |
| 6       | 1563         | 946                 | 249                   | 697               | Russian Caviar | Luxury           | 4          |
| 7       | 1469         | 924                 | 217                   | 707               | Black Truffle  | Luxury           | 5          |
| 8       | 1525         | 932                 | 233                   | 699               | Abalone        | Shellfish        | 6          |
| 9       | 1547         | 968                 | 214                   | 754               | Lobster        | Shellfish        | 7          |
| 10      | 1564         | 949                 | 230                   | 719               | Crab           | Shellfish        | 8          |
| 11      | 1568         | 943                 | 217                   | 726               | Oyster         | Shellfish        | 9          |

~~~sql
SELECT product_category, SUM(times_viewed) AS tot_view, SUM(times_added_to_cart) AS tot_added_to_cart, SUM(cartadd_not_purchases) AS tot_added_not_purch, 
SUM(cartadd_purchases) AS tot_added_purch
FROM all_things_products
GROUP BY product_category;
~~~

| product_category | tot_view | tot_added_to_cart | tot_added_not_purch | tot_added_purch |
| ---------------- | -------- | ----------------- | ------------------- | --------------- |
| Luxury           | 3032     | 1870              | 466                 | 1404            |
| Shellfish        | 6204     | 3792              | 894                 | 2898            |
| Fish             | 4633     | 2789              | 674                 | 2115            |

Use your 2 new output tables - answer the following questions:
1. Which product had the most views, cart adds and purchases?
Oyster, Lobster, Lobster
~~~sql
SELECT page_name, SUM(times_viewed) AS tot_view
FROM all_things_products
GROUP BY page_name
ORDER BY tot_view DESC
LIMIT 1;
~~~
| page_name | tot_view |
| --------- | -------- |
| Oyster    | 1568     |

~~~sql
SELECT page_name, SUM(times_added_to_cart) AS tot_a2c
FROM all_things_products
GROUP BY page_name
ORDER BY tot_a2c DESC
LIMIT 1;
~~~
| page_name | tot_a2c |
| --------- | ------- |
| Lobster   | 968     |

~~~sql
SELECT page_name, SUM(cartadd_purchases) AS tot_purch
FROM all_things_products
GROUP BY page_name
ORDER BY tot_purch DESC
LIMIT 1;
~~~
| page_name | tot_purch |
| --------- | --------- |
| Lobster   | 754       |

2. Which product was most likely to be abandoned?
Russian Caviar
~~~sql
SELECT page_name, SUM(cartadd_purchases) AS tot_purch,
SUM(times_added_to_cart) AS tot_a2c, 
ROUND(100.0*SUM(cartadd_purchases)/SUM(times_added_to_cart)) AS pct_bought
FROM all_things_products
GROUP BY page_name
ORDER BY pct_bought 
LIMIT 1;
~~~
| page_name      | tot_purch | tot_a2c | pct_bought |
| -------------- | --------- | ------- | ---------- |
| Russian Caviar | 697       | 946     | 74         |

4. Which product had the highest view to purchase percentage?
~~~sql
SELECT page_name, product_category, 
ROUND(100.0*cartadd_purchases/times_viewed, 1) AS pct_purch_view
FROM all_things_products;
~~~
| page_name      | product_category | pct_purch_view |
| -------------- | ---------------- | -------------- |
| Salmon         | Fish             | 45.6           |
| Kingfish       | Fish             | 45.3           |
| Tuna           | Fish             | 46.0           |
| Russian Caviar | Luxury           | 44.6           |
| Black Truffle  | Luxury           | 48.1           |
| Abalone        | Shellfish        | 45.8           |
| Lobster        | Shellfish        | 48.7           |
| Crab           | Shellfish        | 46.0           |
| Oyster         | Shellfish        | 46.3           |

4. What is the average conversion rate from view to cart add?
5. What is the average conversion rate from cart add to purchase?
~~~sql
SELECT ROUND(AVG(100.0*times_added_to_cart/times_viewed),1) AS avg_cart_conv,
ROUND(AVG(100.0*cartadd_purchases/times_added_to_cart),1) AS avg_purch_conv
FROM all_things_products
~~~
| avg_cart_conv | avg_purch_conv |
| ------------- | -------------- |
| 61.0          | 75.9           |

1. How many users are there?
~~~sql
SELECT COUNT(DISTINCT user_id)  AS count_of_user
FROM clique_bait.users;
~~~
| count_of_user |
| ------------- |
| 500           |

### 3. Campaigns Analysis
~~~sql
CREATE TEMP TABLE row_numb_events AS
 (SELECT *, ROW_NUMBER() OVER(PARTITION BY visit_id ORDER BY visit_id, event_time)
 FROM clique_bait.events 
 JOIN clique_bait.page_hierarchy USING(page_id)
 WHERE event_type = 2);
~~~
~~~sql
CREATE TEMP TABLE concatenated_purchases AS
 (SELECT visit_id, cart_products FROM
  (SELECT visit_id, 
  CONCAT(page_name,',',page_name_2,',',page_name_3,',',page_name_4,',',page_name_5,',',page_name_6,',',page_name_7,',',page_name_8,',',page_name_9) AS cart_products, ROW_NUMBER() OVER(PARTITION BY visit_id ORDER BY visit_id)     
  FROM
   (SELECT *, 
   LEAD(page_name, 1) OVER(PARTITION BY visit_id ORDER BY visit_id, event_time) AS page_name_2,
   LEAD(page_name, 2) OVER(PARTITION BY visit_id ORDER BY visit_id, event_time) AS page_name_3,
   LEAD(page_name, 3) OVER(PARTITION BY visit_id ORDER BY visit_id, event_time) AS page_name_4,
   LEAD(page_name, 4) OVER(PARTITION BY visit_id ORDER BY visit_id, event_time) AS page_name_5,
   LEAD(page_name, 5) OVER(PARTITION BY visit_id ORDER BY visit_id, event_time) AS page_name_6,
   LEAD(page_name, 6) OVER(PARTITION BY visit_id ORDER BY visit_id, event_time) AS page_name_7,
   LEAD(page_name, 7) OVER(PARTITION BY visit_id ORDER BY visit_id, event_time) AS page_name_8,
   LEAD(page_name, 8) OVER(PARTITION BY visit_id ORDER BY visit_id, event_time) AS page_name_9
   FROM row_numb_events) leads_subq) all_num_rows_subq
  WHERE row_number=1);
~~~
~~~sql
SELECT *, 
CASE WHEN visit_start_time BETWEEN (SELECT start_date FROM clique_bait.campaign_identifier WHERE campaign_id = 1) AND (SELECT end_date FROM clique_bait.campaign_identifier WHERE campaign_id = 1) THEN (SELECT campaign_name FROM clique_bait.campaign_identifier WHERE campaign_id = 1) WHEN visit_start_time BETWEEN (SELECT start_date FROM clique_bait.campaign_identifier WHERE campaign_id = 2) AND (SELECT end_date FROM clique_bait.campaign_identifier WHERE campaign_id = 2) THEN (SELECT campaign_name FROM clique_bait.campaign_identifier WHERE campaign_id = 2) WHEN visit_start_time BETWEEN (SELECT start_date FROM clique_bait.campaign_identifier WHERE campaign_id = 3) AND (SELECT end_date FROM clique_bait.campaign_identifier WHERE campaign_id = 3) THEN (SELECT campaign_name FROM clique_bait.campaign_identifier WHERE campaign_id = 3) ELSE NULL END AS campaign_name
FROM
 (SELECT visit_id, 
 MIN(event_time) AS visit_start_time, 
 COUNT(CASE WHEN event_type = 1 THEN 1 ELSE NULL END) AS page_views,
 COUNT(CASE WHEN event_type = 2 THEN 1 ELSE NULL END) AS cart_adds,
 MAX(CASE WHEN event_type = 3 THEN 1 ELSE 0 END) AS purchase,
 COUNT(CASE WHEN event_type = 4 THEN 1 ELSE NULL END) AS impression,
 COUNT(CASE WHEN event_type = 5 THEN 1 ELSE NULL END) AS click
 FROM clique_bait.events 
 GROUP BY visit_id) all_but_user_subq   

JOIN
 (SELECT DISTINCT visit_id, user_id
 FROM
  (SELECT visit_id, cookie_id FROM clique_bait.events) e
  JOIN clique_bait.users
  USING(cookie_id)) user_subq
USING(visit_id)   

LEFT JOIN 
 (SELECT visit_id, cart_products FROM concatenated_purchases) just_first_row_purch
USING(visit_id)
ORDER BY user_id, visit_id
LIMIT 10;
~~~
| visit_id | visit_start_time         | page_views | cart_adds | purchase | impression | click | user_id | cart_products                                                         | campaign_name                     |
| -------- | ------------------------ | ---------- | --------- | -------- | ---------- | ----- | ------- | --------------------------------------------------------------------- | --------------------------------- |
| 02a5d5   | 2020-02-26T16:57:26.260Z | 4          | 0         | 0        | 0          | 0     | 1       |                                                                       | Half Off - Treat Your Shellf(ish) |
| 0826dc   | 2020-02-26T05:58:37.918Z | 1          | 0         | 0        | 0          | 0     | 1       |                                                                       | Half Off - Treat Your Shellf(ish) |
| 0fc437   | 2020-02-04T17:49:49.602Z | 10         | 6         | 1        | 1          | 1     | 1       | Tuna,Russian Caviar,Black Truffle,Abalone,Crab,Oyster,,,              | Half Off - Treat Your Shellf(ish) |
| 30b94d   | 2020-03-15T13:12:54.023Z | 9          | 7         | 1        | 1          | 1     | 1       | Salmon,Kingfish,Tuna,Russian Caviar,Abalone,Lobster,Crab,,            | Half Off - Treat Your Shellf(ish) |
| 41355d   | 2020-03-25T00:11:17.860Z | 6          | 1         | 0        | 0          | 0     | 1       | Lobster,,,,,,,,                                                       | Half Off - Treat Your Shellf(ish) |
| ccf365   | 2020-02-04T19:16:09.182Z | 7          | 3         | 1        | 0          | 0     | 1       | Lobster,Crab,Oyster,,,,,,                                             | Half Off - Treat Your Shellf(ish) |
| eaffde   | 2020-03-25T20:06:32.342Z | 10         | 8         | 1        | 1          | 1     | 1       | Salmon,Tuna,Russian Caviar,Black Truffle,Abalone,Lobster,Crab,Oyster, | Half Off - Treat Your Shellf(ish) |
| f7c798   | 2020-03-15T02:23:26.312Z | 9          | 3         | 1        | 0          | 0     | 1       | Russian Caviar,Crab,Oyster,,,,,,                                      | Half Off - Treat Your Shellf(ish) |
| 0635fb   | 2020-02-16T06:42:42.735Z | 9          | 4         | 1        | 0          | 0     | 2       | Salmon,Kingfish,Abalone,Crab,,,,,                                     | Half Off - Treat Your Shellf(ish) |
| 1f1198   | 2020-02-01T21:51:55.078Z | 1          | 0         | 0        | 0          | 0     | 2       |                                                                       | Half Off - Treat Your Shellf(ish) |
