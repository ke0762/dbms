# 匯入 WHO COVID-19 Open Data
<img width="1432" height="984" alt="螢幕擷取畫面 2026-10-01 225027" src="https://github.com/user-attachments/assets/0baab045-89cb-47e7-8dc9-2c92910f7eee" />
<img width="1210" height="346" alt="螢幕擷取畫面 2026-10-01 225011" src="https://github.com/user-attachments/assets/371decee-a9aa-4d3d-9b13-dcc2ac03276d" />

#SELECT 資料查詢
MariaDB [covid19]> SELECT DISTINCT who_region
    -> FROM who_covid19
    -> ORDER BY who_region;
+------------+
| who_region |
+------------+
| AFR        |
| AMR        |
| EMR        |
| EUR        |
| OTHER      |
| SEAR       |
| WPR        |
+------------+
7 rows in set (0.115 sec)

MariaDB [covid19]> SELECT COUNT(DISTINCT country_code)
    -> FROM who_covid19;
+------------------------------+
| COUNT(DISTINCT country_code) |
+------------------------------+
|                          240 |
+------------------------------+
1 row in set (0.079 sec)
MariaDB [covid19]> SELECT COUNT(DISTINCT country_code) AS num_country
    -> FROM who_covid19;
+-------------+
| num_country |
+-------------+
|         240 |
+-------------+
1 row in set (0.075 sec)
MariaDB [covid19]> SELECT DISTINCT date_reported FROM who_covid19
    -> ORDER BY date_reported DESC
    -> LIMIT 3;
+---------------+
| date_reported |
+---------------+
| 2022-12-31    |
| 2022-12-30    |
| 2022-12-29    |
+---------------+
3 rows in set (0.049 sec)
MariaDB [covid19]> SELECT *
    -> FROM who_covid19
    -> WHERE country_code IN ('JP','US')
    -> and date_reported IN (SELECT MAX(date_reported) FROM who_covid19);
+---------------+--------------+--------------------------+------------+-----------+------------------+------------+-------------------+
| date_reported | country_code | country                  | who_region | new_cases | cumulative_cases | new_deaths | cumulative_deaths |
+---------------+--------------+--------------------------+------------+-----------+------------------+------------+-------------------+
| 2022-12-31    | JP           | Japan                    | WPR        |    148784 |         29105070 |        247 |             57513 |
| 2022-12-31    | US           | United States of America | AMR        |         0 |         99411696 |          0 |           1082456 |
+---------------+--------------+--------------------------+------------+-----------+------------------+------------+-------------------+
2 rows in set (0.098 sec)
MariaDB [covid19]> SELECT date_reported, country_code, new_cases, cumulative_cases FROM who_covid19
    -> WHERE country_code IN ('JP','US')
    -> ORDER BY date_reported DESC, country_code ASC
    -> LIMIT 6;
+---------------+--------------+-----------+------------------+
| date_reported | country_code | new_cases | cumulative_cases |
+---------------+--------------+-----------+------------------+
| 2022-12-31    | JP           |    148784 |         29105070 |
| 2022-12-31    | US           |         0 |         99411696 |
| 2022-12-30    | JP           |    192063 |         28956286 |
| 2022-12-30    | US           |    392203 |         99411696 |
| 2022-12-29    | JP           |         0 |         28764223 |
| 2022-12-29    | US           |         0 |         99019493 |
+---------------+--------------+-----------+------------------+
6 rows in set (0.043 sec)
MariaDB [covid19]> SELECT *
    -> FROM who_covid19
    -> WHERE country_code IN
    ->     (
    ->         SELECT country_code, country, MAX(cumulative_cases) AS max_cumulative_cases
    ->         FROM who_covid19
    ->         GROUP BY country_code
    ->         ORDER BY max_cumulative_cases DESC
    ->         LIMIT 5
    ->     );
ERROR 1235 (42000): This version of MariaDB doesn't yet support 'LIMIT & IN/ALL/ANY/SOME subquery'

#VIEW 檢視表

