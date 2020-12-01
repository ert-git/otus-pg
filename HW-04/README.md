# Домашнее задание

Настройка autovacuum с учетом оптимальной производительности
# Цель: 
* запустить нагрузочный тест pgbench с профилем нагрузки DWH
* настроить параметры autovacuum для достижения максимального уровня устойчивой производительности

# Шаги
* создать GCE инстанс типа e2-medium и standard disk 10GB
* установить на него PostgreSQL 13 с дефолтными настройками
* применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла
* выполнить pgbench -i postgres
* запустить pgbench -c8 -P 60 -T 3600 -U postgres postgres
* дать отработать до конца
* зафиксировать среднее значение tps в последней ⅙ части работы
* а дальше настроить autovacuum максимально эффективно
* так чтобы получить максимально ровное значение tps на горизонте часа

Автовакуум потребляет системные ресурсы, что может быть заметно при большой нагрузке на сервер.
Поэтому логично предположить, что если совсем выключить автоочистку, то получим (почти) идеальный результат.   

По метрикам так оно и есть:
```
autovacuum default 

number of transactions actually processed: 2293476
latency average = 12.556 ms
latency stddev = 12.370 ms
tps = 637.072923 (including connections establishing)
tps = 637.073245 (excluding connections establishing)
```
```
autovacuum off 

number of transactions actually processed: 2706425
latency average = 10.640 ms
latency stddev = 6.597 ms
tps = 751.778424 (including connections establishing)
tps = 751.778985 (excluding connections establishing)
```

По графику тоже видно, что линия выглядит гораздо ближе к трубемой прямой (не считая провалов, приходящихся на контрольные точки раз в 5 минут, но они отношения к делу не имеют).

![Графики](diag.png?raw=true "Графики")

Чтобы выровнять линию при включенном автовакууме, необходимо максимально "размазать" его по времени. Т.е. сделать так, чтобы очистка обрабатывала за раз поменьше dead tuples. С другой стороны, слишком "выкрученные" параметры тоже не приводят к хорошему результату.  
Поэтому меняем сразу несколько параметров, ориентируясь на bset practices. 

* autovacuum_max_workers - число одновременных сессий очистки, по сути, число таблиц, если я правильно понимаю. 
* autovacuum_vacuum_cost_limit - некоторый коэффициент, связанный с autovacuum_max_workers, который показывает, что пора начать чистку. Чем он больше, тем вероятнее, что чистка запустится. Документация велит увеличивать его при увеличении autovacuum_max_workers
* autovacuum_vacuum_cost_delay - интервал расчета веса. Чем меньше, тем чаще проверяем, не пора ли почиститься.
* autovacuum_vacuum_threshold - минимальное число кортежей, после изменения которых надо начинать чистку. Чем меньше, тем чаще.
* autovacuum_vacuum_scale_factor - процент измененных кортежей от общего числа кортежей в таблице, по достижении которого надо чиститься. Здесь, пожалуй, пространство для маневра. Если у нас разнокалиберные таблицы, то его имеет смысл выставлять для каждой таблицы отдельно. Например,
```
alter table <table_name> set (autovacuum_vacuum_scale_factor = 0.01)
```
Но в данной работе такое исследование не проводилось.
* autovacuum_naptime - как часло будет просыпаться автоочистка. Чем чаще, тем меньшее число записей за раз надо будет удалять.

```
Оптимальный варинат настроек (спасибо за подсказку :)
autovacuum_naptime = 15s
autovacuum_vacuum_threshold = 25
autovacuum_vacuum_scale_factor = 0.1
autovacuum_max_workers = 10
autovacuum_vacuum_cost_delay = 10
autovacuum_vacuum_cost_limit = 1000
```

Производительность
```
number of transactions actually processed: 2629459
latency average = 10.952 ms
latency stddev = 6.868 ms
tps = 730.399766 (including connections establishing)
tps = 730.400323 (excluding connections establishing)
```

```
Выкрученный варинат настроек (больше - на значит лучше)
autovacuum_naptime = 10s
autovacuum_vacuum_threshold = 20
autovacuum_vacuum_scale_factor = 0.05
autovacuum_max_workers = 10
autovacuum_vacuum_cost_delay = 10
autovacuum_vacuum_cost_limit = 1000
```

Производительность (близка к результатам дефолтных настроек)
```
number of transactions actually processed: 2428557
latency average = 11.858 ms
latency stddev = 7.651 ms
tps = 674.594305 (including connections establishing)
tps = 674.594893 (excluding connections establishing)
```


