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
