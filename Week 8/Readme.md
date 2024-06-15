### Data Exploration and Cleansing
1. Update the fresh_segments.interest_metrics table by modifying the month_year column to be a date data type with the start of the month
~~~sql
CREATE TEMP TABLE interest_metrics_cleaned AS
 (SELECT interest_id, composition, index_value, ranking, percentile_ranking, 
 CASE WHEN month_year IS NULL THEN CAST(month_year AS TIMESTAMP)
 ELSE CAST(CONCAT(_year,'-',RIGHT(CONCAT('00', _month), 2),'-01') AS TIMESTAMP)
 END AS month_year
 FROM fresh_segments.interest_metrics);
~~~

2. What is count of records in the fresh_segments.interest_metrics for each month_year value sorted in chronological order (earliest to latest) with the null values appearing first?
~~~sql
SELECT month_year, COUNT(*)
FROM interest_metrics_cleaned
WHERE month_year IS NULL
GROUP BY month_year
    
UNION ALL
    
SELECT * 
FROM
 (SELECT month_year, COUNT(*)
 FROM interest_metrics_cleaned
 WHERE month_year IS NOT NULL
 GROUP BY month_year
 ORDER BY month_year) chr_subq;
~~~
| month_year               | count |
| ------------------------ | ----- |
|                          | 1194  |
| 2018-07-01T00:00:00.000Z | 729   |
| 2018-08-01T00:00:00.000Z | 767   |
| 2018-09-01T00:00:00.000Z | 780   |
| 2018-10-01T00:00:00.000Z | 857   |
| 2018-11-01T00:00:00.000Z | 928   |
| 2018-12-01T00:00:00.000Z | 995   |
| 2019-01-01T00:00:00.000Z | 973   |
| 2019-02-01T00:00:00.000Z | 1121  |
| 2019-03-01T00:00:00.000Z | 1136  |
| 2019-04-01T00:00:00.000Z | 1099  |
| 2019-05-01T00:00:00.000Z | 857   |
| 2019-06-01T00:00:00.000Z | 824   |
| 2019-07-01T00:00:00.000Z | 864   |
| 2019-08-01T00:00:00.000Z | 1149  |

3. What do you think we should do with these null values in the fresh_segments.interest_metrics
~~~sql
# Either fill them with a reasonable assumption or drop them
~~~

4. How many interest_id values exist in the fresh_segments.interest_metrics table but not in the fresh_segments.interest_map table? What about the other way around?
~~~sql
SELECT COUNT(CASE WHEN eheh.interest_id IS NOT NULL AND eheh.id IS NULL THEN 1 ELSE NULL END) AS missing_interests_in_interest_map
FROM
 (SELECT DISTINCT ime.interest_id, ima.id
 FROM 
 fresh_segments.interest_metrics ime
 LEFT JOIN fresh_segments.interest_map ima
 ON CAST(ime.interest_id AS INTEGER) = ima.id) eheh;
~~~
| missing_interests_in_interest_map |
| --------------------------------- |
| 0                                 |

~~~sql
SELECT COUNT(CASE WHEN eheh.interest_id IS NULL AND eheh.id IS NOT NULL THEN 1 ELSE NULL END) AS missing_interests_in_interest_metrics
FROM
 (SELECT DISTINCT ime.interest_id, ima.id
 FROM 
 fresh_segments.interest_metrics ime
 RIGHT JOIN fresh_segments.interest_map ima
 ON CAST(ime.interest_id AS INTEGER) = ima.id) eheh;
~~~
| missing_interests_in_interest_metrics |
| ------------------------------------- |
| 7                                     |

5. Summarise the id values in the fresh_segments.interest_map by its total record count in this table
~~~sql
    SELECT COUNT(id) AS id_count
    FROM fresh_segments.interest_map;
~~~
| id_count |
| -------- |
| 1209     |

6. What sort of table join should we perform for our analysis and why? Check your logic by checking the rows where interest_id = 21246 in your joined output and include all columns from fresh_segments.interest_metrics and all columns from fresh_segments.interest_map except from the id column.
~~~sql
# An inner join, since there are records in a table that are missing in the other one.
~~~

7. Are there any records in your joined table where the month_year value is before the created_at value from the fresh_segments.interest_map table? Do you think these values are valid and why?
~~~sql
# Yes, yes, because we forced the dates to be on the first day of every month, and month_year preceeds the date in the created_at column just because of that. 
~~~

### Interest Analysis
1. Which interests have been present in all month_year dates in our dataset?
~~~sql
SELECT COUNT(interest_id)
FROM
 (SELECT interest_id, COUNT(month_year)
 FROM
  (SELECT DISTINCT month_year, interest_id 
  FROM interest_metrics_cleaned
  GROUP BY month_year, interest_id) dist_group_subq
 GROUP BY interest_id
 HAVING COUNT(month_year) = (SELECT COUNT(DISTINCT month_year) FROM interest_metrics_cleaned)) all_months_subq;
~~~
| count |
| ----- |
| 480   |

### Segment Analysis
1. Using our filtered dataset by removing the interests with less than 6 months worth of data, which are the top 10 and bottom 10 interests which have the largest composition values in any month_year? Only use the maximum composition value for each interest but you must keep the corresponding month_year
~~~sql
SELECT * FROM
 (SELECT interest_id, month_year, MAX(composition) FROM
  (SELECT interest_id
  FROM
   (SELECT DISTINCT month_year, interest_id 
   FROM interest_metrics_cleaned
   GROUP BY month_year, interest_id) dist_group_subq
  GROUP BY interest_id
  HAVING COUNT(month_year) >= 6) at_least_6_subq
    
 JOIN
 interest_metrics_cleaned
 USING(interest_id)
    
 GROUP BY interest_id, month_year
 ORDER BY MAX(composition) DESC
 LIMIT 10) max_subq
    
UNION ALL
    
 (SELECT interest_id, month_year, MAX(composition) FROM
  (SELECT interest_id
  FROM
   (SELECT DISTINCT month_year, interest_id 
   FROM interest_metrics_cleaned
   GROUP BY month_year, interest_id) dist_group_subq
  GROUP BY interest_id
  HAVING COUNT(month_year) >= 6) at_least_6_subq
    
 JOIN interest_metrics_cleaned
 USING(interest_id)
    
 GROUP BY interest_id, month_year
 ORDER BY MAX(composition) 
 LIMIT 10);
~~~
| interest_id | month_year               | max   |
| ----------- | ------------------------ | ----- |
| 21057       | 2018-12-01T00:00:00.000Z | 21.2  |
| 21057       | 2018-10-01T00:00:00.000Z | 20.28 |
| 21057       | 2018-11-01T00:00:00.000Z | 19.45 |
| 21057       | 2019-01-01T00:00:00.000Z | 18.99 |
| 6284        | 2018-07-01T00:00:00.000Z | 18.82 |
| 21057       | 2019-02-01T00:00:00.000Z | 18.39 |
| 21057       | 2018-09-01T00:00:00.000Z | 18.18 |
| 39          | 2018-07-01T00:00:00.000Z | 17.44 |
| 77          | 2018-07-01T00:00:00.000Z | 17.19 |
| 12133       | 2018-10-01T00:00:00.000Z | 15.15 |
| 45524       | 2019-05-01T00:00:00.000Z | 1.51  |
| 35742       | 2019-06-01T00:00:00.000Z | 1.52  |
| 39336       | 2019-05-01T00:00:00.000Z | 1.52  |
| 4918        | 2019-05-01T00:00:00.000Z | 1.52  |
| 34083       | 2019-06-01T00:00:00.000Z | 1.52  |
| 20768       | 2019-05-01T00:00:00.000Z | 1.52  |
| 44449       | 2019-04-01T00:00:00.000Z | 1.52  |
| 6314        | 2019-06-01T00:00:00.000Z | 1.53  |
| 6127        | 2019-05-01T00:00:00.000Z | 1.53  |
| 36877       | 2019-05-01T00:00:00.000Z | 1.53  |

2. Which 5 interests had the lowest average ranking value?
~~~sql
SELECT interest_id, ROUND(AVG(ranking)) AS avg_ranking
FROM interest_metrics_cleaned
GROUP BY interest_id
ORDER BY AVG(ranking)
LIMIT 5;
~~~
| interest_id | avg_ranking |
| ----------- | ----------- |
| 41548       | 1           |
| 42203       | 4           |
| 115         | 6           |
| 48154       | 8           |
| 171         | 9           |

3. Which 5 interests had the largest standard deviation in their percentile_ranking value?
~~~sql
SELECT interest_id, STDDEV(percentile_ranking) AS sdv_ranking
FROM interest_metrics_cleaned
GROUP BY interest_id
ORDER BY STDDEV(percentile_ranking) DESC
LIMIT 5
OFFSET 13;
~~~
| interest_id | sdv_ranking        |
| ----------- | ------------------ |
| 6260        | 41.27382281785878  |
| 131         | 30.720767894048482 |
| 150         | 30.363974871548024 |
| 23          | 30.175047086403474 |
| 20764       | 28.97491995962485  |

4. For the 5 interests found in the previous question - what was minimum and maximum percentile_ranking values for each interest and its corresponding year_month value? Can you describe what is happening for these 5 interests?
~~~sql
# The interactions for all the considered interest_id peaked in the first month
SELECT * FROM
 (SELECT interest_id, month_year AS month_min, min
 FROM
  (SELECT *, ROW_NUMBER() OVER(PARTITION BY interest_id ORDER BY interest_id, max)
  FROM
   (SELECT interest_id, month_year, MAX(percentile_ranking), MIN(percentile_ranking)
   FROM
    (SELECT interest_id, STDDEV(percentile_ranking) AS sdv_ranking
   FROM interest_metrics_cleaned
   GROUP BY interest_id
   ORDER BY STDDEV(percentile_ranking) DESC
   LIMIT 5
   OFFSET 13) top_5_subq
    
  JOIN interest_metrics_cleaned
  USING (interest_id)
  GROUP BY month_year, interest_id) min_max_subq) just_the_min
 WHERE row_number = 1) min_agg
    
JOIN
   
 (SELECT interest_id, month_year AS month_max, max
 FROM
  (SELECT *, ROW_NUMBER() OVER(PARTITION BY interest_id ORDER BY interest_id, max DESC)
  FROM
   (SELECT interest_id, month_year, MAX(percentile_ranking), MIN(percentile_ranking)
   FROM
    (SELECT interest_id, STDDEV(percentile_ranking) AS sdv_ranking
    FROM interest_metrics_cleaned
    GROUP BY interest_id
    ORDER BY STDDEV(percentile_ranking) DESC
   LIMIT 5
   OFFSET 13) top_5_subq
    
  JOIN interest_metrics_cleaned
  USING (interest_id)
  GROUP BY month_year, interest_id) min_max_subq) just_the_max
 WHERE row_number = 1) max_agg
USING(interest_id);
~~~
| interest_id | month_min                | min   | month_max                | max   |
| ----------- | ------------------------ | ----- | ------------------------ | ----- |
| 131         | 2019-03-01T00:00:00.000Z | 4.84  | 2018-07-01T00:00:00.000Z | 75.03 |
| 150         | 2019-08-01T00:00:00.000Z | 10.01 | 2018-07-01T00:00:00.000Z | 93.28 |
| 20764       | 2019-08-01T00:00:00.000Z | 11.23 | 2018-07-01T00:00:00.000Z | 86.15 |
| 23          | 2019-08-01T00:00:00.000Z | 7.92  | 2018-07-01T00:00:00.000Z | 86.69 |
| 6260        | 2019-08-01T00:00:00.000Z | 2.26  | 2018-07-01T00:00:00.000Z | 60.63 |

### Index Analysis
1. What is the top 10 interests by the average composition for each month?
~~~sql
SELECT * FROM
 (SELECT *, ROW_NUMBER() OVER(PARTITION BY month_year ORDER BY month_year, avg_composition DESC)
 FROM
  (SELECT interest_id, month_year, ROUND(AVG(composition)) AS avg_composition
  FROM interest_metrics_cleaned
  WHERE interest_id IS NOT NULL AND month_year IS NOT NULL
  GROUP BY interest_id, month_year) avg_subq) rank_subq
  WHERE row_number <= 10
  LIMIT 20;
~~~
| interest_id | month_year               | avg_composition | row_number |
| ----------- | ------------------------ | --------------- | ---------- |
| 6284        | 2018-07-01T00:00:00.000Z | 19              | 1          |
| 77          | 2018-07-01T00:00:00.000Z | 17              | 2          |
| 39          | 2018-07-01T00:00:00.000Z | 17              | 3          |
| 171         | 2018-07-01T00:00:00.000Z | 15              | 4          |
| 17786       | 2018-07-01T00:00:00.000Z | 14              | 5          |
| 6286        | 2018-07-01T00:00:00.000Z | 14              | 6          |
| 4           | 2018-07-01T00:00:00.000Z | 14              | 7          |
| 4898        | 2018-07-01T00:00:00.000Z | 14              | 8          |
| 5968        | 2018-07-01T00:00:00.000Z | 13              | 9          |
| 4897        | 2018-07-01T00:00:00.000Z | 13              | 10         |
| 6284        | 2018-08-01T00:00:00.000Z | 14              | 1          |
| 77          | 2018-08-01T00:00:00.000Z | 13              | 2          |
| 21057       | 2018-08-01T00:00:00.000Z | 12              | 3          |
| 39          | 2018-08-01T00:00:00.000Z | 12              | 4          |
| 5969        | 2018-08-01T00:00:00.000Z | 11              | 5          |
| 12133       | 2018-08-01T00:00:00.000Z | 10              | 6          |
| 115         | 2018-08-01T00:00:00.000Z | 9               | 7          |
| 6206        | 2018-08-01T00:00:00.000Z | 9               | 8          |
| 6286        | 2018-08-01T00:00:00.000Z | 9               | 9          |
| 19298       | 2018-08-01T00:00:00.000Z | 9               | 10         |

2. For all of these top 10 interests - which interest appears the most often?
~~~sql
SELECT interest_id, COUNT(interest_id)
FROM
 (SELECT * FROM
  (SELECT *, ROW_NUMBER() OVER(PARTITION BY month_year ORDER BY month_year, avg_composition DESC)
  FROM
   (SELECT interest_id, month_year, ROUND(AVG(composition)) AS avg_composition
   FROM interest_metrics_cleaned
   WHERE interest_id IS NOT NULL AND month_year IS NOT NULL
  GROUP BY interest_id, month_year) avg_subq) rank_subq
 WHERE row_number <= 10) row_nr_subq
GROUP BY interest_id
ORDER BY count DESC
LIMIT 1;
~~~
| interest_id | count |
| ----------- | ----- |
| 6284        | 12    |

3. What is the average of the average composition for the top 10 interests for each month?
~~~sql
SELECT month_year, AVG(avg_composition) As avg_avg_composition FROM
 (SELECT *, ROW_NUMBER() OVER(PARTITION BY month_year ORDER BY month_year, avg_composition DESC)
 FROM
  (SELECT interest_id, month_year, ROUND(AVG(composition)) AS avg_composition
  FROM interest_metrics_cleaned
  WHERE interest_id IS NOT NULL AND month_year IS NOT NULL
  GROUP BY interest_id, month_year) avg_subq) rank_subq
 WHERE row_number <= 10
 GROUP BY month_year
 ORDER BY month_year;
~~~
| month_year               | avg_avg_composition |
| ------------------------ | ------------------- |
| 2018-07-01T00:00:00.000Z | 15                  |
| 2018-08-01T00:00:00.000Z | 10.8                |
| 2018-09-01T00:00:00.000Z | 12.2                |
| 2018-10-01T00:00:00.000Z | 13.6                |
| 2018-11-01T00:00:00.000Z | 12.3                |
| 2018-12-01T00:00:00.000Z | 13.3                |
| 2019-01-01T00:00:00.000Z | 12.1                |
| 2019-02-01T00:00:00.000Z | 12.6                |
| 2019-03-01T00:00:00.000Z | 10.8                |
| 2019-04-01T00:00:00.000Z | 9.5                 |
| 2019-05-01T00:00:00.000Z | 6.4                 |
| 2019-06-01T00:00:00.000Z | 5.3                 |
| 2019-07-01T00:00:00.000Z | 5.9                 |
| 2019-08-01T00:00:00.000Z | 6.4                 |

4. What is the 3 month rolling average of the max average composition value from September 2018 to August 2019 and include the previous top ranking interests in the same output shown below.
~~~sql
SELECT * 
FROM
 (SELECT month_year, interest_name, max_index_composition, ROUND((max_composition + max_comp_month_before + max_comp_2_months_before)/3) AS three_month_moving_avg, CONCAT(LAG(interest_name, 1) OVER(ORDER BY month_year), ':', LAG(max_index_composition, 1) OVER(ORDER BY month_year)) AS one_month_ago, CONCAT(LAG(interest_name, 2) OVER(ORDER BY month_year), ':', LAG(max_index_composition, 2) OVER(ORDER BY month_year)) AS two_months_ago
 FROM
  (SELECT subq.month_year, max_composition, LAG(max_composition, 1) OVER(ORDER BY subq.month_year) AS max_comp_month_before, LAG(max_composition, 2) OVER(ORDER BY subq.month_year) AS max_comp_2_months_before, index_value AS max_index_composition, interest_id
  FROM
   (SELECT month_year, MAX(composition) AS max_composition
   FROM interest_metrics_cleaned
   WHERE interest_id IS NOT NULL AND month_year IS NOT NULL AND
   month_year BETWEEN '2018-07-01' AND '2019-08-01'
   GROUP BY month_year) subq

  JOIN interest_metrics_cleaned imc
  ON subq.month_year = imc.month_year
  AND subq.max_composition = imc.composition) all_info_subq
    
 JOIN fresh_segments.interest_map ima
 ON CAST(all_info_subq.interest_id AS INTEGER) = ima.id
 ORDER BY month_year) thats_all_folks
WHERE three_month_moving_avg IS NOT NULL;
~~~
| month_year               | interest_name                     | max_index_composition | three_month_moving_avg | one_month_ago                          | two_months_ago                         |
| ------------------------ | --------------------------------- | --------------------- | ---------------------- | -------------------------------------- | -------------------------------------- |
| 2018-09-01T00:00:00.000Z | Work Comes First Travelers        | 2.2                   | 17                     | Gym Equipment Owners:2.1               | Gym Equipment Owners:2.71              |
| 2018-10-01T00:00:00.000Z | Work Comes First Travelers        | 2.22                  | 17                     | Work Comes First Travelers:2.2         | Gym Equipment Owners:2.1               |
| 2018-11-01T00:00:00.000Z | Work Comes First Travelers        | 2.35                  | 19                     | Work Comes First Travelers:2.22        | Work Comes First Travelers:2.2         |
| 2018-12-01T00:00:00.000Z | Work Comes First Travelers        | 2.55                  | 20                     | Work Comes First Travelers:2.35        | Work Comes First Travelers:2.22        |
| 2019-01-01T00:00:00.000Z | Work Comes First Travelers        | 2.48                  | 20                     | Work Comes First Travelers:2.55        | Work Comes First Travelers:2.35        |
| 2019-02-01T00:00:00.000Z | Work Comes First Travelers        | 2.4                   | 20                     | Work Comes First Travelers:2.48        | Work Comes First Travelers:2.55        |
| 2019-03-01T00:00:00.000Z | Luxury Boutique Hotel Researchers | 2.35                  | 17                     | Work Comes First Travelers:2.4         | Work Comes First Travelers:2.48        |
| 2019-04-01T00:00:00.000Z | Luxury Bedding Shoppers           | 1.82                  | 14                     | Luxury Boutique Hotel Researchers:2.35 | Work Comes First Travelers:2.4         |
| 2019-05-01T00:00:00.000Z | Luxury Bedding Shoppers           | 2.32                  | 10                     | Luxury Bedding Shoppers:1.82           | Luxury Boutique Hotel Researchers:2.35 |
| 2019-06-01T00:00:00.000Z | Gym Equipment Owners              | 2.72                  | 8                      | Luxury Bedding Shoppers:2.32           | Luxury Bedding Shoppers:1.82           |
| 2019-07-01T00:00:00.000Z | Gym Equipment Owners              | 2.58                  | 7                      | Gym Equipment Owners:2.72              | Luxury Bedding Shoppers:2.32           |
| 2019-08-01T00:00:00.000Z | Gym Equipment Owners              | 2.61                  | 7                      | Gym Equipment Owners:2.58              | Gym Equipment Owners:2.72              |

5. Provide a possible reason why the max average composition might change from month to month? Could it signal something is not quite right with the overall business model for Fresh Segments?
~~~sql
"In July 2018, the composition metric is 11.89, meaning that 11.89% of the client’s customer list interacted with the interest interest_id = 32486"
The max composition is the max percentage of clients who interacted with a given interest. If it goes down, it means interactions are going down and it might underline a problem with Fresh Segments business model/value proposition.
~~~
