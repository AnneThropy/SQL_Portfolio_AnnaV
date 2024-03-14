
#### Revenue

For each day in the `orders` table, calculate the following indicators
1. Revenue generated on that day.
2. The cumulative revenue for the current day.
3. The increase in revenue received this day over the previous day's revenue. Calculate the revenue growth as a percentage and round the values to two decimal places.
The result should be sorted in ascending date order.
After creating the query, visualize the results and create graphs showing the calculated indicators' dynamics.
```SQL
SELECT date,
       revenue,
       sum(revenue) OVER(rows between unbounded preceding and current row) as total_revenue,
       round((revenue - lag(revenue) OVER())*100/(lag(revenue) OVER()),
             2) as revenue_change
FROM   (with p1 as (SELECT creation_time::date as date,
                           unnest(product_ids) as product_id
                    FROM   orders
                    WHERE  order_id not in (SELECT order_id
                                            FROM   user_actions
                                            WHERE  action = 'cancel_order'))
SELECT date,
       sum(price) as revenue
FROM   p1
    LEFT JOIN products using(product_id)
GROUP BY date
ORDER BY date) t1
```

 **Result**

|date|revenue|total_revenue|revenue_change|
|---|---|---|---|
|24/08/22|49924.0|49924.0||
|25/08/22|430860.0|480784.0|763.03|
|26/08/22|534766.0|1015550.0|24.12|
|27/08/22|817053.0|1832603.0|52.79|
|28/08/22|1133370.0|2965973.0|38.71|
|29/08/22|1279891.0|4245864.0|12.93|
|30/08/22|1279377.0|5525241.0|-0.04|
|31/08/22|1312720.0|6837961.0|2.61|
|01/09/22|1406101.0|8244062.0|7.11|
|02/09/22|1907107.0|10151169.0|35.63|
|03/09/22|2210988.0|12362157.0|15.93|
|04/09/22|2294009.0|14656166.0|3.75|
|05/09/22|1784690.0|16440856.0|-22.2|
|06/09/22|1330931.0|17771787.0|-25.43|
|07/09/22|1807800.0|19579587.0|35.83|
|08/09/22|2099508.0|21679095.0|16.14|

Daily Revenue:

![](https://storage.yandexcloud.net/klms-public/production/learning-content/152/1881/19951/57851/272088/Daily%20Revenue%20and%20Daily%20Revenue%20Change.png)

Revenue:

![](https://storage.yandexcloud.net/klms-public/production/learning-content/152/1881/19951/57851/272088/Total%20Revenue.png)

---
#### Retention

Calculate the daily retention for all users based on the data in the `user_actions` table, dividing them into cohorts based on the date of their first interaction with our application.
The result should include four columns: the month of the first interaction, the date of the first interaction, the number of days since the date of the first interaction (day number starting from 0), and the Retention value itself.
The retention metric should be expressed as a fraction, rounding the resulting values to two decimal places.
Specify the month of the first interaction as a date rounded to the first day of the month.
The result should be sorted first in ascending order of the date of the first interaction, then in ascending order of the day number.
After creating the query, visualize the results and create graphs showing the calculated retention dynamics.

```SQL
SELECT date_trunc('month', start_date)::date as start_month,
       start_date,
       dt - start_date as day_number,
       round(count(distinct user_id)::decimal / (max(count(distinct user_id)) OVER(PARTITION BY start_date)),
                                                                                                                                   2) as retention FROM(SELECT user_id,
                            min(time::date) OVER (PARTITION BY user_id) as start_date,
                            time::date as dt
                     FROM   user_actions) t1
GROUP BY dt, start_date
ORDER BY start_date, day_number
```

 **Result**

|start_month|start_date|day_number|retention|
|---|---|---|---|
|01/08/22|24/08/22|0|1.0|
|01/08/22|24/08/22|1|0.14|
|01/08/22|24/08/22|2|0.15|
|01/08/22|24/08/22|3|0.18|
|01/08/22|24/08/22|4|0.19|
|...|...|...|...|
|01/08/22|28/08/22|8|0.13|
|01/08/22|28/08/22|9|0.11|
|01/08/22|28/08/22|10|0.12|
|01/08/22|28/08/22|11|0.11|
|01/08/22|29/08/22|0|1.0|
|...|...|...|...|
|01/09/22|06/09/22|1|0.12|
|01/09/22|06/09/22|2|0.12|
|01/09/22|07/09/22|0|1.0|
|01/09/22|07/09/22|1|0.17|
|01/09/22|08/09/22|0|1.0|


![](https://storage.yandexcloud.net/klms-public/production/learning-content/152/1881/19951/57852/277128/Retention%20Practice.png)

---
#### ARPU, ARPPU, AOV

For each day in the `orders` and `user_actions` tables, output the following calculations:
1. The average revenue per user (ARPU) for the current day.
2. Average Revenue Per Paying User (ARPPU) for the current day.
3. Revenue per order or average order value (AOV) for the current day.
When calculating all indicators, round the values to two decimal places.
The results should be sorted in ascending date order. After creating the query, visualize the results and create a graph showing the dynamics of the calculated metrics.
```SQL
with users_active as (SELECT time::date as date,
                             count(distinct user_id) filter(WHERE action = 'create_order' and order_id not in (SELECT order_id
                                                                                                        FROM   user_actions
                                                                                                        WHERE  action = 'cancel_order')) as paying_users
                      FROM   user_actions
                      GROUP BY date), users_joined as (SELECT count(distinct user_id) as total_users,
                                        time::date as date
                                 FROM   user_actions
                                 GROUP BY date), daily_orders as (SELECT count(distinct order_id) as orders,
                                        time::date as date
                                 FROM   user_actions
                                 WHERE  order_id not in (SELECT order_id
                                                         FROM   user_actions
                                                         WHERE  action = 'cancel_order')
                                 GROUP BY date)
SELECT date,
       round(revenue/total_users, 2) as arpu,
       round(revenue/paying_users, 2) as arppu,
       round(revenue/orders, 2) as aov
FROM   (with rev as (SELECT creation_time::date as date,
                            unnest(product_ids) as product_id
                     FROM   orders
                     WHERE  order_id not in (SELECT order_id
                                             FROM   user_actions
                                             WHERE  action = 'cancel_order'))
SELECT date,
       sum(price) as revenue
FROM   rev
    LEFT JOIN products using(product_id)
GROUP BY date
ORDER BY date) t1
    LEFT JOIN users_active using(date)
    LEFT JOIN users_joined using(date)
    LEFT JOIN daily_orders using(date)
```


 **Result**

|date|arpu|arppu|aov|
|---|---|---|---|
|24/08/22|372.57|393.1|361.77|
|25/08/22|508.09|525.44|406.86|
|26/08/22|452.04|470.33|369.57|
|27/08/22|509.38|527.81|381.62|
|28/08/22|528.38|544.1|378.04|
|29/08/22|559.15|581.24|391.76|
|30/08/22|546.74|567.85|379.52|
|31/08/22|517.63|540.21|384.96|
|01/09/22|499.33|518.86|381.26|
|02/09/22|537.67|556.17|381.35|
|03/09/22|565.9|582.76|387.28|
|04/09/22|541.55|558.97|381.7|
|05/09/22|512.84|530.84|381.75|
|06/09/22|475.33|492.75|385.67|
|07/09/22|496.65|514.02|378.44|
|08/09/22|521.1|536.68|383.54|

![](https://storage.yandexcloud.net/klms-public/production/learning-content/152/1881/19951/57851/272089/ARPU%2C%20ARPPU%20and%20AOV%20by%20Day.png)


---
#### Orders Analysis

For each day shown in the `user_actions` table, calculate the following:
1. Total number of orders.
2. Number of first orders (orders placed by users for the first time).
3. The number of new user orders (orders placed by users on the same day they first used the service).
4. Share of first orders in the total number of orders (share of p.2 in p.1).
5. Share of new users' orders in the total number of orders (share of item 3 in item 1).  
In all cases, the number of orders should be expressed as an integer. All fractions should be expressed as percentages rounded to two decimal places.
The results should be sorted in ascending order of date. After creating the query, visualize the results and create a graph showing the dynamics of the calculated metrics.

```SQL
with all_orders as (SELECT time::date as date,
                           count(distinct order_id) filter (WHERE order_id not in (SELECT order_id
                                                                            FROM   user_actions
                                                                            WHERE  action = 'cancel_order')) as orders
                    FROM   user_actions
                    GROUP BY date), new_user_order as (with new as (SELECT min(time)::date as join_date,
                                                       user_id as new_user_id
                                                FROM   user_actions
                                                GROUP BY user_id)
SELECT time::date as date,
       count(distinct order_id) as new_users_orders
FROM   user_actions
    LEFT JOIN new
        ON user_actions.time::date = new.join_date
WHERE  user_id = new_user_id
   and order_id not in (SELECT order_id
                     FROM   user_actions
                     WHERE  action = 'cancel_order')
GROUP BY date)
SELECT date,
       orders,
       count(distinct new_user_id) as first_orders,
       new_users_orders,
       round(count(distinct new_user_id)*100::decimal / orders, 2) as first_orders_share,
       round(new_users_orders*100::decimal / orders, 2) as new_users_orders_share
FROM   (SELECT min(time)::date as date,
               user_id as new_user_id
        FROM   user_actions
        WHERE  order_id not in (SELECT order_id
                                FROM   user_actions
                                WHERE  action = 'cancel_order')
        GROUP BY user_id) t1
    LEFT JOIN all_orders using (date)
    LEFT JOIN new_user_order using (date)
GROUP BY date, orders, new_users_orders
ORDER BY date
```

 **Result**

|date|orders|first_orders|new_users_orders|first_orders_share|new_users_orders_share|
|---|---|---|---|---|---|
|2022-08-24|138|127|138|92.03|100.0|
|2022-08-25|1059|802|1032|75.73|97.45|
|2022-08-26|1447|984|1250|68.0|86.39|
|2022-08-27|2141|1192|1624|55.67|75.85|
|2022-08-28|2998|1460|2102|48.7|70.11|
|...|...|...|...|...|...|
|2022-08-30|3371|1180|1714|35.0|50.85|
|2022-08-31|3410|1380|1908|40.47|55.95|
|2022-09-01|3688|1492|1988|40.46|53.9|
|2022-09-02|5001|1864|2655|37.27|53.09|
|2022-09-03|5709|1907|2830|33.4|49.57|
|...|...|...|...|...|...|
|2022-09-04|6010|1943|2763|32.33|45.97|
|2022-09-05|4675|1387|1865|29.67|39.89|
|2022-09-06|3451|1012|1264|29.32|36.63|
|2022-09-07|4777|1416|1865|29.64|39.04|
|2022-09-08|5474|1661|2300|30.34|42.02|

Dynamics of the total number of orders, the number of first orders and the number of orders of new users:

![](https://storage.yandexcloud.net/klms-public/production/learning-content/152/1881/19868/57329/269880/Orders%2C%20First%20Orders%20and%20Orders%20From%20New%20Users.png)

Dynamics of the share of first orders and the share of new users' orders in the total number of orders:

![](https://storage.yandexcloud.net/klms-public/production/learning-content/152/1881/19868/57329/269880/First%20Orders%20and%20Orders%20From%20New%20Users%20Share%2C%20%25.png)
