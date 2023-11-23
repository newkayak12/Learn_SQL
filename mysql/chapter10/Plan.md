## 실행 계획 확인
이전 버전에는 `EXPLAIN EXTENDED`, `EXPLAIN PARTITION`이 구분돼 있었지만 8.0부터는 통합됐다. 
```mysql
EXPLAIN ANALYZE
SELECT e.emp_no, avg(s.salary)
FROM employees e 
JOIN salaries s on e.emp_no = s.emp_no
    AND s.salary > 5000
    AND s.from_date <= '1990-01-01'
WHERE e.first_name = "Matt"
GROUP BY e.hire_date;


  -> Table scan on <temporary>  (actual time=511..511 rows=71 loops=1)
    -> Aggregate using temporary table  (actual time=511..511 rows=71 loops=1)
        -> Nested loop inner join  (cost=709 rows=365) (actual time=99.7..509 rows=198 loops=1)
            -> Index lookup on e using ix_firstname (first_name='Matt')  (cost=255 rows=233) (actual time=97.3..248 rows=233 loops=1)
            -> Filter: ((s.salary > 5000) and (s.from_date <= DATE'1990-01-01'))  (cost=1.01 rows=1.57) (actual time=1.09..1.12 rows=0.85 loops=233)
                -> Index lookup on s using PRIMARY (emp_no=e.emp_no)  (cost=1.01 rows=9.4) (actual time=1.08..1.11 rows=9.53 loops=233)
 |

```