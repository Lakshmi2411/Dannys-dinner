# Dannys-dinner SQL Case Study
## Introduction

Danny seriously loves Japanese food so in the beginning of 2021, he decides to embark upon a risky venture and opens up a cute little restaurant that sells his 3 favorite foods: sushi, curry, and ramen.

Danny’s Diner is in need of your assistance to help the restaurant stay afloat - the restaurant has captured some very basic data from their few months of operation but have no idea how to use their data to help them run the business.

## Problem Statement

Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent, and which menu items are their favorite. Having this deeper connection with his customers will help him deliver a better and more personalized experience for his loyal customers.

He plans on using these insights to help him decide whether he should expand the existing customer loyalty program - additionally he needs help to generate some basic datasets so his team can easily inspect the data without needing to use SQL.

Danny has provided you with a sample of his overall customer data due to privacy issues - but he hopes that these examples are enough for you to write fully functioning SQL queries to help him answer his questions!

## Datasets

Danny has shared with you 3 key datasets for this case study:

- `sales`
- `menu`
- `members`

## Queries

CREATE DATABASE dannys_diner;
GO

USE dannys_diner;
GO

CREATE TABLE sales (
  customer_id VARCHAR(1),
  order_date DATE,
  product_id INTEGER
);

INSERT INTO sales (customer_id, order_date, product_id)
VALUES
  ('A', '2021-01-01', 1),
  ('A', '2021-01-01', 2),
  ('A', '2021-01-07', 2),
  ('A', '2021-01-10', 3),
  ('A', '2021-01-11', 3),
  ('A', '2021-01-11', 3),
  ('B', '2021-01-01', 2),
  ('B', '2021-01-02', 2),
  ('B', '2021-01-04', 1),
  ('B', '2021-01-11', 1),
  ('B', '2021-01-16', 3),
  ('B', '2021-02-01', 3),
  ('C', '2021-01-01', 3),
  ('C', '2021-01-01', 3),
  ('C', '2021-01-07', 3);

CREATE TABLE menu (
  product_id INTEGER,
  product_name VARCHAR(5),
  price INTEGER
);

INSERT INTO menu (product_id, product_name, price)
VALUES
  (1, 'Sushi', 10),
  (2, 'Curry', 15),
  (3, 'Ramen', 12);

CREATE TABLE members (
  "customer_id" VARCHAR(1),
  "join_date" DATE
);

INSERT INTO members
  ("customer_id", "join_date")
VALUES
  ('A', '2021-01-07'),
  ('B', '2021-01-09');

  -- 1. What is the total amount each customer spent at the restaurant?

  SELECT
    s.customer_id,
    SUM(m.price) AS total_spent
FROM
    sales s
JOIN
    menu m ON s.product_id = m.product_id
GROUP BY
    s.customer_id;
 
-- 2. How many days has each customer visited the restaurant?

SELECT 
  customer_id,
  COUNT(DISTINCT order_date) AS visit_days
FROM 
  sales
GROUP BY 
  customer_id;
 
-- 3. What was the first item from the menu purchased by each customer?

WITH cte AS (
    SELECT
        s.customer_id,
        s.order_date,
        m.product_name,
        ROW_NUMBER() OVER (PARTITION BY s.customer_id ORDER BY s.order_date) AS rn
    FROM
        sales s
    JOIN
        menu m ON s.product_id = m.product_id
)

SELECT
    customer_id,
    product_name
FROM
    cte
WHERE
    rn = 1;
 

-- 4. What is the most purchased item on the menu and how many times was it purchased by all customers?

SELECT 
  m.product_name,
  COUNT(s.product_id) AS purchase_count
FROM 
  sales s
JOIN 
  menu m ON s.product_id = m.product_id
GROUP BY 
  m.product_name
ORDER BY 
  purchase_count DESC;
 
-- 5. Which item was the most popular for each customer?
WITH cte AS (
    SELECT
        s.customer_id,
        m.product_name,
        COUNT(s.product_id) AS order_count,
        ROW_NUMBER() OVER (PARTITION BY s.customer_id ORDER BY COUNT(s.product_id) DESC) AS rn
    FROM
        sales s
    JOIN
        menu m ON s.product_id = m.product_id
    GROUP BY
        s.customer_id, m.product_name
)

SELECT
    customer_id,
    product_name,
    order_count
FROM
    cte
WHERE
    rn = 1;
 

-- 6. Which item was purchased first by the customer after they became a member?

WITH ranked_sales AS (
  SELECT 
    s.customer_id,
    m.product_name,
    s.order_date,
    ROW_NUMBER() OVER (PARTITION BY s.customer_id ORDER BY s.order_date) AS rn
  FROM 
    sales s
  JOIN 
    menu m ON s.product_id = m.product_id
  JOIN 
    members mem ON s.customer_id = mem.customer_id
  WHERE 
    s.order_date >= mem.join_date
)
SELECT 
  customer_id,
  product_name,
  order_date
FROM 
  ranked_sales
WHERE 
  rn = 1;
 

-- 7. Which item was purchased just before the customer became a member?

WITH ranked_sales AS (
  SELECT 
    s.customer_id,
    m.product_name,
    s.order_date,
    ROW_NUMBER() OVER (PARTITION BY s.customer_id ORDER BY s.order_date) AS rn
  FROM 
    sales s
  JOIN 
    menu m ON s.product_id = m.product_id
  JOIN 
    members mem ON s.customer_id = mem.customer_id
  WHERE 
    s.order_date < mem.join_date
)
SELECT 
  customer_id,
  product_name,
  order_date
FROM 
  ranked_sales
WHERE 
  rn = 1;

 
-- 8. What is the total items and amount spent for each member before they became a member?

SELECT 
  s.customer_id,
  COUNT(s.product_id) AS total_items,
  SUM(m.price) AS total_amount
FROM 
  sales s
JOIN 
  menu m ON s.product_id = m.product_id
JOIN 
  members mem ON s.customer_id = mem.customer_id
WHERE 
  s.order_date < mem.join_date
GROUP BY 
  s.customer_id;

 
-- 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

  SELECT 
  s.customer_id,
  SUM(
    CASE 
      WHEN m.product_name = 'Sushi' THEN m.price * 20
      ELSE m.price * 10
    END
  ) AS total_points
FROM 
  sales s
JOIN 
  menu m ON s.product_id = m.product_id
GROUP BY 
  s.customer_id;

 

-- 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

SELECT 
  s.customer_id,
  SUM(
    CASE 
      WHEN s.order_date BETWEEN mem.join_date AND DATEADD(day, 6, mem.join_date) THEN m.price * 20
      WHEN m.product_name = 'Sushi' THEN m.price * 20
      ELSE m.price * 10
    END
  ) AS total_points
FROM 
  sales s
JOIN 
  menu m ON s.product_id = m.product_id
JOIN 
  members mem ON s.customer_id = mem.customer_id
WHERE 
  s.order_date BETWEEN '2021-01-01' AND '2021-01-31'
GROUP BY 
  s.customer_id;

 
