# Data Type
## 문자열(CHAR, VARCHAR)
1. 저장 공간
CHAR, VARCHAR는 문자열을 저장할 수 있는 데이터 타입이다. 차이점은 고정, 가변 길이냐가 있다.
 - 고정 길이는 실제 입력되는 컬럼 값의 길이에 따라 사용하는 저장 공간의 크기가 변하지 않는다. CHAR 타입은 고정이다. 
 - 가변 길이는 최대로 저장할 수 있는 길이가 제한돼 있지만, 그 이하의 크기 값이 저장되면 저장 공간이 줄어든다. 하지만 VARCHAR는 유효 크기가 얼마인지를 별도를 저장해야하므로 1~2 바이트의 저장 공간이 추가로 더 필요하다.

> CHAR(1) vs. VARCHAR(1)
> CHAR는 1, VARCHAR는 255 바이트 이하면 1바이트만 추가 사용하고, 256 넘아가면 2바이트를 사용한다.
> VARCHAR 최대 길이는 2바이트로 표현할 수 있는 이상은 사용할 수 없다.(65,536)

 
> MySQL에서는 레코드 하나에서 TEXT, BLOB을 제외하면 64KB를 초과할 수 없다. 
> 테이블에 VARCHAR 하나만 컬럼으로 있다면 64KB까지 사용할 수 있다. 이미 다른 컬럼에서 40KB를 사용한다면 VARCHAR는 24KB만 쓸 수 있다. 이 경우 그 이상으로 잡으면 알아서 TEXT로 바뀐다.
 
2. CHAR, VARCHAR의 사용 
항상 길이가 일정하면 CHAR, 아니라면 VARCHAR로 쓰는 것이 일반적이다. 굳이 그럴 필요가 있을까?
- 저장되는 문자열의 길이가 대개 비슷한가?
- 컬럼의 값이 자주 변경되는가?

길이도 중요하지만 값이 자주 변경되는지가 중요하다.
CHAR는 고정 바이트가 있으므로 그냥 변경하면 된다. 그러나 VARCHAR는 기존 값보다 더 큰 값으로 변경되면 레코드 자체를 다른 공간으로 (Row Migration) 저장한다.


#### 저장 공간과 스키마 변경(online DDL)
VARCHAR를 늘리는 작업은 그냥 할 수도 있지만 테이블 락을 걸고 레코드 복사하는 작업이 수반될 수도 있다. 이유는 VARCHAR가 가지는 길이 저장 공간의 크기 때문이다.
VARCHAR(60) utf8mb4는 최대가 240(60*40)다. VARCHAR(64)는 256까지 가능하기에 문자열 길이 공간이 2바이트로 늘어야 한다. 이렇게 되면 락을 걸고 레코드를 복사하는 방식으로 처리한다.

#### CHARSET
테이블 컬럼별로 문자열 값을 저장할 수 있다. 문자 집합은 CHAR, VARCHAR, TEXT에 설정할 수 있다. MySQL 서버와 DB 그리고 테이블 단위로 기본 문자 집합을 설정 할 수 있는 기능을 제공한다.
MySQL에서 "SHOW CHARACTER SET"으로 선택 가능한 문자 집합을 확인할 수 있다.
- latin : 알파벳, 숫자, 키보드 특수 문자
- euckr : 한국어 전용 문자 집합
- utf8mb4 : 다국어 문자를 포함할 수 있는 컬럼 디스크에 한 글자를 저장하기 위해서 1 ~ 4바이트까지 사용한다.
- utf8 : uft8mb4 이전에 주로 사용했다.

MySQL 문자 집합을 설정하는 시스템 변수가 여러 가지가 있다. 
- character_set_system : 식별자(identifier, 테이블, 컬럼명 등)을 저장할 때 사용하는 문자 집합 (기본값 utf8)
- character_set_server : DB, 테이블, 컬럼에 아무런 문자 집합이 설정되지 않을 떄 이 변수에 명시된 문자 집합이 기본으로 사용된다. ( 기본값 utf8mb4 )
- character_set_database : MySQL 기본 문자 집합 DB 생성시 문자 집합이 명시되지 않으면 여기를 참조한다.  ( 기본값 utf8mb4 )
- character_set_filesystem : `LOAD DATA INFILE ...` 또는 `SELECT ... INTO OUTFILE`을 실행할 때 인자로 지정하는 파일 이름을 해석할 떄 사용하는 문자 집합  ( 기본값 utf8mb4 )
- character_set_client : MySQL 클라이언트가 보낸 SQL 문장은 이 변수에 설정된 문자 집합으로 인코딩해서 서버로 전송한다. 
- character_set_connect : SQL 문장을 처리하기 위해서 이 변수에 설정된 문자 집합으로 변환한다.  ( 기본값 utf8mb4 )
- character_set_result : 서버가 쿼리의 처리 결과를 클라이언트에 보낼 때 사용하는 문자 집합을 설정하는 시스템 변수  ( 기본값 utf8mb4 )


#### Collation
콜레이션은 문자열 컬럼의 값에 대한 비교나 정렬 순서를 위한 규칙을 의미한다. 즉, 비교나 정렬 작업에서 영문 대소문자를 같은 것으로 처리할지, 아니면 
더 크거나 작은 것으로 판단할지에 대한 규칙을 정의하는 것이다. 그래서 각 문자열 컬럼의 값을 비교하거나 정렬할 때는 항상 문자 집합 뿐아니라 콜레이션의
일치 여부에 따라 결과가 달라지며, 쿼리 성능 또한 상당한 영향을 받는다.

```sql

mysql> SHOW COLLATION;
+-----------------------------+----------+-----+---------+----------+---------+---------------+
| Collation                   | Charset  | Id  | Default | Compiled | Sortlen | Pad_attribute |
+-----------------------------+----------+-----+---------+----------+---------+---------------+
| armscii8_bin                | armscii8 |  64 |         | Yes      |       1 | PAD SPACE     |
| ascii_general_ci            | ascii    |  11 | Yes     | Yes      |       1 | PAD SPACE     |
| big5_bin                    | big5     |  84 |         | Yes      |       1 | PAD SPACE     |
| big5_chinese_ci             | big5     |   1 | Yes     | Yes      |       1 | PAD SPACE     |
| binary                      | binary   |  63 | Yes     | Yes      |       1 | NO PAD        |

                                            ...
                                                
| utf8mb4_unicode_ci          | utf8mb4  | 224 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_vietnamese_ci       | utf8mb4  | 247 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_vi_0900_ai_ci       | utf8mb4  | 277 |         | Yes      |       0 | NO PAD        |
| utf8mb4_vi_0900_as_cs       | utf8mb4  | 300 |         | Yes      |       0 | NO PAD        |
| utf8mb4_zh_0900_as_cs       | utf8mb4  | 308 |         | Yes      |       0 | NO PAD        |
+-----------------------------+----------+-----+---------+----------+---------+---------------+
286 rows in set (0.01 sec)
    
-- 3파트 콜레이션
-- (문자 집합 - 하위 분류 - ci면 Case Insensitive cs는 Case Sensitive)
    
-- 2파트 콜레이션
-- (문자 집합 - bin) bin은 이진 데이터를 의미한다. 비교 및 정렬은 실제 문자 데이터의 바이트 값을 기준으로 수행한다. 
```
#### 테이블 설정에서 문자셋 설정
```sql
CREATE DATABASE db_test CHARACTER SET = utf8mb4;

CREATE TABLE tb_member (
    member_id VARCHAR(20) NOT NULL COLLATE latin1_general_ci,
    member_name VARCHAR(20) NOT NULL COLLATE utf8_bin,
    member_email VARCHAR(100) NOT NULL,
    ...
);
```


#### 이스케이프
| 이스케이프 | 의미              | 
|:------|:----------------|
| \0    | 아스키  NULL       |
| \'    | '               |
| \"    | "               |
| \b    | 백스페이스           |
| \n    | 개행              |
| \r    | carriage return |
| \t    | tab             |
| \\    | \               |
| \%    | %(LIKE에서 사용)    |
| \_    | _(LIKE에서 사용)    |


#### 숫자
숫자는 정확도에 따라 Exac랑 근삿값으로 나뉜다.
- Exact는 소수점 이하 값의 유무와 관계없이 정확히 그 값을 그대로 유지하는 것을 의미한다. INTEGER, INT, DECIMAL이 있다.
- 근삿값은 부동 소수점이라고 불리는 것이다. FLOAT, DOUBLE이 있다.

#### AUTO_INCREMENT 옵션 사용
`auto_increment` 와 `auto_increment_offset` 설정 값으로 자동 증가 값이 얼마가 도리지 변경할 수 있다.

- MyISAM은 자동 증가 옵션이 사용된 컬럼이 PK, UNIQUE 아무데나 사용할 수 있다.
- innoDB 스토리지에는 AUTO_INCREMENT로 PK를 생성해야 한다. PK 뒤에 AUTO_INCREMENT를 배치하면 오류가 발생한다. 

#### 날짜와 시간
| 데이터 타입    | MySQL 5.6.4 이전 | MySQL 5.6.4 이후       |
|:----------|:---------------|:---------------------|
| YEAR      | 1byte          | 1byte                |
| DATE      | 3byte          | 3byte                |
| TIME      | 3byte          | 3byte + 밀리초 단위 저장 공간 |
| DATETIME  | 8byte          | 5byte + 밀리초 단위 저장 공간 |
| TIMESTAMP | 4byte          | 4byte + 밀리초 단위 저장 공간 |

밀리초 단위 자릿수가 1 ~ 2면 1바이트, 3 ~ 4는 2바이트, 5 ~ 6 3바이트

#### 타임존
```sql
SET time_zone='America/Los_Angeles';
SHOW VARIABLES LIKE '%time_zone%';
```

#### 자동 업데이트 
MySQL 5.6이전 까지는 TIMESTAMP는 레코드의 다른 컬럼 데이터가 변겨될 때마다 시간이 자동 없이트 되고, DATETIME은 그렇지 않았다. 5.6부터는 고정이고, 업데이트 떄마다
자동 업데이트 되게 하려면 옵션을 정의해야 한다.

```sql
CREATE TABLE tb_autoupdate (
    id BIGINT NOT NULL AUTO_INCREMENT,
    title VARCHAR(20),
    created_at_ts TIMESTAMP DEFAULT  CURRENT_TIMESTAMP,
    updated_at_ts TIMESTAMP DEFAULT  CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_at_dt TIMESTAMP DEFAULT  CURRENT_TIMESTAMP,
    update_at_dt TIMESTAMP DEFAULT  CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id)
        
);
```

#### ENUM 
ENUM은 테이블 구조에 나열된 목록 중 하나의 값을 가질 수 있다.
```sql
CRATE TABLE tb_enum (fd_enum ENUM("PROCESSING", "FAILURE", "SUCCESS"));
INSERT INTO tb_enum VALUES ("PROCESSING"), ("FAILRUE");

-- 실제로는 문자열이 아닌 매핑된 정수 값을 사용한다. ENUM 최대는 65,535개이고 255 미만이면 1바이트를 사용하고 그 이상이면 2바이트를 사용한다. 


mysql> ALTER TABLE tb_enum MODIFY fd_enum ENUM('PROCESSING','FAILURE','SUCCESS','REFUND'), ALGORITHM=INSTANT;
mysql> ALTER TABLE tb_enum MODIFY fd_enum ENUM('PROCESSING','FAILURE','REFUND','SUCCESS'), ALGORITHM=COPY, LOCK=SHARED;
```

#### SET
테이블 구조에 정의된 아이템을 정수 값으로 매핑해서 쓰는 것은 똑같다. 큰 차이는 하나의 컬럼에 1 이상의 값을 저장할 수 있다는 것이다. MySQL은 내부적으로 BIT-OR 연산으로 1개 이상의 선택된 값을 저장한다.

```sql
CREATE TABLE tb_set ( fd_set SET('TENNIS','SOCCER','GOLF','TABLE-TENNIS','BASKETBALL','BILLIARD') );
INSERT INTO tb_set (fd_set) VALUES ('SOCCER'), ('GOLF,TENNIS');
SELECT * FROM tb_set;
+-------------+
|    fd_set   |
+-------------+ 
|    SOCCER   |
| TENNIS,GOLF | 
+-------------+

```

#### TEXT, BLOB
TEXT, BLOB은 거의 똑같은 설정이나 방식을포 작동한다. 유일한 차이점은 TEXT는 문자열을 저장하는ㄷ 대용량 컬럼이라 문자 집합이나 콜레이션을 가진다는 것이고,
BLOB은 이진 데이터 타입이라 별도의 문자 집합이나 콜레이션을 갖지 않는다는 것이다. 추가로 TEXT나 BLOB은 아래와 같은 상황에서 사용한다.

- 컬럼 하나에 저장되는 문자열이나 이진 값의 길이가 예측할 수 없이 클 때 TEXT 또는 BLOB을 사용한다. 4000바이트를 넘는다고 무조건 사용해야하는 것은 아니다. 레코드 전체 크기가 64KB를 넘지 않는다면 VARCHAR, VARBINARY의 길이 제한이 없다. 
- MySQL 버전에 따라 차이는 있을 수 있지만 보통 레코드 최대 크기가 64KB다. 이 용량을 다썻다면 일부 컬럼을 TEXT, BLOB으로 변경하는 것을 고려해야한다.

추가로 BLOB, TEXT 크기가 크면 `max_allowed_packet` 변수 값보다 큰 문장은 서버로 전송되지 못할 수도 있다. 따라서 이런 경우 이 시스템 변수를 조정해야 한다. 

------

##공간 데이터 
### 공간 데이터 타입
MySQL에서 제공하는 공간 정보 저장용 데이터 타입은 POINT, LINESTRING, POLYGON, GEOMETRY, MULTIPOINT, MULTILINESTRING, MULTIPOLYGON, GEOMETRYCOLLECTION이다.
- POINT : 하나의 점 정보
- LINESTRING : 하나의 라인
- POLYGON : 하나의 다각형
- GEOMETRY : 위 타입들의 SUPER TYPE 그러나 하나의 객체만 저장할 수 있다. 
- MULTIPOINT : 내용은 같으나 여러 개를 저장할 수 있다.
- MULTILINESTRING : 내용은 같으나 여러 개를 저장할 수 있다.
- MULTIPOLYGON : 내용은 같으나 여러 개를 저장할 수 있다.
- GEOMETRYCOLLECTION : 내용은 같으나 여러 개를 저장할 수 있다.

GEOMETRY의 모든 자식은 서버의 메모리에는 BLOB 객체로 관리된다. 클라이언트 전송도 BLOB으로 된다. 결과적으로 GEOMETRY가 BLOB을 감싼다고 보면 된다.


## JSON 타입
5.7부터 JSON타입이 지원됐다. 이렇게 저장하면 MongoDB같이 바이너리 포맷(BSON)으로 변환해서 저장한다.
### 저장 방식
BSON으로 저장하기 때문에 BLOB, TEXT보다 공간 효율이 높은 편이다. MySQL에서는 큰 용량의 JSON이 저장되면 16KB로 쪼개서 저장한다.

### 부분 업데이트
8.0부터 JSON의 PartialUpdate가 지원된다. `JSON_SET()`, `JSON_REPLACE()`, `JSON_REMOVE()`로 이행할 수 있다.

```sql
UPDATE  tb_json
SET fd=JSON_SET(fd, '$.user_id', "1234")
WHERE id = 2;
```

여기서 정확히 부분 업데이트를 했는지는 알 수 없다. 하지만 `JSON_STORAGE_SIZE()`, `JSON_STORAGE_FREE()`로 대략 예측은 가능하다.


### JSON 타입 콜레이션과 비교
JSON 컬럼으로 저장되는 데이터는 모두 utf8mb4 charset, utf8mb4_bin collation 을 가진다.

### JSON 컬럼 선택
BLOB, TEXT에 JSON을 저장하면 변환 없이 그대로 저장한다. 그러나 JSON 타입은 JSON을 BSON으로 컴팩션해서 저장할 뿐만 아니라 필요한 경우 부분 업데이트도 지원한다.
그렇다면 일반적으로 정규화한 컬럼, JSON 컬럼 중에는 어떤 것을 사용해야 하는가? JSON으로만 구성된 테이블과 정규화된 컬럼으로 사용하는 테이블 예시를 보자

```sql
CREATE TABLE tb_json (
    doc JSON NOT NULL,
    id BIGINT AS (doc ->> `$.id`) STORED NOT NULL,
    PRIMARY KEY (id)
);

CREATE TABLE tb_column (
    id BIGINT NOT NULL,
    name VARCHAR(50) NOT NULL,
    PRIMARY KEY (id)
);
```

성능적인 부분에서 정규화된 컬럼이 낫다. 정규화된 컬럼을 컬럼 이름을 메타 정보로만 저장하기 떄문에 컬럼 이름이 별도 데이터 파일의 공간을 차지하지 않는다.
그러나 JSON은 매번 저장돼야 한다. 물론 압축을 사용해서 디스크 공간을 줄일 수는 있지만 메모리 사용 효율을 높이는 것은 아니다.

또한 MySQL에서 정규화된 컬럼 사용의 경우 BLOB, TEXT는 외부 페이지로 관리된다. 이러한 장점을 잘 활용하면 메모리 효율, 쿼리 성능을 올릴 수 있다. 
그러나  모든 데이터를 JSON 타입 컬럼에 저장하면 쿼리가 필요한 데이터를 선별적으로 접근할 수 있는 이점을 잃게 된다.

물론 장점도 있다. 레코드가 가지는 속성이 너무 상이하고 레코드별로 선택적 값을 가지는 경우라면 JSON 컬럼을 고려할 만하다. 또한 너무 정규화된 테이블 구조를 유지하면
테이블 개수가 늘고 그에 따라 응용 프로그램 코드도 비대해진다. 


## 가상 컬럼(파생 컬럼)
다른 DBMS는 `VirtualColumn`으로 부르고 MySQL은 `GeneratedColumn`이라고 명명했다. MySQL의 가상 컬럼은 크게 VirtualColumn, StoredColumn으로 나뉜다.
```sql
-- virtual
CREATE TABLE tb_virtual_column (
    id INT NOT NULL AUTO_INCREMENT,
    price DECIMAL(10,2) NOT NULL DEFUALT '0.00',
    quantity INT NOT NULL DEFAULT 1, 
    total_price DECIMAL(10,2) AS (quantity * price) VIRTUAL,
    PRIMARY KEY (id)
);

-- stored
CREATE TABLE tb_virtual_column (
   id INT NOT NULL AUTO_INCREMENT,
   price DECIMAL(10,2) NOT NULL DEFUALT '0.00',
   quantity INT NOT NULL DEFAULT 1,
   total_price DECIMAL(10,2) AS (quantity * price) STORED,
   PRIMARY KEY (id)
);
```

둘 다 `AS` 뒤 계산 식을 정의한다. 가상 컬럼의 기본 값은 VIRTUAL이다. 가상 컬럼의 표현식은 입력이 동일하면 시점과 관계없이 결과가 동일한(DETERMINISTIC) 표현식만 사용할 수 있다. 

1. 가상 컬럼
- 컬럼 값이 디스크에 저장되지 않는다.
- 컬럼 구조 변경은 테이블 리빌드를 필요로 하지 않는다.
- 컬럼은 레코드가 읽히기 전 또는 BEFORE 트리거 실행 직후 계산된다.

2. 스토어드 컬럼
- 물리적으로 저장된다.
- 컬럼 구조 변경은 테이블 리빌드를 수반한다.
- INSERT, UPDATE 시에 계산된다.

둘의 차이는 실제 저장 여부다. 가상 컬럼은 디스크에 저장되지는 않지만 항상은 아니다. 가상 컬럼에 인덱스를 생성하면 테이블 레코드는 가상 컬럼을 포함하지는 않지만
해당 인덱스는 계산된 값을 저장한다.그래스 인덱스가 생성된 가상 컬럼의 경우 필요하다면 인덱스 리빌드가 필요하다. 

두 가지 중 하나를 선택하는 경우는 계산 복잡도가 크면 STORED, 아니라면 VIRTUAL이 낫다 정도다. CPU 부하를 줘서 디스크 부하를 낮출 것이냐, 디스크 부하를 줘서 CPU 부하를 낮출 것이냐의 선택지다 
