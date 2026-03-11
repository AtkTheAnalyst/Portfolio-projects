# SQL Restaurant Data Analysis 

This file presents the analysis from `SQL Restaurant Data Analysis.sql` 

---

## Skills used

- Joins (**LEFT JOIN**)
- Subqueries (derived tables)
- `WHERE` (including `IN`)
- `ORDER BY`, `GROUP BY`
- Aggregate functions: `COUNT`, `AVG`, `SUM`, `MIN`, `MAX`
- `DISTINCT`
- `HAVING`
- Aliases
- `LIMIT`
- Database selection: `USE`

---

## Setup

```sql
USE restaurant_db;
```

---

## Objective 1: Explore the `menu_items` table

Goal: understand the menu by checking item counts, price extremes, and category breakdowns.

### View the `menu_items` table

```sql
SELECT *
FROM menu_items;
```

### Count the number of menu items

```sql
SELECT COUNT(*)
FROM menu_items;
```

### Least expensive items (ascending)

```sql
SELECT *
FROM menu_items
ORDER BY price ASC;
```

### Most expensive items (descending)

```sql
SELECT *
FROM menu_items
ORDER BY price DESC;
```

### Count Italian dishes

```sql
SELECT COUNT(*)
FROM menu_items
WHERE category = 'Italian';
```

### Least and most expensive Italian dishes

```sql
SELECT category, price
FROM menu_items
WHERE category = 'Italian'
ORDER BY price ASC;
```

```sql
SELECT category, price
FROM menu_items
WHERE category = 'Italian'
ORDER BY price DESC;
```

### Number of dishes per category

```sql
SELECT category, COUNT(item_name) AS num_of_items
FROM menu_items
GROUP BY category;
```

### Average dish price per category

```sql
SELECT category, AVG(price) AS avg_dish_price
FROM menu_items
GROUP BY category;
```

---

## Objective 2: Explore the `order_details` table

Goal: understand ordering activity (date range, order counts, item counts per order).

### View the `order_details` table

```sql
SELECT *
FROM order_details;
```

### View orders by date

```sql
SELECT *
FROM order_details
ORDER BY order_date;
```

### Date range (min/max)

```sql
SELECT MIN(order_date), MAX(order_date)
FROM order_details;
```

### Number of distinct orders in the date range

```sql
SELECT COUNT(DISTINCT(order_id))
FROM order_details;
```

### Items ordered (count per `item_id`)

```sql
SELECT item_id, COUNT(item_id) AS number_of_orders
FROM order_details
GROUP BY item_id;
```

### Items per order (count per `order_id`)

```sql
SELECT order_id, COUNT(item_id) AS num_of_items
FROM order_details
GROUP BY order_id;
```

### How many orders had more than 12 items?

```sql
SELECT COUNT(*)
FROM (
  SELECT COUNT(item_id) AS num_items, order_id
  FROM order_details
  GROUP BY order_id
  HAVING num_items > 12
) AS num_orders;
```

---

## Objective 3: Analyze customer behavior (menu + orders)

Goal: combine items and orders, find most/least ordered items, and inspect the highest spend orders.

### Join `order_details` to `menu_items`

```sql
SELECT *
FROM order_details AS od
LEFT JOIN menu_items AS mi ON od.item_id = mi.menu_item_id;
```

### Most/least ordered items (by purchase count)

```sql
SELECT item_name, COUNT(order_details_id) AS num_purchases
FROM order_details AS od
LEFT JOIN menu_items AS mi ON od.item_id = mi.menu_item_id
GROUP BY item_name
ORDER BY num_purchases DESC;
```

### Categories for ordered items

```sql
SELECT item_name, COUNT(order_details_id) AS num_purchases, category
FROM order_details AS od
LEFT JOIN menu_items AS mi ON od.item_id = mi.menu_item_id
GROUP BY item_name, category
ORDER BY num_purchases DESC;
```

### Top 5 highest spend orders

```sql
SELECT order_id, SUM(price) AS total_spend
FROM order_details AS od
LEFT JOIN menu_items AS mi ON od.item_id = mi.menu_item_id
GROUP BY order_id
ORDER BY total_spend DESC
LIMIT 5;
```

### Details for the highest spend order (example `order_id = 440`)

```sql
SELECT *
FROM order_details AS od
LEFT JOIN menu_items AS mi ON od.item_id = mi.menu_item_id
WHERE order_id = 440;
```

### Category breakdown for the highest spend order

```sql
SELECT category, COUNT(item_id) AS num_items
FROM order_details AS od
LEFT JOIN menu_items AS mi ON od.item_id = mi.menu_item_id
WHERE order_id = 440
GROUP BY category;
```

### Bonus: details for the top 5 highest spend orders

```sql
SELECT *
FROM order_details AS od
LEFT JOIN menu_items AS mi ON od.item_id = mi.menu_item_id
WHERE order_id IN (440, 2075, 1957, 330, 2675);
```

