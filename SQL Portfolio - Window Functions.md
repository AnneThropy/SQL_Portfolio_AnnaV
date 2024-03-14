

#### Moving Average


First, create a new table with the total number of orders by day based on the `orders` table. When calculating the number of orders, do not take into account canceled orders (they can be determined from the `user_actions` table). Name the column containing the number of orders `orders_count`.
Then, calculate the moving average of the number of orders and the moving average for each record over the last three days. Consider setting the frame boundaries correctly to get the correct calculations.
Round the resulting moving average values to two decimal places. Name the column containing the calculated value `moving_avg`. There is no need to sort the resulting table.

```SQL
SELECT date,
       orders_count,
       round((avg(orders_count) OVER(ORDER BY date rows between 3 preceding and 1 preceding)),
             2) as moving_avg
FROM   (SELECT creation_time::date as date,
               count(order_id) as orders_count
        FROM   orders
        WHERE  order_id not in (SELECT order_id
                                FROM   user_actions
                                WHERE  action = 'cancel_order')
        GROUP BY date
        ORDER BY date) t1
ORDER BY date
```

 **Result**

| date       | orders_count | moving_avg |
| ---------- | ------------ | ---------- |
| 2022-08-24 | 138          |            |
| 2022-08-25 | 1059         | 138.0      |
| 2022-08-26 | 1447         | 598.5      |
| 2022-08-27 | 2141         | 881.33     |
| 2022-08-28 | 2998         | 1549.0     |
| ...        | ...          | ...        |
| 2022-09-04 | 6010         | 4799.33    |
| 2022-09-05 | 4675         | 5573.33    |
| 2022-09-06 | 3451         | 5464.67    |
| 2022-09-07 | 4777         | 4712.0     |
| 2022-09-08 | 5474         | 4301.0     |

---
#### Ranking with condition

Select the top 10% of couriers by the number of orders delivered over time. Output the couriers' IDs, the number of orders delivered, and the courier's order number according to the number of orders delivered.

The courier who delivered the largest number of orders should be ranked as 1, and the courier who delivered the smallest number of orders should have a rank equal to ten percent of the total number of couriers in the `courier_actions` table.

When calculating the number of the last courier, round the value to a whole number.

Name the columns containing the number of orders delivered and the order number `orders_count` and `courier_rank`, respectively. Sort the result in ascending order of the courier's order number.

```SQL
SELECT courier_id,
       orders_count,
       courier_rank
FROM   (with t1 as (SELECT courier_id,
                           count(order_id) as orders_count
                    FROM   courier_actions
                    WHERE  action = 'deliver_order'
                    GROUP BY courier_id
                    ORDER BY orders_count desc, courier_id)
SELECT courier_id,
       orders_count,
       row_number() OVER() as courier_rank
FROM   t1) t2
WHERE  courier_rank <= (SELECT ceil(count(courier_id)*0.1)
                        FROM   couriers)
```

 **Result**

| courier_id | orders_count | courier_rank |
| ---------- | ------------ | ------------ |
| 492        | 54           | 1            |
| 708        | 53           | 2            |
| 23         | 52           | 3            |
| 252        | 52           | 4            |
| 291        | 52           | 5            |
| ...        | ...          | ...          |
| 540        | 38           | 279          |
| 589        | 38           | 280          |
| 622        | 38           | 281          |
| 693        | 38           | 282          |
| 753        | 38           | 283          |

---
#### Cumulative sum

Based on the orders table, create a new table with the total number of orders by day. Do not take into account canceled orders (they can be determined from the user_actions table) when calculating the number of orders. Then, calculate the cumulative sum of the number of orders.
Name the column containing the cumulative total `orders_cum_count`. As a result of this operation, the value of the cumulative amount for the last day should be equal to the total number of orders for the entire period.

```SQL
SELECT date,
       orders_count,
       (sum(orders_count) OVER(ORDER BY date))::integer as orders_cum_count
FROM   (SELECT creation_time::date as date,
               count(order_id) as orders_count
        FROM   orders
        WHERE  order_id not in (SELECT order_id
                                FROM   user_actions
                                WHERE  action = 'cancel_order')
        GROUP BY date
        ORDER BY date) t1
```

 **Result**

| date       | orders_count | orders_cum_count |
| ---------- | ------------ | ---------------- |
| 2022-08-24 | 138          | 138              |
| 2022-08-25 | 1059         | 1197             |
| 2022-08-26 | 1447         | 2644             |
| 2022-08-27 | 2141         | 4785             |
| 2022-08-28 | 2998         | 7783             |
| ...        | ...          | ...              |
| 2022-09-04 | 6010         | 38239            |
| 2022-09-05 | 4675         | 42914            |
| 2022-09-06 | 3451         | 46365            |
| 2022-09-07 | 4777         | 51142            |
| 2022-09-08 | 5474         | 56616            |
