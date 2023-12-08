# innoDB Cluster
## 아키텍처
- 그룹 복제 : 소스 서버의 데이터를 레플리카 서버로 동기화하는 기본적인 복제 역할뿐만 아니라 복제에 참여하는 MySQL 서버들에 대한 자동화된 멤버십 관리 역할을 담당한다.
- MySQL 라우터 : 애플리케이션과 MySQL 사이의 미들웨어 쿼리를 적절한 서버로 전달하는 프록시 역할을 한다.(로드 밸런싱?)
- MySQL Shell : 클라이언트보다 확장된 기능을 가진 클라이언트 프로그램 쿼리뿐만 아니라 JS 및 파이썬 기반 스크립트 작성 기능과 클러스터 구성 등의 어드민 작업을 할 수 있게 하는 API를 제공한다.


## 그룹 복제
기존 복제는 SOURCE - REPLICA의 단방향 복제가 이뤄지는 반면, 그룹 복제는 복제에 참여하는 서버들이 하나의 복제 그룹으로 묶인 클러스터 형태를 가지며 서로 통신하면서 양방향으로
복제를 처리할 수 있다. 소스 레플리카 대신 프라이머리, 세컨더리라고 부른다. 그룹 복제에 참여하는 서버들을 그룹 멤버라고 지칭한다.

그룹 복제는 별도 플러그인으로 구현돼 있으며, 그룹 복제를 사용하기 위해서는 MySQL 서버에 그룹 복제 플러그인이 설치돼 있어야 한다. 그룹 복제에 참여하는 MySQL 서버들은
그룹 복제 플러그인을 통해서 서로 간의 지속적으로 통신하여 복제 동기화를 처리한다. 그룹 복제 플러그인은 MySQL 서버에 그룹 복제가 설정되면 `group_replication_applier`
복제 채널을 생성하며, 이 채널을 통해 그룹에 참여하고 있는 다른 서버들은 그룹에서 실행된 모든 트랜잭션을 전달받아 적용한다.

## 그룹 복제에서 트랜잭션 처리
그룹 복제에서 트랜잭션은 아래의 단계들을 거친 후 최종적으로 그룹의 각 서버들에 적용된다.
- 합의(Consensus) : 그룹 내 일관된 트랜잭션 적용을 위해서 멤버들에 트랜잭션 적용을 제안하고 승낙 받는 과정 그룹. 멤버간 통신 결과를 바탕으로 처리된다.
- 인증(Certification) : 트랜잭션들은 합의를 거친 후 글로벌하게 정렬되며, 각 멤버들에서 모두 동일한 순서로 인증 단계를 거친다. 

## 트랜잭션 일관성 수준
그룹 복제에서 각 멤버들은 모두 동일한 트랜잭션을 적용하지만 실제 적용 시점까지 일치하는 것은 아니다. 따라서 한 멤버에서 쓰기를 수행한 후 다른 멤버에서 해당 데이터를 읽었을 때
최신 변경 사항이 반영되지 않았을 수 있다. MySQL 8.0.14전까지는 방지할 방법이 없었는데, 8.0.14부터 그룹 복제에서 트랜잭션의 일관성 수준을 설정할 수 있게 됐다.
`group_replication_consistency`를 통해서 일관성 수준을 설정할 수 있다. 적용 범위는 글로벌, 세션 모두 가능하다. 

- EVENTUAL : 해당 변수가 추가되기 전의 그룹 복제에서 트랜잭션 일관성 수준과 동일하다. 해당 읽관성 수준에서 읽기 전용 및 읽기-쓰기 트랜잭션이 별도 제약없이 바로 실행 가능하다. 
- BEFORE_ON_PRIMARY_FAILOVER : 싱글 프라이머리 모드로 설정된 그룹 복제에서 프라이머리 페일오버가 발생해서 신규 프라이머리가 선출됐을 때만 트랜잭션에 영향을 미친다. 이 일관성 수준은 아래와 같은 부분이 보장된다.
  - 신규 프라이머리로 유입된 읽기 전용 및 읽기-쓰기 트랜잭션들은 오래된 데이터가 아닌 최신 데이터를 바탕으로 동작하게 된다. 
  - 신규 프라이머리로 유입된 읽기-쓰기 트랜잭션은 아직 대기 중인 이전 프라이머리의 트랜잭션과 충돌로 롤백될 수 있는데 이 일관성을 사용하면 이같은 롤백은 발생하지 않는다. 
- BEFORE : 읽기 전용 및 읽기-쓰기 트랜잭션은 모든 선행 트랜잭션이 완료될 때까지 대기 후 처리된다. 선행 트랜잭션은 해당 트랜잭션이 실행된 그룹멤버에서의 선행 트랜잭션만 의미한다. 
- AFTER : 트랜잭션이 적용되면 해당 시점에 그룹 멤버들이 모두 동기화된 데이터를 갖게 한다. AFTER 일관성 수준에서 읽기-쓰기 트랜잭션은 다른 모든 멤버들에게서도 해당 트랜잭션이 커밋될 준비가 됐을 때까지 대기한 후 최종적으로 
처리하며, 읽기 전용 트랜잭션은 데이터 변경을 발생시키지 않으므로 별도 제약 없이 처리된다. AFTER 일관성으로 설정된 읽기-쓰기 트랜잭션은 그룹의 다른 멤버들로부터 응답을 받으면 최종적으로 커밋된다. 
- BEFORE_AND_AFTER: BEFORE, AFTER가 결합된 형태다. 이 일관성 수준에서 읽기-쓰기 트랜잭션은 모든 선행 트랜잭션이 적용될 때까지 기다린 후 실행되며, 트랜잭션이 다른 모든 멤버들에게도
커밋이 준비되어 응답을 보내면 최종 커밋된다. 읽기 전용 트랜잭션은 모든 선행 트랜잭션이 적용될 때까지 대기한 후 실행된다. 

## 흐름 제어

그룹 복제에서 일부 멤버가 하드웨어 스팩, 네트워크 대역폭 등의 문제로 다른 멤버보다 트랜잭션 적용이 지연될 수 있다. 이렇게 지연된 멤버에서 트랜잭션이 실행되면 해당 트랜잭션은
최신 데이터가 아닌 오래된 데이터를 읽을 수 있다. 아직 적용되지 않은 트랜잭션과 충돌할 수도 있다. 트랜잭션 일관성을 조정할 수도 있지만 근본적인 해결책은 아니다.

이를 위해서 그룹 멤버들의 쓰기 처리량을 조절하는 흐름제어 메커니즘이 있다. 그룹 복제는 흐름 제어를 통해 멤버 간 트랜잭션 갭을 적게 유지해서 멤버들의 데이터가 최대한
동기화된 상태로 유지될 수 있게 하며, 그룹에 평소와 다른 워크로드가 유입되는 등의 변화에도 빠르게 적응해서 각 멤버들의 쓰기 처리량이 균등할 수 있게 한다. 또한 필요
이상으로 처리량을 줄이지 않음으로써 서버의 자원을 불필요하게 유휴로 두지 않는다. 


## 그룹 복제의 자동 장애 감지 및 대응
그룹 복제에서는 그룹의 일부 멤버에 장애가 발생해 응답 불능에 빠졌다고 해도 그룹이 정상적으로 동작할 수 있게 하는 장애 감지 메커니즘이 구현돼 있다. 장애 감지 메커니즘에서는
문제 상태에 있는 멤버를 식별하고 그룹 복제에서 제외시킴으로써 그룹이 정상적으로 동작하는 멤버로만 구성될 수 있게 한다. 

그룹 복제는 멤버 간 주기적 통신 메시지를 주고 받으면 서로 상태를 확인한다(heartbeat) 5초 이내로 메시지를 받지 못하면 의심하기 시작한다. 그룹 복제에서는 장애가 의심되는
멤버에 대해서 과반수의 멤버가 동의하면 해당 멤버를 그룹에서 추방한다. 

만약 멤버가 추방되고 다시 통신을 재개할 수 있는 경우 해당 멤버는 자신이 그룹에서 추방됐음을 알게 된다. 멤버가 추방되면 그룹 뷰가 변경되므로 그룹 멤버들은 새로운 뷰ID를 갖는다. 
추방된 멤버가 다시 그룹으로 들어오면 그룹의 현재 뷰 ID와 자신의 뷰 ID가 상이함을 보고 추방됐음을 감지한다. 추방 멤버는 자동으로 재가입을 시도할 수 있다. 이는 
`group_replication_autorejoin_tries`에 따라 달라진다. (8.0.16부터 지

그룹에서 추방된 멤버는 다른 그룹 멤버들과 다시 통신이 되지 않으면 본인의 추방여부를 알지 못한다. 네트워크 단절로 분리되는 경우 소수에 속하는 멤버들은 스스로 그룹을 탈퇴하지 않는다. 
사용자는 `group_replication_unreachable_majority_timeout`으로 소수에 속한 멤버들이 과반수의 그룹 멤버들과 통신 단절됐을 때 일정 시간 대기한 후 스스로 그룹을
탈퇴하도록 할 수 있다. `group_replication_unreachable_majority_timeout` 변수 값으로 타임아웃을 확인한다.

멤버가 그룹의 다른 멤버들과 통신 단절로 타의 혹은 자의로 탈퇴하고 자동 재가입에 실패하면 혹은 아예 시도도 안한다면 멤버는 최종적으로 `group_replication_exit_state_action`에 설정된 작업을 한다.
이를 종료 액션이라고 한다. (8.0.12부터 가능하다.)

- READ_ONLY
`super_read_only` 시스템 변수를 ON으로 해서 서버를 슈퍼 읽기 전용모드로 전환시킨다. 슈퍼 읽기 전용은 CONNECTION_ADMIN 권한을 가지고 있어도 서버에서 데이터 변경을 수행할 수 없다.
8.0.16부터 `group_replication_exit_state_action`의 기본값이다.

- OFFLINE_MODE
`offline_mode`를 ON으로 해서 서버를 오프라인으로 전환시키고 `super_read_only`도 ON으로 바꾼다. 오프라인 모드에서는 기존에 연결돼 있는 세션의 경우 다음 요청에서 
연결이 끊어지고 CONNECTION_ADMIN을 가진 사용자를 뺴고 더 이상 연결을 거부한다.

- ABORT_SERVER
MySQL를 종료시킨다. 

위 상황들이 연출되는 구체적인 경우는 아래와 같다. 

- 그룹 복제의 어플라이어 쓰레드(ApplierThread)에 에러가 발생한 경우
- 멤버가 그룹 복제의 분산 복구 프로세스를 정상적으로 완료할 수 없는 경우
- `group_replicatoin_switch_to_single_primary_mdoe()`같은 그룹 복제 UDF를 사용해서 그룹 전체에 대한 설정을 변경하는 중 에러가 발생한 경우
- 싱글 프라이머리 모드의 그룹에서 새 프라이머리 선출 중 에러가 발생한 경우
- 과반수 이상의 다른 그룹 멤버들과 통신이 단절되고 `group_repliaction_unreachable_majority_timeout`이 초과됐으나 재가입 시도를 하지 않도록 된 경우
- 멤버에 문제가 발생해서 `group_replication_member_expel_timeout`에 설정된 대기 시간이 초과된 후 그룹에서 추방되고 나서 다시 그룹의 다른 멤버들과 
통신 재개되어 자신이 추방됐음을 확인했으나 그룹에 재가입 시도를 하지 않도록 설정된 경우
- 멤버가 자의 혹은 타의로 그룹에서 탈퇴하고 `group_replication_autorejoin_tries`에 지정된 횟수를 초과한 경우

## 그룹 복제 요구 사항
- innoDB 사용 : `disabled_storage_engines`로 다른 스토리지 엔진 사용을 방지할 수 있다. 이유는 데이터 일관성을 위해서 트랜잭션 롤백이 필요한데, 트랜잭션 지원되는 스토리지 엔진이 필요하기 때문이다.
- PK 사용 : 그룹에서 복제될 때 모든 테이블이 PK를 가지고 있어야한다. 혹은 NOT NULL + UNIQUE라도 있어야 한다. 그래야 이를 바탕으로 트랜잭션 간의 충돌을 감지한다.
- 바이너리 로그 활성화 : 그룹 복제는 바이너리 로그를 사용한다. (8.0부터는 기본 활성화)
- ROW 형태 바이너리 로그 포맷 사용 : 그룹 복제는 ROW 포맷 기반 복제를 사용해서 트랜잭션으로 인해 변경된 데이터가 그룹 멤버들에게 일관되게 적용될 수 있게 한다.
- 바이너리 로그 체크섬 설정 : 8.0.21부터 체크섬을 지원하므로 해당 변수에 원하는 값을 설정하면 된다.
- log_slave_update 활성화 : 새로운 멤버가 참여하는 해당 멤버는 기존 그룹 멤버들에게 분산 복구 작업을 받는데, 이때 기존 그룹 멤버의 바이너리 로그르 복제해서 자신에 적용하므로 그룹 멤버들은 그룹에서 발생한 트랜잭션을 모두 각자의 바이너리에 기록해야 한다. 
- GTID 사용 : 그룹 복제는 GTID를 사용하므로 설정해야 한다. 
```sql
[mysqld]
gtid_mode=ON 
enfoce_gtid_consistency=ON
```
- 고유한 server_id 사용 : 그룹 복제에 참여하는 서버들은 모두 각기 다른 고유한 server_id를 가져야 한다.
- 복제 메타데이터 저장소 설정 : 그룹 복제에서 복제 관련 메타 데이터는 데이터 일관성으 루이해서 파일이 아닌 테이블에 저장돼야 한다. 따라서 MySQL 서버에서 master_info_repository 및 relay_log_info_repository 시스템 변수의 값이 TABLE로 설정돼 있어야 한다.
- 트랜잭션 WriteSet 설정 : 트랜잭션에서 변경한 데이터에 대한 정보, 즉 트랜잭션의 WriteSet이 수집될 수 있도록 `transaction_write_set_extraction`이 `XXHASH64`로 설정돼야 한다. 트랜잭션의 WriteSet은 그룹 복제에서 트랜잭션 간 충돌을 탐지하는 트랜잭션 인증 단계에서 사용한다.
- 테이블 스페이스 암호화 : `default_table_encryption` 시스템 변수는 모든 그룹 멤버에서 동일한 값으로 설정돼야 한다.
- lower_case_table_names : `lower_case_table_names `이 변수롤 모든 그룹 멤버에서 맞춰야 한다.
- 멀티 쓰레드 복제 설정 : 그룹 복제에서도 멀티 쓰레드 복제 기능을 사용해서 트랜잭션을 병렬로 적용할 수 있다. 이 때 복제 트랜잭션이 멀티 쓰레도로 작동될 때 원본 서버에서 커밋된 순서와 동일한 순서로 커밋되도록 `slave_preserve_commit_order`를 ON으로 해야한다. 
```sql
[mysqld]
slave_parallel_workers=N
slave_parallel_type=LOGICAL_CLOCK
slave_preserve_commit_order=1
```

## MySQL 라우터
innoDB에서 애플리케이션 서버로부터 유입된 쿼리 요청을 클러스터 내 적절한 MySQL 서버로 전달하고 반환된 쿼리 결과를 다시 애플리케이션 서버로 전달하는 proxy 역할을 한다.
주요 기능은
1. innoDB 클러스터의 MySQL 구성 변경 자동 감지
2. 쿼리 부하 분산
3. 자동 failover

## InnoDB 클러스터 구축
### 요구 사항
1. MySQL 5.7.17 이상
2. MySQL 셸 1.0.8 이상
3. 라우터 2.1.2 이상
4. innoDB 클러스터 MySQL 서버들은 모두 Performance 스키마가 활성화돼 있어야 한다. 
5. MySQL 셸을 사용해서 innoDB 클러스터를 구성하기 위해서 설치될 서버에 python 2.7이상이 설치돼 있어야 한다.