# 아키텍쳐

![](img/img.png)

## MySQL 엔진
클라이언트로부터 접속 및 쿼리 요청을 처리하는 커넥션 핸들러, SQL 파서, 전처리기, 쿼리 옵티마이저가 중심을 이룬다. 

## 스토리지 엔진
MySQL 엔진은 요청을 분석, 최적화 하는 등의 처리를 하고 실제 데이터를 디스크 스토리지에 저장하거나 디스크 스토리지로부터 데이터를 읽어 오는 부분은 스토리지 엔진이 전담한다.
MySQL 엔진은 하나지만 스토리지 엔진은 여러 개를 동시에 사용할 수 있다.

## 핸들러 API
MySQL 엔진의 쿼리 실행기에서 데이터를 쓰거나 읽어야 할 때는 각 스토리지 엔진에 쓰기 또는 읽기를 요청하는데 이를 핸들러 요청이라고 한다.
여기서 사용되는 API를 핸들러 API라고 한다. InnoDB도 이 핸들러 API를 이용해서 MySQL 엔진과 데이터를 주고 받는다.

 `SHOW GLOBAL STATUS LIKE 'Handler%';`

| Variable\_name | Value |
| :--- | :--- |
| Handler\_commit | 614 |
| Handler\_delete | 8 |
| Handler\_discover | 0 |
| Handler\_external\_lock | 6421 |
| Handler\_mrr\_init | 0 |
| Handler\_prepare | 0 |
| Handler\_read\_first | 41 |
| Handler\_read\_key | 1760 |
| Handler\_read\_last | 0 |
| Handler\_read\_next | 4088 |

## MySQL 쓰레딩 구조
![](img/thread.png)

프로세스 기반이 아닌 쓰레드 기반으로 작동한다. 크게 포그라운드(Foreground), 백그라운드(Background) 쓰레드로 구분할 수 있다. 

### Foreground Thread(Client Thread)
포그라운드 쓰레드는 최소한 MySQL 서버에 접속된 클라이언트의 수만큼 존재한다. 주로 사용자가 요청하는 쿼리 문장을 처리한다. 클라이언트 사용자가 작업을 마치고 커넥션을 종료하면
해당 커넥션을 담당하던 쓰레드는 다시 쓰레드 캐시(ThreadCache)로 돌아간다. 만일 쓰레드 캐시에 일정 개수 이상의 대기 중인 쓰레드가 있으면 쓰레드 캐시에 넣지 않고
쓰레드를 종료시켜 일정 개수의 쓰레드만 쓰레드 캐시에 존재하게 한다. 이때 쓰레드 캐시에 유지할 수 있는 최대 쓰레드 개수는 `thread_cache_size` 시스템 변수로 설정한다.

포그라운드 쓰레드는 데이터를 MySQL의 데이터 버퍼나 캐시로부터 가져오며, 버퍼나 캐시에 없는 경우 직접 디스크의 데이터나 인덱스 파일로부터 데이터를 읽어와 처리한다. 
MyISAM은 디스크 쓰기까지 포그라운드가 처리하지만 InnoDB는 데이터 버퍼나 캐시까지만 포그라운드, 버퍼에서 디스크는 백그라운드 쓰레드가 처리한다.


### Background Thread
MyISAM은 해당 없다. InnoDB는 아래와 같은 경우가 백그라운드로 처리한다.
- insert buffer를 병합하는 쓰레드
- 로그를 디스크로 기록하는 쓰레드
- innoDB 버퍼 풀의 데이터를 디스크에 기록하는 쓰레드
- 데이터를 버퍼로 읽어 오는 쓰레드
- 잠금이나 데드락을 모니터링하는 쓰레드

가장 중요한 것은 로그 쓰레드(Log Thread)와 버퍼의 데이터를 디스크로 내려쓰는 작업을 처리하는 쓰기 쓰레드일 것이다.
데이터 쓰기, 데이터 읽기 쓰레드 수를 지정할 수 있다. (MySQL 5.5 이상) `innodb_write_io_threads`, `innodb_read_io_threads` 시스템 변수로 쓰레드 개수를 설정한다.
InnoDB에서 읽는 작업은 주로 클라이언트 쓰레드에서 처리되기 떄문에 읽기 쓰레드는 많이 설정할 필요가 없지만 쓰기 쓰레드는 대부분을 백그라운드로 처리하기 떄문에
디스크를 최적으로 사용할 수 있을 만큼 설정하는 것이 좋다. 

사용자 요청을 처리하는 중, 쓰기는 지연 처리할 수 있지만(버퍼링) 데이터 읽기는 지연될 수 없다. 그래서 상용 DBMS는 쓰기는 버퍼링해서 일괄처리하는 기능이 탑재되어 있으며, 
InnoDB 또한 이런 방식으로 처리한다. MyISAM은 사용자 쓰레드가 쓰기까지 처리한다. 그래서 실질 디스크 쓰기까지 기다려야 한다.

## 메모리 할당 및 사용 구조
MySQL 메모리 공간은 글로벌/ 로컬 영역으로 구분된다. 글로벌 메모리 영역의 모든 메모리 공간은 MySQL 서버가 시작되면서 운영체제로부터 할당된다.
글로벌 메모리 영역과 로컬 메모리 영역은 MySQL 서버 내에 존재하는 많은 쓰레드가 공유해서 사용하는 공간인지 여부에 따라 구분된다.

### 글로벌 메모리 영역
일반적으로 클라이언트 쓰레드의 수와 무관하게 하나의 메모리 공간만 할당된다. 생성된 글로벌 영역이 N개라고 해도 모든 쓰레드에 의해서 공유된다. 글로벌 메모리 영역은 대표적으로 다음과 같다.
- 테이블 캐시
- InnoDB 버퍼 풀
- InnoDB 어댑티브 해시 인덱스
- InnoDB 리두 로그 버퍼

### 로컬 메모리 영역
세션 메모리 영역이라고 표현하며 클라이언트 쓰레드가 쿼리를 처리하는데 사용하는 메모리 영역이다. 
- 정렬 버퍼
- 조인 버퍼
- 바이너리 로그 캐시
- 네트워크 버퍼

클라이언트 커넥션으로부터의 요청을 처리하기 위해 쓰레드를 하나씩 할당하게 되는데, 클라이언트 쓰레드가 사용하는 메모리 공간이라고 해서 클라이언트 메모리 영역이라고도 한다.
로컬 메모리는 각 클라이언트 쓰레드별로 독립적으로 할당되며 절대 공유되어 사용되지 않는다는 특징이있다. 또한, 로컬 메모리 공간은 각 쿼리의 용도별로 필요할 때만 공간이 할당되고
필요하지 않은 경우에는 MySQL이 메모리 공간을 할당조차 하지 않을 수도 있다. 예를 들어 정렬 버퍼, 조인 버퍼와 같은 공간이 그러하다.
로컬 메모리 공간은 커넥션이 열려 있는 동안 계속 할당된 상태로 남아 있는 공간도 있고( 커넥션 버퍼, 결과 버퍼 ) 그렇지 않고 쿼리를 실행하는 순간에만 할당했다가
다시 해제하는 공간( 정렬 버퍼, 조인 버퍼 )도 있다.


## 플러그인 스토리지 엔진 모델
MySQL에서는 플로그인 모델을 사용할 수 있다. 플러그인해서 사용할 수 있는 것은 스토리지 엔진만이 아니다. 스토리지 엔진 이외에도 부가적인 기능을 커스터마이징 해서 사용할 수 있다.
MySQL 플러그인을 사용한다고 해도 대부분의 작업이 MySQL 엔진에서 처리되고, 마지막 '데이터 읽기/쓰기' 작업만 스토리지 엔진에 의해 처리된다. (일부분의 기능만 실행된다.)

`SHOW ENGINES;`

| Engine | Support | Comment | Transactions | XA | Savepoints |
| :--- | :--- | :--- | :--- | :--- | :--- |
| ndbcluster | NO | Clustered, fault-tolerant tables | NULL | NULL | NULL |
| FEDERATED | NO | Federated MySQL storage engine | NULL | NULL | NULL |
| MEMORY | YES | Hash based, stored in memory, useful for temporary tables | NO | NO | NO |
| InnoDB | DEFAULT | Supports transactions, row-level locking, and foreign keys | YES | YES | YES |
| PERFORMANCE\_SCHEMA | YES | Performance Schema | NO | NO | NO |
| MyISAM | YES | MyISAM storage engine | NO | NO | NO |
| ndbinfo | NO | MySQL Cluster system information storage engine | NULL | NULL | NULL |
| MRG\_MYISAM | YES | Collection of identical MyISAM tables | NO | NO | NO |
| BLACKHOLE | YES | /dev/null storage engine \(anything you write to it disappears\) | NO | NO | NO |
| CSV | YES | CSV storage engine | NO | NO | NO |

Support 컬럼
- YES : MySQL 서버에 해당 스토리지 엔진이 포함되어 있고, 사용 가능으로 활성화된 상태
- DEFAULT : YES와 같지만 필수 스토리지 엔진이다. (이 엔진이 없으면 MySQL이 작동하지 않을 수도 있다.)
- NO : 현재 mysqld에 포함되어 있지 않다.
- DISABLED : mysqld에 있지만 파라미터에 의해 비활성됨

No를 사용하려면 MySQL 서버를 다시 빌드해야 한다. 물론 준비만 되어 있다면 플러그인 형태로 빌드된 스토리지 엔진 라이브러리를 끼워넣기만 해도 사용할 수 있다.


`SHOW PLUGINS;`

| Name | Status | Type | Library | License |
| :--- | :--- | :--- | :--- | :--- |
| binlog | ACTIVE | STORAGE ENGINE | NULL | GPL |
| mysql\_native\_password | ACTIVE | AUTHENTICATION | NULL | GPL |
| sha256\_password | ACTIVE | AUTHENTICATION | NULL | GPL |
| caching\_sha2\_password | ACTIVE | AUTHENTICATION | NULL | GPL |
| sha2\_cache\_cleaner | ACTIVE | AUDIT | NULL | GPL |
| daemon\_keyring\_proxy\_plugin | ACTIVE | DAEMON | NULL | GPL |
| CSV | ACTIVE | STORAGE ENGINE | NULL | GPL |
| MEMORY | ACTIVE | STORAGE ENGINE | NULL | GPL |
| InnoDB | ACTIVE | STORAGE ENGINE | NULL | GPL |
| INNODB\_TRX | ACTIVE | INFORMATION SCHEMA | NULL | GPL |

## 컴포넌트
MySQL 8.0부터는 플러그인 아키텍처를 컴포넌트가 아키텍처가 대체한다. 컴포넌트는 플러그인의 단점을 보완해서 구현했다.

- 플러그인은 오직 MySQL 서버와 인터페이스할 수 있고, 플러그인끼리는 통신할 수 없다. 
- 플러그인은 MySQL 서버의 변수나 함수를 직접 호출하기 떄문에 안전하지 않다.(캠슐화가 안 됨)
- 플러그인은 상호 의존 관계를 설정할 수 없어 초기화가 어렵다.


## 쿼리 실행 구조
### 쿼리 파서
쿼리 파서는 사용자 요청이 들어온 쿼리 문장을 토큰(MySQL이 인식할 수 있는 최소 단위의 어휘나 기호)로 분리해 트리 형태의 구조로 만들어 내는 작업을 의미한다. 
쿼리 문장의 기본 문법 오류는 이 과정에서 발견되고, 사용자에게 오류 메시지를 던진다.

### 전처리기
파서 과정에서 만들어진 파서 트리를 기반으로 쿼리 문장에 구조적인 문제점이 있는지 확인한다. 각 토큰을 테이블 이름이나 컬럼 이름, 또는 내장 함수와 같은
개체를 매핑해 해당 객체 존재 여부와 객체의 접근 권한 등을 확인하는 과정을 수행한다. 실제 존재하지 않거나 권한상 사용할 수 없는 개체의 토큰은 이 단계에서 걸러진다. 

### 옵티마이저
사용자 요청을 저렴한 비용으로 가장 빠르게 처리할수 있도록 최적하는 역할을 담당한다.

### 실행 엔진
옵티마이저가 두뇌라면 실행 엔진과 핸들러는 손과 발(실무자)에 비유할 수 있다. 옵티마이저가 `GROUP BY`를 처리하기 위해서 임시 테이블을 사용하기로 결정했다고 해보자.
1. 실행 엔진이 핸들러에게 임시 테이블을 만들라고 요청
2. 실행 엔진은 WHERE 절에 일치하는 레코드를 읽어오라고 핸들러에게 요청
3. 읽어온 레코드들을 1번에서 준비한 임시테이블이 저장하라고 핸들러에 요청
4. 데이터가 준비된 임시 테이블에서 필요한 방식으로 데이터를 읽어오라고 핸들러에게 다시 요청
5. 최종적으로 실행 엔진은 결과를 사용자나 다른 모듈로 넘김

### 핸들러(스토리지 엔진)
MySQL 실행 엔진의 요청에 따라 데이터를 디스크로 저장하고 디스크로부터 읽어 오는 역할을 담당한다. 핸들러는 결국 스토리지 엔진을 의미하며, MyISAM  테이블을 조작하면 핸들러가
MyISAM, InnoDB 테이블은 핸들러가 InnoDB 스토리지 엔진이 된다. 


## 복제
MySQL에서 Replication은 매우 중요한 역할을 담당한다. 

## 쿼리 캐시
MySQL에서 QueryCache는 빠른 응답을 필요로하는 웹 기반의 응용프로그램에서 중요한 역할을 담당했다. 쿼리 캐시는 SQL 실행 결과를 메모리에 캐시하고, 동일 SQL 쿼리가
실행되면 테이블을 읽지 않고 즉시 겨로가를 반환하기 떄문에 매우 빠른 성능을 보였다. 그러나 캐시된 내용 중 변경된 내용들이 포함되어 있는 경우 invalidate해야 했다. 
이는 심각한 동시 처리 성능 저하를 유발한다. 이런 이유로 MySQL 8.0에서는 QueryCache가 제거됐다.

## 쓰레드 풀
Enterprise는 ThreadPool을 제공한다. 쓰레드 풀은 내부적으로 사용자의 요청을 처리하는 쓰레드 개수를 줄여서 동시 처리되는 요청이 많다고 하더라도 MySQL 서버의 CPU가 제한된 개수의
쓰레드 처리에만 집중할 수 있게 해서 자원 소모를 줄이는 것이 목적이다. 쓰레드 풀이 만능은 아니다. 쓰레드 풀은 동시에 실행 중인 쓰레드들을 CPU가 최대한 잘 처리할 수 있는 정도로
줄여서 빨리 처리하도록 하는 기능이기 때문에 스케쥴링 과정에서 CPU 시간을 확보하지 못하면 쿼리 자체가 더 느려질 수도 있다. 물론 제한된 수의 쓰레드만으로 CPU가 처리되도록 하면
CPU 프로세서 진화도를 높이고 OS입장에서도 contextSwitch를 줄여서 오버헤드를 낮출 수 있다. 

일반적으로 CPU 코어 개수와 풀 개수를 맞추는 식으로 진행한다. MySQL 서버가 처리해야할 요청이 생기면 쓰레드 풀로 처리를 이관하는데, 만약 이미 쓰레드 풀이 처리 중인 
작업이 있으면 thread_pool_oversubscribe (default : 3)만큼 더 받아서 처리한다(큐의 개념)
...

## 트랜잭션 지원 메타데이터
DB 서버에서 테이블의 구조 정보와 스토어드 프로그램 등의 정보를 데이터 딕셔너리 또는 메타데이터라고 하는데, 5.7까지는 테이블 구조를 FRM에 저장하고 일부 스토어드 프로그램 또한 (.TRN, .TRG, .PAR )
기반으로 관리했다. 하지만 이런 파일 기반의 메타데이터는 생성, 변경 작업이 트랜잭션을 지원하지 않기 때문에 생성, 변경 중 MySQL 서버가 비정상 종료되면 일관되지 않은 상태로 
남는 문제가 있었다. ('데이터베이스나 테이블이 깨졌다.'라고 표현한다.)

MySQL 8.0 부터는 테이블 구조 정보나 스토어드 프로그램의 코드 관련 정보를 InnoDB 테이블에 저장하도록 변경됐다. 이런 테이블을 시스템 테이블이라고 한다. 시스템 테이블과 데이터
딕셔너리 정보를 모두 모아서 mysql DB에 저장하고 있다. mysql DB는 통째로 mysql.ibd라는 테이블 스페이스에 저장한다. 그래서 MySQL 서버의 데이터 디렉토리에 존재하는
mysql.ibd는 다른 .ibd와 함께 주의해야 한다.

MyISAM이나 CSV 등과 같은 스토리지 엔진의 메타 정보는 여전히 저장할 공간이 필요하다. MySQL 서버는 InnoDB 스토리지 엔진 이외의 스토리지 엔진을 사용하는
테이블들을 위해서 SDI(Serialized Dictionary Information) 파일을 사용한다. InnoDB 이외 테이블들에 대해서 SDI 포맷의 *.sdi가 존재하며 *.FRM과 동일한 역할을 한다.
또한 SDI 이름 그대로 직렬화를 위한 포맷이므로 InnoDB 테이블들의 구조도 SDI 파일로 변환할 수 있다.


## InnoDB 스토리지 엔진 아키텍쳐