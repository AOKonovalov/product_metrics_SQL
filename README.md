# Расчет продуктовых метрик службы доставки
---

## Описание продукта и базы данных:

###### Продукт - приложение доставки продуктов, которое запустилось 2 недели назад 

user_actions — действия пользователей с заказами. 

| Столбец  | Тип данных  | Описание                                                                                          |
| -------- | ----------- | ------------------------------------------------------------------------------------------------- |
| user_id  | INT         | id пользователя                                                                                   |
| order_id | INT         | id заказа                                                                                         |
| action   | VARCHAR(50) | действие пользователя с заказом; 'create_order' — создание заказа, 'cancel_order' — отмена заказа |
| time     | TIMESTAMP   | время совершения действия                                                                         |


courier_actions — действия курьеров с заказами.

| Столбец    | Тип данных  | Описание                                                                                        |
| ---------- | ----------- | ----------------------------------------------------------------------------------------------- |
| courier_id | INT         | id курьера                                                                                      |
| order_id   | INT         | id заказа                                                                                       |
| action     | VARCHAR(50) | действие курьера с заказом; 'accept_order' — принятие заказа, 'deliver_order' — доставка заказа |
| time       | TIMESTAMP   | время совершения действия                                                                       |


orders — информация о заказах.

| Столбец       | Тип данных | Описание                   |
| ------------- | ---------- | -------------------------- |
| order_id      | INT        | id заказа                  |
| creation_time | TIMESTAMP  | время создания заказа      |
| product_ids   | integer[]  | список id товаров в заказе |


users  — информация о пользователях.

| Столбец    | Тип данных  | Описание                                  |
| ---------- | ----------- | ----------------------------------------- |
| user_id    | INT         | id пользователя                           |
| birth_date | DATE        | дата рождения                             |
| sex        | VARCHAR(50) | пол; 'male' — мужской, 'female' — женский |


couriers — информация о курьерах.

| Столбец    | Тип данных  | Описание                                  |
| ---------- | ----------- | ----------------------------------------- |
| courier_id | INT         | id курьера                                |
| birth_date | DATE        | дата рождения                             |
| sex        | VARCHAR(50) | пол; 'male' — мужской, 'female' — женский |


products — информация о товарах, которые доставляет сервис.

| Столбец    | Тип данных  | Описание        |
| ---------- | ----------- | --------------- |
| product_id | INT         | id продукта     |
| name       | VARCHAR(50) | название товара |
| price      | FLOAT(4)    | цена товара     |

---

## 1. DAU

``` postgresql
SELECT  CAST(time as date) as day,
        COUNT(DISTINCT user_id) as DAU
FROM    user_actions
GROUP BY day
ORDER BY day

--- если считать активностью совершение заказа, то запрос будет следующим:
SELECT  time::date as day,
        COUNT(distinct user_id) as paying_users
FROM    user_actions
WHERE   order_id not in (SELECT order_id
                        FROM   user_actions
                        WHERE  action = 'cancel_order')
GROUP BY day
```

Визуализация в redash:
![newplot (1)](https://github.com/user-attachments/assets/4b9f7f9f-8be4-462f-836c-13af9fd6f7e7)

---


## 2. ARPU,  ARPPU и AOV по дням


``` POSTGRESQL
SELECT date,
       round(revenue/users::decimal , 2) as arpu,
       round(revenue/paying_users::decimal , 2) as arppu,
       round(revenue/orders_count::decimal , 2) as aov
FROM          (SELECT creation_time::date as date,
		              SUM(price) as revenue
		       FROM   (SELECT creation_time,
		                      unnest(product_ids) as product_id
		           FROM   orders
		           WHERE  order_id not in (SELECT order_id
	                                       FROM   user_actions
    	                                       WHERE  action = 'cancel_order')) t1
		        LEFT JOIN products 
			        USING (product_id)
		        GROUP BY date) t2
		        
LEFT JOIN     (SELECT time::date as date,
	                  COUNT(distinct user_id) as paying_users
	           FROM   user_actions
	           WHERE  order_id not in (SELECT order_id
	                                   FROM   user_actions
	                                   WHERE  action = 'cancel_order')
	           GROUP BY date) t3 USING(date)
	
LEFT JOIN     (SELECT time::date as date,
		                 COUNT(distinct user_id) as users
	           FROM   user_actions
	           GROUP BY date) t4 USING(date)
	
LEFT JOIN     (SELECT creation_time::date as date,
	                  COUNT(distinct order_id) as orders_count
	           FROM   orders
	           WHERE  order_id not in (SELECT order_id
                                   FROM   user_actions
                                   WHERE  action = 'cancel_order')
	           GROUP BY date) t5 USING(date)
```

Визуализация в redash:
![Pasted image 20250121141239](https://github.com/user-attachments/assets/96e5099c-2699-4187-977b-a67921a11655)

ARPU (Average Revenue Per User) и ARPPU (Average Revenue Per Paying User) стабильно демонстрирует  колебания, а AOV (Average Order Value) в первые дни работы сервиса показывал колебания, но затем стабилизировался в районе 375-385 рублей.
ARPU и ARPPU двигаются параллельно с небольшим расстоянием друг от друга говорит о стабильном и достаточно высокоц конверсии в платящего пользователя.

---

## 3.  Кумулятивные ARPU, ARPPU и AOV по дням


```postgresql
WITH 
t_total_revenue as 
	(SELECT date,
           SUM(revenue) OVER(ORDER BY date) as total_revenue
    FROM   (SELECT creation_time::date as date,
                   SUM(price) as revenue
            FROM   (SELECT  creation_time,
                    unnest(product_ids) as product_id
                    FROM   orders
                    WHERE  order_id not in (SELECT order_id
                                            FROM   user_actions
                                            WHERE  action = 'cancel_order')) t1
            LEFT JOIN products 
	                  USING (product_id)
            GROUP BY date) t2),
t_total_users as
	(SELECT date,
	        SUM(count(user_id)) OVER(ORDER BY date)::integer as total_users
	FROM   (SELECT MIN(time::date) as date,
				   user_id
	       FROM   user_actions
	       GROUP BY user_id) t1
	GROUP BY date),
t_total_paying_users as
	(SELECT date,
            SUM(COUNT(user_id)) OVER(ORDER BY date)::integer as total_paying_users
    FROM   (SELECT min(time::date) as date,
                   user_id
            FROM   user_actions
            WHERE  order_id not in (SELECT order_id
                                    FROM   user_actions
                                    WHERE  action = 'cancel_order')
            GROUP BY user_id) t1
	GROUP BY date),
t_total_orders as
	(SELECT date,
		    SUM(orders_count) OVER(ORDER BY date)::integer as total_orders
	FROM   (SELECT  creation_time::date as date,
		            COUNT(distinct order_id) as orders_count
		    FROM   orders
		    WHERE  order_id not in (SELECT order_id
		                            FROM   user_actions
		                            WHERE  action = 'cancel_order')
			GROUP BY date) t1) 

SELECT date,
       round(total_revenue/total_users::decimal , 2) as running_arpu,
       round(total_revenue/total_paying_users::decimal , 2) as running_arppu,
       round(total_revenue/total_orders::decimal , 2) as running_aov
FROM    t_revenue
LEFT JOIN  t_total_users         USING(date)
LEFT JOIN  t_total_paying_users  USING(date)
LEFT JOIN  t_total_orders        USING(date)
```

Визуализация в redash:
![Pasted image 20250122150109](https://github.com/user-attachments/assets/c17eb632-d214-461a-acf5-a710cb1cce2b)

Динамические показатели ARPU и ARPPU растут, что свидетельствует о сопоставимом росте выручки и числа клиентов. При этом AOV остаётся неизменным, что свидетельствует о стабильности среднего чека.

---

## 4. ARPU,  ARPPU и AOV по дням недели

Чтобы понять, с чем может быть связана колеблющаяся динамика ARPU и ARPPU, можно посмотреть на метрики в разрезе дней недели. Это также поволит оценить наличия вляиния фаткора выходного дня на продажи.

```postgresql
WITH
t_revenue as
	(SELECT to_char(creation_time, 'Day') as weekday,
	        MAX(date_part('isodow', creation_time)) as weekday_number,
	        COUNT(distinct order_id) as orders,
	        SUM(price) as revenue
	FROM   (SELECT   order_id,
	             creation_time,
	                 unnest(product_ids) as product_id
	        FROM     orders
	        WHERE    order_id not in (SELECT order_id
	                                 FROM   user_actions
	                                 WHERE  action = 'cancel_order')
	                 AND creation_time BETWEEN '2022-08-26' AND '2022-09-09') t1
	LEFT JOIN products 
	          USING(product_id)
	GROUP BY  weekday),
t_users as
	(SELECT  to_char(time, 'Day') as weekday,
             MAX(date_part('isodow', time)) as weekday_number,
             COUNT(distinct user_id) as users
    FROM     user_actions
    WHERE    time BETWEEN '2022-08-26' AND '2022-09-09'
    GROUP BY weekday),
t_paying_users as
	(SELECT to_char(time, 'Day') as weekday,
            MAX(date_part('isodow', time)) as weekday_number,
            COUNT(distinct user_id) as paying_users
    FROM   user_actions
    WHERE  order_id not in (SELECT order_id
                            FROM   user_actions
                            WHERE  action = 'cancel_order')
           AND time BETWEEN '2022-08-26' AND '2022-09-09'
    GROUP BY weekday)


SELECT weekday,
       t_revenue.weekday_number::int as weekday_number,
       round(revenue::decimal / users, 2) as arpu,
       round(revenue::decimal / paying_users, 2) as arppu,
       round(revenue::decimal / orders, 2) as aov
FROM    t_revenue
LEFT JOIN  t_users using (weekday)
LEFT JOIN  t_paying_users using (weekday)
ORDER BY weekday_number
```

Визуализация в redash:
![Pasted image 20250122151956](https://github.com/user-attachments/assets/cfda47f4-12c1-46ff-b6ed-f972e87c177f)

В среднем выручка на пользователя максимальна в выходные дни, остаётся достаточно высокой в понедельник, в остальные дни недели показатели остаются стабильными (530-550 рублей), средняя стоимость заказа меняется незначительно в разные дни недели. 
Это может быть точкой роста для нашего продукта: в теории доставка продуктов более востребована в будние дни, поэтому, опираясь на этот факт, можно провести анализ действий пользователей (почему доставка актуальна для них в выходные - можем быть у нас не хватает ассортимента (готовой еды, заморозок и т.д.) или требуется сделать слишком много действий для заказа или нельзя выбирать время доставки, что делает заказ в рабочие дни неудобным).

--- 

## 5. Доли старых и новых пользователей в общей выручке

Новыми считаются те пользователи, которые сделали первый заказ в этот день.

``` postgresql
SELECT date,
       revenue,
       new_users_revenue,
       round(new_users_revenue / revenue * 100, 2) as new_users_revenue_share,
       100 - round(new_users_revenue / revenue * 100, 2) as old_users_revenue_share
FROM   (SELECT creation_time::date as date,
               SUM(price) as revenue
        FROM   (SELECT order_id,
                       creation_time,
                       unnest(product_ids) as product_id
                FROM   orders
                WHERE  order_id not in (SELECT order_id
                                        FROM   user_actions
                                        WHERE  action = 'cancel_order')) t3
            LEFT JOIN products using (product_id)
        GROUP BY date) t1
LEFT JOIN 
       (SELECT start_date as date,
               SUM(revenue) as new_users_revenue
        FROM   (SELECT t5.user_id,
                       t5.start_date,
                       coalesce(t6.revenue, 0) as revenue
                FROM   (SELECT user_id,
                               min(time::date) as start_date
                        FROM   user_actions
                        GROUP BY user_id) t5
                LEFT JOIN 
                        (SELECT user_id,
                                date,
                                SUM(order_price) as revenue
                        FROM   (SELECT user_id,
                                       time::date as date,
                                       order_id
                                FROM   user_actions
                                WHERE  order_id not in (SELECT order_id
                                                        FROM   user_actions
                                                        WHERE  action = 'cancel_order')) t7
                        LEFT JOIN 
		                        (SELECT order_id,
                                        SUM(price) as order_price
                                FROM   (SELECT order_id,
                                            unnest(product_ids) as product_id
                                        FROM   orders
                                        WHERE  order_id not in (SELECT order_id
                                                              FROM   user_actions
                                                              WHERE  action = 'cancel_order')) t9
                                LEFT JOIN products using (product_id)
                        GROUP BY order_id) t8 using (order_id)
                GROUP BY user_id, date) t6
                           ON t5.user_id = t6.user_id and
                              t5.start_date = t6.date) t4
        GROUP BY start_date) t2 using (date)
```

Визуализация в redash:
![Pasted image 20250122155553](https://github.com/user-attachments/assets/95583ece-5fed-436b-9343-6a023e47e804)

Спустя 2 недели с запуска проекта показатель выручки от новых пользователей(синяя часть колонок) остается на достаточно высоком уровне, около 40%. Это хороший показатель, так как говорит о хорошей работе маркетинговых каналов и об отсуствии препятствий для совершения первого заказа.
Дополнительно можно было посмотреть retention, чтоб оценить удерживаемость старых пользователей.

--- 

## 6. Доля товаров в выручке


```postgresql
SELECT product_name,
       sum(revenue) as revenue,
       sum(share_in_revenue) as share_in_revenue
FROM   (SELECT case when round(100 * revenue / sum(revenue) OVER (), 2) >= 0.5 then name
                    else 'ДРУГОЕ' end as product_name,
               revenue,
               round(100 * revenue / sum(revenue) OVER (), 2) as share_in_revenue
        FROM   (SELECT name,
                       sum(price) as revenue
                FROM   (SELECT order_id,
                               unnest(product_ids) as product_id
                        FROM   orders
                        WHERE  order_id not in (SELECT order_id
                                                FROM   user_actions
                                                WHERE  action = 'cancel_order')) t1
                LEFT JOIN products using(product_id)
                GROUP BY name) t2) t3
GROUP BY product_name
ORDER BY revenue desc
```

Визуализация в redash:
![Pasted image 20250122155916](https://github.com/user-attachments/assets/3f45df31-e162-47c5-8cba-b9ec423c16c2)

Наибольшую долю в выручке занимает свинина, 'другое' куда были включены товары с долей мене 0,5%, а также оливковое масло и говядина. 

Если разделить товары на более широкие группы товаров, то категория Мясо и птица занимали бы наибольшую долю в выручке.

--- 
## 7. Динамика числа активных курьеров и платящих пользователей


``` postgresql
SELECT date,
       paying_users,
       active_couriers,
       round(100 * paying_users::decimal / total_users, 2) as paying_users_share,
       round(100 * active_couriers::decimal / total_couriers, 2) as active_couriers_share
FROM   (SELECT date,
               count(user_id) as new_users,
               sum(count(user_id)) OVER(ORDER BY date)::integer as total_users
        FROM   (SELECT min(time::date) as date,
                       user_id
                FROM   user_actions
                GROUP BY user_id) t1
        GROUP BY date)t11 join (SELECT date,
                               count(courier_id) as new_couriers,
                               sum(count(courier_id)) OVER(ORDER BY date)::integer as total_couriers
                        FROM   (SELECT min(time::date) as date,
                                       courier_id
                                FROM   courier_actions
                                GROUP BY courier_id) t2
                        GROUP BY date) t22 using(date)
    LEFT JOIN (SELECT time::date as date,
                      count(distinct courier_id) as active_couriers
               FROM   courier_actions
               WHERE  order_id not in (SELECT order_id
                                       FROM   user_actions
                                       WHERE  action = 'cancel_order')
               GROUP BY date) t3 using(date)
    LEFT JOIN (SELECT time::date as date,
                      count(distinct user_id) as paying_users
               FROM   user_actions
               WHERE  order_id not in (SELECT order_id
                                       FROM   user_actions
                                       WHERE  action = 'cancel_order')
               GROUP BY date) t4 using(date)
```

Визуализация в redash:

Динамика числа платящих пользователей и активных курьеров:
![Pasted image 20250122173347](https://github.com/user-attachments/assets/4e248b26-da0e-4f98-8a48-2edba52bf5d5)
Динамика долей платящих пользователей и активных курьеров:
![Pasted image 20250122173356](https://github.com/user-attachments/assets/15486c1c-7dc6-4108-8304-cda7dab4b7cb)

Количество пользователей растёт быстрее, чем количество курьеров. При этом большинство курьеров остаются активными, чего нельзя сказать о пользователях: доля платящих клиентов за 2 недели работы сервиса снизилась. Вероятно большинство пользователей делают один заказ и больше не пользуются сервисом.


---
## 8. Доли пользователей с одним и несколькими заказами среди платящих пользователей:


``` postgresql
SELECT date,
       round(100 * single_order_users::decimal / paying_users,
             2) as single_order_users_share,
       100 - round(100 * single_order_users::decimal / paying_users,
                   2) as several_orders_users_share
FROM   (SELECT time::date as date,
               count(distinct user_id) as paying_users
        FROM   user_actions
        WHERE  order_id not in (SELECT order_id
                                FROM   user_actions
                                WHERE  action = 'cancel_order')
        GROUP BY date) t1
    LEFT JOIN (SELECT date,
                      count(user_id) as single_order_users
               FROM   (SELECT time::date as date,
                              user_id,
                              count(distinct order_id) as user_orders
                       FROM   user_actions
                       WHERE  order_id not in (SELECT order_id
                                               FROM   user_actions
                                               WHERE  action = 'cancel_order')
                       GROUP BY date, user_id having count(distinct order_id) = 1) t2
               GROUP BY date) t3 using (date)
ORDER BY date
```

Визуализация в redash:
![Pasted image 20250122160628](https://github.com/user-attachments/assets/5d29561a-820c-4074-a9db-bd3b62784fa9)

Доля пользователей с несколькими заказами (красная часть графика) в среднем составляет около 20–25%.

Основная часть пользователей (в среднем примерно 70–80%) делают только один заказ, что видно по преобладанию синей части графика. Это подтверждает гипотезу из расчета доли платящих клиентов, которая снижается с ростом числа пользователей.

Значение показателя в первый день можно считать аномально низким, но это закономерно для начальных периодов, когда пользователи только начинают пользоваться сервисом.

---

## 9. Динамика кол-ва и доли отмененных заказов в течение дня


``` postgresql
SELECT hour,
       successful_orders,
       canceled_orders,
       round(canceled_orders::decimal / (successful_orders + canceled_orders),
             3) as cancel_rate
FROM   (SELECT date_part('hour', creation_time)::int as hour,
               count(order_id) as successful_orders
        FROM   orders
        WHERE  order_id not in (SELECT order_id
                                FROM   user_actions
                                WHERE  action = 'cancel_order')
        GROUP BY hour) t1
    LEFT JOIN (SELECT date_part('hour', creation_time)::int as hour,
                      count(order_id) as canceled_orders
               FROM   orders
               WHERE  order_id in (SELECT order_id
                                   FROM   user_actions
                                   WHERE  action = 'cancel_order')
               GROUP BY hour) t2 using (hour)
ORDER BY hour
```

Визуализация в redash:
![Pasted image 20250122163332](https://github.com/user-attachments/assets/c0d14562-60ed-4228-b870-de04d039e3db)

Пиковая активность пользователей приходится на вечернее время (18:00–22:00), минимальная активность — ночью (1:00–6:00). Показатель cancel rate остаётся стабильно низким в районе 5%, что может говорить о качественной работе сервиса даже в пиковые периоды. 

--- 

## 10. Retention по дням

Оценим удерживаемость пользователей по дням (имеет смысл посмотреть по дням, так как доставка вполне вероятно может использоваться каждый день)

``` postgresql
SELECT date_trunc('month', start_date)::date as start_month,
       start_date,
       dt - start_date as day_number,
       round(users::decimal / first_value(users) OVER (PARTITION BY start_date), 2) as retention
FROM   (SELECT start_date,
               dt,
               COUNT(distinct user_id) as users
        FROM   (SELECT user_id,
                       time::date as dt,
                       MIN(time::date) OVER (PARTITION BY user_id) as start_date
                FROM   user_actions) t1
        GROUP BY start_date, dt) t2
```
Визуализация в redash:
![image](https://github.com/user-attachments/assets/0e372de4-fcf1-4553-badd-3826512723fe)


Можно отметить определенную динамику: в первые 7-10 дней retntion колеблется на уровне 15-20%, но после падает до 5-7% (при этом у новых пользоваталей в этот календарный день retention на уровне 14-17%) - это может свидетельствовать о плохой удерживаемости для старых пользователей. То есть возможно есть какая-то проблема на каком-то этапе взаимодействия с продуктом, которая отсеивает часть пользователей. 
Компании важно это исследовать, так как иначе можно не достигнуть product/market-fit  и компании не задержится на рынке.
