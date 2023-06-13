# SQL analysis of web shop performance

## Objectives
- calculate and evaluate KPIs of e-commerce service
- evaluate overall performance of the e-commerce service by profit and profit ratio
- compare effects of two advertising campaigns on these KPIs

## Summary workflow
1. Daily revenue, year-to-date (YTD) revenue, and the daily revenue fluctuation compared to the previous day
2. Daily Average revenue per user (ARPU), Average revenue per paying user (ARPPU) and Average order volume (AOV)
3. Daily running ARPU, ARPPU and AOV
4. ARPU, ARPPU and AOV by weekdays
5. Total daily revenue, daily revenue from new users and shares of new and old users in revenue per day
6. Share and revenue of major products 
7. Daily and running revenue, cost, VAT, gross profit and profit ratio 
8. Comparison of advertising campaigns using Customer Acquisition Cost (CAC)
8. Comparison of advertising campaigns using Customer
9. Comparison of advertising campaigns using Return of Investment (ROI)
10. Comparison of advertising campaigns participants using Average Order Volume (AOV) 
11. Daily Retention Rate
12. Comparison of advertising campaign by Retention Rate
13. Dates when users attracted by advertising campaigns would cover marketing expenses (CAC) by their revenue 

## Techniques used
Cohort analysis, DAU, MAU, ROI, CAC, AOV, ARPU, ARPPU, Retention rate

## Tools
SQL (window functions, sub-queries, etc.)

## Database settings 
* Name: Database of web shop 
* Database contains 6 tables with details about products, orders, couriers and users: 
    * TABLE courier actions
        * action CHARACTER VARYING 
        * courier_id INTEGER 
        * order_id INTEGER
        * time TIMESTAMP WITHOUT TIME ZONE
    * TABLE couriers
        * birth_date DATE
        * courier_id INTEGER
        * sex CHARACTER VARYING
    * TABLE orders
        * creation_time TIMESTAMP WITHOUT TIME ZONE 
        * order_id INTEGER 
        * product_ids ARRAY
    * TABLE products
        * name CHARACTER VARYING 
        * price NUMERIC 
        * product_id INTEGER
    * TABLE user_actions
        * action CHARACTER VARYING 
        * order_id INTEGER
        * time TIMESTAMP WITHOUT TIME ZONE 
        * user_id INTEGER
    * TABLE users
        * birth_date DATE
        * sex CHARACTER VARYING 
        * user_id INTEGER

## 1. Daily revenue, year-to-date (YTD) revenue, and the daily revenue fluctuation compared to the previous day
```
SELECT    date
        , revenue
        , SUM(revenue) OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS total_revenue
        , ROUND(100.0*(revenue-LAG(revenue, 1) OVER (ORDER BY  date))/(LAG(revenue, 1) OVER (ORDER BY  date)), 2) AS revenue_change
FROM      (
          SELECT    date
                  , SUM(products.price) AS revenue
          FROM      (
                    SELECT    creation_time::date AS date
                            , UNNEST(product_ids) AS product_id
                    FROM      orders
                    WHERE     order_id NOT IN (
                              SELECT    order_id
                              FROM      user_actions
                              WHERE     ACTION='cancel_order'
                              )
                    ) AS ords
          LEFT JOIN products USING (product_id)
          GROUP BY  date
          ORDER BY  date
          ) AS ords2
ORDER BY  date
```

## 2. Daily Average revenue per user (ARPU), Average revenue per paying user (ARPPU) and Average order volume (AOV)
```
SELECT date,
    round(revenue / users_count, 2) AS ARPU,
    round(revenue / paying_users_count, 2) AS ARPPU,
    round(revenue / orders_count, 2) AS AOV
FROM      (
          SELECT    date
                  , SUM(products.price) AS revenue
          FROM      (
                    SELECT    creation_time::date AS date
                            , UNNEST(product_ids) AS product_id
                    FROM      orders
                    WHERE     order_id NOT IN (
                              SELECT    order_id
                              FROM      user_actions
                              WHERE     ACTION='cancel_order'
                              )
                    ) AS ords
          LEFT JOIN products USING (product_id)
          GROUP BY  date
          ORDER BY  date
          ) AS rev
LEFT JOIN (
          SELECT    TIME::date AS date
                  , COUNT(DISTINCT user_id) AS users_count
                  , COUNT(DISTINCT user_id) FILTER (
                    WHERE     order_id NOT IN (
                              SELECT    order_id
                              FROM      user_actions
                              WHERE     ACTION='cancel_order'
                              )
                    ) AS paying_users_count
                  , COUNT(DISTINCT order_id) FILTER (
                    WHERE     order_id NOT IN (
                              SELECT    order_id
                              FROM      user_actions
                              WHERE     ACTION='cancel_order'
                              )
                    ) AS orders_count
          FROM      user_actions
          GROUP BY  date
          ) AS usrs USING (date)
ORDER BY  date
```

## 3. Daily running ARPU, ARPPU and AOV
```
SELECT date,
       round(sum(revenue) over(order by date)::decimal / sum(users) over(order by date), 2) as running_arpu,
       round(sum(revenue) over(order by date)::decimal / sum(paying_users) over(order by date), 2) as running_arppu,
       round(sum(revenue) over(order by date)::decimal / sum(orders) over(order by date), 2) as running_aov
FROM      (
          SELECT    creation_time::date AS date
                  , COUNT(DISTINCT order_id) AS orders
                  , SUM(price) AS revenue
          FROM      (
                    SELECT    order_id
                            , creation_time
                            , UNNEST(product_ids) AS product_id
                    FROM      orders
                    WHERE     order_id NOT IN (
                              SELECT    order_id
                              FROM      user_actions
                              WHERE     ACTION='cancel_order'
                              )
                    ) t1
          LEFT JOIN products USING (product_id)
          GROUP BY  date
          ) t2
LEFT JOIN (
          SELECT    date
                  , COUNT(user_id) AS users
          FROM      (
                    SELECT    user_id
                            , MIN(TIME::date) AS date
                    FROM      user_actions
                    GROUP BY  user_id
                    ) AS t31
          GROUP BY  date
          ) t3 USING (date)
LEFT JOIN (
          SELECT    date
                  , COUNT(user_id) AS paying_users
          FROM      (
                    SELECT    user_id
                            , MIN(TIME::date) AS date
                    FROM      user_actions
                    WHERE     order_id NOT IN (
                              SELECT    order_id
                              FROM      user_actions
                              WHERE     ACTION='cancel_order'
                              )
                    GROUP BY  user_id
                    ) AS t41
          GROUP BY  date
          ) t4 USING (date)
ORDER BY  date
```

## 4. ARPU, ARPPU and AOV by weekdays
```
WITH      rev AS (
          SELECT    DATE_PART('isodow', date) AS weekday_number
                  , TO_CHAR(date, 'Day') AS weekday
                  , COUNT(DISTINCT order_id) AS orders
                  , SUM(price) AS revenue
          FROM      (
        
                    SELECT    order_id
                            , creation_time::date AS date
                            , UNNEST(product_ids) AS product_id
                    FROM      orders
                    WHERE     order_id NOT IN (
                              SELECT    order_id
                              FROM      user_actions
                              WHERE     ACTION='cancel_order'
                              )
                    ) prod_ords
          LEFT JOIN products USING (product_id)
          WHERE     date BETWEEN '2022-08-26' AND '2022-09-08'
          GROUP BY  weekday_number
                  , weekday
          )
        , pusr AS (
          SELECT    DATE_PART('isodow', TIME::date) AS weekday_number
                  , COUNT(DISTINCT user_id) AS paying_users
          FROM      user_actions
          WHERE     order_id NOT IN (
                    SELECT    order_id
                    FROM      user_actions
                    WHERE     ACTION='cancel_order'
                    )
          AND       TIME::date BETWEEN '2022-08-26' AND '2022-09-08'
          GROUP BY  weekday_number
          )
        , usr AS (
          SELECT    DATE_PART('isodow', TIME::date) AS weekday_number
                  , COUNT(DISTINCT user_id) AS users
          FROM      user_actions
          WHERE     TIME::date BETWEEN '2022-08-26' AND '2022-09-08'
          GROUP BY  weekday_number
          )
SELECT    weekday
        , weekday_number::INT
        , ROUND(SUM(revenue)::DECIMAL/SUM(users), 2) AS ARPU
        , ROUND(SUM(revenue)::DECIMAL/SUM(paying_users), 2) AS ARPPU
        , ROUND(SUM(revenue)::DECIMAL/SUM(orders), 2) AS AOV
FROM      rev
LEFT JOIN usr USING (weekday_number)
LEFT JOIN pusr USING (weekday_number)
GROUP BY  weekday_number
        , weekday
```

## 5. Total daily revenue, daily revenue from new users and shares of new and old users in revenue per day
```
WITH      cancelled_orders AS (
          SELECT    order_id
          FROM      user_actions
          WHERE     ACTION='cancel_order'
          )
        , orderv AS (
                ## 5. Total daily revenue, daily revenue from new users and shares of new and old users in revenue per
          SELECT    order_id
                  , user_id
                  , date
                  , SUM(price) AS VALUE
          FROM      (
                    SELECT    order_id
                            , creation_time::date AS date
                            , UNNEST(product_ids) AS product_id
                    FROM      orders
                    ) order_products
          LEFT JOIN products USING (product_id)
          LEFT JOIN user_actions USING (order_id)
          WHERE     order_id NOT IN (
                    SELECT    order_id
                    FROM      cancelled_orders
                    )
          GROUP BY  order_id
                  , date
                  , user_id
          )
        , first_actions AS (
          SELECT    user_id
                  , MIN(TIME::date) AS date
          FROM      user_actions
          GROUP BY  user_id
          )
        , user_daily_orders AS (
          SELECT    date
                  , user_id
                  , SUM(VALUE) AS VALUE
          FROM      orderv
          GROUP BY  date
                  , user_id
          )
        , first_orders AS (
          SELECT    date
                  , SUM(VALUE) AS new_users_revenue
          FROM      user_daily_orders
          INNER     JOIN first_actions USING (date, user_id)
          GROUP BY  date
          )
        , daily_orderv AS (
          SELECT    date
                  , SUM(VALUE) AS revenue
          FROM      orderv
          GROUP BY  date
          )
SELECT    date
        , revenue
        , new_users_revenue
        , ROUND(100.0*new_users_revenue/revenue, 2) AS new_users_revenue_share
        , ROUND(100.0*(revenue-new_users_revenue)/revenue, 2) AS old_users_revenue_share
FROM      daily_orderv
LEFT JOIN first_orders USING (date)
ORDER BY  date
```

## 6. Share and revenue of major products 
```
WITH      cancelled_orders AS (
          SELECT    order_id
          FROM      user_actions
          WHERE     ACTION='cancel_order'
          )
        , products_amt AS (
        
          SELECT    product_id
                  , NAME
                  , SUM(price) AS revenue
          FROM      (
                    SELECT    UNNEST(product_ids) AS product_id
                    FROM      orders
                    WHERE     order_id NOT IN (
                              SELECT    order_id
                              FROM      cancelled_orders
                              )
                    ) order_products
          LEFT JOIN products USING (product_id)
          GROUP BY  product_id
                  , NAME
          )
        , products_w_others AS (
          SELECT    CASE
                              WHEN ROUND(100.0*revenue/SUM(revenue) OVER (), 2)<0.5 THEN 'ДРУГОЕ'
                              ELSE NAME
                    END AS product_name
                  , revenue
                  , ROUND(100.0*revenue/SUM(revenue) OVER (), 2) AS share_in_revenue
          FROM      products_amt
          )
SELECT    product_name
        , SUM(revenue) AS revenue
        , SUM(share_in_revenue) AS share_in_revenue
FROM      products_w_others
GROUP BY  product_name
ORDER BY  revenue DESC
```

## 7. Daily and running revenue, cost, VAT, gross profit and profit ratio 
```
WITH      cancelled_orders AS (
          SELECT    order_id
          FROM      user_actions
          WHERE     ACTION='cancel_order'
          )
        , completed_orders AS (
        
          SELECT    date
                  , order_id
                  , SUM(price) AS revenue
          FROM      (
                    SELECT    creation_time::date AS date
                            , order_id
                            , UNNEST(product_ids) AS product_id
                    FROM      orders
                    WHERE     order_id NOT IN (
                              SELECT    order_id
                              FROM      cancelled_orders
                              )
                    ) order_products
          LEFT JOIN products USING (product_id)
          GROUP BY  date
                  , order_id
          )
        , pickups AS (
          SELECT    date
                  , COUNT(order_id) AS orders_pickuped
          FROM      completed_orders
          GROUP BY  date
          )
        , daily_revenue AS (
          SELECT    date
                  , SUM(revenue) AS revenue
          FROM      completed_orders
          GROUP BY  date
          )
        , deliveries AS (
          SELECT    TIME::date AS date
                  , courier_id
                  , COUNT(order_id) AS orders_delivered
          FROM      courier_actions
          WHERE     order_id NOT IN (
                    SELECT    order_id
                    FROM      cancelled_orders
                    )
          AND       ACTION='deliver_order'
          GROUP BY  date
                  , courier_id
          ORDER BY  date
          )
        , courier_bonus AS (
          SELECT    date
                  , COUNT(courier_id) AS couriers_w_bonus
          FROM      deliveries
          WHERE     orders_delivered>=5
          GROUP BY  date
          )
        , orders_deliveries AS (
          SELECT    date
                  , SUM(orders_delivered) AS orders_delivered
          FROM      deliveries
          GROUP BY  date
          )
        , costs AS (
          SELECT    date
                  , CASE
                              WHEN EXTRACT(
                              MONTH
                              FROM      date
                              )=8 THEN 120000
                              ELSE 150000
                    END AS fixed_costs
                  , CASE
                              WHEN EXTRACT(
                              MONTH
                              FROM      date
                              )=8 THEN pickups.orders_pickuped*140
                              ELSE pickups.orders_pickuped*115
                    END AS order_pickup
                  , orders_deliveries.orders_delivered*150 AS order_delivery
                  , CASE
                              WHEN EXTRACT(
                              MONTH
                              FROM      date
                              )=8 THEN courier_bonus.couriers_w_bonus*400
                              ELSE courier_bonus.couriers_w_bonus*500
                    END AS courier_bonus
          FROM      daily_revenue
          LEFT JOIN courier_bonus USING (date)
          LEFT JOIN orders_deliveries USING (date)
          LEFT JOIN pickups USING (date)
          )
        , daily_cost AS (
          SELECT    date
                  , COALESCE(fixed_costs, 0)+COALESCE(order_pickup, 0)+COALESCE(order_delivery, 0)+COALESCE(courier_bonus, 0) AS costs
          FROM      costs
          )
        , product_taxes AS (
          SELECT    date
                  , NAME
                  , price
                  , CASE
                              WHEN NAME IN (
                              'сахар'
                            , 'сухарики'
                            , 'сушки'
                            , 'семечки'
                            , 'масло льняное'
                            , 'виноград'
                            , 'масло оливковое'
                            , 'арбуз'
                            , 'батон'
                            , 'йогурт'
                            , 'сливки'
                            , 'гречка'
                            , 'овсянка'
                            , 'макароны'
                            , 'баранина'
                            , 'апельсины'
                            , 'бублики'
                            , 'хлеб'
                            , 'горох'
                            , 'сметана'
                            , 'рыба копченая'
                            , 'мука'
                            , 'шпроты'
                            , 'сосиски'
                            , 'свинина'
                            , 'рис'
                            , 'масло кунжутное'
                            , 'сгущенка'
                            , 'ананас'
                            , 'говядина'
                            , 'соль'
                            , 'рыба вяленая'
                            , 'масло подсолнечное'
                            , 'яблоки'
                            , 'груши'
                            , 'лепешка'
                            , 'молоко'
                            , 'курица'
                            , 'лаваш'
                            , 'вафли'
                            , 'мандарины'
                              ) THEN ROUND(price*0.1/1.1, 2)
                              ELSE ROUND(price*0.2/1.2, 2)
                    END AS tax
          FROM      (
                    SELECT    creation_time::date AS date
                            , UNNEST(product_ids) AS product_id
                    FROM      orders
                    WHERE     order_id NOT IN (
                              SELECT    order_id
                              FROM      cancelled_orders
                              )
                    ) date_products
          LEFT JOIN products USING (product_id)
          )
        , daily_tax AS (
          SELECT    date
                  , SUM(tax) AS tax
          FROM      product_taxes
          GROUP BY  date
          ORDER BY  date
          )
    
select date
    , revenue
    , costs
    , tax
    , revenue - costs - tax as gross_profit
    , sum(revenue) over (order by date) as total_revenue
    , sum(costs) over (order by date) as total_costs
    , sum(tax) over (order by date) as total_tax
    , sum(revenue) over(order by date) - sum(costs) over(order by date) - sum(tax) over(order by date) as total_gross_profit
    , round(100.0 * (revenue - costs - tax) / revenue, 2) as gross_profit_ratio
    , round(100.0 * (sum(revenue) over(order by date) - sum(costs) over(order by date) - sum(tax) over(order by date)) / sum(revenue) over(order by date), 2) as total_gross_profit_ratio
from daily_revenue
left join daily_cost using(date)
left join daily_tax using(date)
```

## 8. Comparison of advertising campaigns using Customer Acquisition Cost (CAC)
```
WITH participants as (SELECT DISTINCT user_id ,
                                      CASE WHEN user_id IN (8631, 8632, 8638, 8643, 8657, 8673, 8706, 8707, 8715, 8723, 8732,
                                                            8739, 8741, 8750, 8751, 8752, 8770, 8774, 8788,
                                                            8791, 8804, 8810, 8815, 8828, 8830, 8845, 8853,
                                                            8859, 8867, 8869, 8876, 8879, 8883, 8896, 8909,
                                                            ## 8. Comparison of advertising campaigns using Customer
                                                            8911, 8933, 8940, 8972, 8976, 8988, 8990, 9002,
                                                            9004, 9009, 9019, 9020, 9035, 9036, 9061, 9069,
                                                            9071, 9075, 9081, 9085, 9089, 9108, 9113, 9144,
                                                            9145, 9146, 9162, 9165, 9167, 9175, 9180, 9182,
                                                            9197, 9198, 9210, 9223, 9251, 9257, 9278, 9287,
                                                            9291, 9313, 9317, 9321, 9334, 9351, 9391, 9398,
                                                            9414, 9420, 9422, 9431, 9450, 9451, 9454, 9472,
                                                            9476, 9478, 9491, 9494, 9505, 9512, 9518, 9524,
                                                            9526, 9528, 9531, 9535, 9550, 9559, 9561, 9562,
                                                            9599, 9603, 9605, 9611, 9612, 9615, 9625, 9633,
                                                            9652, 9654, 9655, 9660, 9662, 9667, 9677, 9679,
                                                            9689, 9695, 9720, 9726, 9739, 9740, 9762, 9778,
                                                            9786, 9794, 9804, 9810, 9813, 9818, 9828, 9831,
                                                            9836, 9838, 9845, 9871, 9887, 9891, 9896, 9897,
                                                            9916, 9945, 9960, 9963, 9965, 9968, 9971, 9993,
                                                            9998, 9999, 10001, 10013, 10016, 10023, 10030,
                                                            10051, 10057, 10064, 10082, 10103, 10105, 10122,
                                                            10134, 10135) THEN 'Кампания № 1'
                                           WHEN user_id IN (8629, 8630, 8644, 8646, 8650, 8655, 8659, 8660, 8663, 8665, 8670,
                                                            8675, 8680, 8681, 8682, 8683, 8694, 8697, 8700,
                                                            8704, 8712, 8713, 8719, 8729, 8733, 8742, 8748,
                                                            8754, 8771, 8794, 8795, 8798, 8803, 8805, 8806,
                                                            8812, 8814, 8825, 8827, 8838, 8849, 8851, 8854,
                                                            8855, 8870, 8878, 8882, 8886, 8890, 8893, 8900,
                                                            8902, 8913, 8916, 8923, 8929, 8935, 8942, 8943,
                                                            8949, 8953, 8955, 8966, 8968, 8971, 8973, 8980,
                                                            8995, 8999, 9000, 9007, 9013, 9041, 9042, 9047,
                                                            9064, 9068, 9077, 9082, 9083, 9095, 9103, 9109,
                                                            9117, 9123, 9127, 9131, 9137, 9140, 9149, 9161,
                                                            9179, 9181, 9183, 9185, 9190, 9196, 9203, 9207,
                                                            9226, 9227, 9229, 9230, 9231, 9250, 9255, 9259,
                                                            9267, 9273, 9281, 9282, 9289, 9292, 9303, 9310,
                                                            9312, 9315, 9327, 9333, 9335, 9337, 9343, 9356,
                                                            9368, 9370, 9383, 9392, 9404, 9410, 9421, 9428,
                                                            9432, 9437, 9468, 9479, 9483, 9485, 9492, 9495,
                                                            9497, 9498, 9500, 9510, 9527, 9529, 9530, 9538,
                                                            9539, 9545, 9557, 9558, 9560, 9564, 9567, 9570,
                                                            9591, 9596, 9598, 9616, 9631, 9634, 9635, 9636,
                                                            9658, 9666, 9672, 9684, 9692, 9700, 9704, 9706,
                                                            9711, 9719, 9727, 9735, 9741, 9744, 9749, 9752,
                                                            9753, 9755, 9757, 9764, 9783, 9784, 9788, 9790,
                                                            9808, 9820, 9839, 9841, 9843, 9853, 9855, 9859,
                                                            9863, 9877, 9879, 9880, 9882, 9883, 9885, 9901,
                                                            9904, 9908, 9910, 9912, 9920, 9929, 9930, 9935,
                                                            9939, 9958, 9959, 9961, 9983, 10027, 10033, 10038,
                                                            10045, 10047, 10048, 10058, 10059, 10067, 10069,
                                                            10073, 10075, 10078, 10079, 10081, 10092, 10106,
                                                            10110, 10113, 10131) THEN 'Кампания № 2'
                                           ELSE null END as ads_campaign
                      FROM   user_actions
                      ORDER BY ads_campaign), 
                      
unpaid_orders as(SELECT order_id
                        FROM   user_actions
                        WHERE  action like '%cancel%')
SELECT ads_campaign,
       round(250000.0 / count(distinct user_id), 2) as cac
FROM   user_actions
    LEFT JOIN participants using(user_id)
WHERE  ads_campaign is not null
   and order_id not in (SELECT order_id
                     FROM   unpaid_orders)
GROUP BY ads_campaign
```

## 9. Comparison of advertising campaigns using Return of Investment (ROI)
```
WITH      participants AS (
          SELECT    DISTINCT user_id
                  , CASE
                              WHEN user_id IN (
                                8631, 8632, 8638, 8643, 8657, 8673, 8706, 8707, 8715, 8723, 8732,
                                8739, 8741, 8750, 8751, 8752, 8770, 8774, 8788,
                        
                                8791, 8804, 8810, 8815, 8828, 8830, 8845, 8853,
                                8859, 8867, 8869, 8876, 8879, 8883, 8896, 8909,
                                8911, 8933, 8940, 8972, 8976, 8988, 8990, 9002,
                                9004, 9009, 9019, 9020, 9035, 9036, 9061, 9069,
                                9071, 9075, 9081, 9085, 9089, 9108, 9113, 9144,
                                9145, 9146, 9162, 9165, 9167, 9175, 9180, 9182,
                                9197, 9198, 9210, 9223, 9251, 9257, 9278, 9287,
                                9291, 9313, 9317, 9321, 9334, 9351, 9391, 9398,
                                9414, 9420, 9422, 9431, 9450, 9451, 9454, 9472,
                                9476, 9478, 9491, 9494, 9505, 9512, 9518, 9524,
                                9526, 9528, 9531, 9535, 9550, 9559, 9561, 9562,
                                9599, 9603, 9605, 9611, 9612, 9615, 9625, 9633,
                                9652, 9654, 9655, 9660, 9662, 9667, 9677, 9679,
                                9689, 9695, 9720, 9726, 9739, 9740, 9762, 9778,
                                9786, 9794, 9804, 9810, 9813, 9818, 9828, 9831,
                                9836, 9838, 9845, 9871, 9887, 9891, 9896, 9897,
                                9916, 9945, 9960, 9963, 9965, 9968, 9971, 9993,
                                9998, 9999, 10001, 10013, 10016, 10023, 10030,
                                10051, 10057, 10064, 10082, 10103, 10105, 10122,
                                10134, 10135) THEN 'Кампания № 1'
                              WHEN user_id IN (
                                8629, 8630, 8644, 8646, 8650, 8655, 8659, 8660, 8663, 8665, 8670,
                                8675, 8680, 8681, 8682, 8683, 8694, 8697, 8700,
                                8704, 8712, 8713, 8719, 8729, 8733, 8742, 8748,
                                8754, 8771, 8794, 8795, 8798, 8803, 8805, 8806,
                                8812, 8814, 8825, 8827, 8838, 8849, 8851, 8854,
                                8855, 8870, 8878, 8882, 8886, 8890, 8893, 8900,
                                8902, 8913, 8916, 8923, 8929, 8935, 8942, 8943,
                                8949, 8953, 8955, 8966, 8968, 8971, 8973, 8980,
                                8995, 8999, 9000, 9007, 9013, 9041, 9042, 9047,
                                9064, 9068, 9077, 9082, 9083, 9095, 9103, 9109,
                                9117, 9123, 9127, 9131, 9137, 9140, 9149, 9161,
                                9179, 9181, 9183, 9185, 9190, 9196, 9203, 9207,
                                9226, 9227, 9229, 9230, 9231, 9250, 9255, 9259,
                                9267, 9273, 9281, 9282, 9289, 9292, 9303, 9310,
                                9312, 9315, 9327, 9333, 9335, 9337, 9343, 9356,
                                9368, 9370, 9383, 9392, 9404, 9410, 9421, 9428,
                                9432, 9437, 9468, 9479, 9483, 9485, 9492, 9495,
                                9497, 9498, 9500, 9510, 9527, 9529, 9530, 9538,
                                9539, 9545, 9557, 9558, 9560, 9564, 9567, 9570,
                                9591, 9596, 9598, 9616, 9631, 9634, 9635, 9636,
                                9658, 9666, 9672, 9684, 9692, 9700, 9704, 9706,
                                9711, 9719, 9727, 9735, 9741, 9744, 9749, 9752,
                                9753, 9755, 9757, 9764, 9783, 9784, 9788, 9790,
                                9808, 9820, 9839, 9841, 9843, 9853, 9855, 9859,
                                9863, 9877, 9879, 9880, 9882, 9883, 9885, 9901,
                                9904, 9908, 9910, 9912, 9920, 9929, 9930, 9935,
                                9939, 9958, 9959, 9961, 9983, 10027, 10033, 10038,
                                10045, 10047, 10048, 10058, 10059, 10067, 10069,
                                10073, 10075, 10078, 10079, 10081, 10092, 10106,
                                10110, 10113, 10131
                              ) THEN 'Кампания № 2'
                              ELSE NULL
                    END AS ads_campaign
          FROM      user_actions
          ORDER BY  ads_campaign
          )
        , unpaid_orders AS (
          SELECT    order_id
          FROM      user_actions
          WHERE     ACTION LIKE '%cancel%'
          )
        , orders_revenue AS (
          SELECT    order_id
                  , SUM(price) AS revenue
          FROM      (
                    SELECT    order_id
                            , UNNEST(product_ids) AS product_id
                    FROM      orders
                    ) AS order_products
          LEFT JOIN products USING (product_id)
          GROUP BY  order_id
          )
SELECT    ads_campaign
        , ROUND(100.0*(SUM(revenue) - 250000.0)/250000.0, 2) AS roi
FROM      user_actions
LEFT JOIN participants USING (user_id)
LEFT JOIN orders_revenue USING (order_id)
WHERE     ads_campaign IS NOT NULL
AND       order_id NOT IN (
          SELECT    order_id
          FROM      unpaid_orders
          )
GROUP BY  ads_campaign
```

## 10. Comparison of advertising campaigns participants using Average Order Volume (AOV) 

```
WITH      participants AS (
          SELECT    DISTINCT user_id
                  , CASE
                              WHEN user_id IN (
                                8631, 8632, 8638, 8643, 8657, 8673, 8706, 8707, 8715, 8723, 8732,
                        
                                8739, 8741, 8750, 8751, 8752, 8770, 8774, 8788,
                                8791, 8804, 8810, 8815, 8828, 8830, 8845, 8853,
                                8859, 8867, 8869, 8876, 8879, 8883, 8896, 8909,
                                8911, 8933, 8940, 8972, 8976, 8988, 8990, 9002,
                                9004, 9009, 9019, 9020, 9035, 9036, 9061, 9069,
                                9071, 9075, 9081, 9085, 9089, 9108, 9113, 9144,
                                9145, 9146, 9162, 9165, 9167, 9175, 9180, 9182,
                                9197, 9198, 9210, 9223, 9251, 9257, 9278, 9287,
                                9291, 9313, 9317, 9321, 9334, 9351, 9391, 9398,
                                9414, 9420, 9422, 9431, 9450, 9451, 9454, 9472,
                                9476, 9478, 9491, 9494, 9505, 9512, 9518, 9524,
                                9526, 9528, 9531, 9535, 9550, 9559, 9561, 9562,
                                9599, 9603, 9605, 9611, 9612, 9615, 9625, 9633,
                                9652, 9654, 9655, 9660, 9662, 9667, 9677, 9679,
                                9689, 9695, 9720, 9726, 9739, 9740, 9762, 9778,
                                9786, 9794, 9804, 9810, 9813, 9818, 9828, 9831,
                                9836, 9838, 9845, 9871, 9887, 9891, 9896, 9897,
                                9916, 9945, 9960, 9963, 9965, 9968, 9971, 9993,
                                9998, 9999, 10001, 10013, 10016, 10023, 10030,
                                10051, 10057, 10064, 10082, 10103, 10105, 10122,
                                10134, 10135
                              ) THEN 'Кампания № 1'
                              WHEN user_id IN (
                                8629, 8630, 8644, 8646, 8650, 8655, 8659, 8660, 8663, 8665, 8670,
                                8675, 8680, 8681, 8682, 8683, 8694, 8697, 8700,
                                8704, 8712, 8713, 8719, 8729, 8733, 8742, 8748,
                                8754, 8771, 8794, 8795, 8798, 8803, 8805, 8806,
                                8812, 8814, 8825, 8827, 8838, 8849, 8851, 8854,
                                8855, 8870, 8878, 8882, 8886, 8890, 8893, 8900,
                                8902, 8913, 8916, 8923, 8929, 8935, 8942, 8943,
                                8949, 8953, 8955, 8966, 8968, 8971, 8973, 8980,
                                8995, 8999, 9000, 9007, 9013, 9041, 9042, 9047,
                                9064, 9068, 9077, 9082, 9083, 9095, 9103, 9109,
                                9117, 9123, 9127, 9131, 9137, 9140, 9149, 9161,
                                9179, 9181, 9183, 9185, 9190, 9196, 9203, 9207,
                                9226, 9227, 9229, 9230, 9231, 9250, 9255, 9259,
                                9267, 9273, 9281, 9282, 9289, 9292, 9303, 9310,
                                9312, 9315, 9327, 9333, 9335, 9337, 9343, 9356,
                                9368, 9370, 9383, 9392, 9404, 9410, 9421, 9428,
                                9432, 9437, 9468, 9479, 9483, 9485, 9492, 9495,
                                9497, 9498, 9500, 9510, 9527, 9529, 9530, 9538,
                                9539, 9545, 9557, 9558, 9560, 9564, 9567, 9570,
                                9591, 9596, 9598, 9616, 9631, 9634, 9635, 9636,
                                9658, 9666, 9672, 9684, 9692, 9700, 9704, 9706,
                                9711, 9719, 9727, 9735, 9741, 9744, 9749, 9752,
                                9753, 9755, 9757, 9764, 9783, 9784, 9788, 9790,
                                9808, 9820, 9839, 9841, 9843, 9853, 9855, 9859,
                                9863, 9877, 9879, 9880, 9882, 9883, 9885, 9901,
                                9904, 9908, 9910, 9912, 9920, 9929, 9930, 9935,
                                9939, 9958, 9959, 9961, 9983, 10027, 10033, 10038,
                                10045, 10047, 10048, 10058, 10059, 10067, 10069,
                                10073, 10075, 10078, 10079, 10081, 10092, 10106,
                                10110, 10113, 10131
                              ) THEN 'Кампания № 2'
                              ELSE NULL
                    END AS ads_campaign
          FROM      user_actions
          ORDER BY  ads_campaign
          )
        , unpaid_orders AS (
          SELECT    order_id
          FROM      user_actions
          WHERE     ACTION LIKE '%cancel%'
          )
        , orders_revenue AS (
          SELECT    order_id
                  , SUM(price) AS revenue
          FROM      (
                    SELECT    order_id
                            , UNNEST(product_ids) AS product_id
                    FROM      orders
                    ) AS order_products
          LEFT JOIN products USING (product_id)
          GROUP BY  order_id
          )
        , avg_check AS (
          SELECT    user_id
                  , ROUND(SUM(revenue)/COUNT(order_id), 2) AS avg_check
          FROM      user_actions
          LEFT JOIN orders_revenue USING (order_id)
          WHERE     order_id NOT IN (
                    SELECT    order_id
                    FROM      unpaid_orders
                    )
          AND       TIME::date BETWEEN '2022-09-01' AND '2022-09-07'
          GROUP BY  user_id
          )
SELECT    ads_campaign
        , ROUND(AVG(avg_check), 2) AS avg_check
FROM      avg_check
LEFT JOIN participants USING (user_id)
WHERE     ads_campaign IS NOT NULL
GROUP BY  ads_campaign
ORDER BY  avg_check DESC
```

## 11. Daily Retention Rate
```
WITH      first_action AS (
          SELECT    user_id
                  , MIN(TIME::date) OVER (
                    PARTITION BY user_id
                    ) AS start_date
                  , TIME::date AS date
                
          FROM      user_actions
          )
SELECT    DATE_TRUNC('month', start_date)::date AS start_month
        , start_date
        , date-start_date AS day_number
        , ROUND(
          COUNT(DISTINCT user_id)::NUMERIC/MAX(COUNT(DISTINCT user_id)) OVER (
          PARTITION BY start_date
          )
        , 2
          ) AS retention
FROM      first_action
GROUP BY  date
        , start_date
ORDER BY  start_date
        , day_number
```

## 12. Comparison of advertising campaign by Retention Rate

```
WITH      adv AS (
          SELECT    DISTINCT user_id
                  , CASE
                              WHEN user_id IN (8631, 8632, 8638, 8643, 8657, 8673, 8706, 8707, 8715, 8723, 8732,
                                                  8739, 8741, 8750, 8751, 8752, 8770, 8774, 8788,
                                                
                                                  8791, 8804, 8810, 8815, 8828, 8830, 8845, 8853,
                                                  8859, 8867, 8869, 8876, 8879, 8883, 8896, 8909,
                                                  8911, 8933, 8940, 8972, 8976, 8988, 8990, 9002,
                                                  9004, 9009, 9019, 9020, 9035, 9036, 9061, 9069,
                                                  9071, 9075, 9081, 9085, 9089, 9108, 9113, 9144,
                                                  9145, 9146, 9162, 9165, 9167, 9175, 9180, 9182,
                                                  9197, 9198, 9210, 9223, 9251, 9257, 9278, 9287,
                                                  9291, 9313, 9317, 9321, 9334, 9351, 9391, 9398,
                                                  9414, 9420, 9422, 9431, 9450, 9451, 9454, 9472,
                                                  9476, 9478, 9491, 9494, 9505, 9512, 9518, 9524,
                                                  9526, 9528, 9531, 9535, 9550, 9559, 9561, 9562,
                                                  9599, 9603, 9605, 9611, 9612, 9615, 9625, 9633,
                                                  9652, 9654, 9655, 9660, 9662, 9667, 9677, 9679,
                                                  9689, 9695, 9720, 9726, 9739, 9740, 9762, 9778,
                                                  9786, 9794, 9804, 9810, 9813, 9818, 9828, 9831,
                                                  9836, 9838, 9845, 9871, 9887, 9891, 9896, 9897,
                                                  9916, 9945, 9960, 9963, 9965, 9968, 9971, 9993,
                                                  9998, 9999, 10001, 10013, 10016, 10023, 10030,
                                                  10051, 10057, 10064, 10082, 10103, 10105, 10122,
                                                  10134, 10135) THEN 'Кампания № 1'
                              WHEN user_id IN (8629, 8630, 8644, 8646, 8650, 8655, 8659, 8660, 8663, 8665, 8670,
                                                  8675, 8680, 8681, 8682, 8683, 8694, 8697, 8700,
                                                  8704, 8712, 8713, 8719, 8729, 8733, 8742, 8748,
                                                  8754, 8771, 8794, 8795, 8798, 8803, 8805, 8806,
                                                  8812, 8814, 8825, 8827, 8838, 8849, 8851, 8854,
                                                  8855, 8870, 8878, 8882, 8886, 8890, 8893, 8900,
                                                  8902, 8913, 8916, 8923, 8929, 8935, 8942, 8943,
                                                  8949, 8953, 8955, 8966, 8968, 8971, 8973, 8980,
                                                  8995, 8999, 9000, 9007, 9013, 9041, 9042, 9047,
                                                  9064, 9068, 9077, 9082, 9083, 9095, 9103, 9109,
                                                  9117, 9123, 9127, 9131, 9137, 9140, 9149, 9161,
                                                  9179, 9181, 9183, 9185, 9190, 9196, 9203, 9207,
                                                  9226, 9227, 9229, 9230, 9231, 9250, 9255, 9259,
                                                  9267, 9273, 9281, 9282, 9289, 9292, 9303, 9310,
                                                  9312, 9315, 9327, 9333, 9335, 9337, 9343, 9356,
                                                  9368, 9370, 9383, 9392, 9404, 9410, 9421, 9428,
                                                  9432, 9437, 9468, 9479, 9483, 9485, 9492, 9495,
                                                  9497, 9498, 9500, 9510, 9527, 9529, 9530, 9538,
                                                  9539, 9545, 9557, 9558, 9560, 9564, 9567, 9570,
                                                  9591, 9596, 9598, 9616, 9631, 9634, 9635, 9636,
                                                  9658, 9666, 9672, 9684, 9692, 9700, 9704, 9706,
                                                  9711, 9719, 9727, 9735, 9741, 9744, 9749, 9752,
                                                  9753, 9755, 9757, 9764, 9783, 9784, 9788, 9790,
                                                  9808, 9820, 9839, 9841, 9843, 9853, 9855, 9859,
                                                  9863, 9877, 9879, 9880, 9882, 9883, 9885, 9901,
                                                  9904, 9908, 9910, 9912, 9920, 9929, 9930, 9935,
                                                  9939, 9958, 9959, 9961, 9983, 10027, 10033, 10038,
                                                  10045, 10047, 10048, 10058, 10059, 10067, 10069,
                                                  10073, 10075, 10078, 10079, 10081, 10092, 10106,
                                                  10110, 10113, 10131) THEN 'Кампания № 2'
                              ELSE NULL
                    END AS ads_campaign
          FROM      user_actions
          ORDER BY  ads_campaign
          )
        , first_action AS (
          SELECT    user_id
                  , MIN(TIME::date) OVER (
                    PARTITION BY user_id
                    ) AS start_date
                  , TIME::date AS date
                  , ads_campaign
          FROM      user_actions
          LEFT JOIN adv USING (user_id)
          WHERE     ads_campaign IS NOT NULL
          )
SELECT    ads_campaign
        , start_date
        , date-start_date AS day_number
        , ROUND(
          COUNT(DISTINCT user_id)::NUMERIC/MAX(COUNT(DISTINCT user_id)) OVER (
          PARTITION BY start_date
                  , ads_campaign
          )
        , 2
          ) AS retention
FROM      first_action
WHERE     date-start_date IN (0, 1, 7)
AND       ads_campaign IS NOT NULL
GROUP BY  ads_campaign
        , start_date
        , day_number
ORDER BY  ads_campaign
        , day_number
```

## 13. Dates when users attracted by advertising campaigns would cover marketing expenses (CAC) by their revenue 
```
WITH      participants AS (
          SELECT    user_id
                  , ads_campaign
          FROM      (
                    SELECT    DISTINCT user_id
                            , CASE
                            ## 13. Dates when users attracted by advertising campaigns would cover marketing expenses
                                        WHEN user_id IN (8631, 8632, 8638, 8643, 8657, 8673, 8706, 8707, 8715, 8723, 8732,
                8739, 8741, 8750, 8751, 8752, 8770, 8774, 8788,
                8791, 8804, 8810, 8815, 8828, 8830, 8845, 8853,
                8859, 8867, 8869, 8876, 8879, 8883, 8896, 8909,
                8911, 8933, 8940, 8972, 8976, 8988, 8990, 9002,
                9004, 9009, 9019, 9020, 9035, 9036, 9061, 9069,
                9071, 9075, 9081, 9085, 9089, 9108, 9113, 9144,
                9145, 9146, 9162, 9165, 9167, 9175, 9180, 9182,
                9197, 9198, 9210, 9223, 9251, 9257, 9278, 9287,
                9291, 9313, 9317, 9321, 9334, 9351, 9391, 9398,
                9414, 9420, 9422, 9431, 9450, 9451, 9454, 9472,
                9476, 9478, 9491, 9494, 9505, 9512, 9518, 9524,
                9526, 9528, 9531, 9535, 9550, 9559, 9561, 9562,
                9599, 9603, 9605, 9611, 9612, 9615, 9625, 9633,
                9652, 9654, 9655, 9660, 9662, 9667, 9677, 9679,
                9689, 9695, 9720, 9726, 9739, 9740, 9762, 9778,
                9786, 9794, 9804, 9810, 9813, 9818, 9828, 9831,
                9836, 9838, 9845, 9871, 9887, 9891, 9896, 9897,
                9916, 9945, 9960, 9963, 9965, 9968, 9971, 9993,
                9998, 9999, 10001, 10013, 10016, 10023, 10030,
                10051, 10057, 10064, 10082, 10103, 10105, 10122,
                10134, 10135) THEN 'Кампания № 1'
                                        WHEN user_id IN (8629, 8630, 8644, 8646, 8650, 8655, 8659, 8660, 8663, 8665, 8670,
                8675, 8680, 8681, 8682, 8683, 8694, 8697, 8700,
                8704, 8712, 8713, 8719, 8729, 8733, 8742, 8748,
                8754, 8771, 8794, 8795, 8798, 8803, 8805, 8806,
                8812, 8814, 8825, 8827, 8838, 8849, 8851, 8854,
                8855, 8870, 8878, 8882, 8886, 8890, 8893, 8900,
                8902, 8913, 8916, 8923, 8929, 8935, 8942, 8943,
                8949, 8953, 8955, 8966, 8968, 8971, 8973, 8980,
                8995, 8999, 9000, 9007, 9013, 9041, 9042, 9047,
                9064, 9068, 9077, 9082, 9083, 9095, 9103, 9109,
                9117, 9123, 9127, 9131, 9137, 9140, 9149, 9161,
                9179, 9181, 9183, 9185, 9190, 9196, 9203, 9207,
                9226, 9227, 9229, 9230, 9231, 9250, 9255, 9259,
                9267, 9273, 9281, 9282, 9289, 9292, 9303, 9310,
                9312, 9315, 9327, 9333, 9335, 9337, 9343, 9356,
                9368, 9370, 9383, 9392, 9404, 9410, 9421, 9428,
                9432, 9437, 9468, 9479, 9483, 9485, 9492, 9495,
                9497, 9498, 9500, 9510, 9527, 9529, 9530, 9538,
                9539, 9545, 9557, 9558, 9560, 9564, 9567, 9570,
                9591, 9596, 9598, 9616, 9631, 9634, 9635, 9636,
                9658, 9666, 9672, 9684, 9692, 9700, 9704, 9706,
                9711, 9719, 9727, 9735, 9741, 9744, 9749, 9752,
                9753, 9755, 9757, 9764, 9783, 9784, 9788, 9790,
                9808, 9820, 9839, 9841, 9843, 9853, 9855, 9859,
                9863, 9877, 9879, 9880, 9882, 9883, 9885, 9901,
                9904, 9908, 9910, 9912, 9920, 9929, 9930, 9935,
                9939, 9958, 9959, 9961, 9983, 10027, 10033, 10038,
                10045, 10047, 10048, 10058, 10059, 10067, 10069,
                10073, 10075, 10078, 10079, 10081, 10092, 10106,
                10110, 10113, 10131) THEN 'Кампания № 2'
                                        ELSE NULL
                              END AS ads_campaign
                    FROM      user_actions
                    ) all_users
          WHERE     ads_campaign IS NOT NULL
          AND       user_id IN (
                    SELECT    DISTINCT user_id
                    FROM      user_actions
                    WHERE     order_id NOT IN (
                              SELECT    order_id
                              FROM      user_actions
                              WHERE     ACTION LIKE '%cancel%'
                              )
                    )
          )
        , rev AS (
          SELECT    user_id
                  , TIME::date AS date
                  , ads_campaign
                  , SUM(revenue) AS revenue
          FROM      user_actions
          LEFT JOIN (
                    SELECT    order_id
                            , SUM(price) AS revenue
                    FROM      (
                              SELECT    order_id
                                      , UNNEST(product_ids) AS product_id
                              FROM      orders
                              ) AS ord_prods
                    LEFT JOIN products USING (product_id)
                    WHERE     order_id NOT IN (
                              SELECT    order_id
                              FROM      user_actions
                              WHERE     ACTION='cancel_order'
                              )
                    GROUP BY  order_id
                    ) AS order_rev USING (order_id)
          INNER     JOIN participants USING (user_id)
          GROUP BY  user_id
                  , date
                  , ads_campaign
          )
        , cac AS (
          SELECT    ads_campaign
                  , ROUND(250000.0/COUNT(DISTINCT user_id), 2) AS cac
          FROM      participants
          GROUP BY  ads_campaign
          )
        , for_arppu AS (
          SELECT    ads_campaign
                  , date
                  , COUNT(DISTINCT user_id) AS users_count
                  , SUM(revenue) AS revenue
          FROM      rev
          GROUP BY  ads_campaign
                  , date
          ORDER BY  date
                  , ads_campaign
          )
SELECT    ads_campaign
        , CONCAT(
          'Day '
        , date-MIN(date) OVER (
          PARTITION BY ads_campaign
          )
          ) AS DAY
        , ROUND(
          SUM(revenue) OVER (
          PARTITION BY ads_campaign
          ORDER BY  date
          )/MAX(users_count) OVER (
          PARTITION BY ads_campaign
          )
        , 2
          ) AS cumulative_arppu
        , cac
FROM      for_arppu
LEFT JOIN cac USING (ads_campaign)
ORDER BY  ads_campaign
        , DAY
```