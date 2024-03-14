

#### Product with the longest name

Find the product with the longest name from the 'products' table. Output the product's name, length in characters, and price.

```SQL
SELECT name,
       length(name) as name_length,
       price
FROM   products
ORDER BY name_length desc limit 1;
```
---
#### First word

Convert the product names in the `name` column of the `products` table so that only the first upper-case word remains, name this column `first_word`. The resulting table shall output the original product names, the new first-word names, and the price of the products sorted by product name alphabetically.

```SQL
SELECT name,
       upper(split_part(name, ' ', 1)) as first_word,
       price
FROM   products
ORDER BY name;
```
---
#### Discount

Apply a 20% discount to all products in the "Products" table and output the products whose discounted price exceeds $100. Print the products’ IDs, their names, the old price, and the new price, including the discount, sorted by product ID in ascending order.

```SQL
SELECT product_id,
       name,
       price as old_price,
       price*0.8 as new_price
FROM   products
WHERE  price*0.8 > 100
ORDER BY product_id
```
---
#### Top-5 users

Using data from the `user_actions` table, identify the top five users who placed the most orders in August 2022. Output two columns: user ID and the number of orders they have created. Sort the result first by decreasing the number of orders created by five users and then by increasing the number of orders created by these users.

```SQL
SELECT user_id,
       count(order_id) as created_orders
FROM   user_actions
WHERE  date_trunc('month', time) = '2022-08-01'
   and action = 'create_order'
GROUP BY user_id
ORDER BY created_orders desc, user_id 
LIMIT 5
```

---
#### Order size on weekends and weekdays

Calculate the average order size by weekends and weekdays, naming the group with weekends using the data from the `orders` table.

Output two columns: the group column called `week_part' and the average order size column called `avg_order_size'.  Round the average order size to two decimal places. Sort the result by the average order size column in ascending order.


```SQL
SELECT case when to_char(creation_time, 'D')::integer in (1, 7) then 'weekend'
            else 'weekdays' end as week_part,
       round(avg(array_length(product_ids, 1)), 2) as avg_order_size
FROM   orders
GROUP BY week_part
ORDER BY avg_order_size
```
##### Result

|week_part|avg_order_size|
|---|---|
|weekend|3.39|
|weekdays|3.4|

---
#### Weekly orders

For each day of the week in the `user_actions` table, calculate:
1. The total number of orders placed.
2. Total number of orders cancelled.
3. Total number of orders not canceled (i.e., delivered).
4. The percentage of orders not canceled out of the total number of orders (success rate).
Name the new columns `created_orders', `canceled_orders', `actual_orders’, and `success_rate’, respectively. Round the column with the percentage of orders not canceled to three decimal places.
Perform all calculations for the period August 24 through September 6, 2022, inclusive, so that the time interval contains an equal number of different weekdays.
The result should be grouped into two columns: the serial number of the weekdays and their abbreviations, and be sorted in ascending order by day number.


```SQL
SELECT date_part('isodow', time)::integer as weekday_number,
       to_char(time, 'Dy') as weekday,
       count(order_id) filter (WHERE action = 'create_order') as created_orders,
       count(order_id) filter (WHERE action = 'cancel_order') as canceled_orders,
       (count(order_id) filter (WHERE action = 'create_order')) - (count(order_id) filter (WHERE action = 'cancel_order')) as actual_orders,
       round(((count(order_id) filter (WHERE action = 'create_order')) - (count(order_id) filter (WHERE action = 'cancel_order')))::decimal / (count(order_id) filter (WHERE action = 'create_order')),
             3) as success_rate
FROM   user_actions
WHERE  time between '2022-08-24'
   and '2022-09-07'
GROUP BY weekday_number, weekday
ORDER BY weekday_number;
```

 **Result**

|weekday_number|weekday|created_orders|canceled_orders|actual_orders|success_rate|
|---|---|---|---|---|---|
|1|Mon|8374|434|7940|0.948|
|2|Tue|7193|370|6823|0.949|
|3|Wed|3758|210|3548|0.944|
|4|Thu|5004|258|4746|0.948|
|5|Fri|6800|352|6448|0.948|
|6|Sat|8249|399|7850|0.952|
|7|Sun|9454|443|9011|0.953|

---
#### Name contains

In the `Products` table, select all products whose names either begin with the word "tea" or consist of five characters. Print two columns: product id and their names. The result should be sorted in ascending order by product id.

```SQL
SELECT product_id,
       name
FROM   products
WHERE  name like 'tea %'
    or name like '_____'
ORDER BY product_id
```
---
#### One-word name

From the `products` table, select the id and names of only those products whose names begin with the letter "s" and contain only one word. The result should be sorted in ascending order of product id.

```SQL
SELECT product_id,
       name
FROM   products
WHERE  name not like '% %'
   and name like 's%'
ORDER BY product_id
```
