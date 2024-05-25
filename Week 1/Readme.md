### Case study Questions
1. What is the total amount each customer spent at the restaurant?
~~~sql
SELECT customer_id, SUM(price) 
FROM dannys_diner.sales s 
LEFT JOIN dannys_diner.menu m 
ON s.product_id = m.product_id
GROUP BY customer_id;
~~~

| customer_id | sum |
| ----------- | --- |
| B           | 74  |
| C           | 36  |
| A           | 76  |

2. How many days has each customer visited the restaurant?
~~~sql
SELECT customer_id, COUNT(DISTINCT order_date) FROM dannys_diner.sales
GROUP BY customer_id;
~~~

| customer_id | count |
| ----------- | ----- |
| A           | 4     |
| B           | 6     |
| C           | 2     |

3. What was the first item from the menu purchased by each customer?
~~~sql
SELECT DISTINCT left_table.product_name, right_table.customer_id FROM dannys_diner.menu left_table
JOIN (SELECT product_id, customer_id, order_date FROM dannys_diner.sales s
JOIN (SELECT customer_id AS c_id, MIN(order_date) AS minimum
FROM dannys_diner.sales
GROUP BY customer_id) yuhu
ON s.customer_id = yuhu.c_id
WHERE s.order_date = yuhu.minimum) right_table
ON left_table.product_id = right_table.product_id
ORDER BY customer_id;
~~~

| product_name | customer_id |
| ------------ | ----------- |
| curry        | A           |
| sushi        | A           |
| curry        | B           |
| ramen        | C           |

4. What is the most purchased item on the menu and how many times was it purchased by all customers?
~~~sql
SELECT DISTINCT left_table.product_name FROM dannys_diner.menu left_table
JOIN(SELECT product_id, COUNT(order_date) AS nr_ordered FROM dannys_diner.sales
GROUP BY product_id
ORDER BY nr_ordered DESC 
LIMIT 1) right_table
ON left_table.product_id = right_table.product_id;
~~~

| product_name |
| ------------ |
| ramen        |

5. Which item was the most popular for each customer?
~~~sql
SELECT m.product_name, s.customer_id, s.nr_ordered FROM dannys_diner.menu m
JOIN(SELECT customer_id, product_id, COUNT(order_date) as nr_ordered FROM dannys_diner.sales
GROUP BY customer_id, product_id) s
ON m.product_id = s.product_id
ORDER BY s.customer_id, s.nr_ordered DESC;
~~~

| product_name | customer_id | nr_ordered |
| ------------ | ----------- | ---------- |
| ramen        | A           | 3          |
| curry        | A           | 2          |
| sushi        | A           | 1          |
| curry        | B           | 2          |
| sushi        | B           | 2          |
| ramen        | B           | 2          |
| ramen        | C           | 3          |

6. Which item was purchased first by the customer after they became a member?
~~~sql
SELECT m.product_name, top_items.customer_id FROM (SELECT full_subq.product_id, full_subq.customer_id
FROM
 (SELECT customer_id, MIN(subq.diff) AS min_date
 FROM
  (SELECT s.*, memb.join_date, order_date - join_date AS diff FROM dannys_diner.sales s
  JOIN dannys_diner.members memb
  ON s.customer_id = memb.customer_id
  WHERE s.order_date - memb.join_date > 0) subq
  GROUP BY subq.customer_id) grouped_subq
    
JOIN
    
(SELECT s.*, memb.join_date, order_date - join_date AS diff FROM dannys_diner.sales s
JOIN dannys_diner.members memb
ON s.customer_id = memb.customer_id
WHERE s.order_date - memb.join_date > 0) full_subq
ON grouped_subq.customer_id = full_subq.customer_id
AND grouped_subq.min_date = full_subq.diff) top_items
    
JOIN
    
dannys_diner.menu m
ON m.product_id = top_items.product_id;
~~~

| product_name | customer_id |
| ------------ | ----------- |
| sushi        | B           |
| ramen        | A           |

~~~sql
/* Shorter Query */
SELECT subq.diff, m.product_name, subq.customer_id 
FROM
 (SELECT s.*, memb.join_date, order_date - join_date AS diff
 FROM dannys_diner.sales s
 JOIN dannys_diner.members memb
 ON s.customer_id = memb.customer_id
 WHERE s.order_date - memb.join_date > 0) subq
    
JOIN dannys_diner.menu m
ON m.product_id = subq.product_id
ORDER BY customer_id, diff;
~~~

| diff | product_name | customer_id |
| ---- | ------------ | ----------- |
| 3    | ramen        | A           |
| 4    | ramen        | A           |
| 4    | ramen        | A           |
| 2    | sushi        | B           |
| 7    | ramen        | B           |
| 23   | ramen        | B           |

7. Which item was purchased just before the customer became a member?
~~~sql
# Getting the product name and the associated customer_id
SELECT product_name, customer_id, order_date
FROM
 (# Getting the product_id by self-joining the subq
 SELECT * FROM
  (# Getting the customer_id with its max diff, i.e. the one that is closest to 0
  SELECT customer_id, MAX(subq.diff) as max_diff
  FROM
   (# getting the diff
   SELECT s.*, memb.join_date, order_date - join_date AS diff 
   FROM dannys_diner.sales s
   JOIN dannys_diner.members memb
   ON s.customer_id = memb.customer_id
   WHERE s.order_date - memb.join_date < 0) subq
  GROUP BY customer_id) grouped_subq

JOIN 

 (SELECT s.product_id, s.order_date, s.customer_id AS customer_id_to_drop, memb.join_date, order_date - join_date AS diff
 FROM dannys_diner.sales s

 JOIN dannys_diner.members memb
 ON s.customer_id = memb.customer_id
 WHERE s.order_date - memb.join_date < 0) full_subq

ON grouped_subq.customer_id = full_subq.customer_id_to_drop
AND grouped_subq.max_diff = full_subq.diff) top_items

JOIN
    
dannys_diner.menu m
ON m.product_id = top_items.product_id
~~~

| product_name | customer_id | order_date               |
| ------------ | ----------- | ------------------------ |
| sushi        | B           | 2021-01-04T00:00:00.000Z |
| sushi        | A           | 2021-01-01T00:00:00.000Z |
| curry        | A           | 2021-01-01T00:00:00.000Z |

~~~sql
/* Shorter Query */
SELECT subq.diff, m.product_name, subq.customer_id
FROM
 (SELECT s.*, memb.join_date, order_date - join_date AS diff 
 FROM dannys_diner.sales s
 JOIN dannys_diner.members memb
 ON s.customer_id = memb.customer_id
 WHERE s.order_date - memb.join_date < 0) subq
    
JOIN dannys_diner.menu m
ON m.product_id = subq.product_id
ORDER BY customer_id, diff DESC;
~~~

| diff | product_name | customer_id |
| ---- | ------------ | ----------- |
| -6   | sushi        | A           |
| -6   | curry        | A           |
| -5   | sushi        | B           |
| -7   | curry        | B           |
| -8   | curry        | B           |

8. What is the total items and amount spent for each member before they became a member?
9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

### Bonus Questions
1. Join all the things
2. Rank all the things
