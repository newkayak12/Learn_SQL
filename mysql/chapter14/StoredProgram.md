# 스토어드 프로그램

스토어드 프로그램은 스토어드 루틴이라고도 한다. 스토어드 프로시저와 스토어드 함수, 트리거와 이벤트 등을 모두 아우르는 명칭이다.

## 장단점
### 장점
- DB 보안 향상 : MySQL의 스토어드 프로그램은 자체적인 보안 설정 기능을 가지고 있다. 프로그램 단위로 실행 권한을 부여할 수 있다. 이러한 보안 기능을 조합해서 특정 테이블의 읽기와 쓰기 또는 특정 컬럼에 대해서만 권한을 설정하는 등 세밀하게 권한 제어가 가능하다.
- 기능의 추상화 : 개발언어나 도구와 관계없이 스토어드 프로그램으로 작성해서 사용하면 쉽게 활용할 수 있다.
- 네트워크 소요 시간 절감 : 각 쿼리가 큰 데이터를 클라이언트로 가져와서 가공한 후, 다시 서버로 전송해야하면 더 큰 네트워크 경우 시간이 소요될 것이다. 스토어드 프로그램으로 미리 구현하면 한 번에 호출해서 네트워크로 넘기면 네트워크 소요 시간을 줄일 수 있다.
- 절차적 기능 구현 : 스토어드 프로그램은 절치적 기능을 실행할 수 있는 제어 기능을 제공한다. 


### 단점
- 낮은 처리 성능 : 스토어드 프로그램은 MySQL 엔진에서 해석되고 실행된다. MySQL 서버는 스토어드 프로그램같이 절차적으로 처리하는 것에 초점을 두지 않았어서 처리 성능이 떨어진다. 또한 MySQL 스토어드 프로그램은 실행 때마다 스토어 프로그램의 코드가 파싱돼야 한다.
- 애플리케이션 코드의 조각화 : 비즈니스 로직 처리가 분산되면 관리 포인트가 늘어서 유지 보수가 어려워진다.


### 스토어드 프로시저
스토어드 프로시저는 서로 데이터를 주고 받아야 하는 여러 쿼리를 하나의 그룹으로 묶어서 독립적으로 실행하기 위해서 사용한다.

#### 생성/ 삭제
```sql
CREATE PROCEDURE sp_sum (IN param1 INTEGER, IN param2 INTEGER, OUT param3 INTEGER )
BEGIN 
    SET param3 = param1 + param2;
END;;
```

#### 주의
- 스토어드 프로시저는 기본 반환값이 없어 RETURN을 사용할 수 없다.
- IN 타입 파라미터는 입력 전용
- OUT은 출력 전용 파라미터다.
- INOUT는 입/출력 모두 가능하다.

#### 구분자
너무 많은  `;`는 프로시저의 끝을 알 수 없게 한다. 종료문자를 변경해서 이를 피할 수 있다.
```sql
DELIMITER ;;
CREATE PROCEDURE sp_sum (IN param1 INTEGER, IN param2 INTEGER, OUT param3 INTEGER )
BEGIN 
    SET param3 = param1 + param2;
END;;
DELIMITER ;
```

#### 변경/ 삭제
`ALTER PROCEDURE`, `DROP PROCEDURE`를 사용해서 수정, 삭제한다. 

#### 실행
```sql
SET @result := 0;
CALL sp_sum(1, 2, @result);
SELECT @result;
```

#### 스토어드 프로시저 커서 반환
스토어드 프로그램은 명시적으로 커서를 파라미터로 전달받거나 반환할 수 없다. 하지만 프로시저 내에서 커서를 오픈하지 않거나 SELECT 쿼리의 결과 셋을 FETCH 하지
않으면 해당 쿼리의 결과 셋은 클라이언트로 바로 전송된다. (바로 결과가 반환된다.)

프로시저 쿼리의 결과 셋을 클라이언트로 전송하는 기능은 프로시저 디버깅 용도로 사용된다. 
```sql
DELIMITER ;;
CREATE PROCEDURE sp_sum (IN param1 INTEGER, IN param2 INTEGER, OUT param3 INTEGER )
BEGIN 
    SELECT '> Stored procedure started' as debug_msg;
    SELECT CONCAT(' > param1 : ', param1) as debug_msg;
    SELECT CONCAT(' > param2 : ', param2) as debug_msg;
    
    SET param3 = param1 + param2;

    SELECT '> END' as debug_msg;
END;;
DELIMITER ;
```

#### 스토어드 프로시저 딕셔너리
8.0 이전까지는 `proc`에 저장됐지만 8.0부터는 보이지 않는 시스템 테이블로 저장된다. 사용자는 단지 `information_schema` 데이터베이스의 ROUTINES 뷰를
통해서 스토어드 프로시저 정보를 조회할 수 있다.
```sql
SELECT routine_schema, routine_name, routine_type
FROM information_schema.ROUTINES
where routine_schema = 'test';
```

## 스토어드 함수
### 생성/ 삭제
```sql
CREATE FUNCTION sf_sum( param1 INTEGER, param2 INTEGER )
RETURNS  INTEGER
BEGIN 
    DECLARE param3 INTEGER DEFAULT 0;
    SET param3 = param1 + param2;
    RETURN param3;
END;;
```

#### 프로시저 vs 함수
- 함수 정의 부에 RETURN을 명시
- 본문 마지막에 RETURN TYPE에 맞는 값을 RETURN 해야한다.
- PREPARE, EXECUTE 명령을 prepare statement를 사용할 수 없다.
- 명시적 또는 묵시적 Rollback/ Commit을 사용하는 SQL을 사용할 수 ㅇ벗다.
- 재귀 호출이 불가하다.
- 스토어드 함수 내에서 프로시저를 호출할 수 없다. 
- RESULT SET을 반환하는 SQL를 사용할 수 없다.

#### 실행
```sql
SELECT sf_sum(1, 2) as sum;
```

## 트리거
트리거는 테이블의 레코드가 저장되거나 변경될 때 미리 정의해둔 작업을 자동으로 실행해주는 스토어드 프로그램이다. 

### 생성
```sql
CREATE TRIGGER on_delete BEFORE DELETE ON employees
    FOR EACH ROW 
    BEGIN 
        DELETE FROM salaries WHERE salaries.emp_no = OLD.emp_no;
    END;;
```

### 예외
BEGIN ... END에서 사용하지 못하는 몇 가지 유형의 작업이 있다.
- 트리거는 외래키 관계에 의해 자동으로 변경되면 호출되지 않는다.
- 복제에 의해 레플리카 서버에 업데이트 되는 레코드 기반의 복제(Row based replication)에서는 레플리카 서버의 트리거를 기동시키지 않지만 문장 기반의 복제(Statement based replication)에서는 레플리카 서버에서도 트리거를 가동시킨다.
- 명시적, 묵시적 ROLLBACK/COMMIT을 시키는 작업을 할 수 없다.
- RETURN 문장을 사용할 수 없으며, 트리거 종료시 LEAVE를 사용한다.
- mysql과 information_schema, performance_schema 데이터베이스에 존재하는 테이블에 대해서는 트리거를 생성할 수 없다.

### 트리거 딕셔너리
8.0 이전은 데이터베이스 디렉토리의 *.TRG로 기록됐다. 하지만 8.0부터는 보이지 않는 시스템 테이블로 저장되고, 사용자는 `information_schema`의 TRIGGERS 뷰로만 확인할 수 있다
```sql
SELECT trigger_schema, trigger_name, event_manipulation, action_timing
FROM information_schema.TRIGGERS
WHERE trigger_schema = 'employees';
```

## 이벤트
주어진 특정한 시간에 스토어드 프로그램을 실행할 수 있는 스케쥴러 기능을 이벤트라고 한다. 스케쥴링을 전담하는 쓰레드를 MySQL은 가지고 있는데 이 쓰레드가 활성화된
경우에만 이벤트가 실행된다. `event_scheduler`를 on으로 바꿔야한다. 그러면
```sql
SHOW PROCESSLIST;
```
으로 확인할 수 있다.

### 생성
반복 여부에 따라 일회성, 반복성으로 나눌 수 있다.

1. 일회성
```sql
CREATE EVENT onetime_job
ON SCHEDULE AT CURRENT_TIMESTAMP + INTERVAL 1 HOUR 
DO 
INSERT INTO daily_rank_log VALUES ( NOW(), 'Done' );
```

2. 반복성
```sql
CREATE EVENT daily_ranking
ON SCHEDULE EVERY 1 DAY STARTS '2020-09-07 01:00:00' ENDS '2021-01-01 00:00:00'
DO 
INSERT INTO daily_rank_log VALUES ( NOW(), 'Done' );
```

DO에서 프로시저 콜, 단순 쿼리, BEGIN...END 모두 사용할 수 있다.
또한 이벤트의 반복성과 관계없이 `ON COMPLETE`로 완전히 종료된 이벤트를 삭제할지, 유지할지 선택할 수 있다.

#### 이벤트 실행 및 결과 확인
이벤트의 스케쥴링 정보나 최종 실행 시간 정보는 `information_schema` 데이터베이스 EVENT 뷰를 조회하면 알 수 있다.


```sql
DELIMITER ;;
CREATE TABLE daily_rank_log (exec_dttm DATETIME, exec_msg VARCHAR(50));;
CREATE EVENT daily_ranking
    ON SCHEDULE AT CURRENT_TIMESTAMP + INTERVAL 1 MINUTE 
    ON COMPLETION 
DO BEGIN 
    INSERT INTO daily_rank_log VALUES (NOW(), 'DONE');
END ;;

SELECT * FROM information_schema.EVENTS;
```

#### 이벤트 딕셔너리
8.0 이전까지는 생성된 이벤트의 딕셔너리 정보는 MySQL 데이터베이스의 events 테이블에 관리됐지만 8.0부터는 사용자에 노출되지 않는 시스템 테이블로 관리된다.
```sql
SELECT * FROM information_schema.EVENTS;
```

#### 스토어드 프로그램 본문 작성
1. `BEGIN ... END` : 본문은 BEGIN으로 시작하고 END로 끝난다. 이 블록은 여러 개를 중첩해서 사용할 수 있다. 
   - BEGIN
   - START TRANSACTION
BEGIN...END에서 사용하는 BEGIN은 트랜잭션 시작이 아닌 블록 시작으로 인식한다. 그래서 'START TRANSACTION'으로 대체해야 한다.

```sql
CREATE PROCEDURE sp_hello ( IN name VARCHAR(50) ) 
BEGIN 
    START TRANSACTION;
    INSERT INTO tb_hello VALUES (name, CONCAT('Hello ', name));
    COMMIT;
END;
-- 이렇게 프로시저 내부에서 트랜잭션을 완료하면 이 프로시저를 호출한 애플리케이션, SQL 클라이언트에서는 트랜잭션 조절이 불가하다.
```
2. 변수 : 로컬 변수는 `DECLARE`로 정의되고 반드시 타입이 함께 명시돼야 한다. 로컬변수에 값을 할당하는 방법은 `SET` 또는 `SELECT ... INTO ...`로 가능하다.
이 변수는 BEGIN...END 내에서만 유효하며, 사용자 변수보다는 빠르며 다른 쿼리나 스토어드 프로그램과 스코프를 공유하지 않느다. 또한 로컬 변수는 반드시 타입과 정의되기 때문에 컴파일러 수준에서 타입 오류를 체크할 수 있다.
SELECT ... INTO ...는 레코드 컬럼 값을 변수에 할당하는 명령으로 반드시 1개의 레코드를 반환해야만 한다.

만약 변수명, 파라미터명, 테이블 컬럼명이 겹치면
   1. DECALRE로 정의한 변수
   2. 스토어드 프로그램의 입력 파라미터
   3. 테이블 컬럼


3. 제어문 : 
   1. `IF ... ELSEIF ... ELSE ... END IF`
    ```sql
        CREATE FUNCTION sf_greatest(p_value1 INT, p_value2 INT) 
            RETURNS INT
        BEGIN 
            IF p_value1 IS NULL THEN RETURN p_value2;
            ELSEIF p_value2 IS NULL THEN RETURN p_value1;
            ELSEIF p_value1 >= p_value2 THEN RETURN p_value1;
            ELSE RETURN p_value2;
            END IF;
        END;;
    ```