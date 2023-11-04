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