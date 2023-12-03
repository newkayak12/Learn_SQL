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


#### utf8mb4 문자 집합의 콜레이션