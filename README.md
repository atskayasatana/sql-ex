# sql-ex
Решения упражнений с сайта sql-ex

## Задача 92

Выбрать все белые квадраты, которые окрашивались только из баллончиков,
пустых к настоящему времени. Вывести имя квадрата 

### Решение
```` sql
WITH is_empty AS
(SELECT B_V_ID, SUM(B_VOL) as ttl_vol FROM utB
GROUP BY B_V_ID 
HAVING SUM(B_VOL)>=255
)

SELECT Q_NAME FROM(
SELECT Q_NAME, COUNT(DISTINCT V_COLOR) as rgb, SUM(B_VOL) as vol 
FROM utB JOIN utQ ON B_Q_ID = Q_ID JOIN utV ON V_ID = B_V_ID
WHERE B_V_ID IN 
(SELECT B_V_ID FROM is_empty
)
GROUP BY Q_NAME
HAVING COUNT(DISTINCT V_COLOR) =3 AND SUM(B_VOL)=255*3) tmp
````
## Задача 93
Для каждой компании, перевозившей пассажиров, подсчитать время, которое провели в полете самолеты с пассажирами.
Вывод: название компании, время в минутах. 

Комментарий: 
рейсы с одинаковыми номерами могут вылетать в разные дни, есть рейсы без пассажиров.

Используемые функции:

[DATEDIFF](https://www.w3schools.com/sql/func_sqlserver_datediff.asp)

```` sql
WITH trips_durations AS
(SELECT DISTINCT t.trip_no, t.ID_comp, pt.date,
CASE WHEN DATEDIFF(mi, t.time_out, t.time_in) <=0 THEN DATEDIFF(mi, t.time_out, t.time_in+1)
ELSE DATEDIFF(mi, t.time_out, t.time_in)
END min_dur
FROM Trip t RIGHT JOIN Pass_in_trip pt ON t.trip_no = pt.trip_no)

SELECT c.name as company, SUM(min_dur) as minutes FROM trips_durations td LEFT JOIN Company c ON td.ID_comp = c.ID_comp
GROUP BY c.name


````

## Задача 94

Для семи последовательных дней, начиная от минимальной даты, когда из Ростова было совершено максимальное число рейсов, определить число рейсов из Ростова.

Вывод: дата, количество рейсов 

Используемые функции:

Сложение дат:
[INTERVAL](https://stackoverflow.com/questions/26979764/postgres-interval-using-value-from-table)

[GENERATE_SERIES](https://database.guide/how-generate_series-works-in-postgresql/)

```` sql
WITH trips_by_dates AS
(
SELECT DISTINCT t.trip_no as tn,pt.date as date, t.town_from as tf ,t.time_out as time_out FROM Pass_in_trip pt LEFT JOIN Trip t ON t.trip_no = pt.trip_no
WHERE t.town_from= 'Rostov'
),

qty_by_dates AS
(
SELECT date, COUNT(tn) as cnt FROM trips_by_dates
GROUP BY date
ORDER BY COUNT(tn) DESC, date ASC
),
dates AS
(SELECT DISTINCT GENERATE_SERIES(
(SELECT date FROM qty_by_dates LIMIT 1) , (SELECT date FROM qty_by_dates LIMIT 1) + INTERVAL '6 day','1 day') as date)

SELECT d.date, COALESCE(qd.cnt , 0) as qty FROM dates d LEFT JOIN qty_by_dates qd ON d.date = qd.date
````

## Задача 95

Задание: 95 (qwrqwr: 2013-02-08)

На основании информации из таблицы Pass_in_Trip, для каждой авиакомпании определить:

1) количество выполненных перелетов;
   
2) число использованных типов самолетов;
   
3) количество перевезенных различных пассажиров;
   
4) общее число перевезенных компанией пассажиров.
   
Вывод: Название компании, 1), 2), 3), 4).

Комментарий: При подсчете рейсов учесть, что при соединении таблиц номера рейсов могут задублироваться по числу пассажиров.

```` sql
WITH trips AS(
SELECt t.ID_comp as comp,
COUNT( DISTINCT CONCAT(pit.trip_no, pit.date)) as f_count,
COUNT(DISTINCT t.plane) as plane_type,
COUNT(DISTINCT pit.ID_psg) as ps_count,
COUNT(pit.ID_psg) as ttl_psg FROM Pass_in_trip pit LEFT JOIN Trip t ON pit.trip_no = t.trip_no
GROUP BY t.ID_comp
)

SELECT c.name as company_name,
t.f_count as flights,
t.plane_type as planes,
t.ps_count as diff_psngrs,
t.ttl_psg as total_psngrs FROM trips t LEFT JOIN Company c ON c.ID_Comp=t.comp


````

## Задача 96

При условии, что баллончики с красной краской использовались более одного раза, 

выбрать из них такие, которыми окрашены квадраты, имеющие голубую компоненту.

Вывести название баллончика 

```` sql
WITH red_baloons AS
(SELECT utb.B_V_ID as b_id, utv.V_COLOR as color, COUNT(utb.B_DATETIME) as used
FROM utB utb LEFT JOIN utV utv ON utb.B_V_ID=utV.V_ID
GROUP BY utb.B_V_ID,utv.V_COLOR
HAVING utv.V_COLOR='R' AND COUNT(utb.B_DATETIME)>1),
blue_q AS
(SELECT utb.B_Q_ID as Q_id, utb.B_V_ID, utv.V_COLOR as clr, utv.V_NAME as b_name
FROM utB utb LEFT JOIN utV utv ON utb.B_V_ID=utV.V_ID
WHERE utv.V_COLOR = 'B'
)

SELECT DISTINCT V_NAME FROM utB utb LEFT JOIN utV utv ON utb.B_V_ID=utV.V_ID
WHERE 
B_Q_ID IN
(SELECT Q_id FROM blue_q)
AND V_ID IN
(SELECT b_id FROM red_baloons)

````

## Задание: 97 (qwrqwr: 2013-02-15)

Отобрать из таблицы Laptop те строки, для которых выполняется следующее условие:

значения из столбцов speed, ram, price, screen возможно расположить таким образом, что каждое последующее значение будет превосходить предыдущее в 2 раза или более.

Замечание: все известные характеристики ноутбуков больше нуля.

Вывод: code, speed, ram, price, screen.



````sql

WITH unpivoted_data AS
(SELECT code, param, info 
FROM
(SELECT 
      CAST (code AS FLOAT) code, 
      CAST (speed AS FLOAT) speed, 
      CAST (price AS FLOAT) price, 
      CAST (ram AS FLOAT) ram, 
      CAST (screen AS FLOAT) screen 
 FROM Laptop) t
UNPIVOT
(info FOR param IN(speed, price, ram, screen))upv
),
unpivoted_ranked AS
(
SELECT code, 
       param, 
       info,
       RANK() OVER(PARTITION BY code ORDER BY info) r_info FROM unpivoted_data
)

SELECT code, speed, ram, price, screen FROM Laptop 
WHERE code IN(
SELECT code FROM 
(
SELECT code,[1],[2],[3],[4] FROM 
(SELECT code, info, r_info FROM unpivoted_ranked) t
PIVOT
(MIN(info) FOR r_info IN([1],[2],[3],[4])) pvt
WHERE [1]*2<=[2] AND [2]*2<=[3] AND [3]*2<=[4]
) t
)


````

## Задание 99
Рассматриваются только таблицы Income_o и Outcome_o. Известно, что прихода/расхода денег в воскресенье не бывает.

Для каждой даты прихода денег на каждом из пунктов определить дату инкассации по следующим правилам:

1. Дата инкассации совпадает с датой прихода, если в таблице Outcome_o нет записи о выдаче денег в эту дату на этом пункте.
   
2. В противном случае - первая возможная дата после даты прихода денег, которая не является воскресеньем и в Outcome_o не отмечена выдача денег сдатчикам вторсырья в эту дату на этом пункте.
   
Вывод: пункт, дата прихода денег, дата инкассации.

Комментарий: решение достаточно примитивное, создаем диапазон дат для каждой даты прихода и оставляем только те, которые не воскресение и в которые не было расхода.


```` sql
WITH date_calc AS
(SELECT 0 as cnt, 
        point,
        date, 
        inc,
        date+0 AS next_date, 
        DATENAME(dw, date) AS out_dw 
 FROM Income_o
UNION ALL
SELECT cnt+1,
       point,
       date,
       inc,
      date + cnt,
     DATENAME(dw, date + cnt) AS out_dw        
FROM date_calc
WHERE cnt<100
),
dates_calculated AS
(
SELECT dc.point,
       dc.date AS inc_date, 
       dc.inc AS income, 
       dc.next_date as next_date,
       dc.out_dw AS day_of_week, 
       out.point AS o_pnt, 
       out.date AS o_date, 
       out.out AS out_m,
       ROW_NUMBER() OVER (PARTITION BY CONCAT(dc.point, dc.date) ORDER BY dc.date) as d_rnk 
FROM date_calc dc FULL JOIN Outcome_o out ON dc.point=out.point AND dc.next_date=out.date
WHERE dc.out_dw <> 'Sunday' AND out.out IS NULL
)

SELECT dc.point AS point, dc.inc_date AS DP, dc.next_date AS DI FROM dates_calculated dc WHERE d_rnk = 1

````

## Задача 102

Определить имена разных пассажиров, которые летали

только между двумя городами (туда и/или обратно).

```` sql
WITH flights_history AS
(
SELECT pit.ID_psg as pid,
       t.town_from as town
       FROM Pass_in_trip pit LEFT JOIN Trip t 
ON pit.trip_no = t.trip_no

UNION 

SELECT pit.ID_psg as pid,
       t.town_to as town
       FROM Pass_in_trip pit LEFT JOIN Trip t 
ON pit.trip_no = t.trip_no
),
pid_2 AS
(
SELECT pid, COUNT(DISTINCT town) as t_cnt FROM flights_history
GROUP BY pid
HAVING COUNT(DISTINCT town)=2)

SELECT name FROM Passenger 
WHERE ID_Psg IN
(SELECT pid FROM pid_2)
````

## Задача 103

Выбрать три наименьших и три наибольших номера рейса. Вывести их в шести столбцах одной строки, расположив в порядке от наименьшего к наибольшему.

Замечание: считать, что таблица Trip содержит не менее шести строк. 

```` sql
SELECT MIN(t.trip_no) as min1, 
       MIN(t2.trip_no) as min2, 
       MIN(t3.trip_no) as min3, 
       MAX(t.trip_no) as max1,
       MAX(t2.trip_no) as max2,
       MAX(t3.trip_no) as max3
FROM Trip t, Trip t2, Trip t3
WHERE t.trip_no<t2.trip_no AND t3.trip_no>t2.trip_no
```










