# -8-Week-SQL-Challenge-Case-Study-1---Danny-s-Diner
Introduction:
Danny seriously loves Japanese food so in the beginning of 2021, he decides to embark upon a risky venture and opens up a cute little restaurant that sells his 3 favourite foods: sushi, curry and ramen.Danny’s Diner is in need of your assistance to help the restaurant stay afloat - the restaurant has captured some very basic data from their few months of operation but have no idea how to use their data to help them run the business.

Problem Statement:
Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. Having this deeper connection with his customers will help him deliver a better and more personalised experience for his loyal customers.He plans on using these insights to help him decide whether he should expand the existing customer loyalty program - additionally he needs help to generate some basic datasets so his team can easily inspect the data without needing to use SQL.Danny has provided you with a sample of his overall customer data due to privacy issues - but he hopes that these examples are enough for you to write fully functioning SQL queries to help him answer his questions!

Danny has shared with you 3 key datasets for this case study:
sales
menu
members


-- 1.What is the total amount each customer spent at the restaurant?--

SELECT s.customer_id,
       Sum(m.price) AS money_spent
FROM   sales s
       INNER JOIN menu m
               ON s.product_id = m.product_id
GROUP  BY s.customer_id; 

-- 2.How many days has each customer visited the restaurant?--

SELECT customer_id,
       Count(DISTINCT( order_date )) no_of_days_visited
FROM   sales
GROUP  BY customer_id; 

-- 3.What was the first item from the menu purchased by each customer?--

SELECT DISTINCT( s.customer_id ),
               m.product_name
FROM   menu m
       INNER JOIN sales s
               ON m.product_id = s.product_id
WHERE  s.order_date = ANY (SELECT Min(order_date)
                           FROM   sales
                           GROUP  BY customer_id); 

-- 4.What is the most purchased item on the menu and how many times was it purchased by all customers?--

SELECT product_name,
       Count(product_name) AS no_of_times
FROM   menu m
       JOIN sales s
         ON m.product_id = s.product_id
GROUP  BY product_name
ORDER  BY no_of_times DESC
LIMIT  1; 


-- 5.Which item was the most popular for each customer?--

WITH cte
     AS (SELECT s.customer_id,
                m.product_name,
                Count(s.product_id)                    AS count,
                Dense_rank()
                  OVER(
                    partition BY s.customer_id
                    ORDER BY Count(s.product_id) DESC) AS r
         FROM   sales s
                INNER JOIN menu m
                        ON s.product_id = m.product_id
         GROUP  BY s.customer_id,
                   s.product_id,
                   m.product_name)
SELECT customer_id,
       product_name,
       count
FROM   cte
WHERE  r = 1;  


-- 6.Which item was purchased first by the customer after they became a member?--

WITH ranks
     AS (SELECT s.customer_id,
                m.product_name,
                Dense_rank()
                  OVER(
                    partition BY s.customer_id
                    ORDER BY s.order_date) AS r
         FROM   sales s
                JOIN menu m
                  ON s.product_id = m.product_id
                JOIN members AS mem
                  ON mem.customer_id = s.customer_id
         WHERE  s.order_date >= mem.join_date)
SELECT *
FROM   ranks
WHERE  r = 1; 

-- 7.Which item was purchased just before the customer became a member?--

WITH ranks
     AS (SELECT s.customer_id,
                m.product_name,
                s.order_date,
                mem.join_date,
                Dense_rank()
                  OVER(
                    partition BY s.customer_id
                    ORDER BY s.order_date) AS r
         FROM   sales s
                JOIN menu m
                  ON s.product_id = m.product_id
                JOIN members AS mem
                  ON mem.customer_id = s.customer_id
         WHERE  s.order_date < mem.join_date)
SELECT *
FROM   ranks
WHERE  r = 1; 

-- 8.What is the total items and amount spent for each member before they became a member?--

SELECT s.customer_id,
       Count(s.product_id) AS no_of_items,
       Sum(m.price)        AS money_spent
FROM   sales s
       JOIN menu m
         ON s.product_id = m.product_id
       JOIN members AS mem
         ON mem.customer_id = s.customer_id
WHERE  s.order_date < mem.join_date
GROUP  BY s.customer_id 

-- 9.If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?--


WITH points
     AS (SELECT *,
                CASE
                  WHEN m.product_name = 'sushi' THEN price * 20
                  WHEN m.product_name != 'sushi' THEN price * 10
                END AS points
         FROM   menu m)
SELECT customer_id,
       Sum(points) AS points
FROM   sales s
       JOIN points p
         ON p.product_id = s.product_id
GROUP  BY s.customer_id 

/*10.In the first week after a customer joins the program (including their join date) 
they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?*/

SELECT customer_id,
       SUM(total_points)
FROM   (WITH points
             AS (SELECT s.customer_id,
                        ( s.order_date - mem.join_date ) AS first_week,
                        m.price,
                        m.product_name,
                        s.order_date
                 FROM   sales s
                        join menu m
                          ON s.product_id = m.product_id
                        join members AS mem
                          ON mem.customer_id = s.customer_id)
        SELECT customer_id,
               order_date,
               CASE
                 WHEN ( first_week BETWEEN 0 AND 7 ) THEN price * 20
                 WHEN ( first_week > 7
                         OR first_week < 0 )
                      AND product_name = 'sushi' THEN price * 20
                 WHEN ( first_week > 7
                         OR first_week < 0 )
                      AND product_name != 'sushi' THEN price * 10
               END AS total_points
         FROM   points
         WHERE  Extract(month FROM order_date) = 1) AS t
GROUP  BY customer_id
