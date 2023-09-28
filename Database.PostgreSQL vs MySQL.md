### PostgreSQL vs MySQL

### 참고자료

- [우아한 형제들 기술블로그 - Aurora MySQL vs Aurora PostgreSQL](https://techblog.woowahan.com/6550/)
- [MySQL vs PostgreSQL 비교](https://smoh.tistory.com/370)
- [Oracle, MySQL, PostgreSQL 차이점은?](https://velog.io/@jisoo1170/Oracle-MySQL-PostgreSQL-%EC%B0%A8%EC%9D%B4%EC%A0%90%EC%9D%80)

<br>



### RDBMS, ORDBMS

- PostgreSQL 은 객체 관계형 DB다. (ORDBMS)
- 테이블 상속기능이 제공된다. 자식 테이블은 부모테이블로부터 컬럼을 상속받아 사용가능하다.
- 기본적으로는 관계형 데이터베이스지만, 객체 데이터베이스와 연관되는 기능(테이블 상속/함수 오버로딩) 역시 포함하고 있다.

<br>



### 멀티스레드 vs 멀티프로세스

- PostgreSQL 은 멀티 프로세스 방식이다.
- MySQL 은 멀티 스레드 방식이다.

<br>



### PostgreSQL 의 단순 CRUD 성능 ↓ 복잡한 쿼리 ↑

- PostgreSQL은 단순 CRUD시에 MySQL에 비해서 성능이 떨어진다.
- PostgreSQL 은 복잡한 쿼리를 요구하거나 대규모 서비스에 특화되어 있다.

<br>



### MVCC 지원방식

MySQL

- Undo Segment 방식
- 최신 데이터는 기존 데이터 블록의 레코드에 반영한다.
- 변경 전의 값은 undo 영역이라는 별도의 공간에 저장하고, 갱신에 대한 버전관리를 한다.

PostgreSQL

- MGA 방식
- 튜플을 Update 할 때 새로운 값으로 replce 하지 않고 이전 튜플은 유효범위를 마킹해서 처리하는 방식

<br>



### Update 방식 : PostgreSQL 은 Insert/Select 위주의 서비스에 적합

MySQL 은 Update 를 그대로 수행한다.

PostgreSQL 은 Insert & Delete (삭제표시) 를 수행한다.

- PostgreSQL 은 <u>UPDATE 시 내부적으로는 새 행이 INSERT 되고 이전 데이터는 삭제 표시</u>된다.
- 모든 인덱스에는 보통 행의 실제 위치값에 대한 링크가 표기된다. 따라서 행이 업데이트 되면 변경된 위치값 (새로 추가된 행)에 대한 인덱스 정보도 업데이트가 필요하다. 이런 과정 때문에 UPDATE 시에는 MySQL 보다 성능이 떨어지게 된다.

<br>



PostgreSQL 은 Update 시 변경 전 값을 삭제 마크 처리한 후 새로운 행을 INSERT 한다. 새롭게 INSERT 된 값이 변경 후의 값이 된다.<br>

이런 이유로 PostgreSQL은 보통 Insert, Select 위주의 서비스에 사용되는 것이 선호된다.<br>

<br>



### 지원되는 JOIN

MySQL 은 NL JOIN, HASH JOIN 을 지원한다.

PostgreSQL 은 NL JOIN, HASH JOIN, SORT JOIN 을 지원한다.<br>

<br>



### Parallel Query for SELECT

MySQL 5.7.2.09.2 버전부터 지원되기 시작했다.

PostgreSQL 9.6 버전부터 지원되기 시작했다.

- PostgreSQL 은 오래 전부터(9버전) 대부분의 SELECT 쿼리에서 parallel 기능이 지원되고 있다.



<br>



### 테이블 기본 구성 인덱스

MySQL 은 기본키에 대해 Clustered Index를 지원한다.

PostgreSQL 은 기본키에 대해 Non Clustered Index 가 적용된다.

<br>



### 성능비교

#### PostgreSQL 의 Partial Index

PostgreSQL 에는 전체데이터의 부분 집합에 대해서만 인덱스를 생성하는 Partial Index 라는 기능이 있다.

특정 범위에 대해서만 인덱싱을 하는 기능이다. 

대량 데이터의 일부 값/범위에 대해 인덱스를 생성할 경우 인덱스 크기도 작고 관리하는 리소스도 줄일 수 있다는 장점이 있다.

조회 시에는 PostgreSQL의 성능과 MySQL의 조회성능이 큰 차이를 보여주지 않았다고 한다.

인덱스 크기를 확인해보면, PostgreSQL 의 크기가 10배정도 더 작다. 필요한 부분만 인덱스를 생성하기 때문에 저장공간에 대한 이점이 크며, 데이터 삭제,추가,갱신에 따른 인덱스 유지관리 비용도 절약된다.<br>

<br>



#### Secondary Index 생성

**참고) online ddl 은 꽤 부담스러운 작업에 속한다.**<br>

테이블에 이미 저장되어 있는 데이터가 많기에 시간소요 예측이 힘들고, 작업이 실패한다면 rollback 작업에 따른 위험도도 크다. 

> 시간예측을 할때는 보통 백업 데이터로 테스트를 진행하지만, 라이브 환경에서는 시간 소요가 더 오래 걸리는 경우가 많다.

<br>

**이미 존재하는 테이블에 인덱스,컬럼 추가작업 시의 성능 실험**

Aurora MySQL

- 100G 테이블의 인덱스, 컬럼 추가를 하는 데에 1시간이 넘는 시간이 소요된다.
- 100G 만 넘어도 인댁스/컬럼 추가시 1시간이 넘게 소요된다.

Aurora PostgreSQL

- 200G 테이블의 인덱스,컬럼 추가를 하는 데에 40분 만에 인덱스 추가가 완료되고 컬럼 추가는 바로 된다.
- PostgreSQL 의 online DDL 컬럼 추가는 시스템 카탈로그에 추가될 정보만 반영하므로 꽤 빠른 작업이 가능하다.

- 시스템 카탈로그는 meta data 를 저장하는 역할을 수행한다.

<br>



### 그 밖의 PostgreSQL 특징들

- SP
- Vaccum
- PostGIS
- Materialized View
- 상속기능
- `pg_trgm`

<br>



#### SP

c/c++, Java, Javascript, .Net, R, Perl, Python, Ruby, Tcl 등으로 PostgreSQL 에 SP 생성이 가능하다.<br>



### PostGIS

geographic object를 지원 가능하다. oracle의 GIS 와 성능이 비견할 정도로 뛰어나다.

<br>



#### Vaccum

PostgreSQL 은 MVCC 를 MGA 방식으로 구현한다. <br>

그래서 UPDATE , DELETE시에 물리적으로 공간을 UPDATE하여 사용하지 않고 새로운 영역을 할당하여 사용하게 된다. <br>

즉 이전 공간이 재사용 될 수 없는 dead tuple 상태로 저장공간을 두게 되어서 이러한 현상이 지속될 경우, 공간 부족 및 데이터IO의 비효율을 유발하여 성능저하의 원인이 된다.<br>

따라서, 주기적으로 vacuum 기능을 수행하여 재사용 가능하도록 관리해 주어야 한다.<br>

<br>



#### Materialized View 지원

일반 View 와는 다르게 snapshot이라고 불리는 Materialized View는 view 생성 시 설정한 조건의 쿼리 결과를 별도의 공간에 저장하고 쿼리가 실행될 때 미리 저장된 결과를 보여주어 성능을 향상시킨다.

실시간 노출 필요성이 적은 통계성 쿼리나, 자주 update 되지 않는 테이블에 생성할 때 성능효과를 볼 수 있다.<br>

<br>



#### 상속 기능

부모테이블을 생성 후 상속기능을 이용해 하위 테이블을 만들 수 있다.

- 하위 테이블은 상속받은 부모테이블의 컬럼을 제외한 컬럼만 추가로 생성하면 된다.
- 상위 테이블에서 조회 시 기본적으로 하위 테이블의 데이터까지 모두 조회 가능하다.
- 데이터 변경 시에도 하위 테이블까지 모두 반영된다.

<br>



#### 다양한 사용자 기반 활용 가능

연산자, 복합 자료형, 집계함수, 자료형 변환자, 확장 기능 등 다양한 데이터베이스 객체를 사용자가 임의로 만들 수 있는 기능을 제공한다.<br>

<br>



#### `pg_trgm`

trigram매칭을 기반으로 한 모듈로 데이터 간 유사성 파악 및 like %pattern%(3자이상) 인덱스 검색이 가능하다.<br>

<br>



### PostgreSQL 을 사용하면 좋은 경우 및 단점들

(부가적으로 조금 더 참고한자료 : [Oracle, MySQL, PostgreSQL 차이점](https://velog.io/@jisoo1170/Oracle-MySQL-PostgreSQL-%EC%B0%A8%EC%9D%B4%EC%A0%90%EC%9D%80))

- 테이블 상속 기능등을 활용하고 싶은 경우
- 확장성, 호환성이 필요한 경우 PostgreSQL 이 필요하다.
- Materialized View, `pg_trgm`, Sort Join 등과 같은 검색에 유용한 기능을 사용하고자 할 때
- PostGIS 를 지원한다.
- Insert/Select 위주의 서비스에 적합 
  - (Update가 잦은 경우는 PostgreSQL 에서는 성능이 불안정)
  - 새로운 행을 insert 한 후에 과거의 행은 delete 마크를 붙이기에 update가 잦으면 계속해서 과거의 행의 양이 계속 늘어난다. 이것을 dead tuple 이라고 부른다. 이런 현상으로 인해 Vaccum 을 주기적으로 수행해줘야 한다.

- 다양한 join 방법을 제공한다.
  - nested loop join, hash join, sort merge join
  - 결합할 데이터가 많을 경우는 Hash Join, Merge Join 을 사용한다.
  - 데이터가 이미 정렬되어 있는 경우 Sort Merge 를 사용
  - 데이터가 정렬되어 있지 않다면 Hash Join 이 권장된다.


<br>

부가적인 장점

- 컬럼 추가 등과 같은 online ddl 사용시 MySQL에 비해 월등히 성능이 우월하다.
- 데이터베이스 클러스터 백업 기능을 제공한다.

<br>

> 클러스터 : 디스크로부터 데이터를 읽어오는 시간을 줄이기 위해 자주 사용되는 테이블의 데이터를 디스크의 같은 위치에 저장시키는 방법

<br>

단점 역시 있다.

- update를 할 때, 과거 행을 삭제된 표시를 하고 변경된 데이터를 가진 새로운 행을 추가하는 형태라서 update가 느리다.
- 처리 속도를 빠르게 하기 위해 여러 CPU를 활용하여 쿼리를 실행한다.
- 설정이 복잡하다. 광범위한 기능들과 표준 SQL에 대한 강력한 준수로 인해 PostgreSQL은 간단한 데이터베이스 설정에 대해서도 많은 설정이 필요하다.
- 속도가 중요하고 읽기 작업이 많으면 MySQL 이 더 실용적인 선택이 될 수 있다.

- 단순한 레플리케이션 작업에는 적합하지 않다.
  - PostgreSQL에서도 레플리케이션을 잘 제공하고 있지만 여전히 비교적 새로운 기능이다.
  - 레플리케이션은 MySQL 에서 더 성숙한 기능이고 많은 사용자들이 필요한 데이터베이스 및 시스템 관리 경험이 부족한 사용자들은 MySQL 의 레플리케이션 기능을 구현하는 것이 더 쉽다.

<br>



### PostgreSQL 에 비해 MySQL 을 고려해야 하는 경우

Update 가 잦은 경우

- MVCC 수행시 Update 가 잦은 경우 dead tuple 들이 자주 발생하는 이유로 Database 머신 내에서 Disk Full 등과 같은 이슈가 자주 발생한다. 따라서 PostgreSQL을 사용할 때 Vaccum을 자주 수행해줘야 한다는 단점이 있다.



멀티 스레드 기반의 Database 엔진을 사용하고자 할때

- MySQL 은 멀티스레드 기반의 엔진이다.

<br>



### MySQL 의 장단점

오픈소스로 무료로 사용가능하다.

- top n개의 레코드를 가지고 오는 케이스에 특화되어 있다.
- update 성능이 postgre보다 우수하다.
- Nested Loop Join만 지원한다.

- 복잡한 알고리즘은 가급적 지원하지 않는다.
- 문자열 비교에서 대소 문자를 구분하지 않는다.
- 간단한 처리 속도를 향상 시키는 것을 추구함

간단한 데이터 트랜잭션을 위한 데이터베이스가 필요한 웹 기반 프로젝트에 널리 사용된다. 로드가 많거나 복잡한 쿼리는 성능이 저하된다.<br>



> **Nested Loop Join**
> 바깥 테이블의 처리 범위를 하나씩 접근하면서 추출된 값으로 안쪽 테이블을 조인하는 방식이다.
>
> - 중첩 루프문과 동일한 원리
> - 좁은 범위에 유리하다.
> - 순차적으로 처리한다.

<br>

