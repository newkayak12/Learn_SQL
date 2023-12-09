# Performance Schema & Sys Schema
어떤 경우든지 사용 중인 DB 성능을 향상시켜야 한다면 가장 먼저 해야할 일은 현재 DB에 대해서 분석하고 튜닝포인트를 찾는 것이다.
MySQL에서는 이런 이유로 상세한 정보를 수집하서 한 곳에 모아서 손쉽게 접근해서 확인할 수 있게 하는 기능을 제공한다. Performance와 Sys가 그 예시다.
사용자는 이를 통해 일반 테이블에 저장된 데이터를 조회하는 것처럼 SQL문을 사용해서 수집된 정보를 조회할 수 있다. 

## Performance
서버 내부 동작 및 쿼리 처리와 관련된 세부 정보들이 저장되는 테이블들이 존재한다. 두 가지로 분류하면 Performance 스키마 설정과 관련된 테이블과 Performance 스키마가 수집한 데이터가 저장되는 테이블로 나눌 수 있다.

###  1. Setup 테이블 
 
 Performance 스키마의 데이터 수집 및 저장과 관련된 설정 정보가 정돼 있으며, 사용자는 이 테이블을 통해서 Performance 스키마 설정을 동적으로 변경할 수 있다.

- setup_actors : Performance 스키마가 모니터링하여 데이터를 수집할 대상 유저 목록이 저장돼 있다.
- setup_consumers : Performance 스키마가 얼마나 상세한 수준으로 데이터를 수집하고 저장할 것인지를 결정하는 데이터 저장 레벨 설정이 저장돼 있다.
- setup_instruments : Performance 스키마가 데이터가 수집할 수 있는 MySQL 내부 객체들이 클래스 목록과 클래별 데이터 수집 여부 설정이 저장돼 있다. 
- setup_objects : performance 스키마가 모니터링하여 데이터를 수집할 대상 데이터베이스 객체(프로시저, 테이블, 트리거 등과 같은) 목록이 저장돼 있다.
- setup_threads : Performance 스키마에서 모니터링하여 데이터를 수집할 수 있는 MySQL 내부 쓰레드들의 몰골과 쓰레드별 데이터 수집 여부 설정이 저장돼 있다.

### 2. Instance 테이블

 Performance 스키마가 데이터를 수집하는 대상인 실체화된 객체들, 즉 인스턴스들에 대한 정보를 제공하며, 인스턴스 종류별로 테이블이 구분돼 있다. 

- cond_instances : 서버에서 동작 중인 쓰레드들이 대기하는 조건(Condition) 인스턴스들의 목록을 확인할 수 있다. 조건은 쓰레드 간 동기화 처리와 관련해서 특정 이벤트가 발생했음을 알리기 위해 사용되는 것으로, 쓰레드들은 자신이 기다리고 있는 조건이 참이 되면 작업을 재개한다.
- file_instances : 서버가 열어서 사용 중인 파일들의 목록을 확인할 수 있다. 사용하던 파일이 삭제되면 이 테이블에서도 데이터가 삭제된다.
- mutex_instances : 서버에서 사용 중인 mutex 인스턴스 목록을 확인할 수 있다.
- rwlock_instances : 서버에서 사용 중인 읽기 잠금 및 쓰기 잠금 인스턴스들의 목록을 확인할 수 있다.
- socket_instances : 클라이언트 요청을 대기하고 있는 소켓 인스턴스들의 목록을 확인할 수 있다.  

### 3. Connection 테이블

  생성 커넥션에 대한 통계 및 속성 정보를 제공한다. 

- accounts : DB 계정명과 MySQL 서버로 연결한 클라이언트 호스트 단위의 커넥션 통계 정보를 확인할 수 있다.
- hosts : 호스트별 커넥션 통계정보를 확인할 수 있다.
- users : DB 계정명별 커넥션 통계 정보를 확인할 수 있다.
- session_account_connection_attr : 현재 세션 및 현재 세션에서 MySQL에 접속하기 위해 사용한 DB 계정과 동일한 계정으로 접속한 다른 커넥션 정보를 확인할 수 있다.
- session_connect_attrs : MySQL에 연결된 전체 세션들의 커넥션 속성 정보를 확인할 수 있다.

### 4. Variable 테이블

  시스템 변수 및 사용자 정의 변수와 상태 변수들에 대한 정보를 제공한다.

- global_variables : 전역 시스템 변수들에 대한 정보가 저장돼 있다.
- session_variables: 현재 세션에 대한 세션 범위의 시스템 변수들 정보가 저장돼 있으며, 현재 세션에 설정한 값들을 확인할 수 있다.
- variables_by_thread : 현재 MySQL에 연결돼 있는 전체 세션에 대한 세션 범위의 시스템 변수들의 정보가 저장돼 있다.
- persisted_variables : SET PERSIST 또는 SET PERSIST_ONLY 구문을 통해서 영구적으로 설정된 시스템 변수에 대한 정보가 저장돼 있다. persisted_variables 테이블은 mysql-auto.cnf 파일에 저장돼 있는 내용을 테이블 형태로 나타낸 것으로, 사용자가 SQL문으로 해당 파일을 수정할 수 있게 한다.
- variables_info : 전체 시스템 변수에 대해 설정 가능한 값 범위 및 가장 최근에 변수의 값을 변경한 계정 정보 등이 저장돼 있다.
- user_variables_by_thread : 연결돼 있는 세션에서 생성한 사용자 변수들에 대한 정보
- global_status : 전역 상태 변수들에 대한 정보
- session_status : 현재 세션에 대한 세션 범위의 상태 변수드르이 정보가 저장돼 있다.
- status_by_thread : 서버에 연결돼 있는 전체 세션들에 대한 세션 범위의 상태 변수들의 정보가 저장돼있으며, 세션벼롤 구분될 수 있는 상태 변수만 저장된다. 

### 5. Event 테이블

  Event는 Wait, Stage, Statment, Transaction으로 구분돼 있다. 

> Transaction Events
> > Statement Events
> > > Stage Events
> > > > Wait Events (io, lock, sync ...)

 또한 각 이벤트는 세 가지 유형의 테이블을 가지는데 테이블 명 후미에 해당 테이블에 속해있는 유형의 이름이 표시된다. 

- current : 쓰레드 별로 가장 최신의 이벤트 1 건만 저장되며, 쓰레드가 종료되면 해당 쓰레드의 이벤트 데이터는 바로 삭제된다.
- history : 쓰레드 별로 가장 최신의 이벤트가 저장된 최대 개수만큼 저장된다. 쓰레드가 종료되면 해당 쓰레드의 이벤트 데이터는 바로 삭제되며, 쓰레드가 계속 사용 중이면서 쓰레드별 최대 저장 개수를 넘은 경우 이전 이벤트를 삭제하고 최근 이벤트를 새로 저장함으로써 최대 개수를 유지한다.
- history_log : 전체 쓰레드에 대한 최근 이벤트들을 모두 저장하며, 지정한 전체 최대 개수만큼 데이터가 저장된다. 쓰레드가 종료되는 것과 관계없이 지정된 최대 개수만큼 이벤트 데이터를 가지고 있으며, 저장된 이벤트 데이터가 전체 최대 저장 개수를 넘어가면 이전 이벤트들을 삭제하고 최근 이벤트를 새로 저장함으로써 최대 개수를 유지한다.

#### Wait Event
각 쓰레드에서 대기하고 있는 이벤트들에 대한 정보를 확인할 수 있다. 일반적으로 잠금 경합 또는 I/O 작업 등으로 인해 쓰레드가 대기한다.
- events_waits_current
- events_waits_history
- events_waits_history_long

#### Stage Event 테이블
각 쓰레드에서 실행한 쿼리들의 처리 단계에 대한 정보를 확인할 수 있다. 이를 통해 실행된 쿼리가 구문 분석, 테이블 열기, 정렬 등과 같은 쿼리 단계 중 현재 어느 단계를 수행하고 있는지와 처리 단계별 소요 시간 등을 알 수 있다.
- events_stages_current
- events_stages_history
- events_stages_history_long

#### Statement Event 테이블
각 쓰레드에서 실행한 쿼리들에 대한 정보를 확인할 수 있다. 실행된 쿼리와 쿼리에서 반환된 레코드의 수, 인덱스 사용 유무 및 처리된 방식 등의 다양한 정보를 함계 확인할 수 있다.
- events_statements_current
- events_statements_history
- events_statements_history_long

#### Transaction Event 테이블
각 쓰레드에서 실행한 트랜잭션에 대한 정보를 확인할 수 있다. 트랜잭션별로 트랜잭션 종류와 현재 상태, 격리 수준등을 알 수 있다.
- events_transactions_current
- events_transactions_history
- events_transactions_history_long
 