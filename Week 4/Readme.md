### A. Customer Nodes Exploration

1. How many unique nodes are there on the Data Bank system?
~~~sql
SELECT COUNT(DISTINCT node_id) AS count_of_nodes
FROM data_bank.customer_nodes;
~~~

| count_of_nodes |
| -------------- |
| 5              |

2. What is the number of nodes per region?
~~~sql
SELECT region_name, COUNT(node_id) AS nodes_per_region
FROM data_bank.customer_nodes 
JOIN data_bank.regions
USING (region_id)
GROUP BY region_name
ORDER BY region_name;
~~~

| region_name | nodes_per_region |
| ----------- | ---------------- |
| Africa      | 714              |
| America     | 735              |
| Asia        | 665              |
| Australia   | 770              |
| Europe      | 616              |

3. How many customers are allocated to each region?
~~~sql
SELECT region_name, COUNT(DISTINCT customer_id) AS customers_per_region
FROM data_bank.customer_nodes 
JOIN data_bank.regions
USING (region_id)
GROUP BY region_name
ORDER BY region_name;
~~~

| region_name | customers_per_region |
| ----------- | -------------------- |
| Africa      | 102                  |
| America     | 105                  |
| Asia        | 95                   |
| Australia   | 110                  |
| Europe      | 88                   |

4. How many days on average are customers reallocated to a different node?
~~~sql
SELECT ROUND(AVG(end_date-start_date)) AS avg_delta_dates
FROM data_bank.customer_nodes 
WHERE end_date != '9999-12-31';
~~~

| avg_delta_dates |
| --------------- |
| 15              |

5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?
~~~sql
SELECT * 
FROM
 (SELECT region_id, ROUND(AVG(end_date-start_date)) AS avg_delta_dates, PERCENTILE_CONT(0.5) WITHIN GROUP(ORDER BY end_date-start_date) AS median, PERCENTILE_CONT(0.8) WITHIN GROUP(ORDER BY end_date-start_date) AS eighty_pct, PERCENTILE_CONT(0.95) WITHIN GROUP(ORDER BY end_date-start_date) AS ninety5_pct
 FROM data_bank.customer_nodes 
 WHERE end_date != '9999-12-31'
 GROUP BY region_id) all_stats_subq
JOIN data_bank.regions
USING (region_id);
~~~

| region_id | avg_delta_dates | median | eighty_pct | ninety5_pct | region_name |
| --------- | --------------- | ------ | ---------- | ----------- | ----------- |
| 1         | 15              | 15     | 23         | 28          | Australia   |
| 2         | 15              | 15     | 23         | 28          | America     |
| 3         | 15              | 15     | 24         | 28          | Africa      |
| 4         | 14              | 15     | 23         | 28          | Asia        |
| 5         | 15              | 15     | 24         | 28          | Europe      |

### B. Customer Transactions
1. What is the unique count and total amount for each transaction type?
~~~sql
SELECT txn_type, COUNT(txn_type) AS nr_txn, SUM(txn_amount) AS sum_txn
FROM data_bank.customer_transactions
GROUP BY (txn_type);
~~~

| txn_type   | nr_txn | sum_txn |
| ---------- | ------ | ------- |
| purchase   | 1617   | 806537  |
| deposit    | 2671   | 1359168 |
| withdrawal | 1580   | 793003  |

2. What is the average total historical deposit counts and amounts for all customers?
~~~sql
SELECT tot_amt/distinct_customers AS avg_amt, cnt_deposits/distinct_customers AS avg_nr_deposits
FROM
 (SELECT SUM(txn_amount) AS tot_amt, COUNT(txn_amount) cnt_deposits, COUNT(DISTINCT customer_id) AS distinct_customers 
 FROM data_bank.customer_transactions
 WHERE txn_type IN ('deposit')) tot_cnt_subq;
~~~

| avg_amt | avg_nr_deposits |
| ------- | --------------- |
| 2718    | 5               |

3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?
~~~sql
SELECT txn_month, COUNT(customer_id) AS cust_cnt
FROM
 (SELECT customer_id, txn_month, SUM(nr_deposits) AS deposits_sum, SUM(nr_purchases) AS purchases_sum, SUM(nr_withdrawals) AS withdrawals_sum 
  FROM
  (SELECT *, EXTRACT (MONTH FROM txn_date) AS txn_month,
  CASE WHEN txn_type = 'deposit' THEN 1 ELSE 0 END AS nr_deposits,
  CASE WHEN txn_type = 'purchase' THEN 1 ELSE 0 END AS nr_purchases,
  CASE WHEN txn_type = 'withdrawal' THEN 1 ELSE 0 END AS nr_withdrawals
  FROM data_bank.customer_transactions) nr_transact_subq
 GROUP BY customer_id, txn_month
 ORDER BY customer_id) nr_transact_per_month_subq
    
WHERE deposits_sum > 1 AND (purchases_sum = 1 OR withdrawals_sum = 1)
GROUP BY txn_month
ORDER BY txn_month;
~~~

| txn_month | cust_cnt |
| --------- | -------- |
| 1         | 115      |
| 2         | 108      |
| 3         | 113      |
| 4         | 50       |

4. What is the closing balance for each customer at the end of the month?
~~~sql
SELECT customer_id, txn_month, (deposits_sum - purchases_sum - withdrawals_sum) AS closing_balance
FROM 
 (SELECT customer_id, txn_month, SUM(sum_deposits) AS deposits_sum, SUM(sum_purchases) AS purchases_sum, SUM(sum_withdrawals) AS withdrawals_sum 
 FROM
  (SELECT *, EXTRACT (MONTH FROM txn_date) AS txn_month,
  CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE 0 END AS sum_deposits,
  CASE WHEN txn_type = 'purchase' THEN txn_amount ELSE 0 END AS sum_purchases,
  CASE WHEN txn_type = 'withdrawal' THEN txn_amount ELSE 0 END AS sum_withdrawals
  FROM data_bank.customer_transactions) nr_transact_subq
 GROUP BY customer_id, txn_month) all_sums_subq
ORDER BY customer_id, txn_month
LIMIT 10;
~~~

| customer_id | txn_month | closing_balance |
| ----------- | --------- | --------------- |
| 1           | 1         | 312             |
| 1           | 3         | -952            |
| 2           | 1         | 549             |
| 2           | 3         | 61              |
| 3           | 1         | 144             |
| 3           | 2         | -965            |
| 3           | 3         | -401            |
| 3           | 4         | 493             |
| 4           | 1         | 848             |
| 4           | 3         | -193            |

5. What is the percentage of customers who increase their closing balance by more than 5%?
~~~sql
CREATE TEMP TABLE monthly_balance AS
 (SELECT customer_id, txn_month, (deposits_sum - purchases_sum - withdrawals_sum) AS closing_balance
 FROM 
  (SELECT customer_id, txn_month, SUM(sum_deposits) AS deposits_sum, SUM(sum_purchases) AS purchases_sum, SUM(sum_withdrawals) AS withdrawals_sum 
  FROM 
   (SELECT *, EXTRACT (MONTH FROM txn_date) AS txn_month,
   CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE 0 END AS sum_deposits,
   CASE WHEN txn_type = 'purchase' THEN txn_amount ELSE 0 END AS sum_purchases,
   CASE WHEN txn_type = 'withdrawal' THEN txn_amount ELSE 0 END AS sum_withdrawals
   FROM data_bank.customer_transactions) nr_transact_subq
  GROUP BY customer_id, txn_month) all_sums_subq
 ORDER BY customer_id, txn_month);
~~~
~~~sql
# ASSUMPTION: the first month is not to be considered in the result since you do not have infos about the previous month balance. The result will be splitted in months and months i+1 balance will be compared to month i balance to get whether or not it increases by 5%.

SELECT txn_month, COUNT(DISTINCT customer_id) AS denominator, COUNT(increase_or_not) AS numerator, ROUND(COUNT(increase_or_not)*100.0/COUNT(DISTINCT customer_id)) AS pct
FROM
 (SELECT *, CASE WHEN ((closing_balance - lagged_balance)*100.0/ABS(lagged_balance)) > 5 THEN 1 ELSE NULL END AS increase_or_not
 FROM
  (SELECT *, LAG(closing_balance, 1) OVER(PARTITION BY customer_id ORDER BY customer_id, txn_month) AS lagged_balance FROM monthly_balance) lagged_balance_subq
  # if null, the customer has changed
 WHERE lagged_balance IS NOT NULL) increase_subq
 GROUP BY txn_month
ORDER BY txn_month
~~~

| txn_month | denominator | numerator | pct |
| --------- | ----------- | --------- | --- |
| 2         | 455         | 147       | 32  |
| 3         | 456         | 202       | 44  |
| 4         | 309         | 158       | 51  |

### C. Data Allocation Challenge
For this multi-part challenge question - you have been requested to generate the following data elements to help the Data Bank team estimate how much data will need to be provisioned for each option:
- running customer balance column that includes the impact each transaction
- customer balance at the end of each month
- minimum, average and maximum values of the running balance for each customer

~~~sql
CREATE TEMP TABLE running_balance AS
 (SELECT *, SUM(signed_amt) OVER(PARTITION BY customer_id ORDER BY customer_id, txn_date) AS running_customer_balance
 FROM
  (SELECT *, CASE WHEN txn_type IN ('purchase', 'withdrawal') THEN -1*txn_amount ELSE txn_amount END AS signed_amt
  FROM data_bank.customer_transactions) cash_flow_subq);
~~~
~~~sql
SELECT * FROM running_balance LIMIT 10;
~~~

| customer_id | txn_date                 | txn_type   | txn_amount | signed_amt | running_customer_balance |
| ----------- | ------------------------ | ---------- | ---------- | ---------- | ------------------------ |
| 1           | 2020-01-02T00:00:00.000Z | deposit    | 312        | 312        | 312                      |
| 1           | 2020-03-05T00:00:00.000Z | purchase   | 612        | -612       | -300                     |
| 1           | 2020-03-17T00:00:00.000Z | deposit    | 324        | 324        | 24                       |
| 1           | 2020-03-19T00:00:00.000Z | purchase   | 664        | -664       | -640                     |
| 2           | 2020-01-03T00:00:00.000Z | deposit    | 549        | 549        | 549                      |
| 2           | 2020-03-24T00:00:00.000Z | deposit    | 61         | 61         | 610                      |
| 3           | 2020-01-27T00:00:00.000Z | deposit    | 144        | 144        | 144                      |
| 3           | 2020-02-22T00:00:00.000Z | purchase   | 965        | -965       | -821                     |
| 3           | 2020-03-05T00:00:00.000Z | withdrawal | 213        | -213       | -1034                    |
| 3           | 2020-03-19T00:00:00.000Z | withdrawal | 188        | -188       | -1222                    |

~~~sql
SELECT customer_id, txn_month, (deposits_sum - purchases_sum - withdrawals_sum) AS closing_balance  
FROM     
 (SELECT customer_id, txn_month, SUM(sum_deposits) AS deposits_sum, SUM(sum_purchases) AS purchases_sum, SUM(sum_withdrawals) AS withdrawals_sum
 FROM
  (SELECT *, EXTRACT (MONTH FROM txn_date) AS txn_month,
  CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE 0 END AS sum_deposits,
  CASE WHEN txn_type = 'purchase' THEN txn_amount ELSE 0 END AS sum_purchases,
  CASE WHEN txn_type = 'withdrawal' THEN txn_amount ELSE 0 END AS sum_withdrawals
  FROM data_bank.customer_transactions) nr_transact_subq
 GROUP BY customer_id, txn_month) all_sums_subq    
ORDER BY customer_id, txn_month
LIMIT 10;
~~~

| customer_id | txn_month | closing_balance |
| ----------- | --------- | --------------- |
| 1           | 1         | 312             |
| 1           | 3         | -952            |
| 2           | 1         | 549             |
| 2           | 3         | 61              |
| 3           | 1         | 144             |
| 3           | 2         | -965            |
| 3           | 3         | -401            |
| 3           | 4         | 493             |
| 4           | 1         | 848             |
| 4           | 3         | -193            |

~~~sql
SELECT customer_id, MIN(running_customer_balance) AS min_run_balance,
MAX(running_customer_balance) AS max_run_balance,
ROUND(AVG(running_customer_balance)) AS avg_run_balance
FROM running_balance
GROUP BY customer_id
ORDER BY customer_id
LIMIT 10;
~~~

| customer_id | min_run_balance | max_run_balance | avg_run_balance |
| ----------- | --------------- | --------------- | --------------- |
| 1           | -640            | 312             | -151            |
| 2           | 549             | 610             | 580             |
| 3           | -1222           | 144             | -732            |
| 4           | 458             | 848             | 654             |
| 5           | -2413           | 1780            | -135            |
| 6           | -552            | 2197            | 624             |
| 7           | 887             | 3539            | 2269            |
| 8           | -1029           | 1363            | 174             |
| 9           | -91             | 2030            | 1022            |
| 10          | -5090           | 556             | -2230           |
