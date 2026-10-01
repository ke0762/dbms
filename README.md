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
````[cite: 2, 3]

# COVID-19 資料查詢與練習紀錄

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
```
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
`因為資料太多筆所以很難跑完然後會一直有不同資料跳出來`

#VIEW 檢視表

