## 테이블 암호화

### 테이블 생성
```mysql
CREATE TABLE tab_encrypted (
    id INT,
    data VARCHAR(100),
    PRIMARY KEY (id)
) ENCRYPTION ='Y';
```
`ENCRYPTION ='Y'` 옵션을 붙이면 디스크에 기록될 때 암호화, 메모리에서 읽어올 떄 복호화 된다. 암호화된 테이블만 검색할 떄는 `information_schema`의 TABLES 뷰를 이용하면 된다.

