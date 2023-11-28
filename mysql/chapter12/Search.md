# 확장 검색
## 전문 검색
MySQL은 용량이 큰 문서를 단어 수준으로 잘개 쪼개서 검색해주는 기능이 있었다. 이를 전문 검색이라고 한다.
문서의 단어들을 분리해서 형태소를 찾고 그 형태소를 인덱싱하는 방법은 동양권에서는 무리가 있다. 그래서 8.0에서는 형태소, 어원과 관계 없이 특정 길이의 조각(Token)으로 
인덱싱하는 `n-gram`파서도 도입됐다. 

앞서 밝힌 바와 같이 

1. 형태소 분석
2. n-gram 파서

두 가지를 지원한다. 형태소 분석은 문장의 공백, 띄어쓰기로 단어를 분리하고 각 단어의 조사를 제거해서 명사 또는 어근을 찾아서 인덱싱하는 알고리즘이다. 하지만 MySQL에서는
단순히 공백과 같은 띄어쓰기 기준으로 토큰을 분리해서 인덱싱한다. 즉, MySQL에서는 형태소 분석이나 어근 분석 기능은 구현돼 있지 않다. 그리고 n-gram은
문장 자체에 대한 이해 없이 공백과 같은 띄어쓰기 단위로 단어를 분석하고, 그 단어를 단순이 주어진 길이 (n-gram (1 ~ 10))로 쪼개서 인덱싱하는 알고리즘이다.

n-gram의 n은 `ngram_token_size`로 변경할 수 있다. 기본 값은 2이고 1 ~ 10 사이 수를 설정할 수 있다.
```sql
SELECT COUNT(*)
FROM tb_bi_gram
WHERE MATCH ( title, body ) AGAINST ('단편적인', IN BOOLEAN MODE);
```

### 전문 검색 쿼리 모드
MySQL 서버의 전문 검색 쿼리는 자연어(NATURAL LANGUAGE) 검색 모드와 불리언(BOOLEAN) 검색 모드를 지원한다.

#### 자연어 검색(NATURAL LANGUAGE MODE)
```sql
SELECT id, title, body, MATCH( title, body ) AGAINST ('MySQL' IN NATURAL LANGUAGE MODE ) as score
FROM tbl_bi_gram
WHERE MATCH( title, body ) AGAINST ( 'MySQL' IN NATURAL LANGUAGE MODE );
```


#### 불리언 검색(BOOLEAN MODE)
자연어 검색은 검색어에 포함된 단어들이 존재하는 결과만 가져오지만, 불리언 검색은 쿼리에 사용되는 검색어의 존재 여부에 대해서 논리 연산이 가능하다.
```sql
SELECT id, title, body,
       MATCH( title, body ) AGAINST ('+MySQL -manual' IN BOOLEAN MODE ) as score
FROM tbl_bi_gram
WHERE MATCH( title, body ) AGAINST ('+MySQL -manual' IN BOOLEAN MODE );
```

#### 검색어 확장(Query Expansion)
검색어 확장은 사용자가 쿼리에 사용한 검색어로 검색된 결과에서 공통으로 발견되는 단어들을 모아서 다시 한 번 더 검색을 수행하는 방식이다.
```sql
SELECT * FROM tb_bi_gram
WHERE MATCH(title, body) AGAINST ('database', WITH QUERY EXPANSION);
```

#### 전문 검색 인덱스 디버깅
MySQL는 최근 innoDB에서도 전문 검색 인덱스를 사용할 수 있게 개선됐다. 이로 인해 불용어나 토큰 파서 등의 기능을 제어하는 시스템 변수가 다양해졌다. 
```sql
SET GLOBAL innodb_ft_aux_table = 'test/tb_bi_gram'

SELECT * FROM information_schema.innodb_ft_config; -- 전문 검색 인덱스의 설정 내용을 보여준다. 

SELECT * FROM information_schema.innodb_ft_index_table;
-- 전문 검색 인덱스가 가지고 있는 인덱스 엔트리의 목록을 보여준다. 전문 검색 인덱스의 각 엔트리는
-- 토큰들이 어떤 레코드에 몇 번 사용했는지, 그리고 레코드별로 문자 위치가 어디인지 등의 정보를 관리한다. 

SELECT * FROM information_schema.innodb_ft_index_cache;
-- 테이블에 레코드가 새롭게 INSERT되면 MySQL은 전문 검색 인덱스를 위한 토큰을 분리해서 즉시 디스크로 저장하지 ㅇ낳고 메모리에 임시로 저장한다.
-- 이때 임시로 저장되는 공간이 innodb_ft_index_cache 테이블이다. 그리고 이 테이블이 사용하는 메모리 공간이 innodb_ft_cache_size 크기를 넘어서면
-- 한꺼번에 모아서 디스크 파일로 저장한다. 길이가 긴 텍스트 컬럼에 대해서 인덱스 전문을 생성하면 innodb_ft_cache_size를 늘리는 것이 좋을 수도 있다.

SELECT * FROM information_schema.innodb_ft_deleted;
-- 테이블의 레코드가 삭제되면 어떤 레코드가 삭제됐는지, 그리고 어떤 레코드가 현재 전문 검색 인덱스에서 삭제되고 있는지를 보여준다.
```


#### 공간 검색

##### 용어
> OGC(Open Geospatial Consortium)
> OGC는 위치 기반 데이터 표준을 수립하는 단체
> 
> OpenGIS
> OGC에서 제정한 지리 정보 시스템(GIS, Geographic Information System) 표준
> 
> SRS, GCS, PCS
> SRS(Spatial Reference System) : 좌표계
>   - GCS(Geographic Coordinate System) : 지구 구체상의 특정 위치나 공간을 표현하는 좌표계 (위도, 경도, 각도 단위의 숫자로 표시)
> 
>   - PCS(Projected Coordinate System) : 지도와 같은 평면으로 Projection시킨 좌표계를 의미 미터와 같은 선형적인 단위로 표시
> 
> SRID, SRS-ID
> Sptial Reference ID :  SRS를 지칭하는 고유 번호 SRS-ID와 SRID는 동의어
> 
> WKT, WKB
> WKT(Well-Known Text format), WKB(Well-Known Binary format) OpenGIS에서 명시한 좌표 표현 방법
> 
> MBR, R-Tree
> MBR(Minimum Bounding Rectangle) : 어떤 도형을 감싸는 최소의 사각 상자 MySQL의 공간 인덱스가 도형의 포함 관계를 이용하기 때문이다. 이렇게 만들어진 인덱스를 R-tree라고 한다.
> 

```sql
DESC information_schema.ST_SPATIAL_REFERENCE_SYSTEMS;
+--------------------------+---------------+------+-----+---------+-------+
| Field                    | Type          | Null | Key | Default | Extra |
+--------------------------+---------------+------+-----+---------+-------+
| SRS_NAME                 | varchar(80)   | NO   |     | NULL    |       |
| SRS_ID                   | int unsigned  | NO   |     | NULL    |       |
| ORGANIZATION             | varchar(256)  | YES  |     | NULL    |       |
| ORGANIZATION_COORDSYS_ID | int unsigned  | YES  |     | NULL    |       |
| DEFINITION               | varchar(4096) | NO   |     | NULL    |       |
| DESCRIPTION              | varchar(2048) | YES  |     | NULL    |       |
+--------------------------+---------------+------+-----+---------+-------+
6 rows in set (0.01 sec)

```