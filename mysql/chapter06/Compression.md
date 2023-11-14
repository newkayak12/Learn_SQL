# 데이터 압축

디스크 데이터 파일이 크면 백업이 오래 걸리고, 복구도 그만큼 오래 걸린다. 또한 커리를 처리하기 위해서 더 많은 페이지를 innoDB로 읽어야 할 수도 있고
새로운 페이지가 버퍼 풀로 적재되기 때문에 그만큼 더티 페이지가 더 자주 디스크로 기록돼야한다. DBMS 이런 문제를 해결하기 위해서 데이터 압축을 제공한다.

## 페이지 압축
페이지 압축`Transparent Page Compression`은 MySQL 서버가 디스크에 저장하는 시점에 데이터 페이지가 압축되어 저장되고, 반대로 디스크에서 페이지를 
읽어올 떄 압축이 해제된다. 즉, 버퍼 풀에 데이터 페이지가 한 번 적재되면 innoDB 엔진은 압축이 해제된 상태로만 데이터 페이지를 관리한다. 

## 테이블 압축
테이블 압축은 운영체제나 하드웨어에 대한 제약 없이 사용할 수 있기 떄문에 일반적으로 더 활용도가 높다. 테이블 압축은 우선 디스크의 데이터 파일을 줄일 수 있다.
하지만 단점도 있는데

- 버퍼 풀 공간 활용률이 낮다.
- 쿼리 처리 성능이 낮다.
- 빈번한 데이터 변경 시 압축률이 떨어진다. 

### 압축 테이블 생성
테이블 압축을 사용하기 위해서는 테이블이 별도 테이블 스페이스를 사용해야 한다. `innodb_file_per_table`이 ON으로 설정된 상태에서 테이블이 생성돼야 한다.
테이블 압축을 사용하는 테이블은 `ROW_FORMAT=COMPRESSED`를 명시해야 한다. 추가로 `KEY_BLOCK_SIZE` 옵션으로 압축되페이지의 타깃 크기를 명시하는데, 
2진수로만 설정할 수 있다. 

```mysql
SET GLOBAL innodb_file_per_table=ON;

CREATE TABLE compressed_table (
    c1 INT PRIMARY KEY 
)
ROW_FORMAT=COMPRESSED
KEY_BLOCK_SIZE=8;

-- ROW_FORMAT 명시 없으면 COMPRESSED
-- KEY_BLOCK_SIZE=8;는 압축된 페이지가 저장될 페이지 크기를 지정한 것이다.
-- 너무 작으면 분할하는 작업으로 처리 성능이 떨어질 수 있다.
```

### KEY_BLOCK_SIZE 결졍
테이블 압축에서 중요한 부분은 압축된 결과 사이즈를 예측해서 `KEY_BLOCK_SIZE`를 정하는 것이다. 작은 값에서 서서히 늘리는 것이 좋다. 

### 압축된 페이지의 버퍼 풀 적재 및 사용
innoDB 엔진은 압축된 테이블의 데이터 페이지를 버퍼 풀에 적재하면 압추고딘 상태와 압축이 해제된 상태 2개 버전을 관리한다. 그래서 디스크에서 읽은 상태 그대로의
데이터 페이지 목록을 관리하는 LRU, 압축된 페이지들의 압축 해제 버전인 Unzip_LRU를 별도로 관리한다. 

MySQL은 압축된 테이블, 압축되지 않은 테이블이 공존하므로 LRU 리스트는 아래와 같이 압축된 페이지와 압축되지 않은 페이지를 모두 가질 수 있다. 
- 압축이 적용되지 않은 테이블의 데이터 페이지
- 압축이 적용된 테이블의 데이터 페이지

Unzip_LRU는 압축이 적용되지 않은 테이블의 데이터 페이지는 가지지 않으면 압축이 적용된 테이블에서 읽은 데이터 페이지만 관리한다. 물론 Unzip_LRU 리스트에는
압축을 해제한 상태의 데이터 페이지 목록이 관리된다.

결국 innoDB는 압축된 테이블에 대해서는 버퍼 풀의 공간을 이중으로 사용함으로써 메모리를 낭비하는 효과를 가진다. 또 다른 문제점으로는 압축된 페이지에서
데이터를 읽거나 변경하기 위해서는 압축 해제가 필요한데, 압축 해제는 CPU를 상대적으로 많이 소모한다. 이러한 두 가지 단점을 보완하기 위해서 Unzip_LRU 리스트를
별도로 관리하고 있다가 MySQL 유입 요청 패턴에 따라서 적절히(Adaptive) 요청을 처리한다. 

- innoDB 버퍼 풀의 공간이 필요한 경우에는 LRU 리스트에서 원본 데이터 페이지( 압축된 )는 유지하고 Unzip_LRU 리스트에서 압출 해제된 버전을 제거해서 버퍼 풀 공간을 확보한다.
- 압축된 데이터 페이지가 자주 사용되는 경우, Unzip_LRU에 압축 해제된 페이지를 유지하면서 압축 및 압축 해제 작업을 최소화 한다. 
- 압축된 데이터 페이지가 사용되지 않아서 LRU 리스트에서 제거되면 Unzip_LRU도 함께 제거된다. 

innoDB 엔진은 버퍼 풀에서 압축 해제된 버전의 데이터를 적절한 수준으로 유지하기 위해서 아래와 갑은 적응형 알고리즘을 사용한다.

- CPU 사용량이 높은 서버에서는 가능한 압축과 해제를 피해기 위해서 Unzip_LRU 비율을 높여서 유지한다.
- DISK IO가 높으면 Unzip_LRU를 낮춰서 innoDB 버퍼풀 공간을 더 확보하도록 한다.



### 테이블 압축 관련 설정
테이블 압축을 사용할 떄 연관된 시스템 변수가 몇 가지 있는데, 모두 페이지의 압축 실패율을 낮추기 위한 튜닝 포인트를 제공한다. 

- innodb_cmp_index_enabled: 테이블 압축이 사용된 테이블의 모든 인덱스별로 압축 성공 및 압축 실행 횟수를 수집하도록 설정한다. `innodb_cmp_per_index_enabled`
이 비활성화되면 테이블 단위 압축 성공 및 압축 실행 횟수만 수집된다. 테이블 단위 수집된 정보는 information_schema.INNODB_CMP 테이블이 기록되며,
인덱스 단위로 수집된 정보는 information_schema.INNODB_CMP_PER_INDEX 테이블에 기록된다.
- innodb_compression_level: innoDB 압축은 zlib 압축 알고리즘을 지원하는데 이 변수로 압축률을 지정할 수 있다. 0 ~ 9까지의 값이다. 기본값은 6이다.
- innodb_compression_failure_threshold_pct와 innodb_compression_pad_pct_max: 테이블 단위로 압축 실패율이 `innodb_compression_failure_threshold_pct`
보다 크면 압축 실행 전 원본 데이터 페이지의 끝에 의도적으로 빈 공간을 추가한다. 추가된 빈 공간은 압축률을 높여서 압축 결과가 `KEY_BLOCK_SIZE`보다 작아지게 만드는
효과를 낸다. 여기서 추가하는 공간을 패딩(Padding)이라고 하고, 패딩은 압축 실패율이 높아질수록 증가된 크기를 가진다. 추가할 수 있는 최대 패딩 크기는 `innodb_compression_pad_pct_max`
를 넘을 수 없다. 
- innodb_log_compressed_pages: MySQL이 비정상 종료했다 재기동 되면 압축 알고리즘 버전 차이가 있더라도 복구가 실패하지 않도록 압축된 데이터 페이지를 그대로
리두 로그에 기록한다. 압축 알고리즘 업그레이드에는 도움이 되지만, 데이터 페이지를 통째로 리두 로그에 추가하는 것은 리두 로그 증가량이 영향을 미친다. 압축 적용 후 
리두 로그 용량이 급증한다거나 버퍼 풀로부터 더티 페이지가 한꺼번에 많이 기록된다면 `innodb_log_compressed_pages`를 OFF로 하고 모니터링해보자.
