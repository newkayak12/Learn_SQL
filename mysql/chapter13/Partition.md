# 파티션
파티션은 논리적으로는 하나의 테이블이지만 물리적으로는 여러 개의 테이블로 분해서 관리할 수 있게 해준다. 파티션은 주로 대용량 테이블을 물리적으로 
여러 개의 소규모 테이블로 분산하는 목적으로 사용한다. 

## 사용 이유 
하나의 테이블이 너무 커서 인덱스의 크기가 물리적인 메모리보다 훨씬 크거나 데이터 특성상 주기적인 삭제 작업이 필요한 경우가 필요한 예시다.

### 단일 INSERT, 단일 혹은 범위 SELECT의 빠른 처리
UPDATE, DELETE 처리를 위해서 레코드를 검색하려면 인덱스가 필수다. 하지만 인덱스가 커지면 커질수록 SELECT, UPDATE, DELETE 할거 없이 다 느려진다.
한 테이블의 크기가 물리적으로 MySQL이 사용 가능한 메모리 공간보다 크면 영향은 더 심각해진다. 테이블 데이터는 실질적인 물리 메모리보다 대부분 크겠지만
인덱스의 워킹셋(Working set)이 실질적인 물리 메모리보다 크면 쿼리 처리가 상당히 느려질 것이다.

파티션은 데이터와 인덱스를 조각화해서 물리적 메모리를 효율적으로 사용할 수 있게 해준다.

### 데이터의 물리적인 저장소를 분리
데이터 파일, 인덱스 파일이 커지면 백업, 관리 등이 어려워진다. 이런 문제를 파티션을 통해 파일의 크기를 조절하거나 파티션별 파일들이 저장될 위치나 디스크를 구분해서
지정해 해결하는 것도 가능하다. 

### MySQL의 파티션 내부 처리

```sql
CREATE TABLE tb_article (
    article_id INT NOT NULL,
    reg_date DATETIME NOT NULL,
    
    ...,
    
    PRIMARY KEY(article_id, reg_date) 
                    
) PARTITION BY ( YEAR ( reg_date) )(
    
    PARTIION p2009 VALUES LESS THEN (2010),
    PARTIION p2010 VALUES LESS THEN (2011),
    PARTIION p2011 VALUES LESS THEN (2012),
    PARTIION p9999 VALUES LESS THEN MAXVALUE
    
);
```

1. INSERT : reg_date로 파티션 표현식을 평가하고 어디에 저장할지 결정한다. 
2. UPDATE : 변경 대상 레코드가 어떤 파티션에 있는지 확인한다. WHERE에 파티셔 키 컬럼이 있으면 빠르게 접근한다. 그렇지 않다면 모든 파티션을 검색한다. 
3. SELECT : 파티션 테이블을 검색할 때 성능에 영향을 미치는 조건은
   1. WHERE 조건으로 파티션을 선택할 수 있는가?
   2. WHERE 조건으로 인덱스를 효율적으로 사용할 수 있는가?

   (2) 내용은 파티션되지 않은 일반 테이블에서도 똑같이 성능 영향을 미친다. 하지만 파티션 테이블에서는 (1)의 결과에 따라 (2) 선택사항의 작업 내용이 달라질 수 있다.
    
    - 파티션 선택 가능 + 인덱스 효율적 사용 가능 : 두 선택 사항이 모두 사용가능하면 효율적으로 처리된다. 파티션 개수와 관계없이 파티션의 인덱스만 레인지 스캔
    - 파티션 선택 불가 + 인덱스 효율적 사용 가능 : WHERE로 파티션을 걸러낼 수 없으니 모든 파티션을 대상으로 검색한다. 각 파티션에 대해서는 인덱스 레인지 스캔이 가능하기에 파티션 개수만큼 인덱스 레인지 스캔을 수행한다.
    - 파티션 선택 가능 + 인덱스 효율적 사용 불가 : 파티션 선별은 가능하지만 인덱스 이용이 불가해서 풀 테이블 스캔을 한다. 
    - 파티션 선택 불가 + 인덱스 효율적 사용 불가 : 파티션 선별도 불가하고 풀 스캔을 해야된다.


### 파티션 프루닝
옵티마이저 판단에 따라 3개 파티션 중 2개만 읽어도 된다고 하면 불필요한 파티션에 접근하지 않는다. 이렇게 최적화 단계에서 불필요한 것을 배제하는 것을
파티션 프루닝 (Partition pruning)이라고 한다. 실행 계획을 확인하면 파티션 프루닝에 대한 정보를 알 수 있다.

```sql
EXPLAIN 
    SELECT * FROM tb_article
    WHERE reg_date > '2010-01-01'
    AND reg_date < '2010-02-01';
```

## 주의 사항
1. 파티션 제약 사항
   - 스토어드 루틴, UDF, 사용자 변수 등을 파티션 표현식에 사용할 수 없다.
   - 파티션은 컬럼 자체 또는 MySQL 내장함수를 사용할 수 있는데 이러면 생성은 가능하지만 파티션 프루닝이 불가할 수 있다.
   - PK를 포함해서 테이블의 모든 유니크 인덱스는 파티션 키 컬럼을 포함해야 한다.
   - 파티션된 테이블 인덱스는 모두 로컬 인덱스이며, 동일 테이블에 소속된 모든 파티션은 같은 구조의 인덱스만 가질 수 있다. 또한, 파티션 개별로 인덱스를 변경하거나 추가할 수 없다.
   - 동일 테이블에 속한 모든 파티션은 동일 스토리지 엔진만 가질 수 있다.
   - 최대 (서브 파티션 포함) 8192개만 가질 수 있다. 
   - 파티션 생성 이후 sql_mode 변경은 파티션의 일관성을 깨뜨릴 수 있다.
   - 파티션 테이블에는 외래키 사용이 불가하다.
   - 파티션 테이블은 전문 검색 인덱스 생성이나 전문 검색 쿼리가 불가하다.
   - 공간 데이터를 사용하는 컬럼은 파티션 테이블에서 사용 불가하다.
   - 임시 테이블은 파티션할 수 없다.

2. 주의 사항
   - 파티션과 UNIQUE(PK 포함) : 종류 관계 없이 테이블이 UNIQUE, PK가 있으면 파티션 키는 모든 유니크 인덱스의 일부 또는 모든 컬럼을 포함해야 한다.
   - open_files_limit 설정 : 파티션을 사용하는 경우 `open_files_limit`을 적절히 높은 값으로 설정해야 한다.(서버에서 동시에 오픈된 파일 개수가 많아질 수 있다.)


## 파티션 종류
1. 레인지 파티션
2. 리스트 파티션
3. 해시 파티션
4. 키 파티션

### 레인지 파티션 
파티션 키의 연속된 범위로 파티션을 결정하는 방법이다. 다른 파티션 방법과 달리 MAXVALUE라는 키워드로 명시되지 않은 범위의 키 값이 담긴 레코드를 저장하는
파티션을 정의할 수 있다.

#### 주 용도 
1. 날짜를 기반으로 데이터가 누적되고 연도나 월, 또는 일 단위로 분석하고 삭제할 떄
2. 범위 기반으로 데이터를 여러 파티션에 균등하게 나눌 수 있을 때
3. 파티션 키 위주로 검색이 자주 될 때

#### 생성
```sql
CREATE TABLE employees (
    id INT NOT NULL, 
    first_name VARCHAR(30),
    last_name VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    
    ...
    
) PARTITION BY RANGE ( YEAR(hired) ) (
    PARTITION p0 VALUES LESS THAN (1991), 
    PARTITION p1 VALUES LESS THAN (1996), 
    PARTITION p2 VALUES LESS THAN (2001), 
    PARTITION p3 VALUES LESS THAN MAXVALUE
);
```

#### 추가
```sql
ALTER TABLE employees
ADD PARTITION (PARTITION p4 VALUES LESS THAN (2011));


```

#### 삭제
```sql
ALTER TABLE employees DROP PARTITION p0;
```

#### 기존 파티션 분리
```sql
ALTER TABLE employees ALGORITHM = INPLACE, LOCK = SHARED,
REORGANIZE PARTITION p3 INTO (
    PARTITION p3 VALUES LESS THAN (2011),
    PARTITION p4 VALUES LESS THAN MAXVALUE
);
```

#### 파티션 병합
```sql
ALTER TABLE employees ALGORITHM = INPLACE, LOCK = SHARED,
REORGANIZE PARTITION p2, p3 INTO  (
    PARTITION p23 VALUES LESS THAN (2011) 
);
```

### 리스트 파티션
- 파티션 키 값이 코드 값이나 카테고리 같이 고정적일 때
- 키 값이 연속되지 않고 정렬 순서와 관계없이 파티션해야 할 때
- 파티션 키 값을 기준으로 레코드 건수가 균일하고 검색 조건에 파티션 키가 자주 사용될 때

##### 생성
```sql
CREATE TABLE product (
    id INT NOT NULL ,
    name VARCHAR(30),
    category_id INT NOT NULL,
    ...
) PARTITION BY LIST ( category_id ) (
    PARTITION p_appliance VALUES IN (3),
    PARTITION p_computer VALUES IN (1,9),
    PARTITION p_sports VALUES IN (2,6,7),
    PARTITION p_etc VALUES IN (4,5,8,NULL)
);
```

#### 파티션 분리/ 병합
```sql
ALTER TABLE product ALGORITHM = INPLACE, LOCK = SHARED,
REORGANIZE PARTITION p_appliance, p_computer INTO  (
    PARTITION p23 VALUES IN (1, 3, 9)
);
```

#### 주의 사항
1. 명시되지 않은 나머지 값을 저장하는 MAXVALUE 파티션을 지정할 수 없다. 
2. 레인지 파티션과는 달리 NULL을 저장하는 파티션을 별도 생성할 수 있다.

### 해시 파티션
해시 함수에 의해 레코드가 저장될 파티션을 결정하는 방법이다.

- 레인지 파티션이나 리스트 파티션으로 데이터를 균등하게 나누는 것이 어려울 떄
- 테이블의 모든 레코드가 비슷한 사용 빈도를 보이지만 테이블이 너무 커서 파티션을 적용해야 할 떄

#### 생성
```sql
CREATE TABLE employees (
    id INT NOT NULL,
    first_name VARCHAR(30),
    last_name VARCHAR(30),
    hired DATE NOT NULL  DEFAULT '1970-01-01'
) PARTITION BY HASH ( id ) PARTITIONS 4;

-- or

CREATE TABLE employees (
                          id INT NOT NULL,
                          first_name VARCHAR(30),
                          last_name VARCHAR(30),
                          hired DATE NOT NULL  DEFAULT '1970-01-01'
) PARTITION BY HASH ( id ) PARTITIONS 4 (
    PARTITION p0 ENGINE = INNODB, 
    PARTITION p1 ENGINE = INNODB, 
    PARTITION p2 ENGINE = INNODB, 
    PARTITION p3 ENGINE = INNODB 
);

```

#### 분리/ 병합
해시 함수의 알고리즘을 변경하는 것이므로 전체 파티션이 영향을 받는다. 레코드를 재분배하기 때문이다.

```sql
ALTER TABLE employees ALGORITHM = INPLACE, LOCK = SHARED, 
ADD PARTITION ( PARTITION p5 ENGINE = INNODB );


ALTER TABLE employees ALGORITHM = INPLACE, LOCK = SHARED,
ADD PARTITION PARTITIONS 6;
```

#### 삭제
```sql
ALTER TABLE employees DROP PARTITION p0;

-- Error Code: 1512
-- 파티션 드롭은 RANGE/LIST만 가능하다.
```

#### 분할
특정 파티션을 두 개 이상으로 분할하는 기능은 없다. 테이블 전체적으로 파티션 개수를 늘리는 것만 가능하다.

#### 병합
```sql
ALTER TABLE employees ALGORITHM = INPLACE, LOCK = SHARED
COALESCE PARTITION 1;
```

#### 주의 사항
- 특정 파티션만 DROP은 불가하다.
- 새로운 파티션을 추가하는 작업은 단순히 파티션만 추가하는 것이 아니라 기존 모든 데이터의 재배치가 필요하다.
- 해시 파티션은 레인지 파티션이나 리스트 파티션과는 굉장히 다른 방식으로 관리하므로 해시 파티션이 용도에 적합한지 확인이 필요하다.
- 일반적으로 사용자에게 익숙한 파티션의 조작이나 특성은 대부분 리스트 파티션, 레인지 파티션에만 해당하는 것들이 만핟. 해시 파티션이나 키 파티션을 사용하거나 조작할 떄는 주의가 필요하다.

### 키 파티션
키 파티션은 해시 파티션과 사용법이 거의 비슷하다. 키 파티션에서는 정수 타입이나 정숫값을 반환하는 표현식뿐만 아니라 대부분의 데이터 타입에 대해서 파티션 키를 적용할 수 있다.
이게 유일한 차이점이다.

#### 생성
```sql
CREATE TABLE k1 (
    id INT NOT NULL,
    name VARCHAR(20),
    PRIMARY KEY (id)
) PARTITION BY KEY ()
PARTITIONS 2;

-- 괄호를 비우면 자동으로 PK가 모든 컬럼의 파티션키가 된다.

CREATE TABLE k2 (
    emp_no INT NOT NULL,
    dept_no CHAR(4) NOT NULL ,
    name VARCHAR(20),
    PRIMARY KEY (emp_no, dept_no)
) PARTITION BY KEY (emp_no)
   PARTITIONS 2;

-- PK, UNIQUE 중 일부만 선택할 수도 있다.
```

#### 주의/ 특이 사항
- 키 파티션은 서버 내부적으로 MD5()를 이용해서 파티션 하기 때문에 파티션 키가 반드시 정수 타입이 아니어도 된다. 해시 파티션으로 파티셔닝하기 어려우면 키 파티션 적용을 고려하자.
- PK나 UNIQUE를 구성하는 컬럼 중 일부만으로 파티셔닝할 수 있다.
- UNIQUE키를 파티션 키로 사용하면 `NOT NULL`이어야만 한다.
- 해시 파티션에 비해서 파티션 간의 레코드를 균등하게 분배할 수 있기 때문에 키 파티션이 더 효율적이다.

### 리니어 해시 파티션/ 리니어 키 파티션
해시 파티션이나 키 파티션은 새로운 파티션을 추가하거나 파티션을 통합해서 개수를 줄일 때 파티션 만이 아니라 테이블의 전체 파티션에 저장된 레코드의 재분배 작업이 발생한다.
이러한 단점을 최소화하기 위해서 Linear 해시 파티션, Linear 키 파티션 알고리즘이 고안됐다. (Power-of-two 알고리즘을 이용한다.)

#### 리니어 해시 파티션/ 리니어 키 파티션 추가, 통합
추가는 특정 파티션 레코드만 재분배하는 방식으로 작동한다. 다른 파티션은 재분배 작업과 관련이 없기 때문에 일반 해시 작업보다 빠르다. 통합 역시 통합을 하는 파티션만
재분배 작업을 진행한다.

#### 주의 사항
일반 해시 파티션이나 키 파티션보다는 덜 균등하게 배치될 수 있다. 그러나 새로운 파티션을 추가/ 삭제할 요건이 많으면 리니어를 사용하는 것이 좋다.

### 파티션 테이블 쿼리 성능
파티션 테이블에 쿼리가 실행될 때 모든 파티션을 읽을지 일부만 읽을지는(파티션 프루닝) 성능적으로 큰 영향을 미친다. 

