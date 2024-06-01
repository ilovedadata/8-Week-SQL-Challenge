### A. Pizza Metrics

0. Cleaning stuff: cleaning data and data engineering
~~~sql
CREATE TEMP TABLE customers_cleaned AS
 (SELECT order_id, customer_id, pizza_id,
 CASE WHEN exclusions IS NULL or exclusions='null' or exclusions = ' '
 or exclusions = '' THEN NULL
 ELSE exclusions END AS exclusions,
 CASE WHEN extras IS NULL or extras='null' or extras = ' ' or extras = '' 
 THEN NULL
 ELSE extras END AS extras,
 order_time
 FROM pizza_runner.customer_orders);
~~~

| order_id | customer_id | pizza_id | exclusions | extras | order_time               |
| -------- | ----------- | -------- | ---------- | ------ | ------------------------ |
| 1        | 101         | 1        |            |        | 2020-01-01T18:05:02.000Z |
| 2        | 101         | 1        |            |        | 2020-01-01T19:00:52.000Z |
| 3        | 102         | 1        |            |        | 2020-01-02T23:51:23.000Z |
| 3        | 102         | 2        |            |        | 2020-01-02T23:51:23.000Z |
| 4        | 103         | 1        | 4          |        | 2020-01-04T13:23:46.000Z |
| 4        | 103         | 1        | 4          |        | 2020-01-04T13:23:46.000Z |
| 4        | 103         | 2        | 4          |        | 2020-01-04T13:23:46.000Z |
| 5        | 104         | 1        |            | 1      | 2020-01-08T21:00:29.000Z |
| 6        | 101         | 2        |            |        | 2020-01-08T21:03:13.000Z |
| 7        | 105         | 2        |            | 1      | 2020-01-08T21:20:29.000Z |
| 8        | 102         | 1        |            |        | 2020-01-09T23:54:33.000Z |
| 9        | 103         | 1        | 4          | 1, 5   | 2020-01-10T11:22:59.000Z |
| 10       | 104         | 1        |            |        | 2020-01-11T18:34:49.000Z |
| 10       | 104         | 1        | 2, 6       | 1, 4   | 2020-01-11T18:34:49.000Z |

~~~sql
CREATE TEMP TABLE runners_cleaned AS
 SELECT order_id, runner_id,
 CASE WHEN distance != 'null' AND distance LIKE '%.%' THEN CAST (LEFT(distance, 4) AS FLOAT)
 WHEN distance != 'null' THEN CAST (LEFT(distance, 2) AS FLOAT)
 ELSE NULL END AS distance_km,
    
 CASE WHEN cancellation IS NULL OR cancellation = 'null' OR cancellation = '' OR cancellation = ' ' THEN NULL
 ELSE cancellation END AS cancellation,
    
 CASE WHEN duration = 'null' OR duration IS NULL THEN NULL
 ELSE CAST(LEFT(duration, 2) AS INTEGER) END AS duration_min,
    
 CASE WHEN pickup_time = 'null' OR pickup_time IS NULL THEN NULL
 ELSE CAST(pickup_time AS TIMESTAMP) END AS pickup_time_ts
    
 FROM pizza_runner.runner_orders;
~~~

| order_id | runner_id | distance_km | cancellation            | duration_min | pickup_time_ts           |
| -------- | --------- | ----------- | ----------------------- | ------------ | ------------------------ |
| 1        | 1         | 20          |                         | 32           | 2020-01-01T18:15:34.000Z |
| 2        | 1         | 20          |                         | 27           | 2020-01-01T19:10:54.000Z |
| 3        | 1         | 13.4        |                         | 20           | 2020-01-03T00:12:37.000Z |
| 4        | 2         | 23.4        |                         | 40           | 2020-01-04T13:53:03.000Z |
| 5        | 3         | 10          |                         | 15           | 2020-01-08T21:10:57.000Z |
| 6        | 3         |             | Restaurant Cancellation |              |                          |
| 7        | 2         | 25          |                         | 25           | 2020-01-08T21:30:45.000Z |
| 8        | 2         | 23.4        |                         | 15           | 2020-01-10T00:15:02.000Z |
| 9        | 2         |             | Customer Cancellation   |              |                          |
| 10       | 1         | 10          |                         | 10           | 2020-01-11T18:50:20.000Z |

1. How many pizzas were ordered?
~~~sql
SELECT COUNT(order_time) 
FROM customers_cleaned
~~~

| count |
| ----- |
| 14    |

2. How many unique customer orders were made?
~~~sql
SELECT COUNT(DISTINCT order_id) 
FROM customers_cleaned
~~~

| count |
| ----- |
| 10    |

3. How many successful orders were delivered by each runner?
~~~sql
SELECT runner_id, COUNT(DISTINCT customers_cleaned.order_id)
FROM customers_cleaned
JOIN runners_cleaned
ON customers_cleaned.order_id = runners_cleaned.order_id
WHERE cancellation IS NULL
GROUP BY runner_id;
~~~

| runner_id | count |
| --------- | ----- |
| 1         | 4     |
| 2         | 3     |
| 3         | 1     |

4. How many of each type of pizza was delivered?
~~~sql
SELECT pizza_id, COUNT(pizza_id) 
FROM customers_cleaned
JOIN runners_cleaned
ON customers_cleaned.order_id = runners_cleaned.order_id
WHERE cancellation IS NULL
GROUP BY pizza_id;
~~~

| pizza_id | count |
| -------- | ----- |
| 1        | 9     |
| 2        | 3     |

5. How many Vegetarian and Meatlovers were ordered by each customer?
~~~sql
SELECT customers_cleaned.customer_id, pizza_name, COUNT(pizza_name)
FROM customers_cleaned
JOIN runners_cleaned
ON customers_cleaned.order_id = runners_cleaned.order_id
  
JOIN pizza_runner.pizza_names 
ON customers_cleaned.pizza_id = pizza_names.pizza_id
GROUP BY customers_cleaned.customer_id, pizza_name
ORDER BY customer_id, pizza_name;
~~~

| customer_id | pizza_name | count |
| ----------- | ---------- | ----- |
| 101         | Meatlovers | 2     |
| 101         | Vegetarian | 1     |
| 102         | Meatlovers | 2     |
| 102         | Vegetarian | 1     |
| 103         | Meatlovers | 3     |
| 103         | Vegetarian | 1     |
| 104         | Meatlovers | 3     |
| 105         | Vegetarian | 1     |

6. What was the maximum number of pizzas delivered in a single order?
~~~sql
SELECT customers_cleaned.order_id, COUNT(pizza_id) as cnt_pzz
FROM customers_cleaned
JOIN runners_cleaned
ON customers_cleaned.order_id = runners_cleaned.order_id
WHERE cancellation IS NULL
GROUP BY customers_cleaned.order_id
ORDER BY cnt_pzz DESC;
~~~

| order_id | cnt_pzz |
| -------- | ------- |
| 4        | 3       |
| 3        | 2       |
| 10       | 2       |
| 7        | 1       |
| 1        | 1       |
| 5        | 1       |
| 2        | 1       |
| 8        | 1       |

7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?
~~~sql
SELECT customer_id, change, COUNT(change) 
FROM
 (SELECT customer_id, extras, exclusions,
 CASE WHEN exclusions IS NULL and extras IS NULL THEN 'No'
 ELSE 'Yes' END AS "change" 
 FROM customers_cleaned
 JOIN runners_cleaned
 ON customers_cleaned.order_id = runners_cleaned.order_id
 WHERE cancellation IS NULL) change_query
GROUP BY customer_id, change
ORDER BY customer_id, change;
~~~

| customer_id | change | count |
| ----------- | ------ | ----- |
| 101         | No     | 2     |
| 102         | No     | 3     |
| 103         | Yes    | 3     |
| 104         | No     | 1     |
| 104         | Yes    | 2     |
| 105         | Yes    | 1     |

8. How many pizzas were delivered that had both exclusions and extras?
~~~sql
SELECT COUNT(*)
FROM
 (SELECT customer_id, extras, exclusions,
 CASE WHEN exclusions IS NOT NULL and extras IS NOT NULL THEN 'Both'
 ELSE 'Either_or_not' END AS "extras_excl" 
 FROM customers_cleaned
 JOIN runners_cleaned
 ON customers_cleaned.order_id = runners_cleaned.order_id
 WHERE cancellation IS NULL) changes_query
WHERE changes_query.extras_excl = 'Both';
~~~

| count |
| ----- |
| 1     |

9. What was the total volume of pizzas ordered for each hour of the day?
~~~sql
SELECT order_hour, COUNT(pizza_id) AS pzz_ordered, ROUND(COUNT(pizza_id)/SUM(COUNT(pizza_id)) OVER()*100, 1) AS vol
FROM
 (SELECT *, EXTRACT(HOUR FROM order_time) AS order_hour 
 FROM customers_cleaned
 JOIN runners_cleaned
 ON customers_cleaned.order_id = runners_cleaned.order_id) changes_query
GROUP BY order_hour
ORDER BY order_hour
~~~

| order_hour | pzz_ordered | vol  |
| ---------- | ----------- | ---- |
| 11         | 1           | 7.1  |
| 13         | 3           | 21.4 |
| 18         | 3           | 21.4 |
| 19         | 1           | 7.1  |
| 21         | 3           | 21.4 |
| 23         | 3           | 21.4 |

10. What was the volume of orders for each day of the week?
~~~sql
SELECT dow, COUNT(pizza_id) AS pzz_ordered,
ROUND(COUNT(pizza_id)/SUM(COUNT(pizza_id)) OVER()*100, 1) AS vol
FROM
# dow = day of week
 (SELECT *, EXTRACT(DOW FROM order_time) AS dow 
 FROM customers_cleaned
 JOIN runners_cleaned
 ON customers_cleaned.order_id = runners_cleaned.order_id) changes_query
GROUP BY dow
ORDER BY dow;
~~~

| dow | pzz_ordered | vol  |
| --- | ----------- | ---- |
| 3   | 5           | 35.7 |
| 4   | 3           | 21.4 |
| 5   | 1           | 7.1  |
| 6   | 5           | 35.7 |
