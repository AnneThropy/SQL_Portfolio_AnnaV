

#### Number of users

Count the number of unique customers in the `user_actions` table who have placed at least one order in the last week. Name the resulting column with the number of customers `users_count`. Use the last date in the same `user_actions` table as the current date from which to move the week.

```SQL
SELECT count(distinct user_id) as users_count
FROM   user_actions
WHERE  time < (SELECT max(time)
               FROM   user_actions) and time > (SELECT max(time) - interval '1 week'
                                 FROM   user_actions)
```

 **Result**

|users_count|
|---|
|17353|

---
#### Orders not canceled

Use a subquery to select all orders that have not been canceled by users from the `user_actions` table. Print a column with the IDs of these orders. Sort the query result in ascending order by ID.

```SQL
SELECT order_id
FROM   user_actions
WHERE  order_id not in (SELECT order_id
                        FROM   user_actions
                        WHERE  action = 'cancel_order')
ORDER BY order_id limit 1000
```

---
#### Orders by User vs Average

Using the data from the `user_actions` table, calculate the number of orders placed by each user. In a separate `orders_avg` column, output the average number of orders for all users rounded to two decimal places. Also, calculate the deviation of the number of orders from the average. Name the column containing the deviation `orders_diff`. Sort the result by ascending user ID. Add the `LIMIT` operator to the query and output only the first 1000 rows of the resulting table.

```SQL
with t1 as (SELECT user_id,
                   count(order_id) as orders_count
            FROM   user_actions
            WHERE  action = 'create_order'
            GROUP BY user_id)
SELECT user_id,
       orders_count,
       (SELECT round(avg(orders_count), 2)
 FROM   t1) as orders_avg, (orders_count - (SELECT round(avg(orders_count), 2)
                                           FROM   t1)) as orders_diff
FROM   t1
GROUP BY user_id, orders_count
ORDER BY user_id limit 1000
```

 **Result**

| user_id | orders_count | orders_avg | orders_diff |
| ------- | ------------ | ---------- | ----------- |
| 1       | 4            | 2.78       | 1.22        |
| 2       | 2            | 2.78       | -0.78       |
| 3       | 4            | 2.78       | 1.22        |
| 4       | 2            | 2.78       | -0.78       |
| 5       | 1            | 2.78       | -1.78       |
| ...     | ...          | ...        | ...         |
| 996     | 4            | 2.78       | 1.22        |
| 997     | 4            | 2.78       | 1.22        |
| 998     | 2            | 2.78       | -0.78       |
| 999     | 3            | 2.78       | 0.22        |
| 1000    | 5            | 2.78       | 2.22        |

---
#### Discounts with condition

Apply a 15% discount to the goods whose price is $50 or more above the average price of all goods, and apply a 10% discount to the goods whose price is $50 or more below the average price. Goods whose price falls within the range of (average - $50) to (average + $50) should not be changed. When calculating the average price, round it to two decimal places. After applying the discounts, please display all goods with their original and new prices. Name the column with the new price as `new_price`. Finally, sort the result first by decreasing the previous price in the `price` column and then by increasing the product ID.

```SQL
SELECT product_id,
       name,
       price,
       case when price >= ((SELECT round(avg(price), 2)
                     FROM   products) + 50) then price*0.85 when price <= ((SELECT round(avg(price), 2)
                                                       FROM   products) - 50) then price*0.9 else price end as new_price
FROM   products
ORDER BY price desc, product_id
```

 **Result**

| product_id | name            | price | new_price |
| ---------- | --------------- | ----- | --------- |
| 13         | caviar          | 800.0 | 680.0     |
| 37         | lamb            | 559.0 | 475.15    |
| 15         | olive oil       | 450.0 | 382.5     |
| 57         | pork            | 450.0 | 382.5     |
| 45         | green leaf tea  | 78.0  | 70.2      |
| 53         | flour           | 78.0  | 70.2      |
| 38         | oranges         | 76.0  | 68.4      |
| 28         | cream           | 75.0  | 67.5      |
| 5          | coffee          | 15.0  | 13.5      |
| 73         | flatbread       | 15.0  | 13.5      |
| 10         | sunflower seeds | 12.0  | 10.8      |
| 54         | paper bag       | 1.0   | 0.9       |

---
#### Locating errors in data

Determine the number of canceled orders in the `courier_actions` table and find out if users canceled any orders in this table but were still delivered. Count the number of such orders. Name the column with canceled orders `orders_canceled`. Name the column with canceled and delivered orders `orders_canceled_and_delivered`.

```SQL
SELECT count(distinct order_id) filter(WHERE action = 'accept_order' and order_id not in (SELECT order_id
                                                                                          FROM   courier_actions
                                                                                          WHERE  action = 'deliver_order')) as orders_canceled, count(distinct order_id) filter(
WHERE  action = 'deliver_order'
   and order_id in (SELECT order_id
                 FROM   user_actions
                 WHERE  action = 'cancel_order')) as orders_canceled_and_delivered
FROM   courier_actions
```

 **Result**

|orders_canceled|orders_canceled_and_delivered|
|---|---|
|2979|0|

---
#### Condition from another table

Retrieve all information about couriers who delivered 30 or more orders in September 2022 from the `couriers` table. Sort the result in ascending order by courier ID.

```SQL
SELECT courier_id,
       birth_date,
       sex
FROM   couriers
WHERE  courier_id in (SELECT courier_id
                      FROM   courier_actions
                      WHERE  action = 'deliver_order'
                         and date_trunc('month', time) = '2022-09-01'
                      GROUP BY courier_id having count(order_id) >= 30)
ORDER BY courier_id
```

 **Result**

|courier_id|birth_date|sex|
|---|---|---|
|23|1990-03-26|male|
|869|2001-08-25|female|
|1466|1994-04-07|male|
|1664|1987-12-16|male|


---
#### Delivery time

Calculate the time taken to deliver each order with more than 5 items that has not been canceled. The output should include the order ID, the time the courier accepted the order, the time the order was delivered, and the time spent on delivery. The delivery time should be displayed in minutes, rounded to a whole number. Sort the result by ascending order ID.

```SQL
SELECT order_id,
       time_accepted,
       time_delivered,
       (extract(epoch
FROM   (time_delivered - time_accepted))/60)::integer as delivery_time
FROM(SELECT order_id,
            time as time_accepted
     FROM   courier_actions
     WHERE  order_id in (SELECT order_id
                         FROM   orders
                         WHERE  array_length(product_ids, 1) > 5)
        and action = 'accept_order') t1 join (SELECT order_id,
                                             time as time_delivered
                                      FROM   courier_actions
                                      WHERE  order_id in (SELECT order_id
                                                          FROM   orders
                                                          WHERE  array_length(product_ids, 1) > 5)
                                         and action = 'deliver_order') t2 using(order_id)
ORDER BY order_id
```

 **Result**

| order_id | time_accepted  | time_delivered | delivery_time |
| -------- | -------------- | -------------- | ------------- |
| 9        | 24/08/22 14:45 | 24/08/22 15:05 | 20            |
| 34       | 24/08/22 19:04 | 24/08/22 19:25 | 21            |
| 36       | 24/08/22 19:12 | 24/08/22 19:33 | 21            |
| 42       | 24/08/22 19:25 | 24/08/22 19:43 | 18            |
| 44       | 24/08/22 19:35 | 24/08/22 19:55 | 20            |
| ...      | ...            | ...            | ...           |
| 59521    | 08/09/22 23:37 | 08/09/22 23:54 | 17            |
| 59535    | 08/09/22 23:41 | 08/09/22 23:59 | 18            |
| 59541    | 08/09/22 23:43 | 08/09/22 23:59 | 17            |
| 59551    | 08/09/22 23:45 | 08/09/22 23:59 | 15            |
| 59561    | 08/09/22 23:48 | 08/09/22 23:59 | 12            |

---
#### Most expensive orders

Output the orders that contain at least one of the five most expensive products available in our service, along with their ID and contents. Sort the result by order ID in ascending order.

```SQL
with max_prices_table as (SELECT product_id,
                                 price
                          FROM   products
                          ORDER BY price desc limit 5), ids_table as (SELECT unnest(product_ids) as id,
                                                   product_ids,
                                                   order_id
                                            FROM   orders
                                            GROUP BY id, product_ids, order_id)
SELECT DISTINCT order_id,
                product_ids
FROM   ids_table
WHERE  id in (SELECT product_id
              FROM   max_prices_table)
ORDER BY order_id
```

 **Result**

| order_id | product_ids              |
| -------- | ------------------------ |
| 7        | [35, 74, 15, 34, 80]     |
| 9        | [40, 27, 24, 39, 62, 15] |
| 11       | [84, 14, 57]             |
| 14       | [65, 1, 26, 43, 6]       |
| 26       | [19, 37, 72, 9]          |
| ...      | ...                      |
| 59573    | [2, 55, 37]              |
| 59576    | [33, 5, 28, 37]          |
| 59578    | [82, 25, 50, 37]         |
| 59580    | [75, 40, 50, 57, 67]     |
| 59586    | [18, 70, 15, 24]         |

