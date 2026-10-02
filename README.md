# 匯入 WHO COVID-19 Open Data

```text
MariaDB [covid19]> LOAD DATA LOCAL INFILE 'C:/Users/USER/Downloads/WHO-COVID-19-global-daily-data-2022.csv'
    -> INTO TABLE who_covid19
    -> CHARACTER SET utf8mb4
    -> FIELDS TERMINATED BY ','
    -> OPTIONALLY ENCLOSED BY '"'
    -> LINES TERMINATED BY '\r\n'
    -> IGNORE 1 LINES;
Query OK, 262320 rows affected, 65535 warnings (0.661 sec)
Records: 262320  Deleted: 0  Skipped: 0  Warnings: 224726

MariaDB [(none)]> USE covid19;
Database changed
MariaDB [covid19]> CREATE TABLE who_covid19 (
    ->   date_reported date NOT NULL,
    ->   country_code char(3) NOT NULL,
    ->   country varchar(80) NOT NULL,
    ->   who_region varchar(5) NOT NULL,
    ->   new_cases int,
    ->   cumulative_cases int,
    ->   new_deaths int,
    ->   cumulative_deaths int
    -> );
Query OK, 0 rows affected (0.014 sec)

MariaDB [covid19]> DESCRIBE who_covid19;
+-------------------+-------------+------+-----+---------+-------+
| Field             | Type        | Null | Key | Default | Extra |
+-------------------+-------------+------+-----+---------+-------+
| date_reported     | date        | NO   |     | NULL    |       |
| country_code      | char(3)     | NO   |     | NULL    |       |
| country           | varchar(80) | NO   |     | NULL    |       |
| who_region        | varchar(5)  | NO   |     | NULL    |       |
| new_cases         | int(11)     | YES  |     | NULL    |       |
| cumulative_cases  | int(11)     | YES  |     | NULL    |       |
| new_deaths        | int(11)     | YES  |     | NULL    |       |
| cumulative_deaths | int(11)     | YES  |     | NULL    |       |
+-------------------+-------------+------+-----+---------+-------+
8 rows in set (0.022 sec)

MariaDB [covid19]> SELECT count(*) FROM who_covid19;
+----------+
| count(*) |
+----------+
|   262320 |
+----------+
1 row in set (0.035 sec)
```

---

# SELECT 資料查詢

## 1. 查詢所有的 WHO 區域別 (who_region)
```sql
SELECT DISTINCT who_region 
FROM who_covid19 
ORDER BY who_region;
```
**執行結果：**
```text
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
```

---

## 2. 統計共有多少個國家/地區代碼 (country_code)
```sql
SELECT COUNT(DISTINCT country_code) AS num_country 
FROM who_covid19;
```
**執行結果：**
```text
+-------------+
| num_country |
+-------------+
|         240 |
+-------------+
1 row in set (0.075 sec)
```

---

## 3. 查詢最新公佈日期的前 3 天
```sql
SELECT DISTINCT date_reported 
FROM who_covid19 
ORDER BY date_reported DESC 
LIMIT 3;
```
**執行結果：**
```text
+---------------+
| date_reported |
+---------------+
| 2022-12-31    |
| 2022-12-30    |
| 2022-12-29    |
+---------------+
3 rows in set (0.049 sec)
```

---

## 4. 查詢美、日兩國在最新日期的所有疫情數據 (使用子查詢)
```sql
SELECT * 
FROM who_covid19 
WHERE country_code IN ('JP', 'US') 
  AND date_reported IN (
      SELECT MAX(date_reported) 
      FROM who_covid19
  );
```
**執行結果：**
```text
+---------------+--------------+--------------------------+------------+-----------+------------------+------------+-------------------+
| date_reported | country_code | country                  | who_region | new_cases | cumulative_cases | new_deaths | cumulative_deaths |
+---------------+--------------+--------------------------+------------+-----------+------------------+------------+-------------------+
| 2022-12-31    | JP           | Japan                    | WPR        |    148784 |         29105070 |        247 |             57513 |
| 2022-12-31    | US           | United States of America | AMR        |         0 |         99411696 |          0 |           1082456 |
+---------------+--------------+--------------------------+------------+-----------+------------------+------------+-------------------+
2 rows in set (0.098 sec)
```

---

## 5. 查詢美、日兩國近 3 天的確診數據明細
```sql
SELECT date_reported, country_code, new_cases, cumulative_cases 
FROM who_covid19 
WHERE country_code IN ('JP', 'US') 
ORDER BY date_reported DESC, country_code ASC 
LIMIT 6;
```
**執行結果：**
```text
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
```

---

## 6. 查詢累計確診前 5 大國家的所有歷史資料 (排除 ERROR 1235 限制)

```text
MariaDB [covid19]> SELECT * 
    -> FROM who_covid19 
    -> WHERE country_code IN 
    -> ( 
    -> SELECT country_code, country, MAX(cumulative_cases) AS max_cumulative_cases 
    -> FROM who_covid19 
    -> GROUP BY country_code 
    -> ORDER BY max_cumulative_cases DESC 
    -> LIMIT 5 
    -> ); 
ERROR 1235 (42000): This version of MariaDB doesn't yet support 'LIMIT & IN/ALL/ANY/SOME subquery'
```

> **錯誤紀錄：**
> 直接在 `IN (...)` 子查詢內使用 `LIMIT` 會觸發 MariaDB 限制：  
> `ERROR 1235 (42000): This version of MariaDB doesn't yet support 'LIMIT & IN/ALL/ANY/SOME subquery'`

### 修正寫法（使用 INNER JOIN 衍生資料表）：
```sql
SELECT w.*
FROM who_covid19 w
JOIN (
    SELECT country_code
    FROM who_covid19
    GROUP BY country_code
    ORDER BY MAX(cumulative_cases) DESC
    LIMIT 5
) top5 ON w.country_code = top5.country_code
ORDER BY w.country_code, w.date_reported;
```

**嘗試紀錄：**
因為資料太多筆所以很難跑完，然後會一直有不同資料跳出來。

---

# VIEW 檢視表
## 1. CREATE VIEW ... AS SELECT ... FROM ..
```text
MariaDB [covid19]> CREATE VIEW vw_country AS
    ->      SELECT DISTINCT country_code, country
    ->      FROM who_covid19
    ->      ORDER BY country_code;
Query OK, 0 rows affected (0.011 sec)

MariaDB [covid19]> SELECT *
    -> FROM vw_country;
+--------------+------------------------------------------------------------------------+
| country_code | country                                                                |
+--------------+------------------------------------------------------------------------+
| AD           | Andorra                                                                |
| AE           | United Arab Emirates                                                   |
| AF           | Afghanistan                                                            |
| AG           | Antigua and Barbuda                                                    |
| AI           | Anguilla                                                               |
| AL           | Albania                                                                |
...
| XI           | International conveyance (Kiribati)                                    |
| XJ           | International conveyance (Diamond Princess)                            |
| XK           | Kosovo (in accordance with UN Security Council Resolution 1244 (1999)) |
| XL           | International commercial vessel                                        |
| YE           | Yemen                                                                  |
| YT           | Mayotte                                                                |
| ZA           | South Africa                                                           |
| ZM           | Zambia                                                                 |
| ZW           | Zimbabwe                                                               |
+--------------+------------------------------------------------------------------------+
240 rows in set (0.277 sec)
```
## 2. CREATE VIEW ... AS SELECT ... FROM ... LIMIT ...
```
MariaDB [covid19]> CREATE VIEW vw_country_top5 AS
    ->     SELECT country_code, MAX(cumulative_cases) AS max_cumulative_cases
    ->     FROM who_covid19
    ->     GROUP BY country_code
    ->     ORDER BY max_cumulative_cases DESC
    ->     LIMIT 5;
ERROR 2006 (HY000): Server has gone away
No connection. Trying to reconnect...
Connection id:    5
Current database: covid19

Query OK, 0 rows affected (0.028 sec)

MariaDB [covid19]> SELECT *
    -> FROM vw_country_top5;
+--------------+----------------------+
| country_code | max_cumulative_cases |
+--------------+----------------------+
| US           |             99411696 |
| CN           |             84925042 |
| IN           |             44678384 |
| FR           |             37989547 |
| DE           |             37241936 |
+--------------+----------------------+
5 rows in set (0.175 sec)
```
## 3.SELECT ... FROM ... WHERE ... IN (SELECT view FROM ... )
```
-- 選取所有欄位
MariaDB [covid19]> SELECT *
    -> FROM who_covid19
    -> WHERE country_code IN
    ->     (SELECT country_code FROM vw_country_top5);
| 2022-12-30    | US           | United States of America | AMR        |    392203 |         99411696 |       2480 |           1082456 |
| 2022-12-30    | FR           | France                   | EUR        |         0 |         37989547 |          0 |            161667 |
| 2022-12-30    | IN           | India                    | SEAR       |       243 |         44678158 |          1 |            530699 |
| 2022-12-30    | DE           | Germany                  | EUR        |         0 |         37241936 |          0 |            165738 |
| 2022-12-30    | CN           | China                    | WPR        |   2929223 |         82495770 |       2850 |             48738 |
| 2022-12-31    | CN           | China                    | WPR        |   2429272 |         84925042 |       3806 |             52544 |
| 2022-12-31    | FR           | France                   | EUR        |         0 |         37989547 |          0 |            161667 |
| 2022-12-31    | DE           | Germany                  | EUR        |         0 |         37241936 |          0 |            165738 |
| 2022-12-31    | IN           | India                    | SEAR       |       226 |         44678384 |          3 |            530702 |
| 2022-12-31    | US           | United States of America | AMR        |         0 |         99411696 |          0 |           1082456 |
+---------------+--------------+--------------------------+------------+-----------+------------------+------------+-------------------+
10930 rows in set (0.361 sec)

-- 僅列出四個欄位
MariaDB [covid19]> SELECT date_reported, country_code, new_cases, cumulative_cases
    -> FROM who_covid19
    -> WHERE country_code IN
    ->     (SELECT country_code FROM vw_country_top5);
| 2022-12-29    | DE           |         0 |         37241936 |
| 2022-12-29    | IN           |       268 |         44677915 |
| 2022-12-29    | US           |         0 |         99019493 |
| 2022-12-30    | US           |    392203 |         99411696 |
| 2022-12-30    | FR           |         0 |         37989547 |
| 2022-12-30    | IN           |       243 |         44678158 |
| 2022-12-30    | DE           |         0 |         37241936 |
| 2022-12-30    | CN           |   2929223 |         82495770 |
| 2022-12-31    | CN           |   2429272 |         84925042 |
| 2022-12-31    | FR           |         0 |         37989547 |
| 2022-12-31    | DE           |         0 |         37241936 |
| 2022-12-31    | IN           |       226 |         44678384 |
| 2022-12-31    | US           |         0 |         99411696 |
+---------------+--------------+-----------+------------------+
10930 rows in set (0.333 sec)
```
## 4.CREATE VIEW ... AS SELECT ... FROM ... WHERE ... IN (SELECT view FROM ... )
```
MariaDB [covid19]> CREATE VIEW vw_covid19_top5country AS
    ->     SELECT date_reported, country_code, new_cases, cumulative_cases
    ->     FROM who_covid19
    ->     WHERE country_code IN
    ->         (SELECT country_code FROM vw_country_top5);

MariaDB [covid19]> SELECT *
    ->    FROM vw_covid19_top5country ;
...
| 2022-12-30    | CN           |   2929223 |         82495770 |
| 2022-12-31    | CN           |   2429272 |         84925042 |
| 2022-12-31    | FR           |         0 |         37989547 |
| 2022-12-31    | DE           |         0 |         37241936 |
| 2022-12-31    | IN           |       226 |         44678384 |
| 2022-12-31    | US           |         0 |         99411696 |
+---------------+--------------+-----------+------------------+
10930 rows in set (0.334 sec)

===查詢「各個國家各自」的資料筆數
MariaDB [covid19]> SELECT country_code, COUNT(*) AS record_count
    -> FROM vw_covid19_top5country
    -> GROUP BY country_code;
+--------------+--------------+
| country_code | record_count |
+--------------+--------------+
| CN           |         2186 |
| DE           |         2186 |
| FR           |         2186 |
| IN           |         2186 |
| US           |         2186 |
+--------------+--------------+
5 rows in set (0.329 sec)
```
# JOIN 連接查詢 整合應用
## 1.利用 JOIN 連結 vw_country_top5, vw_country 兩個檢視表
```text
MariaDB [covid19]> SELECT A.*, B.country
    -> FROM vw_country_top5 A
    -> JOIN vw_country B
    -> USING (country_code);
+--------------+----------------------+--------------------------+
| country_code | max_cumulative_cases | country                  |
+--------------+----------------------+--------------------------+
| US           |             99411696 | United States of America |
| CN           |             84925042 | China                    |
| IN           |             44678384 | India                    |
| FR           |             37989547 | France                   |
| DE           |             37241936 | Germany                  |
+--------------+----------------------+--------------------------+
5 rows in set (0.451 sec)
```
## 2. 利用 JOIN 連結 vw_covid19_top5country, vw_country 兩個檢視表
```text
MariaDB [covid19]> SELECT A.*, B.country
    -> FROM vw_covid19_top5country A
    -> JOIN vw_country B
    -> USING (country_code)
    -> WHERE YEAR(date_reported) = 2021
    ->   AND MONTH(date_reported) = 12;
+---------------+--------------+-----------+------------------+--------------------------+
| date_reported | country_code | new_cases | cumulative_cases | country                  |
+---------------+--------------+-----------+------------------+--------------------------+
| 2021-12-01    | US           |     93479 |         48178162 | United States of America |
| 2021-12-01    | FR           |         0 |          7203450 | France                   |
| 2021-12-01    | IN           |      8954 |         34596776 | India                    |
| 2021-12-01    | DE           |         0 |          5820646 | Germany                  |
| 2021-12-01    | CN           |        84 |           128022 | China                    |
| 2021-12-02    | CN           |       119 |           128141 | China                    |
...
| 2021-12-30    | US           |    389514 |         53059977 | United States of America |
| 2021-12-31    | US           |    474309 |         53534286 | United States of America |
| 2021-12-31    | FR           |         0 |          8709926 | France                   |
| 2021-12-31    | IN           |     16764 |         34838804 | India                    |
| 2021-12-31    | DE           |         0 |          7014043 | Germany                  |
| 2021-12-31    | CN           |       291 |           132071 | China                    |
+---------------+--------------+-----------+------------------+--------------------------+
310 rows in set (0.562 sec)
```
## 3. 利用 JOIN 連結兩個 vw_covid19_top5country 相同的檢視表
```
MariaDB [covid19]> SELECT A.*, B.new_cases AS new_cases_nextday
    -> FROM vw_covid19_top5country A
    -> JOIN vw_covid19_top5country B
    -> USING (country_code)
    -> WHERE DATE_ADD(A.date_reported, INTERVAL 1 DAY) = B.date_reported
    ->   AND YEAR(A.date_reported) = 2021
    ->   AND MONTH(A.date_reported) = 12;
+---------------+--------------+-----------+------------------+-------------------+
| date_reported | country_code | new_cases | cumulative_cases | new_cases_nextday |
+---------------+--------------+-----------+------------------+-------------------+
| 2021-12-01    | CN           |        84 |           128022 |               119 |
| 2021-12-01    | CN           |        84 |           128022 |               119 |
| 2021-12-01    | FR           |         0 |          7203450 |                 0 |
| 2021-12-01    | FR           |         0 |          7203450 |                 0 |
| 2021-12-01    | DE           |         0 |          5820646 |                 0 |
...
| 2021-12-31    | DE           |         0 |          7014043 |                 0 |
| 2021-12-31    | IN           |     16764 |         34838804 |             22775 |
| 2021-12-31    | IN           |     16764 |         34838804 |             22775 |
| 2021-12-31    | US           |    474309 |         53534286 |            584647 |
| 2021-12-31    | US           |    474309 |         53534286 |            584647 |
+---------------+--------------+-----------+------------------+-------------------+
620 rows in set (14.119 sec)
```
# 綜合練習
## 1.  SELECT ... FROM ... GROUP BY ... ORDER BY ...
```
MariaDB [covid19]> SELECT country_code, country, MAX(new_cases) as max_new_cases
    -> FROM who_covid19
    -> GROUP BY country_code
    -> ORDER BY max_new_cases DESC;
+--------------+------------------------------------------------------------------------+---------------+
| country_code | country                                                                | max_new_cases |
+--------------+------------------------------------------------------------------------+---------------+
| CN           | China                                                                  |       6966046 |
| FR           | France                                                                 |       2417043 |
| DE           | Germany                                                                |       1588891 |
| US           | United States of America                                               |       1265520 |
| ES           | Spain                                                                  |        956506 |
...
| XF           | International conveyance (American Samoa)                              |             3 |
| XG           | International conveyance (Solomon Islands)                             |             2 |
| XI           | International conveyance (Kiribati)                                    |             1 |
| TM           | Turkmenistan                                                           |             0 |
| KP           | Democratic People's Republic of Korea                                  |             0 |
+--------------+------------------------------------------------------------------------+---------------+
240 rows in set (0.199 sec)
```
## 2. CREATE VIEW ... AS SELECT ... FROM ... GROUP BY ...
```
MariaDB [covid19]> CREATE VIEW vw_max_new_cases AS
    ->     SELECT country_code, MAX(new_cases) as max_new_cases
    ->     FROM who_covid19
    ->     GROUP BY country_code;
Query OK, 0 rows affected (0.010 sec)

MariaDB [covid19]> SELECT *
    -> FROM vw_max_new_cases;
+--------------+---------------+
| country_code | max_new_cases |
+--------------+---------------+
| AD           |          1676 |
| AE           |          3977 |
| AF           |          3243 |
| AG           |           468 |
| AI           |           196 |
...
| YT           |          6040 |
| ZA           |         37875 |
| ZM           |          5555 |
| ZW           |          6181 |
+--------------+---------------+
240 rows in set (0.174 sec)
```
## 3.SELECT ... FROM ... A, ... B WHERE ... ORDER BY ...
```
MariaDB [covid19]> SELECT A.date_reported, A.country_code, A.new_cases, B.max_new_cases,
    ->     (A.new_cases / B.max_new_cases) AS severity_rate
    -> FROM vw_covid19_top5country A, vw_max_new_cases B
    -> WHERE A.country_code = B.country_code
    ->   AND YEAR(date_reported) = 2021
    ->   AND MONTH(date_reported) = 12
    -> ORDER BY date_reported, country_code;
+---------------+--------------+-----------+---------------+---------------+
| date_reported | country_code | new_cases | max_new_cases | severity_rate |
+---------------+--------------+-----------+---------------+---------------+
| 2021-12-01    | CN           |        84 |       6966046 |        0.0000 |
| 2021-12-01    | CN           |        84 |       6966046 |        0.0000 |
| 2021-12-01    | DE           |         0 |       1588891 |        0.0000 |
| 2021-12-01    | DE           |         0 |       1588891 |        0.0000 |
| 2021-12-01    | FR           |         0 |       2417043 |        0.0000 |
| 2021-12-01    | FR           |         0 |       2417043 |        0.0000 |
...
| 2021-12-31    | FR           |         0 |       2417043 |        0.0000 |
| 2021-12-31    | FR           |         0 |       2417043 |        0.0000 |
| 2021-12-31    | IN           |     16764 |        414188 |        0.0405 |
| 2021-12-31    | IN           |     16764 |        414188 |        0.0405 |
| 2021-12-31    | US           |    474309 |       1265520 |        0.3748 |
| 2021-12-31    | US           |    474309 |       1265520 |        0.3748 |
+---------------+--------------+-----------+---------------+---------------+
310 rows in set (0.455 sec)
```
## 4.SELECT ..., (CASE WHEN ... THEN ... ELSE ... END) AS ... FROM ... WHERE ... ORDER BY ...
```
MariaDB [covid19]> SELECT A.date_reported, A.country_code, A.new_cases, B.max_new_cases,
    ->     (A.new_cases / B.max_new_cases) AS severity_rate,
    ->     CASE
    ->         WHEN (A.new_cases / B.max_new_cases >= 0.1) THEN "H"
    ->         ELSE "L"
    ->     END AS severity_level
    -> FROM vw_covid19_top5country A, vw_max_new_cases B
    -> WHERE A.country_code = B.country_code
    ->   AND YEAR(date_reported) = 2021
    ->   AND MONTH(date_reported) = 12
    -> ORDER BY date_reported, country_code;
+---------------+--------------+-----------+---------------+---------------+----------------+
| date_reported | country_code | new_cases | max_new_cases | severity_rate | severity_level |
+---------------+--------------+-----------+---------------+---------------+----------------+
| 2021-12-01    | CN           |        84 |       6966046 |        0.0000 | L              |
| 2021-12-01    | CN           |        84 |       6966046 |        0.0000 | L              |
| 2021-12-01    | DE           |         0 |       1588891 |        0.0000 | L              |
| 2021-12-01    | DE           |         0 |       1588891 |        0.0000 | L              |
...
| 2021-12-31    | FR           |         0 |       2417043 |        0.0000 | L              |
| 2021-12-31    | IN           |     16764 |        414188 |        0.0405 | L              |
| 2021-12-31    | IN           |     16764 |        414188 |        0.0405 | L              |
| 2021-12-31    | US           |    474309 |       1265520 |        0.3748 | H              |
| 2021-12-31    | US           |    474309 |       1265520 |        0.3748 | H              |
+---------------+--------------+-----------+---------------+---------------+----------------+
310 rows in set (0.457 sec)
```

## 5.SELECT ..., CASE WHEN ... THEN ... ELSE (CASE ... END) END AS ... FROM ... WHERE ... ORDER BY ...
```
MariaDB [covid19]> SELECT A.date_reported, A.country_code, A.new_cases, B.max_new_cases,
    ->     (A.new_cases / B.max_new_cases) AS severity_rate,
    ->     CASE
    ->         WHEN (A.new_cases / B.max_new_cases >= 0.1) THEN "H"
    ->         WHEN (A.new_cases / B.max_new_cases <= 0.01) THEN "L"
    ->         ELSE "M"
    ->     END AS severity_level
    -> FROM vw_covid19_top5country A, vw_max_new_cases B
    -> WHERE A.country_code = B.country_code
    ->   AND YEAR(date_reported) = 2021
    ->   AND MONTH(date_reported) = 12
    -> ORDER BY date_reported, country_code;
+---------------+--------------+-----------+---------------+---------------+----------------+
| date_reported | country_code | new_cases | max_new_cases | severity_rate | severity_level |
+---------------+--------------+-----------+---------------+---------------+----------------+
| 2021-12-01    | CN           |        84 |       6966046 |        0.0000 | L              |
| 2021-12-01    | CN           |        84 |       6966046 |        0.0000 | L              |
| 2021-12-01    | DE           |         0 |       1588891 |        0.0000 | L              |
| 2021-12-01    | DE           |         0 |       1588891 |        0.0000 | L              |

| 2021-12-31    | FR           |         0 |       2417043 |        0.0000 | L              |
| 2021-12-31    | IN           |     16764 |        414188 |        0.0405 | M              |
| 2021-12-31    | IN           |     16764 |        414188 |        0.0405 | M              |
| 2021-12-31    | US           |    474309 |       1265520 |        0.3748 | H              |
| 2021-12-31    | US           |    474309 |       1265520 |        0.3748 | H              |
+---------------+--------------+-----------+---------------+---------------+----------------+
310 rows in set (0.454 sec)
```
