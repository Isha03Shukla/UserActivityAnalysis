# Project Title:  User Activity Analysis Using SQL

## Project Overview:
#### This project focuses on analyzing user activity data from two tables, `users_id` and `logins`. The goal is to provide valuable insights into user engagement, activity patterns, and overall usage trends over time. 

## Analytical Questions: 

##### 1. Which users did not log in during the past 5 months?
```
SELECT 
  USER_ID,
  MAX(LOGIN_TIMESTAMP) AS MAX_LOGIN_TIMESTAMP 
  -- ,DATE_SUB(NOW(), INTERVAL 5 MONTH) 
FROM 
  LOGINS 
GROUP BY 
  USER_ID
HAVING 
  MAX(LOGIN_TIMESTAMP) < DATE_SUB(NOW(), INTERVAL 5 MONTH);
```
```
SELECT 
	DISTINCT l.USER_ID, u.USER_NAME
FROM
	LOGINS l
JOIN
	USERS u
ON
	l.USER_ID = u.USER_ID
WHERE
	u.USER_ID NOT IN(
    SELECT
		  USER_ID
	  FROM
		  LOGINS
	  WHERE
		  LOGIN_TIMESTAMP > DATE_SUB(NOW(), INTERVAL 5 MONTH) );
```

##### 2. How many users and sessions were there in each quarter, ordered from newest to oldest?
```
SELECT
	STR_TO_DATE(CONCAT(YEAR(LOGIN_TIMESTAMP), '-', (QUARTER(LOGIN_TIMESTAMP) - 1) * 3 + 1, '-01'), '%Y-%m-%d') AS first_quarter_date,
	-- MIN(LOGIN_TIMESTAMP),
	-- QUARTER(LOGIN_TIMESTAMP) AS QUARTER_LOGIN_TIMESTAMP,
    COUNT(*) AS session_count,
    COUNT(DISTINCT USER_ID) AS user_cnt
FROM
	LOGINS
GROUP BY
	QUARTER(LOGIN_TIMESTAMP),first_quarter_date
ORDER BY
	MIN(LOGIN_TIMESTAMP) desc;
```
##### 3. Which users logged in during January 2024 but did not log in during November 2023?
```
SELECT * FROM LOGINS WHERE LOGIN_TIMESTAMP BETWEEN '2024-01-01' AND '2024-01-31'; -- logged in during January 2024 -- USER_ID - 1, 2, 3, 5
SELECT * FROM LOGINS WHERE LOGIN_TIMESTAMP BETWEEN '2023-11-01' AND '2023-11-30'; -- logged in during November 2023-- USER_ID - 7, 2, 4, 6
-- Main query for Q3
SELECT
	DISTINCT l.USER_ID , u.USER_NAME
FROM
	LOGINS l
JOIN
	USERS u
ON
	l.USER_ID = u.USER_ID
WHERE 
	LOGIN_TIMESTAMP BETWEEN '2024-01-01' AND '2024-01-31'
AND l.USER_ID NOT IN(SELECT USER_ID FROM LOGINS WHERE LOGIN_TIMESTAMP BETWEEN '2023-11-01' AND '2023-11-30');
```
##### 4. What is the percentage change in sessions from the last quarter?
```
WITH CTE AS(SELECT
	STR_TO_DATE(CONCAT(YEAR(LOGIN_TIMESTAMP), '-', (QUARTER(LOGIN_TIMESTAMP) - 1) * 3 + 1, '-01'), '%Y-%m-%d') AS first_quarter_date,
	-- MIN(LOGIN_TIMESTAMP),
	-- QUARTER(LOGIN_TIMESTAMP) AS QUARTER_LOGIN_TIMESTAMP,
    COUNT(*) AS session_count,
    COUNT(DISTINCT USER_ID) AS user_cnt
FROM
	LOGINS
GROUP BY
	QUARTER(LOGIN_TIMESTAMP),first_quarter_date)

SELECT *,
	LAG(session_count,1) OVER(ORDER BY first_quarter_date ) AS prev_session_cnt,
    (session_count - (LAG(session_count,1) OVER(ORDER BY first_quarter_date ))) * 100/(LAG(session_count,1) OVER(ORDER BY first_quarter_date )) AS percentage_change
FROM
	CTE;
```
##### 5. Which user had the highest session score each day?
```
WITH CTE AS(
SELECT USER_ID, convert(LOGIN_TIMESTAMP, date) AS login_date,
SUM(session_score) AS score
FROM LOGINS
GROUP BY USER_ID,  convert(LOGIN_TIMESTAMP, date)
ORDER BY  convert(LOGIN_TIMESTAMP, date), score desc)

SELECT * FROM(SELECT * , ROW_NUMBER() OVER(PARTITION BY login_date ORDER BY score desc) as rnk FROM CTE) a WHERE rnk = 1;
```
##### 6. Which users have had a session every single day since their first login?
```
-- 27-12-2024
SELECT 
	USER_ID,
    MIN(DATE(LOGIN_TIMESTAMP)) AS first_login,
    DATEDIFF(NOW(), MIN(DATE(LOGIN_TIMESTAMP))) + 1 AS no_of_login_days_required,
    COUNT(DISTINCT DATE(LOGIN_TIMESTAMP)) AS no_of_login_days
FROM
	LOGINS
GROUP BY
	USER_ID
HAVING 
	DATEDIFF(NOW(), MIN(DATE(LOGIN_TIMESTAMP))) + 1 = COUNT(DISTINCT DATE(LOGIN_TIMESTAMP)) 
ORDER BY
	USER_ID;
```
##### 7. On what dates were there no logins at all?
```
WITH RECURSIVE CTE AS(
SELECT 
	MIN(DATE(LOGIN_TIMESTAMP)) AS first_login_date,
    DATE(NOW()) AS last_date
	-- ,datediff(DATE(NOW()),MIN(DATE(LOGIN_TIMESTAMP))) + 1 AS TOTAL_NO_OF_DAYS -- 532 DAYS 
FROM
	LOGINS
UNION ALL
SELECT 
	DATE_ADD(first_login_date, INTERVAL 1 DAY),
    last_date
FROM
	CTE
WHERE
	first_login_date<last_date)
SELECT * FROM CTE WHERE first_login_date NOT IN(SELECT DISTINCT DATE(LOGIN_TIMESTAMP) FROM LOGINS) -- 34DAYS ACTIVE DATES
-- 532 - 34 = 498 DAYS
```
