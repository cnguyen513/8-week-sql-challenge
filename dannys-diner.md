# Case #1 The Taste of Success - Dannys Diner

### Question 1: What is the total amount each customer spent at the restaurant?
- I want to join menu and sales tables
- aggregate the price according to customer
```SQL
SELECT
        sales.customer_id, SUM(menu.price) AS total_spent
FROM dannys_diner.sales AS sales
JOIN dannys_diner.menu AS menu ON menu.product_id = sales.product_id
GROUP BY sales.customer_id
ORDER BY total_spent DESC
```
| customer_id | total_spent |
| ----------- | ----------- |
| A           | 76          |
| B           | 74          |
| C           | 36          |

### Question 2: How many days has each customer visited the restaurant?
- want sales.customer_id, sales.order_date
- find unique values for order_date for each customer_id
``` SQL
SELECT
        sales.customer_id, COUNT(DISTINCT sales.order_date) as num_days
FROM dannys_diner.sales AS sales
GROUP BY customer_id
```
| customer_id | num_days |
| ----------- | -------- |
| A           | 4        |
| B           | 6        |
| C           | 2        |

### Question 3: What was the first item from the menu purchased by each customer?
- want sales tables
- order by order_date
- select the first entry for each customer_id
```SQL
WITH
  first_items (customer_id, order_date)
  AS
  (
    SELECT
        customer_id, min(order_date) AS order_date
    FROM dannys_diner.sales
    GROUP BY customer_id
   )
   
SELECT
        fi.customer_id, sales.product_id, fi.order_date
FROM first_items fi
JOIN dannys_diner.sales sales ON sales.customer_id = fi.customer_id
WHERE sales.order_date = fi.order_date
```

| customer_id | product_id |	order_date |
| ----------- | ---------- | ----------- | 
| A |	1	| 2021-01-01T00:00:00.000Z |
| A	| 2 |	2021-01-01T00:00:00.000Z |
| B	| 2 |	2021-01-01T00:00:00.000Z |
| C	| 3	| 2021-01-01T00:00:00.000Z |
| C |	3	| 2021-01-01T00:00:00.000Z |

### Question 4: What is the most purchased item on the menu and how many times was it purchased by all customers?
- want most purchased items
- total amount purchased
- FROM sales
```SQL
SELECT
	product_id, COUNT(product_id)
FROM dannys_diner.sales
GROUP BY product_id
```

| product_id | count |
| ---------- | ----- | 
| 3 |	8 |
| 2 |	4 |
| 1 |	3 |
- found which one was the most popular
- now need to find out how many times product_id 3 was purchased by each customer
- maybe use CTE to count the times this product was purchased by each customer
```SQL
SELECT
	product_id, COUNT(product_id)
FROM dannys_diner.sales
GROUP BY product_id
LIMIT 1
```

### Question 5: Which item was the most popular for each customer?
- select product_id, customer_id
- count product_id per customer
- choose the one that appears the most
- output:
  - A: 3
  - B: 1, 2, 3
  - C: 3
```SQL
    SELECT
            customer_id, product_id, COUNT(product_id)
    FROM dannys_diner.sales
    GROUP BY 2, 1
    ORDER BY count DESC;
```
| customer_id | product_id | count |
| ----------- | ---------- | ----- |
| A           | 3          | 3     |
| C           | 3          | 3     |
| A           | 2          | 2     |
| B           | 3          | 2     |
| B           | 2          | 2     |
| B           | 1          | 2     |
| A           | 1          | 1     |

### Question 6: Which item was purchased first by the customer after they became a member?
- join sales, members
  - maybe don't need to join, just subquery
- select customer, product, join_date
- see when member joined
- get order_date where equal to or first after join
- what if I create a new table where I select only the dates after the members join?
```SQL
WITH joined_as_member AS (
  SELECT
    members.customer_id, 
    sales.product_id,
    ROW_NUMBER() OVER(
      PARTITION BY members.customer_id
      ORDER BY sales.order_date) AS row_num
  FROM dannys_diner.members
  JOIN dannys_diner.sales
    ON members.customer_id = sales.customer_id
    AND sales.order_date > members.join_date
)

SELECT 
  customer_id, 
  product_name 
FROM joined_as_member
JOIN dannys_diner.menu
  ON joined_as_member.product_id = menu.product_id
WHERE row_num = 1
ORDER BY customer_id ASC;
```

| customer_id | product_name |
| ----------- | ------------ |
| A           | ramen        |
| B           | sushi        |

### Question 7: Which item was purchased just before the customer became a member?
- create CTE
  - rows BEFORE join member date
  - order by date DESC
  - create new colum with row number
    - select row 1 (since it should be the latest date)
```SQL
WITH before_member_sales AS (
  SELECT
    members.customer_id, 
    sales.product_id,
          sales.order_date,
    ROW_NUMBER() OVER(
      PARTITION BY members.customer_id
      ORDER BY sales.order_date DESC) AS row_num
  FROM dannys_diner.members
  JOIN dannys_diner.sales
    ON members.customer_id = sales.customer_id
    AND sales.order_date < members.join_date
)

SELECT 
        customer_id,
    menu.product_name,
    bms.order_date,
    bms.row_num
FROM before_member_sales AS bms
JOIN dannys_diner.menu
        ON bms.product_id = menu.product_id
WHERE bms.row_num = 1
ORDER BY bms.customer_id
```
| customer_id | product_name | order_date               | row_num |
| ----------- | ------------ | ------------------------ | ------- |
| A           | sushi        | 2021-01-01T00:00:00.000Z | 1       |
| B           | sushi        | 2021-01-04T00:00:00.000Z | 1       |


### 8. What is the total items and amount spent for each member before they became a member?
- get date they joined as member
- get order_dates before they joined as member
- count rows for each customer_id
- get price for each product_id, sum  price
```SQL
SELECT 
  sales.customer_id, 
  COUNT(sales.product_id) AS total_items, 
  SUM(menu.price) AS total_sales
FROM dannys_diner.sales
JOIN dannys_diner.members
  ON sales.customer_id = members.customer_id
  AND sales.order_date < members.join_date
JOIN dannys_diner.menu
  ON sales.product_id = menu.product_id
GROUP BY sales.customer_id
ORDER BY sales.customer_id;
```
| customer_id | total_items | total_sales |
| ----------- | ----------- | ----------- |
| A           | 2           | 25          |
| B           | 3           | 40          |

### 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
- using previous query
- create new column
  - if sushi, price * 20
  - if other, price * 10
- tables needed:
  - sales
    - product_id
    - customer_id
  - menu
    - product_id
    - product_name
    - price
MY SOLUTION
```SQL
WITH points_cte AS (
SELECT
        customer_id,
        CASE
            WHEN sales.product_id = 1 THEN price*20
        ELSE price*10
    END AS points
FROM dannys_diner.sales
JOIN dannys_diner.menu
        ON sales.product_id = menu.product_id
ORDER BY customer_id
)

SELECT customer_id, SUM(points) AS total_points
FROM points_cte
GROUP BY customer_id
ORDER BY customer_id
```
OTHER SOLUTION
```SQL
WITH points_cte AS (
  SELECT 
    menu.product_id, 
    CASE
      WHEN product_id = 1 THEN price * 20
      ELSE price * 10
    END AS points
  FROM dannys_diner.menu
)

SELECT 
  sales.customer_id, 
  SUM(points_cte.points) AS total_points
FROM dannys_diner.sales
JOIN points_cte
  ON sales.product_id = points_cte.product_id
GROUP BY sales.customer_id
ORDER BY sales.customer_id;
```
| customer_id | total_points |
| ----------- | ------------ |
| A           | 860          |
| B           | 940          |
| C           | 360          |


### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
- customers earn points on purchased made before becoming a member
- customers still earn 2x points on all sushi items before becoming a member
- customers earn 2x points on ALL items for 7 days after becoming a member
- we want only the sales from January
```SQL
WITH dates_cte AS (
  SELECT
          customer_id,
          join_date,
          join_date + 6 AS valid_date,
          DATE_TRUNC(
      'month', join_date::DATE)
                  + interval '1 month'
                  - interval '1 day' AS last_date
  FROM dannys_diner.members
)

SELECT
	sales.customer_id,
    SUM(CASE
          WHEN menu.product_name = 'sushi' THEN menu.price*2*10
          WHEN sales.order_date BETWEEN dates_cte.join_date and dates_cte.valid_date THEN menu.price*2*10
          ELSE menu.price*10
        END) AS points
FROM dannys_diner.sales
JOIN dannys_diner.menu
	ON sales.product_id = menu.product_id
JOIN dates_cte
	ON dates_cte.customer_id = sales.customer_id
    AND sales.order_date <= dates_cte.last_date
GROUP BY sales.customer_id
ORDER BY customer_id
```
| customer_id | points |
| ----------- | ------ |
| A           | 1370   |
| B           | 820    |

