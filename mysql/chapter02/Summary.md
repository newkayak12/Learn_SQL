# MySQL

##  mysqld --verbose --help
해당 명령어로 my.cnf의 내용을 열람할 수 있다.
```
Default options are read from the following files in the given order:
/etc/my.cnf /etc/mysql/my.cnf /usr/etc/my.cnf ~/.my.cnf 
```
결과와 같이 해당 순서대 my.cnf를 읽어들인다.


## 설정 구성
MySQL의 설정 파일은 하나의 my.cnf에 여러 개의 설정 그룹을 담을 수 있으며, 대체로 실행 프로그램 이름을 그룹명으로 한다.

```mysql
# http://dev.mysql.com/doc/refman/8.0/en/server-configuration-defaults.html

[mysqld]
#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M
#
# Remove leading # to turn on a very important data integrity option: logging
# changes to the binary log between backups.
# log_bin
#
# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.
# join_buffer_size = 128M
# sort_buffer_size = 2M
# read_rnd_buffer_size = 2M

# Remove leading # to revert to previous value for default_authentication_plugin,
# this will increase compatibility with older clients. For background, see:
# https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_default_authentication_plugin
# default-authentication-plugin=mysql_native_password
        
skip-host-cache
skip-name-resolve
datadir=/var/lib/mysql
socket=/var/run/mysqld/mysqld.sock
secure-file-priv=/var/lib/mysql-files
user=mysql

pid-file=/var/run/mysqld/mysqld.pid
[client]
socket=/var/run/mysqld/mysqld.sock

!includedir /etc/mysql/conf.d/
```

예를 들어 `[mysqldump에]` 은 mysqldump에 대한, `mysqld`는 mysql이 참조한다. mysqld_safe는 `[mysqld_safe]`, `[mysqld]`를 참조한다.


## 시스템 변수
MySQL 서버는 기동하면서 설정 파일을 읽어 메모리나 작동 방식을 초기화하고, 접속된 사용자를 제어하기 위해서 이러한 값을 별도로 저장한다.
MySQL 서버에서는 이렇게 저장된 값을 `System Variables`라고 한다.

이러한 변수 항목은 아래의 표와 같은 형식으로 구성돼 있다.

|            Name             | Cmd-Line |OptionFile|SystemVar| VarScope | Dynamic |
|:---------------------------:|:--------:|:---:|:---:|:--------:|:-------:|
| activate_all_roles_on_login |    Y     |      Y    |    Y     |  Global  |    Y    |
|        admin_address        |      Y    |     Y     |   Y      |  Global  |    N    |
|         admin_port          |     Y     |    Y      |    Y     |  Global  |     N    |
|          time_zone          |          |          |     Y    |   Both   |    Y    |
|         sql_log_bin         |          |          |     Y    | Session  |    Y    |

- Cmd-Line : 서버 명령행 인자로 설정할 수 있는지에 대한 여부, Y이면 명령행 인자로 이 시스템 변수를 변경할 수 있다.
- OptionFile: my.cnf로 제어할 수 있는지에 대한 여부
- SystemVar: 시스템 변수인지 아닌지를 나타낸다. 서버 설정 파일을 작성할 때 각 변수명에 사용된 하이픈, 언더스코어 구분에 주의해야 한다. 
- VarScope: 시스템 변수의 적용 범위를 나타낸다. 글로벌인지, 세션인지 구분한다. 어떤 변수는 둘 다 적용되기도 한다.
- Dynamic: 시스템 변수가 동적인지 정적인지 구분하는 변수다.

## 글로벌 변수, 세션 변수
- 글로벌 범위의 시스템 변수는 하나의 MySQL 서버 인스턴스에서 전체적으로 영향을 미치는 시스템 변수를 의미하며, 주로 MySQL 서버 자체에 관련된 설정일 때가 많다. 
MySQL 서버에서 단 하나만 존재하는 innoDB 버퍼풀 크기(innodb_buffer_pool_size) 또는 MyISAM의 캐시 크기(key_buffer_size) 등이 가장 대표적인 글로벌 영역의 시스템 변수다. 
- 세션 범위 시스템 변수는 클라이언트가 MySQL에 접속할 떄 부여하는 옵션의 기본값을 제어하는데 사용한다. 클라이언트의 필요에 따라 개별 커넥션 단위로 다른 값을 변경할 수 있는
세션 변수다. 기본 값은 글로벌 시스템 변수며, 각 클라이언트가 가지는 값이 세션 시스템 변수다. 
- 세션 범위의 시스템 변수 가운데 MySQL 서버의 설정(my.cnf)에 명시해 초기화할 수 있는 변수는 대부분 `Both`로 명시돼있다. 이렇게 명시된 시스템 변수는 MySQL
서버가 기억하고 있다가 클라이언트와 실제 커넥션이 생성되는 순간에 해당 커넥션의 기본값으로 사용되는 값이다. 그리고 순수하게 Session인 시스템 변수는 초기 값을 설정 파일(my.cnf)에
명시할 수 없으며, 커넥션이 생성되는 순간부터 해당 커넥션에서만 유효한 설정 변수를 의미한다.

## 정적 변수, 동적 변수
MySQL 서버의 시스템 변수는 서버 기동 중에 바꿀 수 있는지에 따라 정적/ 동적 변수로 구분된다. MySQL 서버 시스템 변수는 디스크에 저장된 my.cnf를 변경하는 경우와
이미 기동 중인 서버의 메모리에 있는 서버 시스템 변수를 변경하는 경우로 구분할 수 있다. 디스크에 저장된 내용은 바꾸고 재기동을 해야하며, `SET` 명령으로 바꾼 경우
영구적으로 적용되는 것이 아님을 기억해야 한다. MySQL 8.0부터는 `SET PERSIST`로 해당 my.cnf에도 반영할 수 있다. 또한 `SHOW`에서 `GLOBAL`을 빼면 자동으로
세션 변수를 조회하고 변경한다.


`SHOW [GLOBAL] VARIABLES [LIKE ~]`
`SET [GLOBAL] variable=value`

## SET PERSIST
`SET PERSIST`는 세션 변수에는 적용되지 않으며, 기본적으로는 `GLOBAL` 시스템 변수로 인식하고 변경한다. 
만약 당장 적용하지 않고 my.cnf에만 반영하거나 재시작 이후에 반영하려면 `SET PERSIST_ONLY`로 지정하면 된다. 또한 정적 변수를 변경할 때도 해당 방법을 사용해야 한다.
