

#### Full join

##### Join Subqueries

Join the tables resulting from the above queries using a `FULL JOIN` on the `birth_date` key. Do not modify the subqueries.
The result should include two columns with birth_dates from both tables, naming them users_birth_date and couriers_birth_date, respectively. It should also include the columns with the number of users and couriers, `users_count` and `couriers_count`.
Sort the table by `users_birth_date` in ascending order, then by `couriers_birth_date` in ascending order.

```SQL
SELECT u.birth_date as users_birth_date,
       users_count,
       c.birth_date as couriers_birth_date,
       couriers_count
FROM   (SELECT birth_date,
               count(user_id) as users_count
        FROM   users
        WHERE  birth_date is not null
        GROUP BY birth_date) as u full join (SELECT birth_date,
                                            count(courier_id) as couriers_count
                                     FROM   couriers
                                     WHERE  birth_date is not null
                                     GROUP BY birth_date) as c using(birth_date)
ORDER BY users_birth_date, couriers_birth_date
```

 **Result**

|users_birth_date|users_count|couriers_birth_date|couriers_count|
|---|---|---|---|
|1981-11-05|1|||
|1982-04-05|1|||
|1982-09-12|1|||
|1983-01-06|1|||
|1983-01-23|1|||
|...|...|...|...|
|1993-03-04|5|1993-03-04|1|
|1993-03-05|8|1993-03-05|1|
|1993-03-06|10|1993-03-06|1|
|1993-03-07|7|||
|1993-03-08|8|||
|...|...|...|...|
|||2005-09-13|1|
|||2005-10-28|1|
|||2006-08-01|1|
|||2007-08-09|1|
|||2007-12-10|1|

---
#### Cross Join

##### Top 100 users

Select the IDs of the first 100 users from the users table and use CROSS JOIN to join them with all the product names from the products table. Output two columns: user ID and product name. Sort the result first by the user ID in ascending order, then by the product name—also in ascending order.
```SQL
SELECT user_id,
       name
FROM   (SELECT user_id
        FROM   users limit 100) as u cross join products
ORDER BY user_id, name
```

---
#### Inner Join

Count the number of unique IDs in the merged table. Output this count as a result. Name the column with the counted value `users_count`.

```SQL
SELECT count(distinct user_id) as users_count
FROM   user_actions as a join users as u using(user_id)
```
 **Result**

|users_count|
|---|
|20331|

---
#### Right Join

Pull the user field data from the `users` table so that all users from the `user_actions` table remain in the result. Then, calculate the average cancel_rate for each gender, rounding it down to three decimal places. Name the column with the calculated average value `avg_cancel_rate`.
Remember that some users will have no field information after joining because not all users from the `user_action` table are in the `users` table. Calculate cancel_rate for this group as well, and output 'unknown' for the empty value in the gender column in the resulting table. 
Sort the result by the user's gender column in ascending order.


```SQL
SELECT coalesce(sex, 'unknown') as sex,
       round(avg(cancel_rate), 3) as avg_cancel_rate
FROM   users
    RIGHT JOIN (SELECT user_id,
                       count(order_id) filter (WHERE action = 'create_order') as orders_count,
                       ((count(order_id) filter (WHERE action = 'cancel_order'))::decimal/(count(order_id) filter (WHERE action = 'create_order'))) as cancel_rate
                FROM   user_actions
                GROUP BY user_id
                ORDER BY user_id) as t2 using(user_id)
GROUP BY sex
ORDER BY sex
```

 **Result**

|sex|avg_cancel_rate|
|---|---|
|female|0.051|
|male|0.048|
|unknown|0.046|


---
#### Self Join

##### Pairs of goods bought together

Find out which pairs of products are bought together most often.
Create product pairs based on the `order` table. Do not include canceled orders. As a result, output two columns — a column with pairs of product names and a column with values showing how many times a particular pair was found in users' orders.
Product pairs should be represented as lists of two items. The product pairs within the lists should be sorted in ascending order of name. Sort the result first by decreasing the frequency of occurrence of the product pair in orders, then by increasing the frequency in the `pair` column.


```SQL
with results as (with prod as (SELECT order_id,
                                      unnest(product_names) as product_name
                               FROM   (with t1 as (SELECT order_id,
                                                          unnest(product_ids) as product_id
                                                   FROM   orders
                                                   WHERE  order_id not in (SELECT order_id
                                                                           FROM   orders
                                                                               LEFT JOIN user_actions using(order_id)
                                                                           WHERE  action = 'cancel_order'
                                                                           GROUP BY order_id) and array_length(product_ids, 1) > 1)
SELECT order_id,
                                      array_agg(name) as product_names
                               FROM   t1
                                   LEFT JOIN products using(product_id)
                               GROUP BY order_id) as t11)
SELECT array[p1.product_name,
       p2.product_name] as pairs,
       p1.order_id
FROM   prod as p1
    RIGHT JOIN prod as p2
        ON p1.order_id = p2.order_id and
           p1.product_name <> p2.product_name and
           p1.product_name < p2.product_name
WHERE  p1.product_name is not null
   and p2.product_name is not null
GROUP BY pairs, p1.order_id
ORDER BY p1.order_id desc)
SELECT DISTINCT array_sort(results.pairs) as pair,
                count(results.order_id) as count_pair
FROM   results
GROUP BY pair
ORDER BY count_pair desc, pair
```
---
#### Left Join

##### Uncancelled orders

Merge the `user_actions` and `orders` tables, leaving only unique uncancelled orders, output user and order IDs, and the list of items in the order. Sort the table by user ID in ascending order, then by order ID in ascending order. Output only the first 1000 rows of the resulting table.

```SQL
SELECT user_id,
       order_id,
       product_ids
FROM   user_actions as u
    LEFT JOIN orders using(order_id)
WHERE  order_id not in (SELECT order_id
                        FROM   user_actions
                        WHERE  action = 'cancel_order')
ORDER BY user_id, order_id limit 1000
```

 **Result**

| user_id | order_id | product_ids         |
| ------- | -------- | ------------------- |
| 1       | 1        | [65, 28]            |
| 1       | 4683     | [1, 15, 40]         |
| 1       | 22901    | [65, 72, 83]        |
| 1       | 23149    | [6, 84, 32]         |
| 2       | 2        | [35, 30, 42, 34]    |
| ...     | ...      | ...                 |
| 248     | 13935    | [75, 28, 86]        |
| 248     | 15518    | [67, 79, 63]        |
| 249     | 287      | [26, 74, 53, 23]    |
| 249     | 758      | [45, 57, 78]        |
| 249     | 7347     | [30, 14, 6, 9]      |

---
##### Largest Orders

Find out who ordered and shipped the orders with the largest number of items. Output the order ID, user ID, and courier ID. Also, add the users' and couriers' ages by the number of full years in separate columns. Sort the result in ascending order id.

```SQL
SELECT u_a.order_id,
       u.user_id,
       date_part('year', age((SELECT max(time)
                       FROM   user_actions), u.birth_date))::varchar as user_age, c.courier_id, date_part('year', age((SELECT max(time)
                                                                                                FROM   user_actions), c.birth_date))::varchar as courier_age
FROM   orders as o
    LEFT JOIN user_actions as u_a using(order_id)
    LEFT JOIN users as u using(user_id)
    LEFT JOIN courier_actions as c_a using(order_id)
    LEFT JOIN couriers as c using(courier_id)
WHERE  array_length(o.product_ids, 1) = (SELECT max(array_length(product_ids, 1))
                                         FROM   orders) and c_a.action = 'deliver_order'
ORDER BY u_a.order_id
```

 **Result**

|order_id|user_id|user_age|courier_id|courier_age|
|---|---|---|---|---|
|7949|3804|29|845|19|
|...|...|...|...|...|
|29786|12728|33|431|26|
|49755|18622|29|1619|32|
|51414|17170|31|2564|27|

---
#### Popular Products

Identify the 10 most popular products delivered in September 2022 using the tables `courier_actions` , `orders` and `products`.
Note: The most frequently ordered products will be considered the most popular. If a product appears multiple times in a single order, only one instance of that product will be counted towards the total frequency.
Output the names of products and the number of times they were found in orders. Name the new column with the number of purchases of the product `times_purchased`.

```SQL
SELECT name,
       count(distinct order_id) as times_purchased
FROM   (SELECT unnest(product_ids) as product_id,
               order_id
        FROM   orders) t1
    LEFT JOIN products using(product_id)
    LEFT JOIN courier_actions using(order_id)
WHERE  action = 'deliver_order'
   and date_part('month', time) = '9.00'
   and date_part('year', time) = '2022.00'
GROUP BY name
ORDER BY times_purchased desc limit 10
```

 **Result**

| name    | times_purchased |
| ------- | --------------- |
| Banana  | 2632            |
| Pasta   | 2623            |
| Bread   | 2622            |
| Sugar   | 2617            |
| Chicken | 2585            |
| Milk    | 2564            |
| Cofee   | 2460            |
| Juice   | 2373            |


