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
### autovacuum default 
```
number of transactions actually processed: 2293476
latency average = 12.556 ms
latency stddev = 12.370 ms
tps = 637.072923 (including connections establishing)
tps = 637.073245 (excluding connections establishing)
```

### autovacuum off 
```
number of transactions actually processed: 2706425
latency average = 10.640 ms
latency stddev = 6.597 ms
tps = 751.778424 (including connections establishing)
tps = 751.778985 (excluding connections establishing)
```

По графику тоже видно, что линия выглядит гораздо ближе к трубемой прямой (не считая провалов, приходящихся, по всей вероятности, на контрольные точки раз в 5 минут).

![Графики](diags.png?raw=true "Графики")

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

### Оптимальный варинат настроек (спасибо за подсказку :)
```
autovacuum_naptime = 15s
autovacuum_vacuum_threshold = 25
autovacuum_vacuum_scale_factor = 0.1
autovacuum_max_workers = 10
autovacuum_vacuum_cost_delay = 10
autovacuum_vacuum_cost_limit = 1000
```

#### Производительность
```
number of transactions actually processed: 2629459
latency average = 10.952 ms
latency stddev = 6.868 ms
tps = 730.399766 (including connections establishing)
tps = 730.400323 (excluding connections establishing)
```

### Выкрученный варинат настроек (больше - на значит лучше)
```
autovacuum_naptime = 10s
autovacuum_vacuum_threshold = 20
autovacuum_vacuum_scale_factor = 0.05
autovacuum_max_workers = 10
autovacuum_vacuum_cost_delay = 10
autovacuum_vacuum_cost_limit = 1000
```

#### Производительность (близка к результатам дефолтных настроек)
```
number of transactions actually processed: 2428557
latency average = 11.858 ms
latency stddev = 7.651 ms
tps = 674.594305 (including connections establishing)
tps = 674.594893 (excluding connections establishing)
```

## Измерения за последнюю 1/6 часа:

### autovacuum default 

```
progress: 3060.0 s, 643.5 tps, lat 12.431 ms stddev 10.135
progress: 3120.0 s, 618.2 tps, lat 12.933 ms stddev 11.058
progress: 3180.0 s, 623.2 tps, lat 12.842 ms stddev 10.777
progress: 3240.0 s, 618.1 tps, lat 12.941 ms stddev 11.422
progress: 3300.0 s, 631.5 tps, lat 12.668 ms stddev 11.323
progress: 3360.0 s, 598.8 tps, lat 13.359 ms stddev 10.697
progress: 3420.0 s, 643.8 tps, lat 12.426 ms stddev 9.389
progress: 3480.0 s, 681.8 tps, lat 11.733 ms stddev 9.311
progress: 3540.0 s, 611.9 tps, lat 13.073 ms stddev 10.621
progress: 3600.0 s, 676.1 tps, lat 11.832 ms stddev 9.217
```
### autovacuum off 
```
progress: 3060.0 s, 723.8 tps, lat 11.051 ms stddev 6.919
progress: 3120.0 s, 711.6 tps, lat 11.242 ms stddev 7.510
progress: 3180.0 s, 735.5 tps, lat 10.877 ms stddev 6.828
progress: 3240.0 s, 756.9 tps, lat 10.569 ms stddev 6.360
progress: 3300.0 s, 758.7 tps, lat 10.544 ms stddev 6.299
progress: 3360.0 s, 746.9 tps, lat 10.709 ms stddev 6.331
progress: 3420.0 s, 695.7 tps, lat 11.499 ms stddev 7.593
progress: 3480.0 s, 743.4 tps, lat 10.761 ms stddev 6.770
progress: 3540.0 s, 767.1 tps, lat 10.427 ms stddev 6.285
progress: 3600.0 s, 768.0 tps, lat 10.417 ms stddev 6.272
```
### autovacuum optinal 
```
progress: 3060.0 s, 789.2 tps, lat 10.136 ms stddev 6.033
progress: 3120.0 s, 763.6 tps, lat 10.476 ms stddev 6.629
progress: 3180.0 s, 792.4 tps, lat 10.095 ms stddev 6.052
progress: 3240.0 s, 713.5 tps, lat 11.212 ms stddev 7.380
progress: 3300.0 s, 792.0 tps, lat 10.100 ms stddev 6.033
progress: 3360.0 s, 789.5 tps, lat 10.132 ms stddev 6.116
progress: 3420.0 s, 768.1 tps, lat 10.414 ms stddev 6.496
progress: 3480.0 s, 785.6 tps, lat 10.184 ms stddev 6.089
progress: 3540.0 s, 729.9 tps, lat 10.957 ms stddev 7.059
progress: 3600.0 s, 766.6 tps, lat 10.436 ms stddev 6.505
```
### autovacuum выкрученный
```
progress: 3060.0 s, 740.5 tps, lat 10.803 ms stddev 6.528
progress: 3120.0 s, 600.6 tps, lat 13.314 ms stddev 10.427
progress: 3180.0 s, 645.2 tps, lat 12.403 ms stddev 8.598
progress: 3240.0 s, 698.3 tps, lat 11.456 ms stddev 6.834
progress: 3300.0 s, 693.4 tps, lat 11.536 ms stddev 6.985
progress: 3360.0 s, 691.1 tps, lat 11.576 ms stddev 7.033
progress: 3420.0 s, 685.2 tps, lat 11.673 ms stddev 7.093
progress: 3480.0 s, 551.4 tps, lat 14.508 ms stddev 10.579
progress: 3540.0 s, 693.5 tps, lat 11.534 ms stddev 6.888
progress: 3600.0 s, 688.5 tps, lat 11.618 ms stddev 7.058
```