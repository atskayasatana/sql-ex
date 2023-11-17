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
