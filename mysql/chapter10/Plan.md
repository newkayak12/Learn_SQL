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

````sql
mysql> explain SELECT * FROM UnionGlobal.tblFile;
+----+-------------+---------+------------+------+---------------+------+---------+------+-------+----------+-------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra |
+----+-------------+---------+------------+------+---------------+------+---------+------+-------+----------+-------+
|  1 | SIMPLE      | tblFile | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 25562 |   100.00 | NULL  |
+----+-------------+---------+------------+------+---------------+------+---------+------+-------+----------+-------+
1 row in set, 1 warning (0.00 sec)

````

## id 컬럼 
실행 계획에서 가장 왼쪽 id는 쿠러별로 부여되는 식별자 값이다. 테이블을 조인하면 조인되는 테이블 개수만큼 실행 계획 레코드가 출력되지만 같은 id가 부여된다.
아래 쿼리는 3개 단위 SELECT이 존재하므로 실행 계획의 각 레코드가 각기 다른 id를 지닌다. 
```sql
mysql> explain select ( (select count(*) from tblFile) + (select count(*) from tblUser)) as total;
+----+-------------+---------+------------+-------+---------------+------------------+---------+------+-------+----------+----------------+
| id | select_type | table   | partitions | type  | possible_keys | key              | key_len | ref  | rows  | filtered | Extra          |
+----+-------------+---------+------------+-------+---------------+------------------+---------+------+-------+----------+----------------+
|  1 | PRIMARY     | NULL    | NULL       | NULL  | NULL          | NULL             | NULL    | NULL |  NULL |     NULL | No tables used |
|  3 | SUBQUERY    | tblUser | NULL       | index | NULL          | admin_sign_index | 2       | NULL |  8406 |   100.00 | Using index    |
|  2 | SUBQUERY    | tblFile | NULL       | index | NULL          | PRIMARY          | 8       | NULL | 25562 |   100.00 | Using index    |
+----+-------------+---------+------------+-------+---------------+------------------+---------+------+-------+----------+----------------+
3 rows in set, 1 warning (0.00 sec)
```
여기서 중요한 것은 id가 테이블 접근 순서를 의미하는 것은 아니다. 

## SELECT_TYPE
쿼리가 어떤 타입의 쿼리인지 표시되는 컬럼이다. 
- SIMPLE : UNION, SUBQUERY를 사용하지 않은 단순 쿼리
- PRIMARY : UNION, SUBQUERY를 가지는 SELECT 쿼리의 실행 계획 가장 바깥쪽에 있는 단위 쿼리
- UNION : UNION하는 쿼리 중 첫 번째 다음 쿼리. 첫 번째 단위는 임시테이블(DERIVED)로 표시된다.
- DEPENDENT UNION : UNION, UNION ALL의 경우 표시된다. DEPENDENT는 UNION, UNION ALL로 결합된 단위 쿼리가 외부 쿼리에 의해 영향을 받는 것을 의미한다.
- UNION RESULT : UNION의 결과를 담아두는 테이블을 의미한다. 8.0 이전은 UNION ALL, UNION (UNION DISTINCT)는 모두 UNION의  결과를 임시테이블로 생성했다. 
- SUBQUERY : FROM절 이외에서 사용되는 서브쿼리만을 의미한다. FROM절은 DERIVED로 표시된다. 
- DEPENDENT SUBQUERY: SUBQUERY가 바깥쪽 SELECT에서 정의된 컬럼을 사용하는 경우 표시된다. 외부 쿼리가 먼저 수행된 후 내부 쿼리를 실행하므로 SUBQUERY보다 느리다. 
- DERIVED : 임시 테이블이라고 이해하면 편하다.
- DEPENDENT DERIVED : 8.0이전은 FROM 절의 서브쿼리(INLINE-VIEW)는 외부컬럼을 사용할 수 없었지만 LATERAL JOIN으로 가능하게 됐다.
- UNCACHEABLE SUBQUERY : 하나의 쿼리 문장에 서브쿼리가 하나만 있더라도 실제 그 서브쿼리가 한 번만 실행되는 것은 아니다. 그런데 조건이 같은 서브 쿼리가 실행될 때는 이전 결과를 담아두고 사용한다. 이 경우는 서브쿼리에 포함된 요소 때문에 캐싱이 불가함을 의미한다.
   1. 사용자 변수가 서브쿼리에 사용된 경우
   2. NOT-DETERMINISTIC 속성의 ROUTINE이 서브쿼리 내에 있는 경우
   3. UUID, RAND 같이 호출시 결과 값이 달라지는 함수가 SUBQUERY에 있는 경우 
- UNCACHEABLE UNION : UNION에 사용된 요소가 캐싱이 불가함을 의미한다.
- MATERIALIZED: 5.6부터 도입됐다. FROM, IN 형태의 쿼리에 사용된 서브쿼리 최적화를 위해서 사용한다. 


## table 컬럼
테이블 기준으로 표시되며, 별칭이 있으면 별칭이 출력된다. 

## partition 컬럼
5.7까지는 EXPLAIN PARTITION으로 확인이 가능했지만 8.0부터는 EXPLAIN으로 가능하다. 여기서 파티션 프루닝 결과를 알 수 있다. 

## type 컬럼
MySQL이 각 테이블 레코드를 어떤 방식으로 읽었는지를 나타낸다. 인덱스를 이용했는지, 테이블을 풀스캔을 했는지 말이다.
- system : 레코드가 1건만 존재하거나 한 건도 없는 테이블을 참조하는 형태의 접근 방법 (innoDB X)
- const : 레코드 건수와 상관없이 쿼리가 PK, UNIQUE를 이용하는 WHERE을 가지고 있으면 반드시 1 건만 반환하는 방법
- eq_ref : 여러 테이블이 조인되는 쿼리 실행계획표에서만 표시된다. 조인에서 처음 읽은 테이블 컬럼 값을, 그다음 읽어야할 테이블의 PK, UQ의 검색 조건에 사용할 때
- ref : 조인 순서 없이 사용되며, PK, UQ등의 제약도 없다. 인덱스 종류와 상관없이 동등 조건으로 검색할 떄 사용된다.
- fulltext : 전문 검색 인덱스를 사용해서 레코드를 읽는 접근 방법을 의미한다. 
- ref_or_null : ref와 같은 null 비교가 추가된 형태다. ref || IS NULL 
- unique_subquery : where에 사용될 수 있는 IN SUBQUERY 형태의 쿼리를 위한 접근 방법, 서브쿼리에서 중복되지 않은 유니크한 값만 반환할 떄 이 접근 방법을 사용한다.
- index_subquery IN에서 SUBQUERY가 중복된 값을 반환할 수 있다. 이때 중복된 값을 인덱스를 이용해서 제거할 수 있을 때 사용한다.
- range : 인덱스 레인지 스캔 (<, >, IS NULL, BETWEEN, IN, LIKE)
- index_merge : 2개 이상의 인덱스를 이용해서 각각의 검색 결과를 만들고 병합해서 처리할 때
- index : index를 처음부터 읽는 풀 스캔을 의미한다. 
- ALL : 풀스캔 

## possible_keys 컬럼
이 컬럼에 있는 내용은 옵티마이저가 최적의 실행 계획을 만들기 위해서 후보로 선정했던 접근 방법에서 사용할 법했던 인덱스 목록이다. 

## key
최종 선택된 실행 계획에서 사용하는 인덱스를 의미한다.

## key_len
다중 컬럼으로 구성된 인덱스에서 몇 개의 컬럼까지 사용했는지 알려준다. 인덱스의 각 레코드에서 몇 바이트까지 사용했는지 알려준다.

## ref
참조 조건으로 어떤 값이 제공됐는지 보여준다. 

## rows
실행 계획의 효율성 판단을 위해서 예측했던 레코드 건수를 보여준다. 또한 쿼리를 위해서 얼마나 많은 레코드를 읽고 체크해야하는지를 의미한다.  

## filtered

## extra
- Const row not found : 레코드가 한 건도 존재하지 않으면
- Deleting all rows : 엔진 핸들러 차원에서 테이블의 모든 레코드를 삭제하는 기능이 있고 사용한다면
- Distinct : 꼭 필요한 레코드만 읽었다는 것을 의미한다(특히 join)
- FirstMatch : 세미 조인 여러 최적화 중 FirstMatch가 사용되면 출력된다.
- Full scan on Null key : `col1 IN (SELECT col2 FROM... )`같은 쿼리에서 자주 발생한다. 서버가 쿼리를 실행하는 중 col1이 NULL을 만나면 차선책으로 서브쿼리 테이블에 대해서 풀스캔을 사용할 것임을 알려주는 키워드다.
  - 서브쿼리가 1건이라도 결과 레코드를 가진다면 최종 비교 결과 NULL
  - 서브쿼리가 1건이라도 레코드를 가지지 않는다면 최종 비교 결과는 FALSE
- Impossible Having : HAVING 조건에 만족하는 레코드가 없을 때 
- Impossible Where : where이 항상 FALSE가 된다면 
- LooseScan : LooseScan이 사용되면 
- No matching min/max row : MIN, MAX같은 집합 함수가 있는 쿼리의 조건절에 일치하는 레코드가 없을 때
- No matching row in const table : 조인에서 사용된 테이블에서 const 방법으로 접근할 때 일치하는 레코드가 없으면
- No matching rows after partition pruning : 파티션된 테이블에 대한 UPDATE/ DELETE할 대상 레코드가 없을 때
- No tables used : FROM 절이 없는 경우 
- Not exists : outer join 으로 안티 조인을 수행하는 쿼리에서 
- Plan isn't ready yet: 다른 커넥션에서 읽고 있는 경우 
- Range checked for each record(index map: N) : 
- Recursive : 8.0부터 CTE로 재귀 쿼리를 사용할 때 
- Rematerialize : 8.0부터 Lateral Join 되는 테이블은 성행테이블의 레코드별로 서브쿼리를 실행해서 그 결과를 임시테이블로 밀어 넣는다. 이 과정을 의미한다. 
- Select tables optimized away : MAX, MIN만 SELECT에서 사용되거나 GROUP BY로 MIN, MAX를 조회하는 쿼리가 인덱스 오름/내림차순으로 1건만 읽는 형태의 최적화가 사용되면 
- Start||End Temporary : 세미 조인 최적화 중 Duplicate Weed-out이 사용되면 
- Unique row not found : UQ(PK 포함)로 아우터 조인을 수행하는 쿼리에서 아우터 테이블에 일치하는 레코드가 없으면
- Using filesort : order by처리하기 위해서 적절한 인덱스를 사용하지 못하면
- Using index(covering index) : 데이터 파일을 전혀 읽지 않고 인덱스만 읽어서 쿼리를 모두 처리할 수 있다면
- Using index condition : 옵티마이저가 인덱스 컨디션 푸시 다운(Index Condition Pushdown) 최적화를 사용하면   -> ICP : WHERE 필터 조건(Condition)을 스토리지 엔진으로 밀어넣을(Pushdown) 수 있도록 해주는 기능이다.
- Using index for group-by: grouping 기준 컬럼을 이용해서 정렬 작업을 수행하고 다시 정렬된 결과를 grouping한다. Group by가 인덱스를 탄다면 정렬된 인덱스를 읽으면서 순서대로 Grouping한다. 이때 표시된다.
- tide index scan을 통한 group by : avg, sum, count처럼 다 조회해야 한다면
- loose index scan을 통한 group by : max, min같이 첫 번쨰 또는 마지막 레코드만 읽어도 되는 경우 
- Using index for skip scan : 인덱스 스캡 스캔 최적화를 사용하면 표시한다.
- Using join buffer(Block Nested Loop || Batched Key Access || hash join) : 조인은 인덱스가 없는 테이블을 읽고 있는 테이블(FK)를 읽어서 진행한다. 적절한 인덱스가 없으면 서버는 블록 네스티드 루프 조인이나 해시 조인을 쓴다.
이 때 조인 버퍼를 사용하는데, 실행 계획에서 조인 버퍼가 사용되는 실행 계획은 Extra 컬럼에는 Using join buffer라는 메시지가 나온다.
- Using MRR : innoDB를 포함한 스토리진 엔진 레벨에서는 쿼리 실행의 전체적인 부분을 알지 못한다. 그래서 최적화에 한계가 있다. 이러한 이유로 아무리 많은 레코드를 읽어야 해도 스토리지 엔진은 MySQL 엔진이 넘겨주는
값을 기준으로 레코드를 한 건 한 건 읽어서 반환해야만 한다. 이러한 문제를 해결하기위해서 MRR(Multi Range Read)를 도입했다. MySQL 엔진은 여러 개의 키를 한 번에 스토리지 엔진으로 전달하고, 스토리지 엔진은 넘겨받은
키 값들을 정렬해서 최소한 페이지 접근만으로 레코드를 읽을 수 있도록 최적화한다.
- Using sort_union(...), Using union(...), Using intersect(...) : 쿼리 index_merge 접근 방법으로 실행되는 경우에는 2개 이상의 인덱스가 동시에 사용될 수 있다. 실행계획의 Extra에는
두 인덱스로부터 읽은 결과를 어떻게 병합했는지 나타낸다.
  - Using sort_union(...) : Using union과 같은 작업을 수행하지만 Using union으로 처리할 수 없으면 (OR 연결된 상대적으로 대량의 range 조건) 이 방식으로 처리한다. Using sort_union은 PK만 먼저 읽어서 정렬하고 병합한 이후 비로소 레코드를 읽어서 반환할 수 있다는 것
  - Using union(...) : 각 인덱스를 사용할 수 있는 조건이 OR로 연결된 경우 각 처리 결과에서 합집합을 추출하는 작업을 수행했다는 의미다.
  - Using intersect(...) : 각각의 인덱스를 사용할 수 있는 조건이 AND로 연결돈 경우 각 처리 결과에서 교집합을 추출하는 작업을 했단 의미다.
- Using temporary : 쿼리 처리 중 임시 테이블을 생성하면 출력된다. 메모리 상에 생성될 수도, 디스크 상에 생성될 수도 있다. 
- Using where : MySQL은 내부적으로 MySQL엔진과 스토리지 엔진 두 개의 레이어로 나눠있다. MySQL엔진은 스토리지 엔진으로부터 받은 레코드를 가공, 연산하는 작업을 수행하는데, MySQL 엔진 레이어에서 별도 가공을 하면 출력된다. 
- Zero limit : 때로는 데이터가 아닌 쿼리 결과의 메타데이터만 필요한 경우도 있다. 즉 쿼리가 몇 개의 컬럼을 가지고 컬럼 타입은 무엇인지 등의 정보 말이다. 이런 경우 `LIMIT 0`을 사용하면 되는데 이때는 옵티마이저 사용자의 의도를 파악하고 레코드는 읽지 않고 결과 값의 메타 정보만 반환한다. 
