# 사용자 및 권한

## 사용자 식별
MySQL는 사용자와 호스트도 계정의 일부간 된다. 따라서 항상 아이디, 호스트를 명시해야 한다.

```sql
newkayak12@192.168.0.87 pwd 1234
newakayak12@% pwd abc
```

위와 같이 계정 범위와 비밀번호가 다른 같은 이름의 계정이 있을 때 MySQL은 항상 범위가 작은 것부터 체크 한다.  이렇게 계정의 중첩되는 경우에 
유의해야 한다.


## 계정 관리
MySQL 8.0부터 계정에 권한 개념이 생겼다. SystemAccount(SYSTEM_USER), RegularAccount로 구분된다. 
시스템 계정은 DBA를 위한 계정, 일반 계정은 응용 프로그램, 개발자를 위한 계정 정도로 생각하면 된다. 

- 계정 관리( 계정 생성, 삭제, 권한 부여 및 제거)
- 다른 세션(Connection) 또는 그 세션에서 실행 중인 쿼리 강제 종료
- 스토어드 프로그램 생성시 DEFINER를 타 사용자로 설정

등의 작업은 SystemAccount로만 진행할 수 있다. 

## 내장 계정

- `mysql.sys@localhost` : MySQL 8.0부터 기본으로 내장된 sys 스키마의 객체(뷰, 함수, 프로시저)들의 DEFINER
- `mysql.session@localhost` : MySQL 플러그인이 서버로 접근할 때 사용되는 계정
- `mysql.infoschema@localhost` : information_schema에 정의된 뷰의 DEFINER로 사용

위 세 가지의 계정은 account_locked 상태다. 따라서 악의적인 용도로 사용할 수 없으므로 보안에 걱정하지 않아도 된다.

## 계정 생성 
5.7까지는 GRANT로 생성, 권한 부여 모두 가능헀다. 8.0부터는 CREATE USER, GRANT로 구분하도록 바뀌었다. 계정 생성에는 아래와 같은 다양한 옵션을 설정할 수 있다.

 - 계정의 인증 방식과 비밀번호
 - 비밀번호 관련 옵션( 유효 기간, 이력 개수, 재사용 불가 기간 )
 - 기본 역할( Role )
 - SSL 옵션
 - 계정 잠금 여부

```mysql
CREATE USER `user`@`%`
    IDENTIFIED WITH `mysql_native_password` BY 'password'
    REQUIRE NONE
    PASSWORD EXPIRE INTERVAL 30 DAY
    ACCOUNT UNLOCK
    PASSWORD HISTORY DEFAULT
    PASSWORD REUSE INTERVAL DEFAULT
    PASSWORD REQUIRE CURRENT DEFAULT;
```

### IDENTIFIED WITH
사용자 인증 방식과 비밀번호를 설정한다. IDENTIFIED WITH 바로 뒤는 인증 방식(인증 플러그인)을 반드시 명시해야 하는데
비어 있으면 MySQL 기본 인증 방식이다. 

- Native Pluggable Authentication: MySQL 5.7까지의 방식이다. 비밀번호에 대한 해시(SHA-1) 값을 저장하고 클라이언트가 보낸 값과 대조하는 방식
- Caching SHA-2 Pluggable Authentication: SHA-2(256bit) 알고리즘을 사용한다. `Native Pluggable Authentication`와는 알고리즘 차이, 내부적으로 Salt를 사용한다는 차이가 있다. 또한 
해시 값을 계산하는데 리소스가 크기 떄문에 캐싱으로 이를 극복한다. 이 인증 방식을 사용하려면 SSL/TLS 또는 RSA 키페어를 기본적으로 사용해야 한다. 
- PAM Pluggable Authentication: 유닉스나 리눅스 패스워드 또는 LDAP(Ligthweigth Directory Access Protocol) 같은 외부 인증을 사용할 수 있게 해준다. (엔터프라이즈)
- LDAP Pluggable Authentication: LDAP를 이용한 외부 인증 (엔터프라이즈)


### REQUIRE
MySQL이 SSL/TLS 채널을 사용할지 여부를 결정한다. Caching SHA-2 Pluggable Authentication는 SSL/TLS를 사용하도록 강제된다.

### PASSWORD EXPIRE
비밀번호의 유효 기간을 설정하는 옵션이며, 별도 명시하지 않으면 `defualt_password_lifetime`에 지정된 기간으로 설정된다.

- PASSWORD EXPIRE : 계정 생성과 동시에 비밀번호 만료
- PASSWORD EXPIRE NEVER : 만료 기간 없음
- PASSWORD EXPIRE DEFAULT : defualt_password_lifetime 시스템 변수에 지정된 값
- PASSWORD EXPIRE INTERVAL n DAY : 오늘부터 n일

### PASSWORD HISTORY
한 번 사용했던 비밀번호를 재사용 하지 못하도록 설정하는 옵션

- PASSWORD HISTORY DEFAULT : password_history에 지정된 개수 만큼 비밀번호 이력을 저장
- PASSWORD HISTORY n : 최근 n 개까지

### PASSWORD REUSE INTERVAL
한 번 사용했던 비밀번호의 재사용 금지 기간을 설정하는 옵션
- PASSWORD REUSE INTERVAL DEFAULT : password_reuse_interval 참조
- PASSWORD REUSE INTERVAL n DAY : n일 이후

### PASSWORD REQUIRE 
비밀번호가 만료되어 새로운 비밀번호로 변경할 떄 현재 비밀번호를 필요로 할지에 대한 옵션

- PASSWORD REQUIRE CURRENT : 변경 전 현재 비밀번호 요구
- PASSWORD REQUIRE OPTIONAL : 비밀번호를 변경할 때 현재 비밀번호 입력하지 않아도 되도록
- PASSWORD REQUIRE DEFAULT : password_require_current의 값에 따름

### ACCOUNT UNLOCK/ LOCK
계정 생성시 또는 ALTER USER 명령을 사용해 계정 정보를 변경할 때 계정을 사용하지 못하게 잠글지 여부

- ACCOUNT LOCK : 계정을 잠금
- ACCOUNT UNLOCK : 잠긴 계정을 다시 사용 가능 상태로 잠금 해제


## 비밀번호 관리
### 고수준 비밀번호
비밀번호를 쉽게 유추하지 못하도록 글자 조합을 강제하거나 금칙어를 설정하는 기능이 있음 validate_password 컴포넌트를 사용한다. 기본으로 탑재외어 있다.
`SELECT * FROM mysql.component`

정책은 크게 3가지 중에서 선택할 수 있다.

- LOW : 길이만 검증
- MEDIUM : 길이 검증, 숫자, 대소문자, 특수문자 배합 검증
- STRONG : 레벨의 검증을 모두 수행, 금칙어 포함여부 검증

### 이중 비밀번호
MySQL 8.0부터는 계정 비밀번호로 2개의 값을 사용할 수 있는 기능을 제공한다. 덕분에 응용 프로그램이 사용 중인 비밀번호 외에 하나 더 사용할 수 있고, 비밀번호 변경
때문에 다운타임이 생길 이유가 없어졌다.


### 권한

글로벌 권한

- FILE / `file_priv` / 파일
- CREATE ROLE / `Create_role_priv` / 서버 관리
- CREATE TABLESPACE / `Create_tablespace_priv` / 서버 관리
- CREATE USER / `Create_usr_priv` / 서버 관리
- DROP ROLE / `Drop_role_priv` / 서버 관리
- PROCESS / `Process_priv` / 서버 관리
- PROXY / `See proxies_priv table` / 서버 관리
- RELOAD / `Reload_priv` / 서버 관리
- REPLICATION CLIENT / `Repl_client_priv` / 서버 관리
- REPLICATION SLAVE / `Repl_slave_priv` / 서버 관리
- SHOW DATABASES / `Show_db_priv` / 서버 관리
- SHUTDOWN / `Shutdown_priv` / 서버 관리
- SUPER / `Super_priv` / 서버 관리
- USAGE / `Synonym for "no privileges"` / 서버 관리

객체 권한

- EVENT / `Event_priv` / 데이터베이스
- LOCK TABLES / `Lock_tables_priv` / 데이터베이스
- REFERENCES / `References_priv` / 데이터베이스 & 테이블
- CREATE / `Create_priv` / 데이터베이스 & 테이블 & 인덱스
- GRANT OPTION / `Grant_option_priv` / 데이터베이스 & 테이블 & 스토어드 프로그램
- DROP / `Drop_priv`  / 데이터베이스 & 테이블, 뷰
- ALTER ROUTINE / `Alter_routine_priv` / 스토어드 프로그램
- CREATE ROUTINE / `Create_routine_priv` / 스토어드 프로그램
- EXECUTE / `Execute_priv` / 스토어드 프로그램
- ALTER / `Alter_priv` / 테이블
- CREATE TEMPORARY TABLES / `Create_temporary_tables_priv` / 테이블
- DELETE / `Delete_priv` / 테이블
- INDEX / `Index_priv` / 테이블
- TRIGGER / `Trigger_priv` / 테이블
- INSERT / `Insert_priv` / 테이블 & 컬럼
- SELECT / `Select_priv` / 테이블 & 컬럼
- UPDATE / `Update_priv` / 테이블 & 컬럼
- CREATE VIEW / `Create_view_priv` / 뷰
- SHOW VIEW / `Show_view_priv` / 뷰

객체 & 글로벌

- ALL [PRIVILEGES] / Synoym for 'all privileges' / 서버 관리


## Role
MySQL부터 권한을 묶어서 역할을 사용할 수 있게 됐다. 역할을 생성하고 권한을 부여하는 식으로 사용한다.
```mysql

CREATE ROLE 
    role_emp_read,
    role_emp_write;

GRANT SELECT ON employees.* TO role_emp_read;
GRANT INSERT, UPDATE, DELETE ON employees.* TO role_emp_write;

GRANT role_emp_read, role_emp_write TO user@%;
```

사실 계정과 큰 차이가 없다. account_locked가 다를 뿐이다.

























 