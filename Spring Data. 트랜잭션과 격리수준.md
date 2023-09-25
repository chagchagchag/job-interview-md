# 트랜잭션과 격리수준

트랜잭션은 여러가지 분야 또는 여러가지 의미로 쓰인다. 이번 문서에서 정리할 트랜잭션의 개념은 **Database 에 대해 수행하는 한 번의 작업의 논리적 단위**이다.<br>

<br>



## 참고자료

- [트랜잭션 - ko.wikipedia.org](https://ko.wikipedia.org/wiki/%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%98)
- [데이터베이스 트랜잭션 - ko.wikipedia.org](https://ko.wikipedia.org/wiki/%EB%8D%B0%EC%9D%B4%ED%84%B0%EB%B2%A0%EC%9D%B4%EC%8A%A4_%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%98)
- [자바 ORM 표준 JPA 프로그래밍](https://ridibooks.com/books/3984000009)
- DBMS 별 트랜잭션 기본 격리수준
  - [Postgresql - Transaction Isolation Level](https://zamezzz.tistory.com/318)
  - [MySQL 의 격리수준](https://velog.io/@jsj3282/11.-MySQL%EC%9D%98-%EA%B2%A9%EB%A6%AC-%EC%88%98%EC%A4%80)
  - [MySQL - 트랜잭션 격리수준](https://zzang9ha.tistory.com/381)
  - [MySQL 트랜잭션 격리수준](https://steady-coding.tistory.com/562)

- MVCC
  - [MySQL MVCC](https://joont92.github.io/db/MVCC/)
  - [PostgreSQL MVCC](https://simpledb.tistory.com/71)



<br>



## 트랜잭션과 트랜잭션의 격리수준

아래에서 정리하고 있지만, 트랜잭션은 기본적으로 ACID 라는 원칙을 지켜야 하는데, 이 중에서 가장 지키기 어려운 원칙은 I(Isolation)다.<br>

그래서 ANSI 에서는 네가지의 트랜잭션 격리 수준을 정의하고 있다.<br>

상용 Database 들 중 유명한 제품군들의 기본 격리수준만 정리해보면 아래와 같다.

- MySQL (InnoDB 스토리지 엔진을 사용할 경우)
  - Repeatable Read
- Oracle
  - Read Committed
- Postgresql
  - Read Committed

<br>

대부분의 Database는 Read Committed 이상의 레벨을 기본 격리수준으로 채택하고 있음을 알 수 있다.<br>

<br>



### MVCC (Multi Version Concurrency Content)

> 참고
>
> - [MySQL MVCC](https://joont92.github.io/db/MVCC/)
> - [PostgreSQL MVCC](https://simpledb.tistory.com/71)

<br>

격리 수준을 어느 정도 보장하기 위해서는 잠금(락, Lock)기능을 사용하게 된다. 그런데 잠금(락, Lock)을 사용하지 않고도 일관된 읽기를 제공하기 위해, 대부분의 데이터베이스들은 MVCC 를 제공한다.<br>

MySQL

- 언두로그를 이용해 이 기능을 제공한다. <br>

Postgresql

- Vaccum 을 이용해 이 MVCC 기능을 제공한다.<br>

<br>

MVCC 를 사용하면, 데이터 조회 요청시 수정/삭제 중인 데이터의 커밋,롤백 전/후에 데이터파일의 내용을 그대로 접근해서 보여 주지 않는다. 대신, 트랜잭션마다 트랜잭션이 시작한 시점에 따라 매겨진 버전으로 관리되는 데이터의 사본(복제본)이 있는데, 이 버전에 따라 해당 버전의 데이터를 조회 결과로 돌려준다.<br>

MySQL 은 Undo 로그로, Postgresql 은 Vaccum 을 이용해서 이 MVCC 기능을 제공한다. 쉽게 생각하면, 조회 데이터의 일관성을 제공하기 위해 데이터의 버전을 매겨서 관리한다고 생각할 수 있을 것 같다.<br>

그리고 이 MVCC기능이 제공되는 이유는 잠금(락, Lock)을 사용하지 않고 동시성을 지원하면서 데이터 접근시 일관성을 제공하기 위해서이다.<br>

<br>



## 트랜잭션을 구성하는 기본 원칙 - ACID

트랜잭션은 아래의 네가지 원칙이 보장되어야 한다.

- 원자성(Atomicity)
  - 트랜잭션 내에서 실행한 작업들은 <u>마치 하나의 작업인 것처럼 모두 성공하거나 모두 실패해야</u> 한다.
  - ex) Spring의 @Transactional 에 구현한 내용들에 여러개의 작업단위가 있어도 모두 하나의 작업단위처럼 취급되어서 실패할 경우 해당 작업단위로 롤백된다.
- 일관성(Consistency)
  - 트랜잭션은 데이터베이스의 <u>무결성 제약조건을 항상 만족시켜야</u> 한다.
  - = 모든 트랜잭션은 일관성있는 데이터베이스 상태를 유지해야 한다.
- 격리성(Isolation)
  - 동시에 실행되는 여러개의 트랜잭션들이 <u>서로에게 영향을 미치지 않도록 격리</u>될 수 있어야 한다.
  - 트랜잭션 로직을 구현할 때 가장 보장하기 힘든 원칙이 **격리성**이다. 이런 이유로 ANSI 에서는 네 가지 수준의 격리수준을 정의하고 있다.
  - 이 네 가지 격리 수준 중 하나를 선택해서 트랜잭션 로직을 작성할 수 있다면 트랜잭션의 격리성을 만족하는 것이다.
  - 참고로 격리성을 완벽하게 보장하려면 트랜잭션을 거의 차례대로 수행해야만 한다.
    - 이것을 Serializable 단계라고 하는데, 트래픽이 많은 시스템은 시스템을 느려지게 만들 수 있는 요인이다.
- 지속성(Durability)
  - 트랜잭션을 성공하면 그 결과는 <u>기록되어야</u> 한다.
  - 트랜잭션이 성공했어도 시스템 문제가 발생할 경우, 데이터베이스 로그 등을 이용해 성공한 트랜잭션 내용을 복구할 수 있어야 한다.

<br>



**참고) ACID (Atomicity - Consistency - Isolation - Durability)**<br>

- ACID ([en.wikipedia.org/wiki/ACID](https://en.wikipedia.org/wiki/ACID))

<br>



## 격리수준

여러 개의 트랜잭션은 서로에게 영향을 미치지 않도록 격리될 수 있어야 한다. **트랜잭션을 구현할 때 가장 보장하기 힘든 원칙**이 **격리성** 이다. 이런 이유로 ANSI에서는 네가지의 격리수준을 정의하고 있다. 



- `READ UNCOMMITTED (커밋되지 않은 읽기)`
  - 커밋되지 않은 것이라도 읽을 수 있다
  - 읽어들인 커밋되지 않은 데이터가 롤백되면 데이터 정합성에 문제가 생긴다.
  - DIRTY READ 발생
    - t1이 데이터를 수정하고 있는데, 커밋하지 않았는데도 t2가 커밋되지 않은 데이터를 읽어들일 수 있는 현상을 DIRTY READ 라고 한다.
    - 만약 t2가 DIRTY READ 한 데이터를 사용하는데 t1이 수정하고 있던 데이터를 롤백하게 될 경우 데이터 정합성에 문제를 일으키게 된다.
- `READ COMMITTED (커밋된 읽기)`
  - t1이 조회작업을 수행 도중에 t2가 수정/커밋을 했을 경우 t2에 의해 수정/커밋된 버전으로 읽어들인다.
  - NON REPEATABLE READ 가 발생한다.
    - 조회작업를 수행 도중에 새롭게 수정된 데이터를 새롭게 읽어들이므로 매번 같은 데이터를 읽어들일 수 없게된다. 
    - 이렇게 수정/커밋된 데이터에 대해 매번 같은 데이터를 읽어들일 수 없는 현상을 NON REPEATABLE READ 라고 부른다.
- `REPEATABLE READ (반복가능한 읽기)`
  - 위의 READ COMMITED 가 보장하지 못하는 **수정/커밋 된 데이터에 대해 REPEATABLE READ 가 가능하도록 해주는 격리수준**이다.
  - t2가 수정/커밋을 완료된 후에 t1의 조회작업을 마무리하는 것.
  - 즉, 다른 트랜잭션에서 수정/커밋했어도 한번 조회했던 데이터를 반복해서 조회해도 같은 데이터가 조회된다.
  - 하지만 수정/커밋된 데이터가 아닌 **새롭게 추가된 데이터**에 대해서는 반복 조회시 같은 결과값을 조회할 수 있음을 보장하지 못한다.
    - 이런 현상을 PHANTOM READ 라고 이야기한다.
    - 새롭게 유령처럼 뭔가 한 행이 추가되어서 읽혀졌다는 의미
  - (= ex. t2가 수정/커밋이 아닌 추가 작업을 했을 경우는 새롭게 추가된 데이터도 함께 조회결과에 포함되게 되어 반복적으로 읽더라도 같은 결과집합 임을 보장하지는 못한다.) 
- `SERIALIZABLE (직렬화 가능)`
  - 가장 엄격한 트랜잭션 격리수준이다. 
  - PHANTOM READ 조차도 발생하지 않는다.
  - 하지만 동시성 처리 성능이 급격히 떨어질 수 있게 된다.

<br>



### ex) DIRTY READ - READ UNCOMMITTED

- t1이 데이터를 수정하고 있는데, 커밋하지 않았는데도 t2가 커밋되지 않은 데이터를 읽어들일 수 있는 현상을 DIRTY READ 라고 한다.
- 만약 t2가 DIRTY READ 한 데이터를 사용하는데 t1이 수정하고 있던 데이터를 롤백하게 될 경우 데이터 정합성에 문제를 일으키게 된다.

<br>



### ex) NON REPEATABLE READ - READ COMMITTED

트랜잭션 t1 이 회원 A 를 조회 중인데, 갑자기 트랜잭션 t2가 회원 A를 수정하고 커밋하면, 트랜잭션 t1이 다시 회원 A를 조회했을 때 수정된 데이터가 조회된다. DIRTY READ는 허용하지 않는다. 커밋한 데이터만을 읽어들이기 때문이다.<br>

<br>



### ex) REPEATABLE READ - PHANTOM READ

트랜잭션 t1이 10살 이하의 회원을 조회했는데 트랜잭션 t2가 5살 회원을 하나 추가하고 커밋하면, 트랜잭션 t1 이 다시 10살 이하의 회원을 조회했을 때 회원 하나가 추가된 상태로 조회된다. 이처럼 반복 조회시 결과 집합이 달라지는 것을 PHANTOM READ라고 한다. (마치 유령처럼 하나의 행이 달라 붙었다. 이런 의미)<br>

<br>



### ex) SERIALIZABLE

가장 엄격한 격리수준.<br>

<br>





