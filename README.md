# index_work
д.з индексы
```
--Для выполнения домашнего задания и последующей практики загрузил на локальный PostgreSQL развернутый на windows
тестовую БД bookings (demo-big) с сайта https://postgrespro.ru/education/demodb и удалил все имеющиеся на ней индексы и ключи

1. Создать индекс к таблице
explain analyze
select tf.ticket_no,
	tf.flight_id,
	tf.fare_conditions,
	tf.amount
from bookings.ticket_flights as tf 
where flight_id = '7683'

/*
Gather  (cost=1000.00..114650.76 rows=102 width=32) (actual time=0.629..451.414 rows=57 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on ticket_flights tf  (cost=0.00..113640.56 rows=42 width=32) (actual time=217.923..367.666 rows=19 loops=3)
        Filter: (flight_id = 7683)
        Rows Removed by Filter: 2797265
Planning Time: 0.081 ms
Execution Time: 451.445 ms
*/
--Без индекса план выполнения строится с паралелльным выполнением запроса, в настройках моего домашнего кластера 
workersplanned = 2 потока это максимально возможное распаралеливание parallel seq scan говорит о том что происходит построчное вычитывание в 2 потока таблицы ticket_flight по фильтру flight_id = 7683   

create index fli_id_indx on bookings.ticket_flights using btree(flight_id);

explain analyze
select tf.ticket_no,
	   tf.flight_id,
	   tf.fare_conditions,
	   tf.amount
from bookings.ticket_flights as tf 
where flight_id = '7683'

/*
Index Scan using fli_id_indx on ticket_flights tf  (cost=0.43..149.79 rows=102 width=32) (actual time=0.088..0.171 rows=57 loops=1)
  Index Cond: (flight_id = 7683)
Planning Time: 2.161 ms
Execution Time: 0.193 ms
 */

--После создания Btree индеса чтение таблицы происходит по индексу узел gather не включается распаралеливание и не требуется и так очень быстро работает в сравнении с предыдущим запросом

--2. Реализовать индекс для полнотекстового поиска
explain analyze
select bt.ticket_no,
	   bt.book_ref,
	   bt.passenger_id,
	   bt.passenger_name,
	   bt.contact_data	   
from bookings.tickets as bt
where passenger_name = 'DENIS YAKOVLEV'

/*
Gather  (cost=1000.00..65805.04 rows=262 width=104) (actual time=0.783..384.551 rows=196 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on tickets bt  (cost=0.00..64778.84 rows=109 width=104) (actual time=7.016..325.958 rows=65 loops=3)
        Filter: (passenger_name = 'DENIS YAKOVLEV'::text)
        Rows Removed by Filter: 983220
Planning Time: 0.113 ms
Execution Time: 384.615 ms
 */

create index pass_name_indx on bookings.tickets (passenger_name);

explain analyze
select bt.ticket_no,
	   bt.book_ref,
	   bt.passenger_id,
	   bt.passenger_name,
	   bt.contact_data	   
from bookings.tickets as bt
where passenger_name = 'DENIS YAKOVLEV'

/*
 * Bitmap Heap Scan on tickets bt  (cost=3.96..390.70 rows=262 width=104) (actual time=0.713..6.289 rows=196 loops=1)
  Recheck Cond: (passenger_name = 'DENIS YAKOVLEV'::text)
  Heap Blocks: exact=196
  ->  Bitmap Index Scan on pass_name_indx  (cost=0.00..3.90 rows=262 width=0) (actual time=0.643..0.643 rows=196 loops=1)
        Index Cond: (passenger_name = 'DENIS YAKOVLEV'::text)
Planning Time: 2.604 ms
Execution Time: 80.710 ms
*/
--В результате создания индекса на текстовый атрибут чтение таблицы начинается с дочернего узла Bitmap c 
нахождением в индексе адреса строк по указанному фильтру, а затем сам выбор этих строк из таблицы в верхнем узле 
beatmap heap 


--Реализовать индекс на часть таблицы 
drop index fli_id_indx

explain analyze
select tf.ticket_no,
	   tf.flight_id,
	   tf.fare_conditions,
	   tf.amount
from bookings.ticket_flights as tf 
where flight_id > '7683';

/*
 Seq Scan on ticket_flights tf  (cost=0.00..174831.15 rows=7917771 width=32) (actual time=0.016..886.149 rows=7907901 loops=1)
  Filter: (flight_id > 7683)
  Rows Removed by Filter: 483951
Planning Time: 1.239 ms
Execution Time: 1124.106 ms
 */

--До создания индекса, план строится на построчном чтении таблицы по указанному фильтру

create index fli_id_indx on bookings.ticket_flights using btree(flight_id) where flight_id > '7683';

explain analyze
select tf.ticket_no,
	   tf.flight_id,
	   tf.fare_conditions,
	   tf.amount
from bookings.ticket_flights as tf 
where flight_id = '8000';
/*
Index Scan using fli_id_indx on ticket_flights tf  (cost=0.43..149.79 rows=102 width=32) (actual time=0.112..0.113 rows=0 loops=1)
  Index Cond: (flight_id = 8000)
Planning Time: 0.139 ms
Execution Time: 0.124 ms
*/

--При указании в условии выборки строк значения попадающего в диапазон работы индекса чтение происходит по индексу  

explain analyze
select tf.ticket_no,
	   tf.flight_id,
	   tf.fare_conditions,
	   tf.amount
from bookings.ticket_flights as tf 
where flight_id = '7000';

/*
 Gather  (cost=1000.00..114650.76 rows=102 width=32) (actual time=147.752..369.504 rows=131 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on ticket_flights tf  (cost=0.00..113640.56 rows=42 width=32) (actual time=100.034..295.641 rows=44 loops=3)
        Filter: (flight_id = 7000)
        Rows Removed by Filter: 2797240
Planning Time: 0.116 ms
Execution Time: 369.533 ms
 */
--В случае когда чтение искомое значение в условии не попдает в диапазон работы индекса, чтение таблицы происходит построчно в 2 потока

--Реализовать индекс на несклолько атрибутов
drop index pass_name_indx

explain analyze
select bt.ticket_no,
	   bt.book_ref,
	   bt.passenger_id,
	   bt.passenger_name,
	   bt.contact_data	   
from bookings.tickets as bt
where 1=1
and bt.passenger_name = 'DENIS YAKOVLEV'
and bt.ticket_no = '0005433873402';

/*
 * Gather  (cost=1000.00..68851.71 rows=1 width=104) (actual time=367.286..372.649 rows=1 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on tickets bt  (cost=0.00..67851.61 rows=1 width=104) (actual time=230.460..317.930 rows=0 loops=3)
        Filter: ((passenger_name = 'DENIS YAKOVLEV'::text) AND (ticket_no = '0005433873402'::bpchar))
        Rows Removed by Filter: 983285
Planning Time: 0.106 ms
Execution Time: 372.668 ms
 */

create index tick_name_indx on bookings.tickets(passenger_name) include (ticket_no);  

explain analyze
select bt.ticket_no,
	   bt.book_ref,
	   bt.passenger_id,
	   bt.passenger_name,
	   bt.contact_data	   
from bookings.tickets as bt
where 1=1
and bt.passenger_name = 'DENIS YAKOVLEV'
and bt.ticket_no = '0005433873402';

/*
 * Bitmap Heap Scan on tickets bt  (cost=5.40..392.79 rows=1 width=104) (actual time=0.330..0.436 rows=1 loops=1)
  Recheck Cond: (passenger_name = 'DENIS YAKOVLEV'::text)
  Filter: (ticket_no = '0005433873402'::bpchar)
  Rows Removed by Filter: 195
  Heap Blocks: exact=196
  ->  Bitmap Index Scan on tick_name_indx  (cost=0.00..5.39 rows=262 width=0) (actual time=0.189..0.189 rows=196 loops=1)
        Index Cond: (passenger_name = 'DENIS YAKOVLEV'::text)
Planning Time: 2.368 ms
Execution Time: 0.469 ms
*/

--Так же как ив прерыдущем примере с текстовым атрибутом планировщик решил что найти сначал адреса строк по индексу, а затем выбрать эти сами строки по условии быстрее



