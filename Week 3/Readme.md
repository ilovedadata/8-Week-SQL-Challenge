### A. Customer Journey
Based off the 8 sample customers provided in the sample from the subscriptions table, write a brief description about each customer’s onboarding journey. Try to keep it as short as possible - you may also want to run some sort of join to make your explanations a bit easier!
~~~sql
CREATE TEMP TABLE subscr_rank AS
 (SELECT customer_id, plan_id, start_date, plan_name, price, RANK() OVER(PARTITION BY customer_id ORDER BY start_date)
 FROM foodie_fi.subscriptions s
 JOIN 
  (SELECT plan_id AS plan_id_to_drop, plan_name, price FROM foodie_fi.plans) p
 ON p.plan_id_to_drop = s.plan_id
 WHERE customer_id IN (1,2,3,11,13,15,16,18,19));

CREATE TEMP TABLE all_subscr_info_on_rows AS
 (SELECT customer_id, plan_id_1, plan_id_2, plan_id_3, start_date_1, start_date_2, start_date_3, plan_name_1, plan_name_2, plan_name_3, rank_1, rank_2, rank_3
 FROM
  (SELECT * FROM
   (SELECT customer_id, plan_id AS plan_id_1, rank AS rank_1, start_date AS start_date_1, plan_name AS plan_name_1
   FROM subscr_rank
   WHERE rank = 1) first_subscr 
  LEFT JOIN
   (SELECT customer_id AS ci_2, plan_id AS plan_id_2, rank AS rank_2, start_date AS start_date_2, plan_name AS plan_name_2
   FROM subscr_rank
   WHERE rank = 2) second_subscr 
   ON first_subscr.customer_id = second_subscr.ci_2 
  LEFT JOIN
   (SELECT customer_id AS ci_3, plan_id AS plan_id_3, rank AS rank_3, start_date AS start_date_3, plan_name AS plan_name_3
   FROM subscr_rank
   WHERE rank = 3) third_subscr
   ON first_subscr.customer_id = third_subscr.ci_3) just_info_needed);
~~~
~~~sql
SELECT CONCAT('Customer number ', customer_id, ' plans were: ', plan_name_1,', ', plan_name_2,', ', plan_name_3,'. ', 'The plans started on: ', start_date_1, ' ', start_date_2, ' ', start_date_3, ' ', 'respectively') AS a_row_that_says_it_all
FROM all_subscr_info_on_rows;
~~~

| a_row_that_says_it_all                                                                                                                |
| ------------------------------------------------------------------------------------------------------------------------------------- |
| Customer number 1 plans were: trial, basic monthly, . The plans started on: 2020-08-01 2020-08-08  respectively                       |
| Customer number 2 plans were: trial, pro annual, . The plans started on: 2020-09-20 2020-09-27  respectively                          |
| Customer number 3 plans were: trial, basic monthly, . The plans started on: 2020-01-13 2020-01-20  respectively                       |
| Customer number 11 plans were: trial, churn, . The plans started on: 2020-11-19 2020-11-26  respectively                              |
| Customer number 13 plans were: trial, basic monthly, pro monthly. The plans started on: 2020-12-15 2020-12-22 2021-03-29 respectively |
| Customer number 15 plans were: trial, pro monthly, churn. The plans started on: 2020-03-17 2020-03-24 2020-04-29 respectively         |
| Customer number 16 plans were: trial, basic monthly, pro annual. The plans started on: 2020-05-31 2020-06-07 2020-10-21 respectively  |
| Customer number 18 plans were: trial, pro monthly, . The plans started on: 2020-07-06 2020-07-13  respectively                        |
| Customer number 19 plans were: trial, pro monthly, pro annual. The plans started on: 2020-06-22 2020-06-29 2020-08-29 respectively    |

### B. Data Analysis Questions

1. How many customers has Foodie-Fi ever had?
~~~sql
SELECT COUNT(DISTINCT customer_id)
FROM foodie_fi.subscriptions;
~~~

| count |
| ----- |
| 1000  |

2. What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value
~~~sql
SELECT starting_month, COUNT(plan_id) AS nr_plans
FROM
 (SELECT *, EXTRACT(MONTH FROM start_date) AS starting_month FROM foodie_fi.subscriptions) month_subq
WHERE plan_id = 0
GROUP BY starting_month
ORDER BY starting_month;
~~~

| starting_month | nr_plans |
| -------------- | -------- |
| 1              | 88       |
| 2              | 68       |
| 3              | 94       |
| 4              | 81       |
| 5              | 88       |
| 6              | 79       |
| 7              | 89       |
| 8              | 88       |
| 9              | 87       |
| 10             | 79       |
| 11             | 75       |
| 12             | 84       |

3. What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name
~~~sql
SELECT plan_id, count, plan_name 
FROM
 (SELECT plan_id, COUNT(plan_id)
 FROM foodie_fi.subscriptions
 WHERE start_date > '2020-12-31'
 GROUP BY plan_id
 ORDER BY plan_id) plan_id_subq        
JOIN       
foodie_fi.plans p
USING (plan_id);
~~~

| plan_id | count | plan_name     |
| ------- | ----- | ------------- |
| 1       | 8     | basic monthly |
| 2       | 60    | pro monthly   |
| 3       | 63    | pro annual    |
| 4       | 71    | churn         |

4. What is the customer count and percentage of customers who have churned rounded to 1 decimal place?
~~~sql
SELECT churned_customers, tot_customers, ROUND(100.0*churned_customers/tot_customers,1) AS PCT
FROM
 (SELECT COUNT(DISTINCT customer_id) AS churned_customers, '1' AS union_key
 FROM foodie_fi.subscriptions s
 WHERE plan_id = 4) churn_subq
JOIN   
 (SELECT COUNT(DISTINCT customer_id) AS tot_customers,
 '1' AS union_key
 FROM foodie_fi.subscriptions s) tot_subq   
USING(union_key);
~~~

| churned_customers | tot_customers | pct  |
| ----------------- | ------------- | ---- |
| 307               | 1000          | 30.7 |

~~~sql
# Done differently
SELECT COUNT(DISTINCT customer_id) AS churned_customers, ROUND(100.0 * COUNT(DISTINCT customer_id) / (SELECT COUNT(DISTINCT customer_id) FROM foodie_fi.subscriptions), 1) AS pct
FROM foodie_fi.subscriptions s
WHERE plan_id = 4;
~~~

| churned_customers | pct  |
| ----------------- | ---- |
| 307               | 30.7 |

5. How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number?
~~~sql
CREATE TEMP TABLE subscr_rank AS
 (SELECT customer_id, plan_id, start_date, plan_name, price, RANK() OVER(PARTITION BY customer_id ORDER BY start_date)
 FROM foodie_fi.subscriptions s
 JOIN 
  (SELECT plan_id AS plan_id_to_drop, plan_name, price FROM foodie_fi.plans) p
 ON p.plan_id_to_drop = s.plan_id);

SELECT MAX(rank) FROM subscr_rank; # it is 4, it is important to size up all_subscr_info_on_rows

CREATE TEMP TABLE all_subscr_info_on_rows AS
 (SELECT customer_id, plan_id_1, plan_id_2, plan_id_3, plan_id_4, start_date_1, start_date_2, start_date_3, start_date_4, plan_name_1, plan_name_2, plan_name_3, plan_name_4, rank_1, rank_2, rank_3, rank_4
 FROM
  (SELECT * FROM
   (SELECT customer_id, plan_id AS plan_id_1, rank AS rank_1, start_date AS start_date_1, plan_name AS plan_name_1
   FROM subscr_rank
   WHERE rank = 1) first_subscr
        
  LEFT JOIN
        
  (SELECT customer_id AS ci_2, plan_id AS plan_id_2, rank AS rank_2, start_date AS start_date_2, plan_name AS plan_name_2
  FROM subscr_rank
  WHERE rank = 2) second_subscr
  ON first_subscr.customer_id = second_subscr.ci_2
        
  LEFT JOIN
        
  (SELECT customer_id AS ci_3, plan_id AS plan_id_3, rank AS rank_3, start_date AS start_date_3, plan_name AS plan_name_3
  FROM subscr_rank
  WHERE rank = 3) third_subscr      
  ON first_subscr.customer_id = third_subscr.ci_3
         
  LEFT JOIN
      
  (SELECT customer_id AS ci_4, plan_id AS plan_id_4, rank AS rank_4, start_date AS start_date_4, plan_name AS plan_name_4
  FROM subscr_rank
  WHERE rank = 4) fourth_subscr
  ON first_subscr.customer_id = fourth_subscr.ci_4) just_info_needed);
~~~
~~~sql
SELECT COUNT(customer_id), ROUND(100.0 * COUNT(customer_id)/(SELECT COUNT(customer_id) FROM all_subscr_info_on_rows)) AS pct
FROM all_subscr_info_on_rows
WHERE plan_id_1 IS NOT NULL and
plan_id_2 = 4;
~~~

| count | pct |
| ----- | --- |
| 92    | 9   |

6. What is the number and percentage of customer plans after their initial free trial?
~~~sql
SELECT plan_name_2, COUNT(plan_name_2) AS nr_subscr, ROUND(100.0 * COUNT(plan_name_2)/(SELECT COUNT(customer_id) FROM all_subscr_info_on_rows WHERE plan_name_1 = 'trial')) AS pct
FROM all_subscr_info_on_rows
WHERE plan_name_1 = 'trial'
GROUP BY plan_name_2;
~~~

| plan_name_2   | nr_subscr | pct |
| ------------- | --------- | --- |
| basic monthly | 546       | 55  |
| churn         | 92        | 9   |
| pro annual    | 37        | 4   |
| pro monthly   | 325       | 33  |

7. What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?
~~~sql
SELECT plan_id, nr_cust, plan_name, ROUND(100.0*nr_cust/(SELECT COUNT(DISTINCT customer_id) FROM foodie_fi.subscriptions))
FROM
 (SELECT plan_id, COUNT(customer_id) AS nr_cust
 FROM
  (SELECT *, RANK() OVER(PARTITION BY customer_id ORDER BY start_date DESC)
  FROM foodie_fi.subscriptions s
  WHERE start_date <= '2020-12-31') rank_subq
 WHERE rank = 1
 GROUP BY plan_id) nr_cust_subq
JOIN     
foodie_fi.plans p
USING (plan_id);
~~~

| plan_id | nr_cust | plan_name     | round |
| ------- | ------- | ------------- | ----- |
| 0       | 19      | trial         | 2     |
| 1       | 224     | basic monthly | 22    |
| 2       | 326     | pro monthly   | 33    |
| 3       | 195     | pro annual    | 20    |
| 4       | 236     | churn         | 24    |

8. How many customers have upgraded to an annual plan in 2020?
~~~sql
SELECT COUNT(customer_id) 
FROM foodie_fi.subscriptions s
WHERE start_date BETWEEN '2020-01-01' AND '2020-12-31'
AND plan_id = 3;
~~~

| count |
| ----- |
| 195   |

9. How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?
~~~sql
SELECT ROUND(AVG(start_date - first_start_date)) AS delta_dates
FROM
 (SELECT * FROM
  (SELECT *
  FROM foodie_fi.subscriptions s
  WHERE plan_id = 3) subq_3
    
  JOIN
    
  (SELECT customer_id, start_date AS first_start_date, RANK() OVER(PARTITION BY customer_id ORDER BY start_date) AS ranking
  FROM foodie_fi.subscriptions su) rank_subq
    
  USING(customer_id)
WHERE ranking = 1) all_info_subq;
~~~

| delta_dates |
| ----------- |
| 105         |

10. Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)
~~~sql
SELECT COUNT(CASE WHEN start_date - first_start_date <= 30 THEN 1 ELSE NULL END) AS "0-30",
COUNT(CASE WHEN start_date - first_start_date BETWEEN 31 AND 60 THEN 1 ELSE NULL END) AS "31-60", 
COUNT(CASE WHEN start_date - first_start_date BETWEEN 61 AND 90 THEN 1 ELSE NULL END) AS "61-90",
COUNT(CASE WHEN start_date - first_start_date BETWEEN 91 AND 120 THEN 1 ELSE NULL END) AS "91-120",
COUNT(CASE WHEN start_date - first_start_date BETWEEN 121 AND 150 THEN 1 ELSE NULL END) AS "121-150",
COUNT(CASE WHEN start_date - first_start_date BETWEEN 151 AND 180 THEN 1 ELSE NULL END) AS "151-180",
COUNT(CASE WHEN start_date - first_start_date BETWEEN 181 AND 210 THEN 1 ELSE NULL END) AS "181-210",
COUNT(CASE WHEN start_date - first_start_date > 210 THEN 1 ELSE NULL END) AS ">210"    
FROM
 (SELECT * FROM
  (SELECT *
  FROM foodie_fi.subscriptions s
  WHERE plan_id = 3) subq_3
 JOIN
  (SELECT customer_id, start_date AS first_start_date, RANK() OVER(PARTITION BY customer_id ORDER BY start_date) AS ranking
  FROM foodie_fi.subscriptions su) rank_subq      
 USING(customer_id)
 WHERE ranking = 1) all_info_subq;
~~~

| 0-30 | 31-60 | 61-90 | 91-120 | 121-150 | 151-180 | 181-210 | >210 |
| ---- | ----- | ----- | ------ | ------- | ------- | ------- | ---- |
| 49   | 24    | 34    | 35     | 42      | 36      | 26      | 12   |

11. How many customers downgraded from a pro monthly to a basic monthly plan in 2020?
~~~sql
SELECT * 
FROM all_subscr_info_on_rows
WHERE (plan_name_1 = 'pro monthly' AND
plan_name_2 = 'basic monthly' AND start_date_2 BETWEEN '2020-01-01' AND '2020-12-31') OR
(plan_name_2 = 'pro monthly' AND
plan_name_3 = 'basic monthly' AND start_date_3 BETWEEN '2020-01-01' AND '2020-12-31') OR
(plan_name_3 = 'pro monthly' AND
plan_name_4 = 'basic monthly' AND start_date_4 BETWEEN '2020-01-01' AND '2020-12-31');
~~~

### C. Challenge Payment Question
The Foodie-Fi team wants you to create a new payments table for the year 2020 that includes amounts paid by each customer in the subscriptions table with the following requirements:
- monthly payments always occur on the same day of month as the original start_date of any monthly paid plan
- upgrades from basic to monthly or pro plans are reduced by the current paid amount in that month and start immediately
- upgrades from pro monthly to pro annual are paid at the end of the current billing period and also starts at the end of the month period
- once a customer churns they will no longer make payments

~~~sql
# Get the first plan change
CREATE TEMP TABLE first_change AS
 (SELECT customer_id, plan_id_1, plan_id_2, start_date_1, start_date_2, EXTRACT(MONTH FROM AGE(start_date_2, start_date_1)) AS delta_months, plan_name_1, plan_name_2
 FROM all_subscr_info_on_rows
 WHERE plan_id_2 IS NOT NULL);

# Get the second plan change
CREATE TEMP TABLE second_change AS
(SELECT customer_id, plan_id_2, plan_id_3, start_date_2, start_date_3, EXTRACT(MONTH FROM AGE(start_date_3, start_date_2)) AS delta_months,  plan_name_2, plan_name_3
 FROM all_subscr_info_on_rows
 WHERE plan_id_3 IS NOT NULL);

/*# This is empty: there are no third changes: the rank is at most 3
CREATE TEMP TABLE third_change AS
(SELECT customer_id, plan_id_3, plan_id_4, start_date_3, start_date_4, EXTRACT(MONTH FROM AGE(start_date_4, start_date_3)) AS delta_months, plan_name_3, plan_name_4
 FROM all_subscr_info_on_rows
 WHERE plan_id_4 IS NOT NULL);*/

# At most, you have 12 months rolling of delta: here you compute it on the second delta
CREATE TEMP TABLE second_change_all_info AS
 (SELECT *, start_date_2 + delta_months * INTERVAL '1 month' AS new_date 
 FROM
  (SELECT * FROM second_change
  UNION ALL
  SELECT customer_id, plan_id_2, plan_id_3, start_date_2, start_date_3, delta_months - 1 AS delta_months, plan_name_2, plan_name_3 FROM second_change
  UNION ALL
  SELECT customer_id, plan_id_2, plan_id_3, start_date_2, start_date_3, delta_months - 2 AS delta_months, plan_name_2, plan_name_3 FROM second_change
  UNION ALL
  SELECT customer_id, plan_id_2, plan_id_3, start_date_2, start_date_3, delta_months - 3 AS delta_months, plan_name_2, plan_name_3 FROM second_change
  UNION ALL
  SELECT customer_id, plan_id_2, plan_id_3, start_date_2, start_date_3, delta_months - 4 AS delta_months, plan_name_2, plan_name_3 FROM second_change
  UNION ALL
  SELECT customer_id, plan_id_2, plan_id_3, start_date_2, start_date_3, delta_months - 5 AS delta_months, plan_name_2, plan_name_3 FROM second_change
  UNION ALL
  SELECT customer_id, plan_id_2, plan_id_3, start_date_2, start_date_3, delta_months - 6 AS delta_months, plan_name_2, plan_name_3 FROM second_change
  UNION ALL
  SELECT customer_id, plan_id_2, plan_id_3, start_date_2, start_date_3, delta_months - 7 AS delta_months, plan_name_2, plan_name_3 FROM second_change
  UNION ALL
  SELECT customer_id, plan_id_2, plan_id_3, start_date_2, start_date_3, delta_months - 8 AS delta_months, plan_name_2, plan_name_3 FROM second_change
  UNION ALL
  SELECT customer_id, plan_id_2, plan_id_3, start_date_2, start_date_3, delta_months - 9 AS delta_months, plan_name_2, plan_name_3 FROM second_change
  UNION ALL
  SELECT customer_id, plan_id_2, plan_id_3, start_date_2, start_date_3, delta_months - 10 AS delta_months, plan_name_2, plan_name_3 FROM second_change
  UNION ALL
  SELECT customer_id, plan_id_2, plan_id_3, start_date_2, start_date_3, delta_months - 11 AS delta_months, plan_name_2, plan_name_3 FROM second_change
  UNION ALL
  SELECT customer_id, plan_id_2, plan_id_3, start_date_2, start_date_3, delta_months - 12 AS delta_months, plan_name_2, plan_name_3 FROM second_change) all_second_delta
 WHERE delta_months >= 1
 ORDER BY customer_id, delta_months * INTERVAL '1 month');

# AT most, you have 12 months rolling of delta: here you compute it on the first delta
CREATE TEMP TABLE first_change_all_info AS
 (SELECT *, start_date_1 + delta_months * INTERVAL '1 month' AS new_date 
  FROM
  (SELECT * FROM first_change
  UNION ALL
  SELECT customer_id, plan_id_1, plan_id_2, start_date_1, start_date_2, delta_months - 1 AS delta_months, plan_name_1, plan_name_2 FROM first_change
  UNION ALL
  SELECT customer_id, plan_id_1, plan_id_2, start_date_1, start_date_2, delta_months - 2 AS delta_months, plan_name_1, plan_name_2 FROM first_change
  UNION ALL
  SELECT customer_id, plan_id_1, plan_id_2, start_date_1, start_date_2, delta_months - 3 AS delta_months, plan_name_1, plan_name_2 FROM first_change
  UNION ALL
  SELECT customer_id, plan_id_1, plan_id_2, start_date_1, start_date_2, delta_months - 4 AS delta_months, plan_name_1, plan_name_2 FROM first_change
  UNION ALL
  SELECT customer_id, plan_id_1, plan_id_2, start_date_1, start_date_2, delta_months - 5 AS delta_months, plan_name_1, plan_name_2 FROM first_change
  UNION ALL
  SELECT customer_id, plan_id_1, plan_id_2, start_date_1, start_date_2, delta_months - 6 AS delta_months, plan_name_1, plan_name_2 FROM first_change
  UNION ALL
  SELECT customer_id, plan_id_1, plan_id_2, start_date_1, start_date_2, delta_months - 7 AS delta_months, plan_name_1, plan_name_2 FROM first_change
  UNION ALL
  SELECT customer_id, plan_id_1, plan_id_2, start_date_1, start_date_2, delta_months - 8 AS delta_months, plan_name_1, plan_name_2 FROM first_change
  UNION ALL
  SELECT customer_id, plan_id_1, plan_id_2, start_date_1, start_date_2, delta_months - 9 AS delta_months, plan_name_1, plan_name_2 FROM first_change
  UNION ALL
  SELECT customer_id, plan_id_1, plan_id_2, start_date_1, start_date_2, delta_months - 10 AS delta_months, plan_name_1, plan_name_2 FROM first_change
  UNION ALL
  SELECT customer_id, plan_id_1, plan_id_2, start_date_1, start_date_2, delta_months - 11 AS delta_months, plan_name_1, plan_name_2 FROM first_change
  UNION ALL
  SELECT customer_id, plan_id_1, plan_id_2, start_date_1, start_date_2, delta_months - 12 AS delta_months, plan_name_1, plan_name_2 FROM first_change) all_first_delta
 WHERE delta_months >= 1
 ORDER BY customer_id, delta_months * INTERVAL '1 month');

/* The lagged rows are needed to impose the following condition, which computes the correct amount paid by each customer in case he/she switches plans in the middle of a month*/
SELECT customer_id, plan_id, plan_name, payment_date, 
CASE WHEN (lagged_plan_name = 'basic monthly' AND (plan_name = 'pro monthly' OR plan_name = 'pro annual') AND customer_id = lagged_customer AND
EXTRACT(MONTH FROM AGE(payment_date, lagged_payment_date)) = 0) THEN amount - lagged_amount ELSE amount END AS amount, ROW_NUMBER() OVER(PARTITION BY customer_id) AS payment_order
FROM
 (SELECT *, 
 LAG(customer_id, 1) OVER() AS lagged_customer,
 LAG(plan_name, 1) OVER() AS lagged_plan_name,
 LAG(amount, 1) OVER() AS lagged_amount,
 LAG(payment_date, 1) OVER() AS lagged_payment_date
 FROM
  (SELECT DISTINCT customer_id, plan_id, plan_name, payment_date, price AS amount
  FROM
   (SELECT customer_id, plan_id_2 AS plan_id, plan_name_2 AS plan_name, new_date AS payment_date
   FROM
    (SELECT * FROM second_change_all_info 
    # if new_date is = to start_date_3, keep on using start_date_3
    WHERE new_date != start_date_3) no_duplicated_dates
   UNION ALL
   SELECT customer_id, plan_id_1 AS plan_id, plan_name_1 AS plan_name, new_date AS payment_date
   FROM
    (SELECT * FROM first_change_all_info 
    # if new_date is = to start_date_2, keep on using start_date_2
    WHERE new_date != start_date_2) no_duplicated_dates_first
   UNION ALL
   # Get rows which were skipped by the queries above. 
   SELECT customer_id, plan_id, plan_name, start_date AS payment_date 
   FROM
    (SELECT * FROM foodie_fi.subscriptions s
    JOIN
    # Needed to get the plan_name, in order to append rows properly
    foodie_fi.plans p
    USING (plan_id)
    WHERE start_date BETWEEN '2020-01-01' AND '2020-12-31' AND
    plan_id != 0 AND plan_id != 4) get_first_rows) all_rows_appended
   JOIN
   # Get the price of the plans
   (SELECT plan_id, price FROM foodie_fi.plans) foodie_prices
   USING(plan_id)
  ORDER BY customer_id, payment_date) all_info_but_prices) all_info_w_prices
LIMIT 32
~~~

| customer_id | plan_id | plan_name     | payment_date             | amount | payment_order |
| ----------- | ------- | ------------- | ------------------------ | ------ | ------------- |
| 1           | 1       | basic monthly | 2020-08-08T00:00:00.000Z | 9.90   | 1             |
| 2           | 3       | pro annual    | 2020-09-27T00:00:00.000Z | 199.00 | 1             |
| 3           | 1       | basic monthly | 2020-01-20T00:00:00.000Z | 9.90   | 1             |
| 4           | 1       | basic monthly | 2020-01-24T00:00:00.000Z | 9.90   | 1             |
| 5           | 1       | basic monthly | 2020-08-10T00:00:00.000Z | 9.90   | 1             |
| 6           | 1       | basic monthly | 2020-12-30T00:00:00.000Z | 9.90   | 1             |
| 7           | 1       | basic monthly | 2020-02-12T00:00:00.000Z | 9.90   | 1             |
| 7           | 1       | basic monthly | 2020-03-12T00:00:00.000Z | 9.90   | 2             |
| 7           | 1       | basic monthly | 2020-04-12T00:00:00.000Z | 9.90   | 3             |
| 7           | 1       | basic monthly | 2020-05-12T00:00:00.000Z | 9.90   | 4             |
| 7           | 2       | pro monthly   | 2020-05-22T00:00:00.000Z | 10.00  | 5             |
| 8           | 1       | basic monthly | 2020-06-18T00:00:00.000Z | 9.90   | 1             |
| 8           | 1       | basic monthly | 2020-07-18T00:00:00.000Z | 9.90   | 2             |
| 8           | 2       | pro monthly   | 2020-08-03T00:00:00.000Z | 10.00  | 3             |
| 9           | 3       | pro annual    | 2020-12-14T00:00:00.000Z | 199.00 | 1             |
| 10          | 2       | pro monthly   | 2020-09-26T00:00:00.000Z | 19.90  | 1             |
| 12          | 1       | basic monthly | 2020-09-29T00:00:00.000Z | 9.90   | 1             |
| 13          | 1       | basic monthly | 2020-12-22T00:00:00.000Z | 9.90   | 1             |
| 14          | 1       | basic monthly | 2020-09-29T00:00:00.000Z | 9.90   | 1             |
| 15          | 2       | pro monthly   | 2020-03-24T00:00:00.000Z | 19.90  | 1             |
| 16          | 1       | basic monthly | 2020-06-07T00:00:00.000Z | 9.90   | 1             |
| 16          | 1       | basic monthly | 2020-07-07T00:00:00.000Z | 9.90   | 2             |
| 16          | 1       | basic monthly | 2020-08-07T00:00:00.000Z | 9.90   | 3             |
| 16          | 1       | basic monthly | 2020-09-07T00:00:00.000Z | 9.90   | 4             |
| 16          | 1       | basic monthly | 2020-10-07T00:00:00.000Z | 9.90   | 5             |
| 16          | 3       | pro annual    | 2020-10-21T00:00:00.000Z | 189.10 | 6             |
| 17          | 1       | basic monthly | 2020-08-03T00:00:00.000Z | 9.90   | 1             |
| 17          | 1       | basic monthly | 2020-09-03T00:00:00.000Z | 9.90   | 2             |
| 17          | 1       | basic monthly | 2020-10-03T00:00:00.000Z | 9.90   | 3             |
| 17          | 1       | basic monthly | 2020-11-03T00:00:00.000Z | 9.90   | 4             |
| 17          | 1       | basic monthly | 2020-12-03T00:00:00.000Z | 9.90   | 5             |
| 17          | 3       | pro annual    | 2020-12-11T00:00:00.000Z | 189.10 | 6             |

### D. Outside The Box Questions
- How would you calculate the rate of growth for Foodie-Fi?
Number of customers at the end of a given period / Number of customers at the end of the previous period - 1. Ex: 120 / 100 - 1 = 20% growth

- What key metrics would you recommend Foodie-Fi management to track over time to assess performance of their overall business? %Type of plans over total plans (is there a more lucrative plan? If so, put particular attention in tracking the number of customers subscribed to it), churn rate, growth rate.

- What are some key customer journeys or experiences that you would analyse further to improve customer retention? Check which is the plan customers subscribed to before churning, track the journeys of satisfied customers, track the journeys of the customers subscribed to the plan you would like all customers to be subscribed to

- If the Foodie-Fi team were to create an exit survey shown to customers who wish to cancel their subscription, what questions would you include in the survey?
1- Explain the main issue with your plan that led you to unsubscribe from our services
2- How would you rate your experience with us? 3-How can we improve our service?

- What business levers could the Foodie-Fi team use to reduce the customer churn rate? How would you validate the effectiveness of your ideas? -Improve the customer service;
1. Identify the customers who are at risk of churning (through predictive modelling or by considering custoemrs that are taking the same steps of customers who churned already);
2. Validate so by analyzing Growth rate and churn rate.
