### High Level Sales Analysis
1. What was the total quantity sold for all products?
~~~sql
SELECT product_name, qty_sold
FROM
 (SELECT product_id, product_name FROM balanced_tree.product_details) name_subq 
    
JOIN
    
 (SELECT prod_id AS product_id, SUM(qty) AS qty_sold
 FROM balanced_tree.sales 
 GROUP BY prod_id) qty_subq    
USING (product_id)
ORDER BY product_name
LIMIT 10;
~~~
| product_name                     | qty_sold |
| -------------------------------- | -------- |
| Black Straight Jeans - Womens    | 3786     |
| Blue Polo Shirt - Mens           | 3819     |
| Cream Relaxed Jeans - Womens     | 3707     |
| Grey Fashion Jacket - Womens     | 3876     |
| Indigo Rain Jacket - Womens      | 3757     |
| Khaki Suit Jacket - Womens       | 3752     |
| Navy Oversized Jeans - Womens    | 3856     |
| Navy Solid Socks - Mens          | 3792     |
| Pink Fluro Polkadot Socks - Mens | 3770     |
| Teal Button Up Shirt - Mens      | 3646     |

2. What is the total generated revenue for all products before discounts?
~~~sql
SELECT product_name, qty_sold*price AS tot_revenue
FROM
 (SELECT product_id, product_name, price FROM balanced_tree.product_details) name_subq 
    
JOIN
    
 (SELECT prod_id AS product_id, SUM(qty) AS qty_sold
 FROM balanced_tree.sales 
 GROUP BY prod_id) qty_subq
 USING (product_id)
 ORDER BY product_name
 LIMIT 10;
~~~
| product_name                     | tot_revenue |
| -------------------------------- | ----------- |
| Black Straight Jeans - Womens    | 121152      |
| Blue Polo Shirt - Mens           | 217683      |
| Cream Relaxed Jeans - Womens     | 37070       |
| Grey Fashion Jacket - Womens     | 209304      |
| Indigo Rain Jacket - Womens      | 71383       |
| Khaki Suit Jacket - Womens       | 86296       |
| Navy Oversized Jeans - Womens    | 50128       |
| Navy Solid Socks - Mens          | 136512      |
| Pink Fluro Polkadot Socks - Mens | 109330      |
| Teal Button Up Shirt - Mens      | 36460       |

3. What was the total discount amount for all products?
~~~sql
SELECT product_id, tot_disc_amt, product_name
FROM
 (SELECT prod_id AS product_id, SUM(price*discount/100) AS tot_disc_amt
 FROM balanced_tree.sales 
 GROUP BY prod_id) disc_subq

JOIN balanced_tree.product_details 
USING(product_id)
ORDER BY product_name;
~~~
| product_id | tot_disc_amt | product_name                     |
| ---------- | ------------ | -------------------------------- |
| e83aa3     | 4281         | Black Straight Jeans - Womens    |
| 2a2353     | 8199         | Blue Polo Shirt - Mens           |
| e31d39     | 986          | Cream Relaxed Jeans - Womens     |
| 9ec847     | 7754         | Grey Fashion Jacket - Womens     |
| 72f5d4     | 2301         | Indigo Rain Jacket - Womens      |
| d5e9a6     | 2772         | Khaki Suit Jacket - Womens       |
| c4a632     | 1386         | Navy Oversized Jeans - Womens    |
| f084eb     | 5017         | Navy Solid Socks - Mens          |
| 2feb6b     | 3748         | Pink Fluro Polkadot Socks - Mens |
| c8d436     | 987          | Teal Button Up Shirt - Mens      |
| b9a74d     | 1999         | White Striped Socks - Mens       |
| 5d267b     | 5683         | White Tee Shirt - Mens           |

### Transaction Analysis
1. How many unique transactions were there?
~~~sql
SELECT COUNT(DISTINCT txn_id) AS tot_transact
FROM balanced_tree.sales;
~~~
| tot_transact |
| ------------ |
| 2500         |

2. What is the average unique products purchased in each transaction?
~~~sql
SELECT ROUND(AVG(products_per_transact)) AS avg_prod_trans_tot
FROM
 (SELECT txn_id, COUNT(DISTINCT prod_id) AS products_per_transact
 FROM balanced_tree.sales
 GROUP BY txn_id) count_transact;
~~~
| avg_prod_trans_tot |
| ------------------ |
| 6                  |

3. What are the 25th, 50th and 75th percentile values for the revenue per transaction?
~~~sql
SELECT PERCENTILE_CONT(0.25) WITHIN GROUP(ORDER BY tot_transact) AS twtfv_pct,
PERCENTILE_CONT(0.5) WITHIN GROUP(ORDER BY tot_transact) AS fifty_pct,
PERCENTILE_CONT(0.75) WITHIN GROUP(ORDER BY tot_transact) AS svtfv_pct
FROM
 (SELECT txn_id, SUM(qty*(price-price*discount/100)) AS tot_transact
 FROM balanced_tree.sales
 GROUP BY txn_id) tot_transact;
~~~
| twtfv_pct | fifty_pct | svtfv_pct |
| --------- | --------- | --------- |
| 333       | 450       | 582.25    |

4. What is the average discount value per transaction?
~~~sql
SELECT ROUND(AVG(tot_disc)) AS avg_discount
FROM
 (SELECT txn_id, SUM(qty*(price*discount/100)) AS tot_disc
 FROM balanced_tree.sales
 GROUP BY txn_id) tot_disc_subq;
~~~
| avg_discount |
| ------------ |
| 54           |

5. What is the percentage split of all transactions for members vs non-members?
~~~sql
SELECT ROUND(100.0*member_transact/(member_transact+not_member_transact)) AS pct_member,
ROUND(100.0*not_member_transact/(member_transact+not_member_transact)) AS pct_not_member
FROM
 (SELECT SUM(CASE WHEN member='t' THEN qty*(price-price*discount/100) ELSE 0 END) AS member_transact,
 SUM(CASE WHEN member='f' THEN qty*(price-price*discount/100) ELSE 0 END) AS not_member_transact
 FROM balanced_tree.sales) member_not_subq;
~~~
| pct_member | pct_not_member |
| ---------- | -------------- |
| 60         | 40             |

6. What is the average revenue for member transactions and non-member transactions?
~~~sql
SELECT member, AVG(tot_transact)
FROM
 (SELECT txn_id, member, SUM(qty*(price-price*discount/100)) AS tot_transact
 FROM balanced_tree.sales
 GROUP BY txn_id, member) memb_subq
 GROUP BY member;
~~~
| member | avg                  |
| ------ | -------------------- |
| false  | 460.3216080402010050 |
| true   | 462.5235880398671096 |

### Product Analysis
1. What are the top 3 products by total revenue before discount?
~~~sql
SELECT product_name, tot_revenue
FROM
 (SELECT prod_id AS product_id, SUM(qty*price) AS tot_revenue
 FROM balanced_tree.sales 
 GROUP BY product_id) tot_rev_subq
JOIN balanced_tree.product_details
USING (product_id)
ORDER BY tot_revenue DESC
LIMIT 3;
~~~
| product_name                 | tot_revenue |
| ---------------------------- | ----------- |
| Blue Polo Shirt - Mens       | 217683      |
| Grey Fashion Jacket - Womens | 209304      |
| White Tee Shirt - Mens       | 152000      |

2. What is the total quantity, revenue and discount for each segment?
~~~sql
SELECT segment_name, SUM(qty) AS tot_qty, SUM(tot_revenue) AS tot_revenue,
SUM(tot_discount) AS tot_discount
FROM
 (SELECT prod_id AS product_id, qty, qty*price AS tot_revenue, qty*discount*price/100 AS tot_discount
 FROM balanced_tree.sales) tot_all_subq
JOIN balanced_tree.product_details
USING (product_id)
GROUP BY segment_name
ORDER BY tot_revenue DESC;
~~~
| segment_name | tot_qty | tot_revenue | tot_discount |
| ------------ | ------- | ----------- | ------------ |
| Shirt        | 11265   | 406143      | 48082        |
| Jacket       | 11385   | 366983      | 42451        |
| Socks        | 11217   | 307977      | 35280        |
| Jeans        | 11349   | 208350      | 23673        |

3. What is the top selling product for each segment?
~~~sql
SELECT segment_name, product_name, tot_qty, tot_revenue, tot_discount
FROM
 (SELECT *, ROW_NUMBER() OVER(PARTITION BY segment_name ORDER BY tot_qty DESC)
 FROM
  (SELECT segment_name, product_name, SUM(qty) AS tot_qty, SUM(tot_revenue) AS tot_revenue, SUM(tot_discount) AS tot_discount
  FROM
   (SELECT prod_id AS product_id, qty, qty*price AS tot_revenue, qty*discount*price/100 AS tot_discount 
   FROM balanced_tree.sales) tot_all_subq
  JOIN balanced_tree.product_details
  USING (product_id)
  GROUP BY segment_name, product_name) sum_subq) row_subq
 WHERE row_number = 1;
~~~
| segment_name | product_name                  | tot_qty | tot_revenue | tot_discount |
| ------------ | ----------------------------- | ------- | ----------- | ------------ |
| Jacket       | Grey Fashion Jacket - Womens  | 3876    | 209304      | 24781        |
| Jeans        | Navy Oversized Jeans - Womens | 3856    | 50128       | 5538         |
| Shirt        | Blue Polo Shirt - Mens        | 3819    | 217683      | 26189        |
| Socks        | Navy Solid Socks - Mens       | 3792    | 136512      | 16059        |

4. What is the total quantity, revenue and discount for each category?
~~~sql
SELECT category_name, SUM(qty) AS tot_qty, SUM(tot_revenue) AS tot_revenue,
SUM(tot_discount) AS tot_discount
FROM
 (SELECT prod_id AS product_id, qty, qty*price AS tot_revenue, qty*discount*price/100 AS tot_discount
 FROM balanced_tree.sales) tot_all_subq
JOIN balanced_tree.product_details
USING (product_id)
GROUP BY category_name
ORDER BY tot_revenue DESC;
~~~
| category_name | tot_qty | tot_revenue | tot_discount |
| ------------- | ------- | ----------- | ------------ |
| Mens          | 22482   | 714120      | 83362        |
| Womens        | 22734   | 575333      | 66124        |

5. What is the top selling product for each category?
~~~sql
SELECT category_name, product_name, tot_qty, tot_revenue, tot_discount
FROM
 (SELECT *, ROW_NUMBER() OVER(PARTITION BY category_name ORDER BY tot_qty DESC)
 FROM
  (SELECT category_name, product_name, SUM(qty) AS tot_qty, SUM(tot_revenue) AS tot_revenue, SUM(tot_discount) AS tot_discount
  FROM
   (SELECT prod_id AS product_id, qty, qty*price AS tot_revenue, qty*discount*price/100 AS tot_discount
   FROM balanced_tree.sales) tot_all_subq
  JOIN balanced_tree.product_details
  USING (product_id)
  GROUP BY category_name, product_name) sum_subq) row_subq
 WHERE row_number = 1;
~~~
| category_name | product_name                 | tot_qty | tot_revenue | tot_discount |
| ------------- | ---------------------------- | ------- | ----------- | ------------ |
| Mens          | Blue Polo Shirt - Mens       | 3819    | 217683      | 26189        |
| Womens        | Grey Fashion Jacket - Womens | 3876    | 209304      | 24781        |

6. What is the percentage split of revenue by product for each segment?
~~~sql
SELECT *, ROUND(100.0*tot_revenue/SUM(tot_revenue) OVER(PARTITION BY segment_name)) AS pct_revenue
FROM
 (SELECT segment_name, product_name, SUM(qty) AS tot_qty, SUM(tot_revenue) AS tot_revenue, SUM(tot_discount) AS tot_discount
 FROM
  (SELECT prod_id AS product_id, qty, qty*price AS tot_revenue, qty*discount*price/100 AS tot_discount
  FROM balanced_tree.sales) tot_all_subq
 JOIN balanced_tree.product_details
 USING (product_id)
 GROUP BY segment_name, product_name) summed_subq;
~~~
| segment_name | product_name                     | tot_qty | tot_revenue | tot_discount | pct_revenue |
| ------------ | -------------------------------- | ------- | ----------- | ------------ | ----------- |
| Jacket       | Indigo Rain Jacket - Womens      | 3757    | 71383       | 8010         | 19          |
| Jacket       | Khaki Suit Jacket - Womens       | 3752    | 86296       | 9660         | 24          |
| Jacket       | Grey Fashion Jacket - Womens     | 3876    | 209304      | 24781        | 57          |
| Jeans        | Navy Oversized Jeans - Womens    | 3856    | 50128       | 5538         | 24          |
| Jeans        | Black Straight Jeans - Womens    | 3786    | 121152      | 14156        | 58          |
| Jeans        | Cream Relaxed Jeans - Womens     | 3707    | 37070       | 3979         | 18          |
| Shirt        | White Tee Shirt - Mens           | 3800    | 152000      | 17968        | 37          |
| Shirt        | Blue Polo Shirt - Mens           | 3819    | 217683      | 26189        | 54          |
| Shirt        | Teal Button Up Shirt - Mens      | 3646    | 36460       | 3925         | 9           |
| Socks        | Navy Solid Socks - Mens          | 3792    | 136512      | 16059        | 44          |
| Socks        | White Striped Socks - Mens       | 3655    | 62135       | 6877         | 20          |
| Socks        | Pink Fluro Polkadot Socks - Mens | 3770    | 109330      | 12344        | 35          |

7. What is the percentage split of revenue by segment for each category?
~~~sql
SELECT *, ROUND(100.0*tot_revenue/SUM(tot_revenue) OVER(PARTITION BY category_name)) AS pct_revenue
FROM
 (SELECT segment_name, category_name, SUM(qty) AS tot_qty, SUM(tot_revenue) AS tot_revenue, SUM(tot_discount) AS tot_discount
 FROM
  (SELECT prod_id AS product_id, qty, qty*(price) AS tot_revenue, qty*discount*price/100 AS tot_discount
  FROM balanced_tree.sales) tot_all_subq
 JOIN balanced_tree.product_details
 USING (product_id)
 GROUP BY segment_name, category_name) summed_subq;
~~~
| segment_name | category_name | tot_qty | tot_revenue | tot_discount | pct_revenue |
| ------------ | ------------- | ------- | ----------- | ------------ | ----------- |
| Shirt        | Mens          | 11265   | 406143      | 48082        | 57          |
| Socks        | Mens          | 11217   | 307977      | 35280        | 43          |
| Jacket       | Womens        | 11385   | 366983      | 42451        | 64          |
| Jeans        | Womens        | 11349   | 208350      | 23673        | 36          |

8. What is the percentage split of total revenue by category?
~~~sql
SELECT category_name, SUM(tot_revenue) AS tot_revenue, ROUND(100.0*SUM(tot_revenue)/(SELECT SUM(qty*(price - discount*price/100)) FROM balanced_tree.sales)) AS pct_on_tot
FROM
 (SELECT prod_id AS product_id, 
 qty*(price - discount*price/100) AS tot_revenue
 FROM balanced_tree.sales) tot_all_subq
JOIN balanced_tree.product_details
USING (product_id)
GROUP BY category_name;
~~~
| category_name | tot_revenue | pct_on_tot |
| ------------- | ----------- | ---------- |
| Mens          | 637683      | 55         |
| Womens        | 516435      | 45         |

9. What is the total transaction “penetration” for each product? (hint: penetration = number of transactions where at least 1 quantity of a product was purchased divided by total number of transactions)
~~~sql
SELECT product_name, penetration
FROM
 (SELECT prod_id AS product_id, COUNT(DISTINCT txn_id) AS cnt_trans, ROUND(100.0*COUNT(DISTINCT txn_id)/(SELECT COUNT(DISTINCT txn_id) FROM balanced_tree.sales), 2) AS penetration
 FROM balanced_tree.sales
 GROUP BY product_id) pen_subq
JOIN balanced_tree.product_details
USING(product_id)
ORDER BY penetration;
~~~
| product_name                     | penetration |
| -------------------------------- | ----------- |
| Teal Button Up Shirt - Mens      | 49.68       |
| White Striped Socks - Mens       | 49.72       |
| Cream Relaxed Jeans - Womens     | 49.72       |
| Black Straight Jeans - Womens    | 49.84       |
| Khaki Suit Jacket - Womens       | 49.88       |
| Indigo Rain Jacket - Womens      | 50.00       |
| Pink Fluro Polkadot Socks - Mens | 50.32       |
| White Tee Shirt - Mens           | 50.72       |
| Blue Polo Shirt - Mens           | 50.72       |
| Navy Oversized Jeans - Womens    | 50.96       |
| Grey Fashion Jacket - Womens     | 51.00       |
| Navy Solid Socks - Mens          | 51.24       |

10. What is the most common combination of at least 1 quantity of any 3 products in a 1 single transaction?
~~~sql
SELECT s1.prod_id, s2.prod_id, s3.prod_id, COUNT(*) AS tot_combinations
FROM balanced_tree.sales s1
JOIN balanced_tree.sales s2
ON s1.txn_id = s2.txn_id
AND s1.prod_id < s2.prod_id
JOIN balanced_tree.sales s3
ON s1.txn_id = s3.txn_id
AND s2.prod_id < s3.prod_id
GROUP BY s1.prod_id, s2.prod_id, s3.prod_id
ORDER BY tot_combinations DESC
LIMIT 3;
~~~
| prod_id | prod_id | prod_id | tot_combinations |
| ------- | ------- | ------- | ---------------- |
| 5d267b  | 9ec847  | c8d436  | 352              |
| 72f5d4  | e83aa3  | f084eb  | 349              |
| 2a2353  | 9ec847  | c8d436  | 347              |

### Reporting Challenge
~~~sql
# To run the query as requested, it is enough to use the following temp table instead of balanced_tree.sales in the previous questions.

# INPUT PART: modify the where clause to get the months you want included in your analysis --> it is as easy as it gets: let's start by considering January. If you need to consider February, just change 1 with 2. If you need to take into account January AND February, include both 1 and 2.
CREATE TEMP TABLE sales_monthly AS
 (SELECT * 
 FROM balanced_tree.sales 
 WHERE EXTRACT(MONTH FROM start_txn_time) IN (1));
# Here it is enough to copy paste each and every answer to the previous questions, separated by ";". Ofc, once again, you need to get data from sales_monthly and not from balanced_tree.sales
~~~

### Bonus Challenge
~~~sql
SELECT product_id, price, CONCAT(level_text_st, ' ', level_text_se, ' - ', ph.level_text) AS product_name, ph.id AS category_id, id_se AS segment_id, id_st AS style_id, ph.level_text AS category_name, level_text_se AS segment_name,  level_text_st AS style_name
FROM
 (SELECT *
 FROM
  (SELECT id AS id_se, parent_id AS parent_id_se, level_name AS level_name_se, level_text AS level_text_se
  FROM balanced_tree.product_hierarchy) phse
 JOIN 
  (SELECT id AS id_st, parent_id AS parent_id_st, level_name AS level_name_st, level_text AS level_text_st
  FROM balanced_tree.product_hierarchy) phst
 ON phst.parent_id_st = phse.id_se
 WHERE phse.level_name_se = 'Segment' AND phst.level_name_st = 'Style') segment_style_subq
    
JOIN balanced_tree.product_prices pp
ON pp.id = segment_style_subq.id_st
    
JOIN balanced_tree.product_hierarchy ph
ON ph.id = segment_style_subq.parent_id_se;
~~~

| product_id | price | product_name                     | category_id | segment_id | style_id | category_name | segment_name | style_name          |
| ---------- | ----- | -------------------------------- | ----------- | ---------- | -------- | ------------- | ------------ | ------------------- |
| c4a632     | 13    | Navy Oversized Jeans - Womens    | 1           | 3          | 7        | Womens        | Jeans        | Navy Oversized      |
| e83aa3     | 32    | Black Straight Jeans - Womens    | 1           | 3          | 8        | Womens        | Jeans        | Black Straight      |
| e31d39     | 10    | Cream Relaxed Jeans - Womens     | 1           | 3          | 9        | Womens        | Jeans        | Cream Relaxed       |
| d5e9a6     | 23    | Khaki Suit Jacket - Womens       | 1           | 4          | 10       | Womens        | Jacket       | Khaki Suit          |
| 72f5d4     | 19    | Indigo Rain Jacket - Womens      | 1           | 4          | 11       | Womens        | Jacket       | Indigo Rain         |
| 9ec847     | 54    | Grey Fashion Jacket - Womens     | 1           | 4          | 12       | Womens        | Jacket       | Grey Fashion        |
| 5d267b     | 40    | White Tee Shirt - Mens           | 2           | 5          | 13       | Mens          | Shirt        | White Tee           |
| c8d436     | 10    | Teal Button Up Shirt - Mens      | 2           | 5          | 14       | Mens          | Shirt        | Teal Button Up      |
| 2a2353     | 57    | Blue Polo Shirt - Mens           | 2           | 5          | 15       | Mens          | Shirt        | Blue Polo           |
| f084eb     | 36    | Navy Solid Socks - Mens          | 2           | 6          | 16       | Mens          | Socks        | Navy Solid          |
| b9a74d     | 17    | White Striped Socks - Mens       | 2           | 6          | 17       | Mens          | Socks        | White Striped       |
| 2feb6b     | 29    | Pink Fluro Polkadot Socks - Mens | 2           | 6          | 18       | Mens          | Socks        | Pink Fluro Polkadot |
~~~
