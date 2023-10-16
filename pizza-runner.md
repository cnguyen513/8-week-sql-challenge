# Case Study #2 - Pizza Runner
[Link to SQL Challenge Prompt](https://8weeksqlchallenge.com/case-study-2/)

## Pre-work
- need to clean the data in some of the tables 
    - remove null values: customer_orders, runner_orders
    - edit data types: runner_orders
        - pickup_time --> date and time
        - distance --> number
        - duration --> number
### Cleaning data
```sql
-- Check the datatype for a table.column
USE pizza_runner;
SELECT
    column_name,
    data_type,
    character_maximum_length AS max_length,
    character_octet_length AS octet_length
FROM
    information_schema.columns
WHERE
    table_name = 'runner_orders' AND
    column_name = 'distance';

-- check null values
SELECT *
FROM pizza_runner.customer_orders
WHERE exclusions LIKE '%null%' OR extras LIKE '%null%' OR extras IS NULL

-- create new temp table without null values
DROP TABLE IF EXISTS pizza_runner.dbo.tmp_customer_orders;
SELECT order_id
	,customer_id
	,pizza_id
	,CASE
  		WHEN exclusions IS NULL OR exclusions = 'null' THEN ''
  		ELSE exclusions
  	END AS exclusions
  	,CASE
  		WHEN extras IS NULL OR extras = 'null' THEN ''
  		ELSE extras
  	END AS extras
    ,order_time
INTO pizza_runner.dbo.tmp_customer_orders
FROM pizza_runner.dbo.customer_orders

-- Edit runner_orders
DROP TABLE IF EXISTS pizza_runner.dbo.tmp_runner_orders;
SELECT order_id
	,runner_id
	,CASE
		WHEN pickup_time = 'null' THEN NULL
		ELSE pickup_time
	END AS pickup_time
	,CASE
		WHEN distance = 'null' THEN NULL
		WHEN distance LIKE '%km' THEN TRIM('km' from distance)
		ELSE distance
	END AS distance
	,CASE
		WHEN duration = 'null' THEN NULL
		WHEN duration LIKE '%mins' THEN TRIM('mins' from duration)
		WHEN duration LIKE '%minute' THEN TRIM('minute' from duration)
		WHEN duration LIKE '%minutes' THEN TRIM('minutes' from duration)
		ELSE duration
	END AS duration
	,CASE
		WHEN cancellation IS NULL OR cancellation = 'null' THEN ''
		ELSE cancellation
	END AS cancellation
INTO pizza_runner.dbo.tmp_runner_orders
FROM pizza_runner.dbo.runner_orders

-- Change datatype for pickup_time, distance, and duration
-- Change the data type of the pickup_time column to DATETIME
ALTER TABLE pizza_runner.dbo.tmp_runner_orders
ALTER COLUMN pickup_time DATETIME;

-- Change the data type of the distance column to NUMERIC
ALTER TABLE pizza_runner.dbo.tmp_runner_orders
ALTER COLUMN distance NUMERIC;

-- Change the data type of the duration column to INT
ALTER TABLE pizza_runner.dbo.tmp_runner_orders
ALTER COLUMN duration INT;
```


## A. Pizza Metrics
### 1. How many pizzas were ordered?
- I'm using SQL Server Management Studio instead of the db-fiddle that's provided, so some syntax is a little different
- want to get sum of pizza count
```sql
-- Total count of pizzas ordered
SELECT 
    COUNT(order_id) AS sum
FROM pizza_runner.dbo.tmp_customer_orders
```
| sum |
| --- |
| 14  |

### 2. How many unique customer orders were made?
- find the count of unique order id's
```sql
-- Count of pizzas ordered per customer_id
SELECT 
    COUNT(DISTINCT(order_id))
FROM pizza_runner.dbo.tmp_customer_orders
```
| unique_orders |
|---------------|
| 10            |

### 3. How many successful orders were delivered by each runner?
- want runner_id and order_id
- count order_id where pickup_time/distance/duration is not null
```sql
SELECT 
    runner_id,
	COUNT(order_id) AS order_count
FROM pizza_runner.dbo.tmp_runner_orders
WHERE pickup_time IS NOT NULL
GROUP BY runner_id
```
| runner_id | order_count |
|-----------|-------------|
| 1         | 4           | 
| 2         | 3           |
| 3         | 1           | 

### 4. How many of each type of pizza was delivered?
- tables: tmp_customer_orders for what was in the order, tmp_runner_orders for the delivered orders, pizza_names if we want to correlate pizza_id
- additional notes: needed to change data type of pizza_names.pizza_name to varchar because I couldn't do a GROUP BY without running into an error
```sql
SELECT 
	p_name.pizza_name,
	COUNT(c_ord.order_id) AS Count
FROM pizza_runner.dbo.tmp_runner_orders r_ord
JOIN pizza_runner.dbo.tmp_customer_orders c_ord
	ON c_ord.order_id = r_ord.order_id
JOIN pizza_runner.dbo.pizza_names p_name
	ON c_ord.pizza_id = p_name.pizza_id
WHERE r_ord.distance IS NOT NULL
GROUP BY p_name.pizza_name
```
| pizza_name | Count |
|------------|-------|
| Meatlovers | 9     |
| Vegetarian | 3     |

5. How many Vegetarian and Meatlovers were ordered by each customer?
- tables: tmp_customer_orders, pizza_names
- GROUP BY customer_id, pizza_id
```sql
SELECT 
    c_ord.customer_id,
	p_name.pizza_name,
	COUNT(p_name.pizza_name) AS pizza_count
FROM pizza_runner.dbo.tmp_customer_orders c_ord
JOIN pizza_runner.dbo.pizza_names p_name
	ON c_ord.pizza_id = p_name.pizza_id
GROUP BY c_ord.customer_id, p_name.pizza_name
ORDER BY c_ord.customer_id
```

| customer_id | pizza_name | pizza_count |
|-------------|------------|-------------|
| 101         | Meatlovers | 2           |
| 101         | Vegetarian | 1           |
| 102         | Meatlovers | 2           |
| 102         | Vegetarian | 1           |
| 103         | Meatlovers | 3           |
| 103         | Vegetarian | 1           |
| 104         | Meatlovers | 3           |
| 105         | Vegetarian | 1           |

### 6. What was the maximum number of pizzas delivered in a single order?
- table: tmp_customer_orders
- looking for order_id
```sql
WITH cte_c_order AS (
SELECT 
    order_id,
	COUNT(customer_id) AS order_count
FROM pizza_runner.dbo.tmp_customer_orders c_ord
GROUP BY order_id
)

SELECT 
    cte_c_order.order_id,
	MAX(order_count) AS order_count
FROM cte_c_order
GROUP BY cte_c_order.order_id
ORDER BY order_count DESC
```

| order_id | order_count |
|----------|-------------|
| 4        | 3           |
| 3        | 2           |
| 10       | 2           |
| 1        | 1           |
| 2        | 1           |
| 5        | 1           |
| 6        | 1           |
| 7        | 1           |
| 8        | 1           |
| 9        | 1           |

### 7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?
- table: tmp_customer_orders
- select column exclusions or extras
	- has a value or no value
- maybe need to do a CASE statement, create a new column for has exclusion, or no exclusion
```sql
WITH cte_exclusions AS (
	SELECT order_id,
		customer_id,
		exclusions,
		extras,
		CASE
			WHEN exclusions != '' OR extras != '' THEN '1'
		END AS changed,
		CASE
			WHEN exclusions = '' AND extras = '' THEN '1'
		END AS unchanged
	FROM pizza_runner.dbo.tmp_customer_orders
)

SELECT 
	customer_id,
	COUNT(changed) AS changed,
	COUNT(unchanged) AS unchanged
FROM cte_exclusions
GROUP BY customer_id
```

| customer_id | changed | unchanged |
|-------------|---------|-----------|
| 101         | 0       | 3         |
| 102         | 0       | 3         |
| 103         | 4       | 0         |
| 104         | 2       | 1         |
| 105         | 1       | 0         |

### 8. How many pizzas were delivered that had both exclusions and extras?
- list order_id where there is exclusion and extras
- don't want distance = 'null'
```sql
SELECT
	COUNT(*) AS excl_extra
FROM pizza_runner.dbo.tmp_customer_orders co
JOIN pizza_runner.dbo.tmp_runner_orders ro
	ON co.order_id = ro.order_id
WHERE distance IS NOT NULL
	AND exclusions != ''
	AND extras != ''
```

| excl_extra |
|------------|
| 1          |

### 9. What was the total volume of pizzas ordered for each hour of the day?
- table: tmp_customer_orders
- want order_time -- really only want the hour
- maybe need to split into date and time, then round to the hour
- learned there are a ton of ways to get the date/time you want (DATEADD, DATETRUNC, EXTRACT, DATEDIFF)
```sql
SELECT 
	DATEPART(hour, order_time) AS hour_of_day,
	COUNT(order_id) AS order_count
FROM pizza_runner.dbo.tmp_customer_orders c_ord
GROUP BY DATEPART(hour, order_time) 
```

| order_hour | order_count |
|------------|-------------|
| 11         | 1           |
| 13         | 3           |
| 18         | 3           |
| 19         | 1           |
| 21         | 3           |
| 23         | 3           |

10. What was the volume of orders for each day of the week?
- want to do the same as the previous question, except get day of week
```sql
SELECT 
	DATENAME(WEEKDAY, order_time) AS day_of_week,
	COUNT(order_id) AS order_count
FROM pizza_runner.dbo.tmp_customer_orders c_ord
GROUP BY DATENAME(WEEKDAY, order_time)
```

| day_of_week | order_count |
|-------------|-------------|
| Friday      | 1           |
| Saturday    | 5           |
| Thursday    | 3           |
| Wednesday   | 5           |

## B. Runner and Customer Experience
### 1. How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)
- want the 1 week period starting on the 1st, which is a Friday. So we are doing Friday to Friday = one week
- only want to count the distinct runners
```sql
SELECT
	COUNT(DISTINCT(runner_id)) as runner_count,
	DATEPART(WEEK, pickup_time) AS week
FROM pizza_runner.dbo.tmp_runner_orders r_ord
WHERE distance IS NOT NULL
GROUP BY DATEPART(WEEK, pickup_time)
```
| runner_count | week |
|--------------|------|
| 2            | 1    |
| 3            | 2    |

### 2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?
- want to take order_time and subtract pickup_time
- needed each distinct order, since additional entries of the same order would change the average
```sql
WITH cte_joined_orders AS (
	SELECT DISTINCT(c_ord.order_id),
		runner_id,
		order_time,
		pickup_time
	FROM pizza_runner.dbo.tmp_customer_orders c_ord
	JOIN pizza_runner.dbo.tmp_runner_orders r_ord
		ON c_ord.order_id = r_ord.order_id
	WHERE r_ord.distance IS NOT NULL
)

SELECT
	AVG(DATEDIFF(MINUTE, order_time, pickup_time)) AS avg_time
FROM cte_joined_orders
```
| avg_time |
|----------|
| 16       |

3. Is there any relationship between the number of pizzas and how long the order takes to prepare?
```sql
WITH cte_joined_orders AS (
	SELECT c_ord.order_id,
		DATEDIFF(SECOND, order_time, pickup_time) AS prep_time,
		COUNT(c_ord.order_id) AS order_count
	FROM pizza_runner.dbo.tmp_customer_orders c_ord
	JOIN pizza_runner.dbo.tmp_runner_orders r_ord
		ON c_ord.order_id = r_ord.order_id
	WHERE r_ord.distance IS NOT NULL
	GROUP BY c_ord.order_id, pickup_time, order_time

)

SELECT
	order_count,
	AVG(prep_time / 60) AS avg_pizza_time
FROM cte_joined_orders
GROUP BY order_count
```
| order_count | avg_pizza_time |
|-------------|----------------|
| 1           | 12             |
| 2           | 18             |
| 3           | 29             |

### 4. What was the average distance travelled for each customer?
- table: runner_orders
- avg(distance)
```sql
SELECT
	customer_id,
	ROUND(AVG(distance), 2) AS avg_distance
FROM pizza_runner.dbo.tmp_customer_orders c_ord
JOIN pizza_runner.dbo.tmp_runner_orders r_ord
	ON c_ord.order_id = r_ord.order_id
WHERE r_ord.distance IS NOT NULL
GROUP BY customer_id
ORDER BY avg_distance DESC
```
| customer_id | avg_distance |
|-------------|--------------|
| 105         | 25.000000    |
| 103         | 23.000000    |
| 101         | 20.000000    |
| 102         | 16.330000    |
| 104         | 10.000000    |

5. What was the difference between the longest and shortest delivery times for all orders?
```sql
SELECT
	MAX(duration) - MIN(duration) AS delivery_time_diff
FROM pizza_runner.dbo.tmp_runner_orders r_ord
```
| min_max_dist |
|--------------|
| 30           |

6. What was the average speed for each runner for each delivery and do you notice any trend for these values?
```sql
SELECT
	DISTINCT(order_id),
	runner_id,
	ROUND(distance / (CAST(duration AS decimal) / 60), 2) AS kmph
FROM pizza_runner.dbo.tmp_runner_orders r_ord
WHERE r_ord.distance IS NOT NULL
ORDER BY runner_id, kmph
```
- most of the time is very close to 1 km = 1 min, but runner 2 has variation across all his runs ranging from slower to faster
| order_id | runner_id | kmph              |
|----------|-----------|-------------------|
| 1        | 1         | 37.50000000000000 |
| 3        | 1         | 39.00000000000000 |
| 2        | 1         | 44.44000000000000 |
| 10       | 1         | 60.00000000000000 |
| 4        | 2         | 34.50000000000000 |
| 7        | 2         | 60.00000000000000 |
| 8        | 2         | 92.00000000000000 |
| 5        | 3         | 40.00000000000000 |

7. What is the successful delivery percentage for each runner?
```sql
SELECT
	runner_id,
	COUNT(distance) AS cnt_distance,
	COUNT(runner_id) AS cnt_runner,
	CAST(COUNT(distance) AS float) / CAST(COUNT(runner_id) AS float) * 100AS percent_delivered
FROM pizza_runner.dbo.tmp_runner_orders r_ord
GROUP BY runner_id
```
| runner_id | cnt_distance | cnt_runner | percent_delivered |
|-----------|--------------|------------|-------------------|
| 1         | 4            | 4          | 100               |
| 2         | 3            | 4          | 75                |
| 3         | 1            | 2          | 50                |

## C. Ingredient Optimisation
1. What are the standard ingredients for each pizza?
2. What was the most commonly added extra?
3. What was the most common exclusion?
4. Generate an order item for each record in the customers_orders table in the format of one of the following:
    - Meat Lovers
    - Meat Lovers - Exclude Beef
    - Meat Lovers - Extra Bacon
    - Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers
5. Generate an alphabetically ordered comma separated ingredient list for each pizza order from the customer_orders table and add a 2x in front of any relevant ingredients
    - For example: "Meat Lovers: 2xBacon, Beef, ... , Salami"
6. What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?

## D. Pricing and Ratings
1. If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?
2. What if there was an additional $1 charge for any pizza extras?
    - Add cheese is $1 extra
3. The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.
4. Using your newly generated table - can you join all of the information together to form a table which has the following information for successful deliveries?
    - customer_id
    - order_id
    - runner_id
    - rating
    - order_time
    - pickup_time
    - Time between order and pickup
    - Delivery duration
    - Average speed
    - Total number of pizzas
5. If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled - how much money does Pizza Runner have left over after these deliveries?

## E. Bonus Questions
If Danny wants to expand his range of pizzas - how would this impact the existing data design? Write an INSERT statement to demonstrate what would happen if a new Supreme pizza with all the toppings was added to the Pizza Runner menu?
