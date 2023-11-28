# 쿼리 작성 및 최적화

## 시스템 변수
### SQL mode
MySQL은 sql_mode는 여러 개틔 값이 동시에 설정될 수 있다. 

- STRICT_ALL_TABLES & STRICT_TRANS_TABLES : UPDATE에 지정한 값의 타입과 컬럼 타입이 다르면 타입 캐스트를 자동으로 진행한다. 만일 원만하게 진행하기 어려우면 (길이가 부족하다거나 ) 에러를 낸다. 이 설정을 켜면 이를 방지한다. 
- ANSI_QUOTES : 홑따옴표만 문자열 값 표기로 사용할 수 있고 쌍따옴표는 컬럼명이나 테이블명과 같은 식별자를 표기하는 데만 사용하게 한다. 
- ONLY_FULL_GROUP_BY : GROUP BY에 포함되지 않은 컬럼이라도 집합 함수 사용 없이 `SELECT`, `HAVING`에 사용할 수 있게 한다. 
- PIPE_AS_CONCAT : 이 옵션을 설정하면 ||를 오라클 같이 CONCAT으로 사용할 수 있다. 
- PAD_CHAR_TO_FULL_LENGTH : MySQL은 CHAR라도 VARCHAR 같이 유효 문자열 뒤 공백을 제거해서 반환한다. 이 옵션을 설정하면 이를 방지한다.
- NO_BACKSLASH_ESCAPE : 설정하면 `\`를 이스케이프 문자로 사용할 수 있게 한다. 
- IGNORE_SPACE : 프로시저나 함수명과 괄호 사이 공백을 무시하는 옵션을 설정할 수 있다. 또한 이 옵션이 활성화되면 서버의 내장 함수는 모두 예약어로 간주되어 테이블이나 컬럼 이름으로 사용될 수 없다. 
- REAL_AS_FLOAT : 부동 소수점 타입은 FLOAT, DOUBLE이 지원되는데 REAL은 DOUBLE과 동의어다. REAL_AS_FLOAT이 설정되면 REAL이 FLOAT이 된다.
- NO_ZERO_IN_DATE & NO_ZERO_DATE : 두 옵션이 활성화되면 DATE, DATETIME에 '2023-00-00'과 같은 잘못된 날자를 저장하지 못하게 한다. 
- ANSI : SQL 표준에 맞게 동작하게 한다. "READ_AS_FLOAT", "PIPES_AS_CONCAT", "ANSI_QUOTES", "IGNORE_SPACE", "ONLY_FULL_GROUP_BY" 모드가 조합된 모드
- TRADITIONAL : STRICT_TRANS_TABLES, STRICT_ALL_TABLES와 비슷하지만 더 엄격하게 작동한다. "STRICT_TRANS_TABLES", "STRICT_ALL_TABLES", "NO_ZERO_IN_DATE", "NO_ZERO_DATE", "ERROR_FOR_DIVISION_BY_ZERO", "NO_ENGINE_SUBSTITUTION"
이 조합된 모드다. 

### 영문 대소문자 구분
윈도우는 기본 대소 구분이 없다. 리눅스는 구분한다. 만약 대소문자 영향을 받지 않게 하려면 `lower_case_tables_names`를 설정하면 된다. 

### MySQL 예약어
생성하는 테이블, 컬럼 이름을 예약어 같은 키워드로 생성하려면 " ` "으로 감싸면 사용할 수 있다. 

### 불리언
BOOL, BOOLEAN이 있지만 TINYINT와 동의어다. 

### 동등 비교 ( =, <=>)
동등 비교를 수행한다. `<=>`는 NULL 값에 대한 비교까지 수행한다. 

### IN
```sql
SELECT *
FROM dept_emp
WHERE (dept_no, emp_no) IN (('d001', 10017), ('d002', 10144), ('d003', 10054));
```
8.0은 튜플을 나열해도 인덱스를 최적으로 찾을 수 있게 개선했다. 


### NOW(), SYSDATE()
```sql
mysql> SELECT NOW(), SLEEP(2), NOW();
+---------------------+----------+---------------------+
| NOW()               | SLEEP(2) | NOW()               |
+---------------------+----------+---------------------+
| 2023-11-23 11:21:43 |        0 | 2023-11-23 11:21:43 |
+---------------------+----------+---------------------+
1 row in set (2.01 sec)
    
mysql> SELECT SYSDATE(), SLEEP(2), SYSDATE();
+---------------------+----------+---------------------+
| SYSDATE()           | SLEEP(2) | SYSDATE()           |
+---------------------+----------+---------------------+
| 2023-11-23 11:22:08 |        0 | 2023-11-23 11:22:10 |
+---------------------+----------+---------------------+
1 row in set (2.00 sec)

```
`SYSDATE()`는 1) 함수가 사용된 SQL은 레플리카 서버에서 안정적으로 복제되지 못한다. 2) 함수와 비교되는 컬럼은 인덱스를 효율적으로 사용하지 못한다. 

### GROUP_CONCAT()
`GROUP_CONCAT()`이 사용하는 메모리 버퍼는 `group_concat_max_len`으로 조정할 수 있다. 

### CASE WHEN ... THEN ... END
```sql
SELECT emp_no, first_name
      CASE gender WHEN 'M' THEN 'Man'
                  WHEN 'F' THEN 'Woman'
                  ELSE 'Unknown' END as gender
FROM employees LIMIT 10;
```

### WHERE 절 인덱스 사용
WHERE에 인덱스 사용 방법은 범위 결정 조건과 체크 조건 두 가지로 구분된다. 두 방식 중 작업 범위 결정 조건은, WHERE 절에ㅔ서 동등 비교 조건이나 IN으로 
구성된 조건에 사용된 칼럼들이 인덱스의 컬럼 구성과 조측에서부터 비교했을 떄 얼마나 일치하는가에 따라 달라진다. 

만일 OR이 있으면 처리 방법이 바뀐다. AND이면 인덱스를 사용하겠지만 OR를 사용하면 풀 스캔을 선택할 수밖에 없다. 

### GROUP BY 절의 인덱스 사용
GROUP BY는 명시된 컬럼의 순서가 인덱스를 구성하는 컬름의 순서와 같으면 GROUP BY는 인덱스를 이용할 수 있다.

### ORDER BY 절의 인덱스 사용
`GROUP BY`, `ORDER BY`는 처리 방법이 비슷하다. ORDER BY는 조건이 하나 더 있는데 정렬되는 각 컬럼의 오름차순, 내림차순 옵션이 인덱스와 같거나
정반대인 경우에만 사용할 수 있다.

### WHERE, ORDER BY 혹은 GROUP BY
WHERE 절과 ORDER BY가 같이 사용된 쿼리는 아래 3가지 중 한 방법으로만 인덱스를 이용한다. 

- WHERE 절과 ORDER BY 절이 동시에 같은 인덱스를 이용 : WHERE 절의 비교조건에서 사용하는 컬럼과 ORDER BY 절의 정렬 대상 컬럼이 모두 하나의 인덱스에 연속해서 포함되면 이 방식으로 인덱스를 사용할 수 있다. 
- WHERE 절만 인덱스 이용 : ORDER BY 절은 인덱스를 이용한 정렬이 불가능하며, 인덱스를 통해 검색된 결과 레코드를 별도의 정렬 처리 과정(Using Filesort)을 거쳐 정렬을 수행한다. WHERE 절의 조건에 일치하는 레코드 건수가 많지 않을 때 효율적인 방식 
- ORDER BY 절만 인덱스 이용 : ORDER BY는 인덱스를 이용해 처리하지만 WHERE은 인덱스를 이용하지 못한다. ORDER BY절의 순서대로 인덱스를 읽으면서 레코드를 한 건씩 WHERE 절의 조건에 일치하는지 비교하고 일치하지 않을 떄 버리는 형태로 처리한다. 

### Short-Circuit Evaluation
`AND`의 순서에 따라서 쿼리 성능이 달라진다.  ` && `의 앞이 TRUE여야 뒤를 평가하는 것과 같은 원리다. 

### COUNT
COUNT(*)의 `*`은 레코드를 의미한다. COUNT(PK)나 COUNT(*) 속도에 별 차이가 없다. COUNT()는 모든 레코드를 읽어야 할 수 있는 작업이기에 큰 테이블에서는
지양하는 것이 좋다. 

### OUTER JOIN
굳이 OUTER JOIN을 사용할 필요가 없는 곳에서 OUTER JOIN을 사용하면 성능이 떨어질 수 있다. MySQL 옵티마이저는 절대로 아우터로 조인되는 테이블을 드라이빙 테이블로 선택하지 못하기 때문에 
풀 스캔이 필요한 테이블을 드라이빙 테이블로 선택한다. 

### Delayed Join
Join을 사용해서 데이터를 조회하는 쿼리에서 GROUP BY 또는 ORDER BY를 사용할 때 각 처리 방법에서 인덱스를 사용하면 최적 처리 중 일 것이다. 그렇지 못하다면
MySQL 서버는 모든 조인을 실행하고 GROUP BY, ORDER BY를 처리할 것이다. 조인 결과를 GROUP BY, ORDER BY하면 조인 전 레코드보다 양이 많아진다.
지연된 조인이란 조인이 실행되기 이전에 GROUP BY, ORDER BY를 처리하는 방식을 의미한다.

### Lateral Join
8.0 이전까지는 그룹별로 몇 건씩만 가져오는 쿼리를 작성할 수 없었다. 8.0부터는 Lateral Join을 이용해서 특정 그룹별로 서브쿼리를 실행해서 그 결과와 조인하는 
것이 가능해졌다. 
```sql
SELECT *
FROM employees e 
LEFT JOIN LATERAL ( SELECT * 
                    FROM salaries s 
                    WHERE s.emp_no = e.emp_no
                    ORDER BY s.from_date DESC
                    LIMIT 2
                  ) s2
ON s2.emp_no = e.emp_no
WHERE e.first_name='MATT';

+--------+------------+------------+-----------------+--------+------------+--------+--------+------------+------------+
| emp_no | birth_date | first_name | last_name       | gender | hire_date  | emp_no | salary | from_date  | to_date    |
+--------+------------+------------+-----------------+--------+------------+--------+--------+------------+------------+
|  10690 | 1962-09-06 | Matt       | Jumpertz        | F      | 1989-08-22 |  10690 | 100949 | 2001-08-19 | 9999-01-01 |
|  10690 | 1962-09-06 | Matt       | Jumpertz        | F      | 1989-08-22 |  10690 | 100200 | 2000-08-19 | 2001-08-19 |
|  12302 | 1962-02-14 | Matt       | Plessier        | M      | 1987-01-28 |  12302 |  60386 | 2002-01-24 | 9999-01-01 |
|  12302 | 1962-02-14 | Matt       | Plessier        | M      | 1987-01-28 |  12302 |  60215 | 2001-01-24 | 2002-01-24 |
|  13163 | 1963-12-11 | Matt       | Heping          | F      | 1987-03-31 |  13163 |  73780 | 2002-03-27 | 9999-01-01 |
|  13163 | 1963-12-11 | Matt       | Heping          | F      | 1987-03-31 |  13163 |  72457 | 2001-03-27 | 2002-03-27 |
|  13507 | 1959-09-05 | Matt       | Wallrath        | M      | 1985-06-28 |  13507 |  99680 | 1997-06-25 | 1997-12-04 |
|  13507 | 1959-09-05 | Matt       | Wallrath        | M      | 1985-06-28 |  13507 |  96140 | 1996-06-25 | 1997-06-25 |

                                                          ...

```

### 실행 계획으로 인한 정렬 흐트러짐
네스티드-루프 조인은 알고리즘 특성상 드라이빙 테이블에서 읽은 레코드의 순서가 다른 테이블이 모든 조인돼도 순서가 유지된다. 그래서 결과는 드라이빙 테이블을 읽은
순서로 정렬된다고 예상할 수 있다. 실제로도 드라이빙 테이블을 인덱스 스캔이나 풀 테이블 스캔을 하고, 그때 드라이빙 테이블을 읽은 순서가 그대로 최종 결과에 반영된다.

하지만 네스티드 루프 조인 대신 해시 조인이 사용되면 쿼리 결과의 레코드 정렬 순서가 달라진다.

### GROUP BY
### WITH ROLLUP
```sql
SELECT dept_no, COUNT(*)
FROM dept_emp
GROUP BY dept_no
WITH ROLLUP;

+---------+----------+
| dept_no | count(*) |
+---------+----------+
| d001    |    20211 |
| d002    |    17346 |
| d003    |    17786 |
| d004    |    73485 |
| d005    |    85707 |
| d006    |    20117 |
| d007    |    52245 |
| d008    |    21126 |
| d009    |    23580 |
| NULL    |   331603 |
+---------+----------+
10 rows in set (0.73 sec)

```
소계를 가져올 수 있다. 

##  서브 쿼리
### SELECT 절에 사용한 서브쿼리
SELECT 절의 서브쿼리는 내부적으로 임시테이블을 만들거나 쿼리를 비효율적으로 실행하게 만들거나 하지 않는다. 따라서 서브쿼리가 적절히 인덱스를 사용하면 주의할 사항은 없다. 

```sql
-- 스칼라 쿼리 : 레코드 컬럼이 하나인 결과를 만드는 서브쿼리
-- 로우 쿼리 : 레코드 건수가 많거나 컬럼 수가 많은 결과를 만들어내는 서브쿼리
```

### FROM 절에 사용한 서브쿼리
이전 버전은 서브쿼리 결과를 임시 테이블로 저장하고 필요할 때 다시 임시 테이블을 읽는 방식으로 처리했다. 5.7 버전부터는 옵티마이저가 FROM 절의 서브 쿼리를
외부 쿼리로 병합하는 최적화하도록 개선됐다. 

단, 집합함수 사용, DISTINCT, GROUP BY 또는 HAVING, LIMIT, UNION (UNION DISTINCT), UNION ALL, SELECT 절에 서브쿼리가 사용된 경우,
사용자 변수 사용(변수 할당)을 하면 서브쿼리는 외부 쿼리로 병합되지 못한다. 

### WHERE 절에 사용한 서브쿼리
SELECT, FROM보다 다양한 형태로 사용될 수 있다. 
1. 동등 비교
2. IN
3. NOT IN


1) equal
````````sql
SELECT * FROM dept_emp de 
WHERE  de.emp_no=
       (
        SELECT e.emp_no
        FROM employees e 
        WHERE e.first_name='Georgi'
        AND e.last_name='Facello'
        LIMIT 1
       );
````````
이러면 5.5에서는 풀스캔하면서 서브 쿼리의 조건에 일치하는지를 체크했다. 이후에는 정반대로 동작한다. 서브쿼리 결과를 상수화 하고 나머지를 처리한다.

2) IN (SemiJoin)
``````sql
SELECT *
FROM dept_emp de 
WHERE (emp_no, from_date) = 
      (
        SELECT emp_no, from_date
        FROM salaries
        WHERE emp_no=10001
        LIMIT 1
      );
``````
5.5까지는 세미 조인 최적화가 부족해서 풀스캔을 했다. 5.6부터는 최적화가 개선되면서 사용할 만해졌다.
- 테이블 풀-아웃(Table Pull-out)
- 퍼스트 매치(Firstmatch)
- 루스 스캔(Loosescan)
- 구체화(Materialization)
- 중복 제거(Duplicated Weed-out)

3) NOT IN( Anti Semi-Join)
Not-Equal을 제대로 활용하기 어렵듯, 안티 세미 조인 역시 최적화하기 어렵다. 
- NOT EXISTS
- 구체화(Materialization)

## CTE(Common Table Expression)
CTE라는 이름을 가지는 임시테이블로 SQL 문장 내에서 한 번 이상 사용될 수 있으며, SQL 문장이 종료되면 CTE는 자동으로 삭제된다. CTE는 재귀 반복 여부로
Non-recursive, Recursive로 구분된다. 

### Non-Recursive CTE
```sql
WITH cte1 AS (SELECT * FROM departments)
SELECT * FROM cte1
```
CTE는 WITH으로 정의한다. 

단, 임시테이블이 여러 번 사용되는 쿼리는 실행 계획이 달라진다.(서브쿼리와)
```sql
EXPLAIN
WITH cte1 AS ( SELECT emp_no, MIN(from_date) FROM salaries GROUP BY  emp_no )
SELECT * FROM employees e 
JOIN cte1 t1 ON t1.emp_no = e.emp_no
JOIN cte1 t2 ON t2.emp_no = e.emp_no;


+----+-------------+------------+------------+--------+-------------------+-------------+---------+-----------+--------+----------+--------------------------+
| id | select_type | table      | partitions | type   | possible_keys     | key         | key_len | ref       | rows   | filtered | Extra                    |
+----+-------------+------------+------------+--------+-------------------+-------------+---------+-----------+--------+----------+--------------------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL              | NULL        | NULL    | NULL      | 282545 |   100.00 | NULL                     |
|  1 | PRIMARY     | e          | NULL       | eq_ref | PRIMARY           | PRIMARY     | 4       | t1.emp_no |      1 |   100.00 | NULL                     |
|  1 | PRIMARY     | <derived2> | NULL       | ref    | <auto_key0>       | <auto_key0> | 4       | t1.emp_no |     10 |   100.00 | NULL                     |
|  2 | DERIVED     | salaries   | NULL       | range  | PRIMARY,ix_salary | PRIMARY     | 4       | NULL      | 282545 |   100.00 | Using index for group-by |
+----+-------------+------------+------------+--------+-------------------+-------------+---------+-----------+--------+----------+--------------------------+
4 rows in set, 1 warning (0.00 sec)
    
EXPLAIN 
SELECT * FROM employees e 
JOIN (SELECT  emp_no, MIN(from_date) FROM salaries GROUP BY emp_no) t1 on t1.emp_no=e.emp_no
JOIN (SELECT  emp_no, MIN(from_date) FROM salaries GROUP BY emp_no) t2 on t2.emp_no=e.emp_no;

+----+-------------+------------+------------+--------+-------------------+-------------+---------+-----------+--------+----------+--------------------------+
| id | select_type | table      | partitions | type   | possible_keys     | key         | key_len | ref       | rows   | filtered | Extra                    |
+----+-------------+------------+------------+--------+-------------------+-------------+---------+-----------+--------+----------+--------------------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL              | NULL        | NULL    | NULL      | 282545 |   100.00 | NULL                     |
|  1 | PRIMARY     | e          | NULL       | eq_ref | PRIMARY           | PRIMARY     | 4       | t1.emp_no |      1 |   100.00 | NULL                     |
|  1 | PRIMARY     | <derived3> | NULL       | ref    | <auto_key0>       | <auto_key0> | 4       | t1.emp_no |     10 |   100.00 | NULL                     |
|  3 | DERIVED     | salaries   | NULL       | range  | PRIMARY,ix_salary | PRIMARY     | 4       | NULL      | 282545 |   100.00 | Using index for group-by |
|  2 | DERIVED     | salaries   | NULL       | range  | PRIMARY,ix_salary | PRIMARY     | 4       | NULL      | 282545 |   100.00 | Using index for group-by |
+----+-------------+------------+------------+--------+-------------------+-------------+---------+-----------+--------+----------+--------------------------+
5 rows in set, 1 warning (0.00 sec)
```
- CTE 임시 테이블은 재사용 가능하므로 FROM절 서브쿼리보다 효율적
- CTE로 선언된 임시 테이블은 다른 CTE에서 참조할 수 있다.
- CTE는 임시테이블 생성 부분과 사용 부분의 코드를 분리할 수 있어 가독성이 높다.

### Recursive CTE

```sql
WITH RECURSIVE cte (no) AS (
        SELECT 1 
        UNION ALL
        SELECT ( no + 1 ) FROM cte WHERE no < 5
)   
SELECT * FROM cte;

+------+
| no   |
+------+
|    1 |
|    2 |
|    3 |
|    4 |
|    5 |
+------+
5 rows in set (0.00 sec)
```

### 윈도우 함수(WINDOW FUNCTION)
현재 레코드를 기준으로 연관된 레코드 집합의 연산을 수행한다. 집계함수는  주어진 그룹별로 하나의 레코드로 묶어서 출력하지만 윈도우 함수는 조건에 일치하는 레코드
건수는 변하지 않고 그대로 유지한다. 풀어서 말하면 GROUP BY는 집계 함수를 사용하면 결과 집합의 모양이 바뀐다. 그에 반해 윈도우 함수는 결과 집합을 그대로
유지하면서 하나의 레코드 연산에 다른 레코드의 컬럼 값을 참조할 수 있다.

### 윈도우 함수 기본 사용법

```sql
AGGREGATE_FUNC () OVER (<partition> <order>) AS window_func_column

SELECT e.*,
RANK() OVER (ORDER BY e.hire_date) AS hire_date_rank
FROM employees e;

+--------+------------+------------+--------------+--------+------------+----------------+
| emp_no | birth_date | first_name | last_name    | gender | hire_date  | hire_date_rank |
+--------+------------+------------+--------------+--------+------------+----------------+
| 110022 | 1956-09-12 | Margareta  | Markovitch   | M      | 1985-01-01 |              1 |
| 110085 | 1959-10-28 | Ebru       | Alpin        | M      | 1985-01-01 |              1 |
| 110183 | 1953-06-24 | Shirish    | Ossenbruggen | F      | 1985-01-01 |              1 |
| 110303 | 1956-06-08 | Krassimir  | Wegerle      | F      | 1985-01-01 |              1 |
                                    ........


SELECT e.*, 
    RANK() OVER ( PARTITION BY de.dept_no ORDER BY e.hire_date) AS hire_date_rank
FROM employees e  
JOIN dept_emp de ON de.emp_no = e.emp_no  
ORDER BY de.dept_no, e.hire_date;


+--------+------------+------------+------------+--------+------------+----------------+
| emp_no | birth_date | first_name | last_name  | gender | hire_date  | hire_date_rank |
+--------+------------+------------+------------+--------+------------+----------------+
| 110022 | 1956-09-12 | Margareta  | Markovitch | M      | 1985-01-01 |              1 |
|  51773 | 1963-06-08 | Eric       | Piazza     | M      | 1985-02-02 |              2 |
|  95867 | 1955-05-06 | Shakhar    | Meszaros   | F      | 1985-02-02 |              2 |
|  98351 | 1962-01-28 | Florina    | Setia      | M      | 1985-02-02 |              2 |
                                        .....

```

윈도우 함수의 각 파티션 안에서도 연산 대상 레코드별로 연산을 수행할 소그룸이 사용되는데, 이를 프레임이라고 한다. 윈도우 함수에서 프레임을 명시적으로 지정하지
않아도 MySQL은 상황에 맞게 프레임을 묵시적으로 선택한다. 

```sql
AGGREGATE_FUNC () OVER (<partition> <order>) AS window_func_column

frame:
    {ROWS | RANGE} {frame_start | frame_between}
frame_between:
    BETWEEN frame_start AND frame_end
```
프레임을 만드는 기준으로 ROWS, RANGE 중 하나를 선택할 수 있다. 
- ROWS : 레코드의 위치를 기준으로 프레임을 생성
- RANGE : ORDER BY에 명시된 컬럼의 값의 범위로 프레임 생성

프레임 시작과 끝을 의미하는 키워드의 의미는 아래와 같다. 
- CURRENT ROW: 현재 레코드
- UNBOUNDED PRECEDING: 파티션의 첫 번째 레코드
- UNBOUNDED FOLLOWING: 파티션의 마지막 레코드
- expr PRECEDING: 현재 레코드로부터 n번째 이전 레코드 
- expr FOLLOWING: 현재 레코드로부터 n번째 이후 레코드 

```
SELECT emp_no, from_date, salary,

--> // 현재 레코드의 from_date를 기준으로 1년 전부터 지금까지 급여 중 최소 급여
MIN(salary) OVER(ORDER BY from_date RANGE INTERVAL 1 YEAR PRECEDING) AS min_1,

--> // 현재 레코드의 from_date를 기준으로 1년 전부터 2년 후까지의 급여 중 최대 급여
MAX(salary) OVER(ORDER BY from_date RANGE BETWEEN INTERVAL 1 YEAR PRECEDING AND INTERVAL 2 YEAR FOLLOWING) AS max_1,

--> // from_date 칼럼으로 정렬 후, 첫 번째 레코드부터 현재 레코드까지의 평균
AVG(salary) OVER(ORDER BY from_date ROWS UNBOUNDED PRECEDING) AS avg_1,

--> // from_date 칼럼으로 정렬 후, 현재 레코드를 기준으로 이전 건부터 이후 레코드까지의 급여 평균
AVG(salary) OVER(ORDER BY from_date ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING) AS avg_2

FROM salaries WHERE emp_no=10001
```

### 윈도우 함수
|                                         |                                         |
|:---------------------------------------:|:----------------------------------------|
|                  AVG()                  | 평균 값 반환                                 |
|                BIT_AND()                | AND 비트 연산 결과 반환                         |
|                BIT_OR()                 | OR 비트 연산                                |
|                BIT_XOR()                | XOR 비트 연산                               |
|                 COUNT()                 | 건수                                      |
|             JSON_ARRAYAGG()             | 결과를 JSON ARRAY로                         |
|            JSON_OBJECTAGG()             | 결과를 JSON OBJECT로                        |
|                  MAX()                  | 최댓값                                     |
|                  MIN()                  | 최소값                                     |
| STDDEV_POP(),<br/> STDDEV(),<br/> STD() | 표준 편차값                                  |
|              STDDEV_SAMP()              | 표본 표준 편차 값                              |
|                  SUM()                  | 합계 값 반환                                 |
|          VAR_POP(), VARIANCE()          | 표준 분산 값 반환                              |
|               VAR_SAMP()                | 표준 분산 값 반환                              |
|               CUME_DIST()               | 누적 분포 값 반환                              |
|              DENSE_RANK()               | 랭킹 값 반환(GAP X)                          |
|              FIRST_VALUE()              | 파티션의 첫 번째 레코드 값 반환                      |
|                  LAG()                  | 파티션 내에서 파라미터(N)을 이용해서 N 번째 이후 레코드 값 반환  |
|              LAST_VALUE()               | 파티션의 마지막 레코드 값 반환                       |
|                 LEAD()                  | 파티션 내에서 파라미터 (N)을 이용해서 N 번쨰 이후 레코드 값 반환 |
|               NTH_VALUE()               | 파티션 N 번째 값 반환                           |
|                 NTILE()                 | 파티션별 전체 건수를 파라미터로 N 등분한 값 반환            |
|             PERCENT_RANK()              | 퍼센트 랭킹 값 반환                             |
|                 RANK()                  | 랭킹값 반환(GAP O)                           |
|              ROW_NUMBER()               | 파티션의 레코드 순번 반환                          |


## 잠금을 사용하는 SELECT
innoDB는 기본적으로 SELECT는 잠금을 하지 않는다. 이를 `Non Locking Consistent Read`라고 한다. 하지만 SELECT 쿼리를 이용해서 읽는 값을 애플리케이션에서
가공해서 업데이트하고자 하면 다른 트랜잭션이 컬럼 값 변경을 하지 못하도록 막아야 한다. 옵션으로  `FOR SHARE`, `FOR UPDATE`를 사용하면 된다.
`FOR SAHRE` 은 읽은 레코드에 읽기 잠금을 걸고 `FOR UPDATE`는 쓰기 잠금을 건다. 

## 잠금 테이블 선택
```sql
SELECT *
FROM employees e 
JOIN dept_emp de on e.emp_no = de.emp_no
JOIN departments d on de.dept_no = d.dept_no
FOR UPDATE;
```
이러면 3개 테이블에서 읽은 레코드에 모두 쓰기 잠금(Exclusive Lock)을 건다. 

## NOWAIT & SKIP LOCKED
8.0부터 추가됐다. 지금까지 MySQL 잠금은 레코드를 잠그고 있으면 트랜잭션은 그 잠금이 해제될 때까지 기다렸다. 최악의 경우 timeout으로 에러를 받을 수도 있었다. 
NOWAIT을 주면 `Statement aborted because lock(s) could not be acquired immediately and NOWAIT is set.`이라는 문구와 함께 에러를
낸다. 

SKIP LOCK은 에러 반환 없이 잠긴 레코드를 무시하고 잠금이 안걸린 레코드만 반환한다.
```sql
SELECT * FROM salaries
WHERE  emp_no = 10001 FOR UPDATE SKIP LOCKED;
```


## INSERT
### INSERT IGNORE
`INSERT IGNORE`은 PK, UNIQUE가 존재해서 중복되는 경우, 테이블과 레코드 컬럼이 호환되지 않는 경우 모두 무시하고 다음 레코드를 처리할 수 있게 해준다. 
BULK INSERT에서 유용하다.

### INSERT ... ON DUPLICATE KEY UPDATE
`INSERT ... ON DUPLICATE KEY UPDATE`는 PK, UNIQUE 중복이 발생하면 UPDATE를 하게 해준다. 

### LOAD DATA
RDBMS에서 데이터를 빠르게 적재할 수 있는 방법으로 `LOAD DATA`가 자주 소개된다. MySQl엔진과 스토리지 엔진의 호출 횟수를 최소화하고 스토리지 엔진이 직접
데이터를 적재하기 때문에 INSERT보다 빠르다. 그러나 아래와 같은 단저미 있다. 
- 단일 쓰레드 실행
- 단일 트랜잭션으로 실행

데이터가 아주 많지 않다면 상관없는데 많다면 온라인 쿼리들이 영향을 받을 수 있다. 또한 LOAD DATA는 하나의 트랜잭션으로 실행되기 때문에 Undo Log가 삭제되지 
않도록 유지해야 한다. 이는 언두 로그를 디스크로 기록해야하는 부하를 만들기도 하지만, 언두가 너무 많이 쌓이면 레코드를 읽는 쿼리들이 필요한 레코드를 찾는 데 
더 많은 오버헤드를 만들어 내기도 한다. 

가능하면 LOAD DATA를 하나로 실행하는 것이 아니라. 동시에 여러 트랜잭션으로 나뉘어 실행되게 하는 것이 좋다. 

### 성능을 위한 테이블 구조
INSERT는 테이블 구조에 의해서 성능이 많이 결정된다. INSERT는 단일 레코드를 저장하는 형태로 사용하기 떄문에 INSERT 자체의 튜닝여지는 적은 편이다.
그래도 노력하자면 

1. INSERT 할 값을 PK 기준으로 정렬해서 INSERT 하는 것이 도움이 될 수 있다.
2. PK 선정에 심혈을 기울인다. ( PK가 B-Tree 전체에 메모리가 적재돼 있어야 빠른 INSERT가 보장된다.)
추가적으로 innoDB 스토리지 엔진은 PK가 클러스터링 키가 기본이다. 이는 INSERT, SELECT 성능 둘 다 잡으면서 PK 설계하기 어렵다는 것을 의미하기도 한다. 
대부분 SELECT 비중이 높기 때문에 SELECT에 맞춰서 설계하는 게 옳다. 
3. Auto-Increment 를 사용하면 클러스터링 효과를 받지 않도록 할 수 있다. 이런 방법으로 INSERT에 알맞도록 설정할 수도 있다. 
   - 단조 증가, 단조 감소 값으로 PK 설정
   - 세컨더리 인덱스 최소화
추가로 채번을 위해서  `AUTO-INC`라는 잠금을 건다. 잠금 방식을 변경할 수 있도록 `innodb_autoinc_lock_mode`를 제공한다.
   - innodb_autoinc_lock_mode = 0 : 항상 AUTO_INC를 걸고 한 번에 1씩만 증가된 값을 가져온다.  (5.1)
   - innodb_autoinc_lock_mode = 1 (Consecutive mode) : 단순히 INSERT하는 쿼리에서 `AUTO_INC`를 사용하지 않고 뮤텍스로 처리한다. 
   - innodb_autoinc_lock_mode = 2 (Interleaved mode) : LOAD DATA나 벌크 INSERT를 포함한 INSERT 계열의 문장을 실행할 때 `AUTO_INC`를 사용하지 않는다. 이때 자동 증가 값을 적당히 미리 받아 처리할 수 있으므로 가장 빠르다.
이 모드에서 채번된 번호는 단조 증가하는 유니크한 번호까지만 보장하며, INSERㅅT 순서와 채번된 번호의 연속성을 보장하지는 않는다. `SBR, Statement Base Replication`을 사용하면 소스, 레플리카 간 AUTO_INC 동기화를 보장하지 못한다. 


## UPDATE, DELETE
보통 하나의 테이블에 대해서 한 건 또는 여러 건의 레코드를 변경/삭제하기 위해서 사용한다. 하지만 MySQL에서는 여러 테이블을 조인해서 한 개 이상의 테이블 레코드를
변경하거나 삭제하는 기능도 제공한다. `JOIN UPDATE`, `JOIN DELETE`이 그 예이다. 

### UPDATE ... ORDER BY ... LIMIT n
UPDATE< DELETE는 WHERE을 걸어서 처리하는 것이 일반적이지만 ORDER BY, LIMIT을 걸어서 특정 컬럼으로 정렬해서 변경, 삭제하는 것도 가능하다. 그러나 
한 번에 너무 많은 양을 처리하면 문제가 될 수 있다. 

```sql
DELETE FROM employees
ORDER BY last_name LIMIT 10;
```
### JOIN UPDATE
두 개이 이상의 테이블을 조인해서 조인된 결과 레코드를 변경/ 삭제하는 쿼리를 JOIN UPDATE라고 한다. 조인된 테이블 중에서 특정 테이블의 컬럼 값을 다른 테이블의
컬럼에 업데이트 해야 할 때 주로 조인 업데이트를 사용한다.

일반적으로 JOIN UPDATE는 조인되는 모든 테이블에 대해서 읽기 참조만 되는 테이블은 읽기 잠금이 걸리고, 컬럼이 변경되는 곳은 쓰기 잠금이 걸린다. 그래서 JOIN UPDATE
는 데드락을 유발할 수도 있다. 

```sql
UPDATE tb_test1 t1, employees e 
SET t1.first_name = e.first_name
WHERE e.emp_no = t1.emp_no;
```
JOIN UPDATE도 JOIN 이기에 순서에 따라서 성능이 달라질 수 있다. 

### 여러 레코드 UPDATE
8.0부터는 레코드 생성(Row Constructor)로 레코드별로 서로 다른 값을 업데이트할 수 있게 됐다. 

### JOIN DELETE

```sql
DELETE e
FROM employees e, dept_emp de, departments d
WHERE e.emp_no = de.emp_no AND de.dept_no = d.dept_no AND d.dept_no=`d001`;
```


## DDL
8.0에는 스키마를 변경하는 작업 도중에도 다른 커넥션에서 해당 테이블의 데이터를 변경하거나 조회하는 작업을 가능하게 해준다. `old_alter_table`로 온라인 DDL로 작동할지
아니면 읽기/쓰기를 막고 스키마를 변경하게 할지 고를 수 있다. 8.0에는 기본 값은 OFF라서 온라인 DDL이 켜져 있다. 온라인 DDL이 켜져 있고, ALTER TABLE을
하면

1. ALGORITHM=INSTANT로 스키마 변경 가능 여부 확인 후, 가능하면 선택
2. ALGORITHM=INPLACE로 스키머 변경 가능 여부 확인 후, 가능하면 선택
3. ALGORITHM=COPY 알고리즘 선택

순으로 적합한 알고리즘을 찾는다. 

- INSTANT : 테이블의 데이터는 변경하지 안혹 메타 데이터만 변경하고 작업을 완료한다.  걸리는 시간은 레코드 건수와 무관하다.  스키마 변경 도중 읽고 쓰기는 대기하게 되지만 무시할 수준이다.
- INPLACE : 임시테이블로 데이터를 복사하지 않고 스키마를 변경한다. 내부적으로 테이블의 리빌드를 실행할 수도 있다. 레코드 복사는 없지만 모든 레코드를 리빌드해야 하기 때문에 테이블 크기에 따라 많은 시간이 걸릴 수 있다. 그러나 스키마 변경 중에도 읽고 쓰기가 된다. 최초, 마지막에만 불가능하다.
- COPY : 변경된 스키마를 적용한 임시 테이블을 생성하고, 테이블의 레코드를 모두 임시 테이블로 복사하고 최종적으로 임시 테이블을 RENAME해서 스키마 변경을 완료한다. 진행되는 도중 읽기만 가능하고 DML은 불가능하다.

INPLACE, COPY라면 LOCK 옵션을 명시할 수 있다.
- NONE : 아무런 잠금 걸지 않음
- SHARED: 읽기 잠금을 걸고 스키마 변경을 실행하기 때문에 스키마 변경 중 읽기는 가능하지만 쓰기는 불가능함
- EXCLUSIVE: 쓰기 잠금을 걸고 스키마 변경을 실행하기에 테이블의 읽고 쓰기가 불가능하다. 

```sql
ALTER TABLE salaries CHANGE to_date end_date DATE NOT NULL,
ALGORITHM = INPLACE, LOCK = NONE; 
```
INPLACE로 진행하면 NONE으로 LOCK은 가능하지만, 가끔 `SHARED`까지 설정해야 할 수도 있다. EXCLUSIVE는 전통적 ALTER TABLE과 동일하므로 LOCK을 명시할 필요가 없다.
만일 리빌드가 필요하면 (PK를 추가하는 작업같은 경우) 레코드 저장위치가 바뀌어야 하기 때문에 테이플 리빌드가 필요한 반면, 단순히 컬럼 이름 변경은 INPLACE를 사용해도
리빌드가 필요가 없다. 리빌드가 필요한 경우를 `Reorganizing` 혹은 `Table Rebuild 라고 한다.

<a href="https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html"> 온라인 DDL 지원 표 확인 </a>

### INPLACE 알고리즘
INPLACE가 임시 테이블로 레코드 복사를 하지 않더라도 내부적으로 테이블의 모든 레코드를 리빌드해야 하는 경우가 많다.
```
1. INPLACE 스키마 변경이 지원되는 스토리지 엔진의 테이블인지 확인
2. INPLACE 스키마 변경 준비( 변경 정보 준비해서 변경 중 바뀌는 데이터 추적 준비)
3. 테이블 스키마 변경 및 새로운 DML 로깅
4. 로그 적용
5. INPLACE 스키마 변경 완료
```

2, 4 단계에서는 배타적 잠금(Exclusive Lock)이 필요하다. 이 시점에는 DML이 대기한다. 3번에서는 대기 없이 처리된다. 그리고 INPLACE 알고리즘으로 온라인 스키마
변경 진행되는 동안 새로 유입된 DML의 쿼리들에는 온라인 변경 로그라는 메모리 공간을 쌓아뒀다가 온라인 스키마 변경이 완료되면 로그 내용을 실제 테이블로 일괄 적용한다.
`innodb_online_alter_log_max_size`로 이 메모리 크기가 결정된다. 


### 온라인 DDL 모니터링
온라인 DDL을 포함한 모든 ALTER는 `performance_schema`를 통해서 진행 상황을 모니터링할 수 있다. `performance_schema`를 이용해서 ALTER TABLE을 모니터링하려면
performance_schema 옵션 (Instrument, Consumer)을 활성화해야 한다.

```sql
SET GLOBAL performance_schema = ON;

UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'stage/innodb/alter%';

UPDATE performance_schema.setup_consumers
SET ENABLED = 'YES'
WHERE NAME LIKE '%stages%';
```


## 데이터베이스 변경
MySQL은 스키마와 데이터베이스를 구분하지 않는다. 동격의 개념이다. 그래서 MySQL은 굳이 스키마를 명시적으로 사용하지 않는다. MySQL의 데이터베이스는 디스크의 물리적인
저장소를 구분하지만 여러 데이터베이스의 테이블을 묶어서 조인쿼리를 사용할 수도 있기 때문에 단순히 논리적인 개념이기도 하다.

```sql
-- 생성
CREATE DATABASE  [IF NOT EXIST] employees;
CREATE DATABASE employees CHARACTER SET utf8mb4;
CREATE DATABASE employees CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;

-- 조회
SHOW DATABASES;

-- 선택
USER employees;

-- 속성 변경
ALTER DATABASE employees CHARACTER SET = 'euckr' COLLATE = euckr_korean_ci;

-- 삭제
DROP DATABASE [IF EXISTS] employees;
```

## 테이블 스페이스 변경
MySQL은 전통적으로 테이블 별로 전용의 테이블 스페이스를 사용했다. 8.0부터는 MySQL 서버에도 사용자 테이블을 제너럴 테이블 스페이스(스토리지 엔진의 시스템 테이블 스페이스)
로 저장하는 기능이 추가되고, 테이블 스페이스를 관리하는 DDL이 추가됐다. 그러나 제너럴 테이블 스페이스에는 여러 제약 사항을 가진다.  

- 파티션 테이블은 제너럴 테이블스페이스를 사용하지 못한다.
- 복제 소스와 레플리카 서버가 동일 호스트에서 실행하는 경우 ADD DATAFILE은 불가
- 테이블 암호화(TDE)는 테이블 스페이스 단위로 설정됨
- 테이블 압축 기능 여부는 테이블 스페이스의 블록 사이즈와 innoDB 페이지 사이즈에 의해서 결정된다.
- 특정 테이블을 삭제하고 (DROP TABLE)해도 운영체제로 바로 반납되지 않는다.

그러나 아래와 같은 장점이 있다.
- 제너럴 테이블 스페이스를 사용하면 파일 핸들러(Open file descriptor) 최소화
- 테이블 스페이스 관리에 필요한 메모리 공간을 최소화

## 테이블 생성

```sql
CREATE [TEMPORARY] TABLE [IF NOT EXISTS] tb_test ( 
    
    member_id BIGINT [UNSIGNED] [AUTO_INCREMENT],
    nickname CHAR(20) [CHARACTER SET 'utf8'] [COLLATE 'utf8_general_ci'] [NOT NULL],
    home_url VARCHAR(200) [COLLATE 'latin1_general_cs'],
    birth_year SMALLINT [(4)] [UNSIGNED] [ZEROFILL],
    member_point INT [NOT NULL] [DEFAULT 0],
    registered_dttm DATETIME [NOT NULL],
    modified_ts TIMESTAMP [NOT NULL] [DEFAULT CURRENT_TIMESTAMP],
    gender ENUM('Female','Male') [NOT NULL], 
    hobby SET('Reading','Game','Sports'),
    profile TEXT [NOT NULL],
    session_data BLOB, PRIMARY KEY (member_id),
    
    UNIQUE INDEX ux_nickname (nickname),
    INDEX ix_registereddttm (registered_dttm)
    
 ) ENGINE=INNODB;
```

### 테이블 구조 조회
```sql
SHOW CREATE TABLE employees;

CREATE TABLE `employees` (
                            `emp_no` int NOT NULL,
                            `birth_date` date NOT NULL,
                            `first_name` varchar(14) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
                            `last_name` varchar(16) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
                            `gender` enum('M','F') CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
                            `hire_date` date NOT NULL,
                            PRIMARY KEY (`emp_no`),
                            KEY `ix_hiredate` (`hire_date`),
                            KEY `ix_gender_birthdate` (`gender`,`birth_date`),
                            KEY `ix_firstname` (`first_name`)
                         
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci STATS_PERSISTENT=0
```

### 테이블 구조 변경 
```sql
ALTER TABLE employees
CONVERT TO 
    CHARACTER SET UTF8MB4 COLLATE utf8mb4_general_ci,
    ALGORITHM=INPLACE , LOCK=NONE;
```
### 테이블 명 변경
```sql
RENAME TABLE table1 TO table2;
```
### 테이블 상태 조회
MySQL의 모든 테이블은 만들어진 시간, 대략의 레코드 건수, 데이터 파일의 크기 등의 정보를 가지고 있다.또한 데이터 파일의 버전, 레코드 포맷 등의 중요한 정보도
가지고 있다.

```sql
SHOW TABLE STATUS LIKE 'employees';

+-----------+--------+---------+------------+--------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+-------------+------------+--------------------+----------+--------------------+---------+
| Name      | Engine | Version | Row_format | Rows   | Avg_row_length | Data_length | Max_data_length | Index_length | Data_free | Auto_increment | Create_time         | Update_time | Check_time | Collation          | Checksum | Create_options     | Comment |
+-----------+--------+---------+------------+--------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+-------------+------------+--------------------+----------+--------------------+---------+
| employees | InnoDB |      10 | Dynamic    | 300141 |             50 |    15220736 |               0 |     22593536 |   4194304 |           NULL | 2023-11-04 09:44:16 | NULL        | NULL       | utf8mb4_general_ci |     NULL | stats_persistent=0 |         |
+-----------+--------+---------+------------+--------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+-------------+------------+--------------------+----------+--------------------+---------+
1 row in set (0.02 sec)

```

### 테이블 구조 복사
```sql
CREATE TABLE temp_employees LIKE employees;

INSERT INTO temp_employees SELECT * FROM employees;
```

### 테이블 삭제 
```sql
DROP TABLE [ IF EXISTS ] table1;
```
테이블 삭제에서 유의해야 할 것은 어댑티브 해시 인덱스(Adaptive hash index)이다. innoDB 버퍼 풀의 각 페이지가 가진 레코드에 대한 해시 인덱스를 제공하는데,
어댑티브 해시 인덱스가 활성화돼 있는 경우 테이블이 삭제되면 어댑티브 해시 인덱스 정보도 모두 삭제해야 한다.

### 인덱스 변경
```sql
ALTER TABLE employees ADD PRIMARY KEY (emp_no)
ALGORITHM = INPLACE, LOCK = NONE;

ALTER TABLE employees ADD UNIQUE INDEX ux_empno (emp_no),
ALGORITHM = INPLACE, LOCK = NONE;

ALTER TABLE employees ADD INDEX ix_lastname (last_name),
ALGORITHM = INPLACE, LOCK = NONE;

ALTER TABLE employees ADD FULLTEXT INDEX fx_firstname_lastname (first_name, last_name)
ALGORITHM = INPLACE, LOCK = SHARED;

ALTER TABLE employees ADD SPATIAL INDEX fx_loc ( last_location ),
ALGORITHM = INPLACE, LOCK = SHARED;
```

### 인덱스 조회
```sql
SHOW INDEX FROM employees;
+-----------+------------+---------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table     | Non_unique | Key_name            | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+-----------+------------+---------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| employees |          0 | PRIMARY             |            1 | emp_no      | A         |      300141 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| employees |          1 | ix_hiredate         |            1 | hire_date   | A         |        4156 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| employees |          1 | ix_gender_birthdate |            1 | gender      | A         |           5 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| employees |          1 | ix_gender_birthdate |            2 | birth_date  | A         |        8669 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| employees |          1 | ix_firstname        |            1 | first_name  | A         |        1309 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
+-----------+------------+---------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
5 rows in set (0.03 sec)
```

### 인덱스 이름 변경
```sql
ALTER TABLE salaries RENAME INDEX ix_salary TO ix_salary2,
ALGORITHM = INPLACE, LOCK = NONE;
```

### 인덱스 가시성 변경
인덱스를 만드는 건 비용이 적으나 지우는 건 비용이 크게 든다. 그래서 8.0부터는 쿼리를 실행할 때 해당 인덱스를 무효화할지 아닐지를 결정하는 쿼리를 만들었다.
```sql
ALTER TABLE employees ALTER INDEX ix_firstname INVISIBLE;

```

### 인덱스 삭제
```sql
ALTER TABLE ... DROP INDEX ...로 삭제할 수 있다.
```
PK는 클러스터링 인덱스 특성 때문에 삭제 시 임시 테이블로 레코드를 복사해서 테이블을 재구축 해야 한다.

### 프로세스 조회 및 강제 종료
```sql
SHOW PROCESSLIST;

+----+-----------------+-----------+-----------+---------+------+------------------------+------------------+
| Id | User            | Host      | db        | Command | Time | State                  | Info             |
+----+-----------------+-----------+-----------+---------+------+------------------------+------------------+
|  5 | event_scheduler | localhost | NULL      | Daemon  | 5979 | Waiting on empty queue | NULL             |
|  8 | root            | localhost | employees | Query   |    0 | init                   | SHOW PROCESSLIST |
+----+-----------------+-----------+-----------+---------+------+------------------------+------------------+
2 rows in set, 1 warning (0.00 sec)

    
KILL [QUERY] 1;
```

### 활성 트랜잭션 조회
```sql
SELECT trx_id, 
       (
         SELECT CONCAT(user, '@', host)
         FROM information_schema.PROCESSLIST
         WHERE id = trx_mysql_thread_id
       ) AS source_info,
       trx_state,
       trx_started,
       now(),
       (unix_timestamp(now() - unix_timestamp(trx_started))) AS lasting_sec,
       trx_requested_lock_id,
       trx_wait_started,
       trx_mysql_thread_id,
       trx_tables_in_use,
       trx_tables_locked
FROM information_schema.innodb_trx
WHERE (UNIX_TIMESTAMP(now()) - unix_timestamp(trx_started)) > 5;
```

## 쿼리 성능 테스트
### 쿼리의 성능에 영향을 미치는 요소
1. 운영체제의 캐시
MySQL 서버는 운영체제의 파일 시스템 관련 기능(시스템 콜)을 이용해서 데이터 파일을 읽어온다. innoDB 엔진은 파일 시스템의 캐시나 버퍼를 거치지 않는 Direct I/O를 
사용하므로 운영체제의 캐시가 그다지 큰 영향을 끼치지 않지만 MyISAM은 그렇지 않다.

2. MySQL 서버의 버퍼풀( innoDB 버퍼 풀과 MyISAM의 키 캐시)
innoDB 버퍼 풀은 인덱스 페이지, 데이터 페이지까지 캐시하고, 쓰기 작업을 위한 버퍼링 작업까지 겸해서 처리한다. MyISAM은 인덱스에만 캐싱을 지원한다. 결국 MyISAM은 운영체제 
캐시에 의존한다. MySQL 서버가 한 번 시작되면 innoDB 버퍼 풀, MyISAM의 키 캐시의 내용을 퍼지할 수는 없다. MySQL 서버에 포함된 키 캐시나 버퍼 푸를 초기화 하려면
MySQL 서버를 재시작해야한다.
특히 innoDB 버퍼 풀은 서버가 종료될 때 자동으로 덤프, 재기동하면 자동으로 적재된다. 그래서 `innodb_buffer_pool_load_at_startup`을 off로 두고 시작해야 한다.

3. 네트워크적 요소

4. 쿼리 테스트 횟수 : MySQL이 웜업된 상탠지 콜드 상태인지에 따라 다른 결과를 낼 것이다.