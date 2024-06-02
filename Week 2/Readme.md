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

### B. Runner and Customer Experience

1. How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)
~~~sql
SELECT COUNT(runner_id) AS runners_registered, week_nr
FROM
 (SELECT *, EXTRACT(WEEK FROM registration_date) AS week_nr
 FROM pizza_runner.runners) week_nr_query
GROUP BY week_nr;
~~~

| runners_registered | week_nr |
| ------------------ | ------- |
| 2                  | 53      |
| 1                  | 1       |
| 1                  | 2       |

2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?
~~~sql
SELECT runner_id, AVG(delta_time) AS avg_time 
FROM
 (SELECT runner_id, pickup_time_ts - order_time AS delta_time
 FROM customers_cleaned
 JOIN runners_cleaned
 ON customers_cleaned.order_id = runners_cleaned.order_id
 WHERE cancellation IS NULL) delta_time_query        
GROUP BY runner_id
ORDER BY runner_id
~~~

| runner_id | avg_time        |
| --------- | --------------- |
| 1         | {"minutes":15,"seconds":40,"milliseconds":666.667} |
| 2         | {"minutes":23,"seconds":43,"milliseconds":200} |
| 3         | {"minutes":10,"seconds":28} |

3. Is there any relationship between the number of pizzas and how long the order takes to prepare?
~~~sql
# Customers cleaned modified as follows to avoid the redundancy of the order_id column
CREATE TEMP TABLE customers_cleaned AS 
 (SELECT order_id AS order_id_2, customer_id, pizza_id,
 CASE WHEN exclusions IS NULL or exclusions='null' or exclusions = ' ' 
 or exclusions = '' THEN NULL
 ELSE exclusions END AS exclusions,
 CASE WHEN extras IS NULL or extras='null' or extras = ' ' or extras = '' 
 THEN NULL
 ELSE extras END AS extras,
 order_time
 FROM pizza_runner.customer_orders); 
~~~
~~~sql
SELECT nr_pizza, AVG(proxy_time)
FROM
 # Get the time spent per order. N.W.: MAX is  a quick way to get a unique value per order 
 (SELECT COUNT(pizza_id) AS nr_pizza, MAX(delta_time) AS proxy_time FROM
  # compute the delta time
  (SELECT *, pickup_time_ts - order_time AS delta_time
  FROM customers_cleaned
  JOIN runners_cleaned
  ON customers_cleaned.order_id_2 = runners_cleaned.order_id
  WHERE cancellation IS NULL) delta_time_query       
 GROUP BY order_id) nr_pizza_query
GROUP BY nr_pizza
ORDER BY nr_pizza;
~~~

| nr_pizza | avg             |
| -------- | --------------- |
| 1        | {"minutes":12,"seconds":21,"milliseconds":400} |
| 2        | {"minutes":18,"seconds":22,"milliseconds":500} |
| 3        | {"minutes":29,"seconds":17} |

4. What was the average distance travelled for each customer?
~~~sql
SELECT customer_id, ROUND(AVG(distance_km)) AS avg_distance
FROM customers_cleaned
JOIN runners_cleaned
ON customers_cleaned.order_id_2 = runners_cleaned.order_id
WHERE cancellation IS NULL
GROUP BY customer_id
ORDER BY customer_id;
~~~

| customer_id | avg_distance |
| ----------- | ------------ |
| 101         | 20           |
| 102         | 17           |
| 103         | 23           |
| 104         | 10           |
| 105         | 25           |

5. What was the difference between the longest and shortest delivery times for all orders?
~~~sql
SELECT MAX(duration_min) - MIN(duration_min) AS delta_duration
FROM customers_cleaned
JOIN runners_cleaned
ON customers_cleaned.order_id_2 = runners_cleaned.order_id
WHERE cancellation IS NULL;
~~~

| delta_duration |
| -------------- |
| 30             |

6. What was the average speed for each runner for each delivery and do you notice any trend for these values?
~~~sql
# AGAIN, I used max but you could use MIN, AVG: the speed is constant for a given order
# The trend one might spot is that the later the order comes, the quicker the runner is. This is especially true for runner number 2.
SELECT runner_id, order_id, ROUND(MAX(speed_metersseconds)) avg_speed
FROM 
 (SELECT runner_id, order_id, (distance_km*1000)/(duration_min*60) AS speed_metersseconds 
 FROM customers_cleaned
 JOIN runners_cleaned
 ON customers_cleaned.order_id_2 = runners_cleaned.order_id
 WHERE cancellation IS NULL) speed_query
GROUP BY runner_id, order_id
ORDER BY runner_id, order_id
~~~

| runner_id | order_id | avg_speed |
| --------- | -------- | --------- |
| 1         | 1        | 10        |
| 1         | 2        | 12        |
| 1         | 3        | 11        |
| 1         | 10       | 17        |
| 2         | 4        | 10        |
| 2         | 7        | 17        |
| 2         | 8        | 26        |
| 3         | 5        | 11        |

7. What is the successful delivery percentage for each runner?Cleaning stuff: cleaning data and data engineering
~~~sql
SELECT runner_id, CAST(succesful AS FLOAT)/(CAST(succesful AS FLOAT) + CAST(notso AS FLOAT)) AS pct_success
FROM
 # Get the succesful and not succesful orders
 (SELECT runner_id,
 COUNT(CASE WHEN cancellation IS NULL THEN 1 ELSE NULL END) AS succesful,
 COUNT(CASE WHEN cancellation IS NOT NULL THEN 1 ELSE NULL END) AS notso
 FROM
  # Get distinct order id, runner_id and cancellation
  (SELECT DISTINCT order_id, runner_id, cancellation 
  FROM customers_cleaned
  JOIN runners_cleaned
  ON customers_cleaned.order_id_2 = runners_cleaned.order_id) distinct_query
 GROUP BY runner_id
 ORDER BY runner_id) succ_not_query
~~~

| runner_id | pct_success |
| --------- | ----------- |
| 1         | 1           |
| 2         | 0.75        |
| 3         | 0.5         |

### C. Ingredient Optimisation

0. Tables that were created to answer to the following questions:
~~~sql
# GOAL: for each pizza, get multiple rows telling you which topping was added on top of it 
CREATE TEMP TABLE toppings_cleaned AS 
 (SELECT pizza_id, CAST(topping_id AS INTEGER) FROM

 (SELECT pizza_id,
 SPLIT_PART(toppings, ',', 1) AS topping_id
 FROM pizza_runner.pizza_recipes

 UNION ALL

 SELECT pizza_id,
 SPLIT_PART(toppings, ',', 2) AS topping_id
 FROM pizza_runner.pizza_recipes

 UNION ALL

 SELECT pizza_id,
 SPLIT_PART(toppings, ',', 3) AS topping_id
 FROM pizza_runner.pizza_recipes

 UNION ALL

 SELECT pizza_id,
 SPLIT_PART(toppings, ',', 4) AS topping_id
 FROM pizza_runner.pizza_recipes

 UNION ALL

 SELECT pizza_id,
 SPLIT_PART(toppings, ',', 5) AS topping_id
 FROM pizza_runner.pizza_recipes

 UNION ALL

 SELECT pizza_id,
 SPLIT_PART(toppings, ',', 6) AS topping_id
 FROM pizza_runner.pizza_recipes

 UNION ALL

 SELECT pizza_id,
 SPLIT_PART(toppings, ',', 7) AS topping_id
 FROM pizza_runner.pizza_recipes

 UNION ALL

 SELECT pizza_id,
 SPLIT_PART(toppings, ',', 8) AS topping_id
 FROM pizza_runner.pizza_recipes) toppings_subq

 WHERE LENGTH(topping_id) > 0);
~~~

| pizza_id | topping_id |
| -------- | ---------- |
| 1        | 1          |
| 2        | 4          |
| 1        | 2          |
| 2        | 6          |
| 1        | 3          |
| 2        | 7          |
| 1        | 4          |
| 2        | 9          |
| 1        | 5          |
| 2        | 11         |
| 1        | 6          |
| 2        | 12         |
| 1        | 8          |
| 1        | 10         |

1. What are the standard ingredients for each pizza?
~~~sql
SELECT pizza_name, topping_name FROM toppings_cleaned tc
JOIN pizza_runner.pizza_toppings pt
ON pt.topping_id = tc.topping_id
JOIN pizza_runner.pizza_names pn
ON tc.pizza_id = pn.pizza_id
ORDER BY pizza_name;
~~~

| pizza_name | topping_name |
| ---------- | ------------ |
| Meatlovers | Bacon        |
| Meatlovers | BBQ Sauce    |
| Meatlovers | Beef         |
| Meatlovers | Cheese       |
| Meatlovers | Chicken      |
| Meatlovers | Mushrooms    |
| Meatlovers | Pepperoni    |
| Meatlovers | Salami       |
| Vegetarian | Onions       |
| Vegetarian | Tomatoes     |
| Vegetarian | Cheese       |
| Vegetarian | Peppers      |
| Vegetarian | Mushrooms    |
| Vegetarian | Tomato Sauce |

2. What was the most commonly added extra?
~~~sql
SELECT extras_id, topping_name, cnt_extras
FROM
 (SELECT CAST(extras_id AS INTEGER), COUNT(extras_id) AS cnt_extras 
 FROM
  (SELECT order_id_2,
  SPLIT_PART(extras, ',', 1) AS extras_id
  FROM customers_cleaned
    
  UNION ALL
    
  SELECT order_id_2,
  SPLIT_PART(extras, ',', 2) AS extras_id
  FROM customers_cleaned) extras_subq
 WHERE length(extras_id) >=1
 GROUP BY (extras_id)) grouped_toppings_count
    
JOIN pizza_runner.pizza_toppings pt
ON extras_id = pt.topping_id;
~~~

| extras_id | topping_name | cnt_extras |
| --------- | ------------ | ---------- |
| 1         | Bacon        | 4          |
| 4         | Cheese       | 1          |
| 5         | Chicken      | 1          |

3. What was the most common exclusion?
~~~sql
SELECT exclusions_id, topping_name, cnt_exc 
 FROM
 (SELECT CAST(exclusions_id AS INTEGER), COUNT(exclusions_id) AS cnt_exc 
  FROM
  (SELECT order_id_2,
  SPLIT_PART(exclusions, ',', 1) AS exclusions_id
  FROM customers_cleaned
    
  UNION ALL
    
  SELECT order_id_2,
  SPLIT_PART(exclusions, ',', 2) AS exclusions_id
  FROM customers_cleaned) exclusions_subq
 WHERE length(exclusions_id) >=1
 GROUP BY (exclusions_id)) grouped_excl_count
    
JOIN pizza_runner.pizza_toppings pt
ON exclusions_id = pt.topping_id;
~~~

| exclusions_id | topping_name | cnt_exc |
| ------------- | ------------ | ------- |
| 2             | BBQ Sauce    | 1       |
| 4             | Cheese       | 4       |
| 6             | Mushrooms    | 1       |

4. Generate an order item for each record in the customers_orders table in the format of one of the following:
   Meat Lovers, Meat Lovers - Exclude Beef, Meat Lovers - Extra Bacon, Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers
~~~sql
CREATE TEMP TABLE topping_extras_cleaned AS
 SELECT order_id_2 AS order_id, pizza_id, exclusions, extras,
 CASE WHEN LENGTH(excl_id_1) < 1 THEN NULL
 ELSE CAST(excl_id_1 AS INTEGER) END AS excl_id_1, 
 CASE WHEN LENGTH(excl_id_2) < 1 THEN NULL
 ELSE CAST(excl_id_2 AS INTEGER) END AS excl_id_2,  
 CASE WHEN LENGTH(extra_id_1) < 1 THEN NULL
 ELSE CAST(extra_id_1 AS INTEGER) END AS extra_id_1,
 CASE WHEN LENGTH(extra_id_2) < 1 THEN NULL
 ELSE CAST(extra_id_2 AS INTEGER) END AS extra_id_2
 FROM
  (SELECT order_id_2, pizza_id, exclusions, extras, 
  SPLIT_PART(exclusions, ',', 1) AS excl_id_1,
  SPLIT_PART(exclusions, ',', 2) AS excl_id_2,
  SPLIT_PART(extras, ',', 1) AS extra_id_1,
  SPLIT_PART(extras, ',', 2) AS extra_id_2
  FROM customers_cleaned) stuff_splitted_subq;
~~~
~~~sql
SELECT * FROM topping_extras_cleaned;
~~~

| order_id | pizza_id | exclusions | extras | excl_id_1 | excl_id_2 | extra_id_1 | extra_id_2 |
| -------- | -------- | ---------- | ------ | --------- | --------- | ---------- | ---------- |
| 1        | 1        |            |        |           |           |            |            |
| 2        | 1        |            |        |           |           |            |            |
| 3        | 1        |            |        |           |           |            |            |
| 3        | 2        |            |        |           |           |            |            |
| 4        | 1        | 4          |        | 4         |           |            |            |
| 4        | 1        | 4          |        | 4         |           |            |            |
| 4        | 2        | 4          |        | 4         |           |            |            |
| 5        | 1        |            | 1      |           |           | 1          |            |
| 6        | 2        |            |        |           |           |            |            |
| 7        | 2        |            | 1      |           |           | 1          |            |
| 8        | 1        |            |        |           |           |            |            |
| 9        | 1        | 4          | 1, 5   | 4         |           | 1          |  5         |
| 10       | 1        |            |        |           |           |            |            |
| 10       | 1        | 2, 6       | 1, 4   | 2         |  6        | 1          |  4         |

~~~sql
SELECT order_id, CONCAT(pizza_name, ' ', '-', ' ', 'Exclude', ' ', excl_1_name, ' ', excl_2_name, ' ', '-', ' ', 'Extra', ' ', extra_1_name, ' ', extra_2_name) AS pizza_order
FROM  
 (SELECT * FROM topping_extras_cleaned tec
 JOIN pizza_runner.pizza_names pn
 ON tec.pizza_id = pn.pizza_id
    
 LEFT JOIN
  (SELECT topping_id, topping_name AS excl_1_name FROM pizza_runner.pizza_toppings) pt 
 ON tec.excl_id_1 = pt.topping_id
 LEFT JOIN
  (SELECT topping_id, topping_name AS excl_2_name FROM pizza_runner.pizza_toppings) pt_1 
 ON tec.excl_id_2 = pt_1.topping_id
 LEFT JOIN
  (SELECT topping_id, topping_name AS extra_1_name FROM pizza_runner.pizza_toppings) pt_2 
 ON tec.extra_id_1 = pt_2.topping_id
 LEFT JOIN
  (SELECT topping_id, topping_name AS extra_2_name FROM pizza_runner.pizza_toppings) pt_3 
 ON tec.extra_id_2 = pt_3.topping_id) all_names_subq
ORDER BY order_id;
~~~

| order_id | pizza_order                                                   |
| -------- | ------------------------------------------------------------- |
| 1        | Meatlovers - Exclude   - Extra                                |
| 2        | Meatlovers - Exclude   - Extra                                |
| 3        | Vegetarian - Exclude   - Extra                                |
| 3        | Meatlovers - Exclude   - Extra                                |
| 4        | Vegetarian - Exclude Cheese  - Extra                          |
| 4        | Meatlovers - Exclude Cheese  - Extra                          |
| 4        | Meatlovers - Exclude Cheese  - Extra                          |
| 5        | Meatlovers - Exclude   - Extra Bacon                          |
| 6        | Vegetarian - Exclude   - Extra                                |
| 7        | Vegetarian - Exclude   - Extra Bacon                          |
| 8        | Meatlovers - Exclude   - Extra                                |
| 9        | Meatlovers - Exclude Cheese  - Extra Bacon Chicken            |
| 10       | Meatlovers - Exclude   - Extra                                |
| 10       | Meatlovers - Exclude BBQ Sauce Mushrooms - Extra Bacon Cheese |

5. Generate an alphabetically ordered comma separated ingredient list for each pizza order from the customer_orders table and add a 2x in front of any relevant ingredients. For example: "Meat Lovers: 2xBacon, Beef, ... , Salami"
~~~sql
SELECT order_ide, CONCAT(pzz_name, ':', ' ', topping, ', ', two_tn, ', ', three_tn, ', ', four_tn, ', ', five_tn, ', ', six_tn, ', ', seven_tn, ', ', eight_tn)
FROM
 (SELECT order_ide, pzz_name, topping, two_tn, three_tn, four_tn, five_tn, six_tn, seven_tn, eight_tn 
 FROM
  (SELECT * 
  FROM
   (SELECT row_nr AS row_num, order_id AS order_ide, pizza_name as pzz_name, topping_name_2x AS topping, unique_index  FROM all_info 
   WHERE unique_index = 1) one
  LEFT JOIN
   (SELECT row_nr, order_id, unique_index, topping_name_2x AS two_tn FROM all_info
   WHERE unique_index = 2) two 
   ON two.row_nr = one.row_num AND
   two.order_id = one.order_ide 
  LEFT JOIN
   (SELECT row_nr, order_id, unique_index, topping_name_2x AS three_tn FROM all_info
   WHERE unique_index = 3) three
   ON three.row_nr = one.row_num AND
   three.order_id = one.order_ide  
  LEFT JOIN
   (SELECT row_nr, order_id, unique_index, topping_name_2x AS four_tn FROM all_info
   WHERE unique_index = 4) four
   ON four.row_nr = one.row_num AND
   four.order_id = one.order_ide 
  LEFT JOIN
   (SELECT row_nr, order_id, unique_index, topping_name_2x AS five_tn FROM all_info
   WHERE unique_index = 5) five 
   ON five.row_nr = one.row_num AND
   five.order_id = one.order_ide 
  LEFT JOIN
   (SELECT row_nr, order_id, unique_index, topping_name_2x AS six_tn FROM all_info
   WHERE unique_index = 6) six 
   ON six.row_nr = one.row_num AND
   six.order_id = one.order_ide 
  LEFT JOIN
   (SELECT row_nr, order_id, unique_index, topping_name_2x AS seven_tn FROM all_info
   WHERE unique_index = 7) seven 
   ON seven.row_nr = one.row_num AND
   seven.order_id = one.order_ide 
  LEFT JOIN
   (SELECT row_nr, order_id, unique_index, topping_name_2x AS eight_tn FROM all_info
   WHERE unique_index = 8) eight_tn 
   ON eight_tn.row_nr = one.row_num AND
   eight_tn.order_id = one.order_ide ) all_ingredients) pzz_ingr;
~~~

| order_ide | concat                                                                               |
| --------- | ------------------------------------------------------------------------------------ |
| 1         | Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami    |
| 2         | Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami    |
| 3         | Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami    |
| 3         | Vegetarian: Cheese, Mushrooms, Onions, Peppers, Tomatoes, Tomato Sauce, ,            |
| 4         | Meatlovers: Bacon, BBQ Sauce, Beef, Chicken, Mushrooms, Pepperoni, Salami,           |
| 4         | Meatlovers: Bacon, BBQ Sauce, Beef, Chicken, Mushrooms, Pepperoni, Salami,           |
| 4         | Vegetarian: Mushrooms, Onions, Peppers, Tomatoes, Tomato Sauce, , ,                  |
| 5         | Meatlovers: 2x Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami |
| 6         | Vegetarian: Cheese, Mushrooms, Onions, Peppers, Tomatoes, Tomato Sauce, ,            |
| 7         | Vegetarian: Bacon, Cheese, Mushrooms, Onions, Peppers, Tomatoes, Tomato Sauce,       |
| 8         | Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami    |
| 9         | Meatlovers: 2x Bacon, BBQ Sauce, Beef, 2x Chicken, Mushrooms, Pepperoni, Salami,     |
| 10        | Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami    |
| 10        | Meatlovers: 2x Bacon, Beef, 2x Cheese, Chicken, Pepperoni, Salami, ,                 |

6. What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?
~~~sql
SELECT topping_name, SUM(ingr_cnt_adjusted) AS ingr_cnt_adjusted
 FROM
 (SELECT 
 CASE WHEN STRPOS(topping_name_2x, '2x') >0
 THEN ingr_cnt*2 ELSE ingr_cnt END AS "ingr_cnt_adjusted",
 CASE WHEN STRPOS(topping_name_2x, '2x') >0
 THEN RIGHT(topping_name_2x, LENGTH(topping_name_2x) - 3) ELSE topping_name_2x END AS "topping_name"
 FROM
  (SELECT topping_name_2x, COUNT(topping_name_2x) AS ingr_cnt FROM all_info
  GROUP BY topping_name_2x
 ORDER BY topping_name_2x) ingr_count_subq) adjusted_subq
GROUP BY topping_name
ORDER BY ingr_cnt_adjusted DESC;
~~~

| topping_name | ingr_cnt_adjusted |
| ------------ | ----------------- |
| Bacon        | 14                |
| Mushrooms    | 13                |
| Chicken      | 11                |
| Cheese       | 11                |
| Pepperoni    | 10                |
| Salami       | 10                |
| Beef         | 10                |
| BBQ Sauce    | 9                 |
| Tomato Sauce | 4                 |
| Onions       | 4                 |
| Tomatoes     | 4                 |
| Peppers      | 4                 |

### D. Pricing and Ratings
1. If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?
~~~sql
SELECT SUM(price) AS earnings 
 FROM
 (SELECT order_id, pizza_id, pizza_name,
 CASE WHEN pizza_name = 'Meatlovers' THEN 12 ELSE 10 END AS price 
  FROM
  (SELECT * 
  FROM 
   (SELECT * FROM customers_cleaned cc
   JOIN 
    (SELECT order_id, cancellation FROM runners_cleaned) rc 
   ON cc.order_id_2 = rc.order_id
   WHERE cancellation is NULL) no_canc_orders
    
   JOIN
    
   (SELECT pizza_id AS pizza_id_2, pizza_name FROM pizza_runner.pizza_names) pn
  ON no_canc_orders.pizza_id = pn.pizza_id_2) no_canc_orders_pzz_name) price_query;
~~~

| earnings |
| -------- |
| 138      |

2. What if there was an additional $1 charge for any pizza extras? Add cheese is $1 extra
~~~sql
SELECT SUM(earnings) AS tot_earnings
FROM
 (SELECT SUM(price) AS earnings FROM
  (SELECT order_id, pizza_id, pizza_name,
  CASE WHEN pizza_name = 'Meatlovers' THEN 12 ELSE 10 END AS price 
  FROM
   (SELECT * FROM 
    (SELECT * FROM customers_cleaned cc
    JOIN 
    (SELECT order_id, cancellation FROM runners_cleaned) rc 
    ON cc.order_id_2 = rc.order_id
    WHERE cancellation is NULL) no_canc_orders
    
   JOIN
    
    (SELECT pizza_id AS pizza_id_2, pizza_name FROM pizza_runner.pizza_names) pn
   ON no_canc_orders.pizza_id = pn.pizza_id_2) no_canc_orders_pzz_name) price_query
    
UNION ALL    
SELECT COUNT(*) AS earnings FROM extra_cleaned) earnings_extra;
~~~

| tot_earnings |
| ------------ |
| 144          |

3. The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.
~~~sql
DROP TABLE IF EXISTS customer_ratings;
CREATE TABLE customer_ratings (
  order_id INTEGER,
  rating_1_5 INTEGER
);

INSERT INTO customer_ratings
  (order_id, rating_1_5)
VALUES
  (1, 5),
  (2, 1),
  (3, 1),
  (4, 4),
  (5, 3),
  (6, 2),
  (7, 3),
  (8, 2),
  (9, 3),
  (10, 1);
~~~
~~~sql
    SELECT * FROM customer_ratings;
~~~

| order_id | Rating_1_5 |
| -------- | ---------- |
| 1        | 5          |
| 2        | 1          |
| 3        | 1          |
| 4        | 4          |
| 5        | 3          |
| 6        | 2          |
| 7        | 3          |
| 8        | 2          |
| 9        | 3          |
| 10       | 1          |

4. Using your newly generated table - can you join all of the information together to form a table which has the following information for successful deliveries? customer_id order_id runner_id rating order_time pickup_time Time between order and pickup Delivery duration Average speed Total number of pizzas
~~~sql
SELECT DISTINCT customer_id, order_id_2 AS order_id, runner_id, rating_1_5, order_time, pickup_time_ts,
pickup_time_ts - order_time AS delta_time, duration_min AS delivery_duration, ROUND(distance_km*1000/(duration_min*60)) AS avg_m_s_speed, pizza_number
FROM
 (SELECT * FROM
  (SELECT * FROM
   (SELECT * FROM
    (SELECT order_id_2 AS order_id_to_drop, COUNT(order_id_2) AS pizza_number
    FROM customers_cleaned
    GROUP BY order_id_to_drop) pzz_subq
   JOIN customers_cleaned cc
   ON pzz_subq.order_id_to_drop = cc.order_id_2) cc_w_pzz
  JOIN customer_ratings cr
  ON cr.order_id = cc_w_pzz.order_id_2) cc_w_rating
 JOIN runners_cleaned rc
 ON rc.order_id = cc_w_rating.order_id_2
 WHERE cancellation IS NULL) all_info_subq
ORDER BY order_id;
~~~

| customer_id | order_id | runner_id | rating_1_5 | order_time               | pickup_time_ts           | delta_time      | delivery_duration | avg_m_s_speed | pizza_number |
| ----------- | -------- | --------- | ---------- | ------------------------ | ------------------------ | --------------- | ----------------- | ------------- | ------------ |
| 101         | 1        | 1         | 5          | 2020-01-01T18:05:02.000Z | 2020-01-01T18:15:34.000Z | {"minutes":10,"seconds":32} | 32                | 10            | 1            |
| 101         | 2        | 1         | 1          | 2020-01-01T19:00:52.000Z | 2020-01-01T19:10:54.000Z | {"minutes":10,"seconds":2} | 27                | 12            | 1            |
| 102         | 3        | 1         | 1          | 2020-01-02T23:51:23.000Z | 2020-01-03T00:12:37.000Z | {"minutes":21,"seconds":14} | 20                | 11            | 2            |
| 103         | 4        | 2         | 4          | 2020-01-04T13:23:46.000Z | 2020-01-04T13:53:03.000Z | {"minutes":29,"seconds":17} | 40                | 10            | 3            |
| 104         | 5        | 3         | 3          | 2020-01-08T21:00:29.000Z | 2020-01-08T21:10:57.000Z | {"minutes":10,"seconds":28} | 15                | 11            | 1            |
| 105         | 7        | 2         | 3          | 2020-01-08T21:20:29.000Z | 2020-01-08T21:30:45.000Z | {"minutes":10,"seconds":16} | 25                | 17            | 1            |
| 102         | 8        | 2         | 2          | 2020-01-09T23:54:33.000Z | 2020-01-10T00:15:02.000Z | {"minutes":20,"seconds":29} | 15                | 26            | 1            |
| 104         | 10       | 1         | 1          | 2020-01-11T18:34:49.000Z | 2020-01-11T18:50:20.000Z | {"minutes":15,"seconds":31} | 10                | 17            | 2            |

5. If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled - how much money does Pizza Runner have left over after these deliveries?
~~~sql
CREATE TEMP TABLE all_info_prices AS   
 (SELECT order_id_2, pizza_name, distance_km*0.30 AS runner_pay,
 CASE WHEN pizza_name = 'Meatlovers' THEN 12
 ELSE 10 END AS pizza_price
 FROM
  (SELECT *
  FROM
   (SELECT * 
   FROM customers_cleaned
   JOIN
    (SELECT order_id, cancellation, distance_km FROM runners_cleaned) km_subq
   ON km_subq.order_id = customers_cleaned.order_id_2
   WHERE cancellation IS NULL) cc_km
  JOIN pizza_runner.pizza_names pn
  ON cc_km.pizza_id = pn.pizza_id) all_info_subq);
~~~
~~~sql
SELECT SUM(total_pizza_price - runner_pay) AS money_made
FROM    
 (# runners are paid once per order
 SELECT order_id_2 AS order_id, runner_pay, total_pizza_price 
 FROM
  ((SELECT DISTINCT order_id_2, runner_pay FROM all_info_prices) runner_pay_subq
  JOIN  
   (SELECT order_id_2 AS order_id_to_drop, SUM(pizza_price) AS total_pizza_price
   FROM all_info_prices
   GROUP BY order_id_to_drop) pzz_price_subq
  ON pzz_price_subq.order_id_to_drop = runner_pay_subq.order_id_2) all_info_subq) final_subq;

| money_made |
| ---------- |
| 94.44      |
~~~

### E. Bonus Questions
If Danny wants to expand his range of pizzas - how would this impact the existing data design? Write an INSERT statement to demonstrate what would happen if a new Supreme pizza with all the toppings was added to the Pizza Runner menu?
# It would not impact the existing db significantly. It is enough to insert a new pizza_id and pizza_name in pizza_runner.pizza_names
~~~sql
INSERT INTO pizza_runner.pizza_names
  ("pizza_id", "pizza_name")
VALUES
  (3, 'Supreme');

# The other table affected by this is pizza_recipes:
INSERT INTO pizza_runner.pizza_recipes
  ("pizza_id", "toppings")
VALUES
  (3, '1, 2, 3, 4, 5, 6, 7, 8, 9, 10');
~~~
