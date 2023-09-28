# 커버링 인덱스

사실 조금만 공부를 시작하다보면, 커버링 인덱스는 기본적인 개념이라는 것을 알게 된다.(내가 그랬다. 요즘 Database를 깊게 공부하려고 하고 있었다.)<br>

그래도... 개발자 입장에서는 한동안 로직쪽 개발만 하다가, 비즈니스 로직을 최적화하거나, 초기 개발을 다시 시작하는 경우 복습을 할만한 커리큘럼이 필요하다.<br>

뭔가 튜토리얼처럼 복습을 할만한 커리큘럼을 만들어두기 위해 정리를 시작했다.<br>

<br>

오늘 정리하려는 개념은 이렇다. 커버링 인덱스가 적용되는 경우와 적용되지 않는 경우 이렇게 예제로 정리해보려 한다. Querydsl, JPQL, JPA 를 사용한다고 하더라도 커버링 인덱스를 잘 활용해서 조회하는 것과 커버링 인덱스를 잘 활용하지 않는것은 조금이나마 성능상에 차이가 있다.<BR>

<BR>

# 참고자료

- MySQL 성능 최적화 : [http://www.yes24.com/Product/Goods/112622445](http://www.yes24.com/Product/Goods/112622445)
- [커버링 인덱스 - tecoble](https://tecoble.techcourse.co.kr/post/2021-10-12-covering-index/)

<br>

# 커버링 인덱스란?

쉽게 설명하면 이렇다.<br>

어떤 테이블의 데이터를 조회할 때  쿼리 내의 `SELECT`, `WHERE`, `GROUP BY`, `ORDER BY` 절 에 인덱스로 지정된 컬럼만을 사용하는 경우가 있다. 이때 사용한 인덱스로 정의되어 있는 컬럼들이 쿼리를 충족하는 데에 모든 데이터를 포함(커버)한다고 해서 커버링 인덱스라고 부른다.<br>

쉽게 설명하면 SQL을 작성하는데, 거기에 SELECT, WHERE, GROUP BY, ORDER BY 에 사용된 컬럼들이 인덱스로 선언되어 있는 컬럼들이라면, 해당 컬럼들은 커버링 인덱스로 쓰였다고 이야기한다.<br>

<br>

쿼리 플랜을 통해 커버링 인덱스가 적용된 쿼리를 확인해보면 `extra` 라는 속성이 `Using Index` 라고 나타난다. (즉, 커버링인덱스 쿼리를 쿼리 콘솔에서 explain 했을 때 extra = 'Using Index' 라고 나타난다는 이야기)<br>

<br>

extra 에 대한 각각의 속성은 아래와 같다.

- Distinct : 중복을 제거하는 쿼리일 경우
- Using Where : Where 절을 사용한 쿼리
- Using temporary : 데이터 중간 결과를 위한 임시테이블을 사용한 쿼리 
  - 보통 Distinct, Group By, Order By 절이 포함된 쿼리에 대해 임시테이블이 생성된다.
  - MySQL의 쿼리 해석 엔진이 표현식을 해석해서 만들어낸 쿼리가 임시테이블을 사용하는 경우를 의미
- Using index : **커버링 인덱스** 가 적용된 쿼리일 때 
- Using filesort : 데이터를 정렬할 때 적용

<br>



# 커버링 인덱스의 장점

쿼리 성능을 최적화할 수 있다는 것이 커버링 인덱스의 장점이다.<br>

자세한 내용은 추후 정리 ㅠㅠ

<br>

# 예제실습

## 테이블 생성 및 bulk 데이터 insert

테이블은 아래와 같다.

```sql
create table study_mysql.soccer_player_test1
(
    id            bigint auto_increment
        primary key,
    name          varchar(30)  not null,
    age           int          null,
    email         varchar(100) null,
    registered_at timestamp    null,
    constraint registered_dt
        unique (registered_at)
);

create index soccer_player_test1_age_index
    on study_mysql.soccer_player_test1 (age);

```

<br>

테스트를 위해 데이터를 10만 건 넣어줬다. 데이터를 10만건 넣어주는 코드는 아래와 같이 굉장히 단순하게 작성했다.

```java
package io.study.database.mysql_study.soccer_player_test1;

// ...

@SpringBootTest
public class SoccerPlayerTest1BulkInsert {

    @Autowired
    private SoccerPlayerTest1Repository repository;

    @Test
    public void 데이터_10만개_insert(){
        List<SoccerPlayerTest1> list = new ArrayList<>();
        LocalDateTime baseTime = LocalDateTime.of(LocalDate.of(2022,11,11), LocalTime.of(11,0,0));

        for(int i=0; i<10_0000; i++){
            list.add(
                    new SoccerPlayerTest1(
                            Long.valueOf(String.valueOf(i)), 
                        	new StringBuilder("PLAYER-").append(String.valueOf(i)).toString(),
                            i+1, 
                        	new StringBuilder("player").append(String.valueOf(i)).append("@spring.io").toString(),
                            baseTime.plusSeconds(i+1)
                    )
            );
        }
        repository.saveAllAndFlush(list);
    }
}
```

<br>

## index 추가

email 필드에 대해서 = 으로 조회하는 것 말고 LIKE 연산에도 커버링인덱스가 적용되는지 테스트해보기 위해 아래와 같이 email 컬럼을 인덱스로 추가해주자

```sql
CREATE INDEX index_soccer_player_test1__email ON soccer_player_test1 (email);

// 10 만건이다 보니 274 ms 정도의 시간이 소요되었다.
// [2022-11-16 18:42:51] completed in 274 ms
```

<br>

## 여러가지 쿼리 실행해보기 - 커버링 인덱스가 적용되는 경우, 안되는 경우

### SELECT *, WHERE 절에는 INDEX 걸린 컬럼으로 조회

select 절에는 모든 컬럼을 조회하고, where 절에는 인덱스가 걸린 컬럼을 사용할 경우

```sql
EXPLAIN
SELECT *
FROM soccer_player_test1
WHERE email = 'player-1@gmail.com';
```

<br>

**출력결과**<BR>

아래 그림처럼 나타난다.(CSV가 지원이 안되어서 그림으로 일일이 캡처를 뜨기 시작...)

커버링 인덱스가 적용된 것도 아니고, 인덱스가 적용되지도 않은 일반적인 조회방식으로 조회됐다.

![1](./img/COVERING-INDEX/eg_1.png)

<BR>

### SELECT [인덱스걸린 컬럼], WHERE [인덱스걸린 컬럼]=조건값

SELECT 절과 WHERE 절 모두에 인덱스가 걸린 컬럼만을 사용했을 때는 아래와 같이 `Extra=Using index` 라는 결과값을 얻었다. `Extra=Using index` 라는 것은 커버링인덱스가 걸린 것을 의미한다.

![1](./img/COVERING-INDEX/eg_2.png)

<br>

### Like 절에 대해 select 시 index 적용되는지 테스트

#### index가 걸려있지 않은 일반 컬럼에 대해 Where [col] LIKE '...' 수행시

먼저 아까 추가해뒀던 email 에 대한 인덱스를 지워두고, 처음의 테이블 상태로 둔다.(쿼리는 생략.)

email 컬럼에 인덱스가 걸려있지 않은 상태에서 Like 절로 조회시에 모든 컬럼을 조회해보자.

```sql
EXPLAIN
SELECT *
FROM soccer_player_test1
WHERE email LIKE 'player-1%'
GROUP BY id, age;


-- 출력 (26MS)
study_mysql> EXPLAIN
             SELECT *
             FROM soccer_player_test1
             WHERE email LIKE 'player-1@%'
             GROUP BY id, age
[2022-11-16 19:33:47] [HY000][1003] /* select#1 */ select `study_mysql`.`soccer_player_test1`.`id` AS `id`,`study_mysql`.`soccer_player_test1`.`name` AS `name`,`study_mysql`.`soccer_player_test1`.`age` AS `age`,`study_mysql`.`soccer_player_test1`.`email` AS `email`,`study_mysql`.`soccer_player_test1`.`registered_at` AS `registered_at` from `study_mysql`.`soccer_player_test1` where (`study_mysql`.`soccer_player_test1`.`email` like 'player-1@%') group by `study_mysql`.`soccer_player_test1`.`id`,`study_mysql`.`soccer_player_test1`.`age`
[2022-11-16 19:33:47] 1 row retrieved starting from 1 in 26 ms (execution: 7 ms, fetching: 19 ms)
```

<Br>

출력결과는 아래와 같다.

Using where 이 사용된 것을 볼 수 있다.

![1](./img/COVERING-INDEX/eg_3.png)

<br>

#### index 가 적용된 컬럼에 대해 where [col] LIKE '...' 수행

 이번에는 email, id , age 순의 조합으로 인덱스를 하나 만들어준다.

```sql
CREATE INDEX index_soccer_player_test1__email_id_age ON soccer_player_test1(email, id, age);

-- 결과
-- study_mysql> CREATE INDEX index_soccer_player_test1__email_id_age ON soccer_player_test1(email, id, age)
[2022-11-16 19:14:28] completed in 285 ms
```

<br>

아래와 같이 인덱스가 적용된 컬럼의 순서대로 아래의 쿼리를 수행해보자.

위의 예제에 비해 22MS 로 더 빠르게 조회되었음을 확인 가능하다.

```SQL
EXPLAIN
SELECT email, id, age
FROM soccer_player_test1
WHERE email LIKE 'player-1%'
GROUP BY id, age;

study_mysql> EXPLAIN
             SELECT email, id, age
             FROM soccer_player_test1
             WHERE email LIKE 'player-1%'
             GROUP BY id, age
[2022-11-16 19:36:46] [HY000][1003] /* select#1 */ select `study_mysql`.`soccer_player_test1`.`email` AS `email`,`study_mysql`.`soccer_player_test1`.`id` AS `id`,`study_mysql`.`soccer_player_test1`.`age` AS `age` from `study_mysql`.`soccer_player_test1` where (`study_mysql`.`soccer_player_test1`.`email` like 'player-1%') group by `study_mysql`.`soccer_player_test1`.`id`,`study_mysql`.`soccer_player_test1`.`age`
[2022-11-16 19:36:46] 1 row retrieved starting from 1 in 22 ms (execution: 7 ms, fetching: 15 ms)
```

<BR>

![1](./img/COVERING-INDEX/eg_4.png)

<BR>

## 복합키 인덱스에 대한 Where절 + Group By 사용시 인덱스 적용 유무 

Group By 절을 사용할 때는 아래와 같은 조건에 대해서 인덱스가 생성된다.

- 인덱스 컬럼과 Group By 에 명시하는 컬럼의 순서는 동일해야 한다.
- 인덱스가 걸린 컬럼 들 중 뒤에 있는 컬럼들은 Group By에 명시하지 않아도 된다.
- 인덱스가 걸린 컬럼 들 중 앞에 있는 컬럼들은 Group By에 명시해야 한다.
- 인덱스에 없는 컬럼을 Group By에 명시하면 인덱스 기반 조회는 수행되지 않고 일반 조회가 수행된다.

<br>

e.g.

```sql
Group By b			-- 인덱스 적용 x
Group By b, a 		-- 인덱스 적용 x
Group By a, c, b	-- 인덱스 적용 x

Group By a			-- 인덱스 적용 o
Group By a, b		-- 인덱스 적용 o
Group By a, b, c	-- 인덱스 적용 o

Group By b, c		-- 인덱스 적용 x

Group By a,b,c,d	-- 인덱스 적용 x
```

<br>

위에 대한 예제를 모두 예로 들기에는 문서가 다소 길어질 것 같아 예제들은 생략한다.<br>

<br>

## where col1 동등비교 + Group By col2, col3 

Where + Group By 가 함께 사용되면, Where 에 있는 컬럼은 Group By 에 없어도 된다.<br>

"Where 에서 이미 컬럼을 하나 사용했으니, Group By에서는 생략해도 된다" 이런 의미로 해석하면 기억에 오래 남지 않을까 싶다. <br>

실행문맥관점에서 생각해보면 SQL이 실행문맥이 들어갈때 Where 절에서 사용한 컬럼이 이미 해석이 되었기에 생략해도 되는 것이라는 의미인것 같기도 하다.<bR>

<br>

위에서 생성했던 인덱스를 만들자. (이미 생성해둔 상태라면 패스)

```sql
CREATE INDEX index_soccer_player_test1__email_id_age ON soccer_player_test1(email, id, age);
```

<br>

조회문을 아래와 같이 작성해보자.

```sql
EXPLAIN
SELECT *
FROM soccer_player_test1
WHERE email = 'player-1@gmail.com'
GROUP BY id, age;

study_mysql> EXPLAIN
             SELECT *
             FROM soccer_player_test1
             WHERE email = 'player-1@gmail.com'
             GROUP BY id, age
[2022-11-16 20:22:04] [HY000][1003] /* select#1 */ select `study_mysql`.`soccer_player_test1`.`id` AS `id`,`study_mysql`.`soccer_player_test1`.`name` AS `name`,`study_mysql`.`soccer_player_test1`.`age` AS `age`,`study_mysql`.`soccer_player_test1`.`email` AS `email`,`study_mysql`.`soccer_player_test1`.`registered_at` AS `registered_at` from `study_mysql`.`soccer_player_test1` where (`study_mysql`.`soccer_player_test1`.`email` = 'player-1@gmail.com') group by `study_mysql`.`soccer_player_test1`.`id`,`study_mysql`.`soccer_player_test1`.`age`
[2022-11-16 20:22:04] 1 row retrieved starting from 1 in 40 ms (execution: 9 ms, fetching: 31 ms)
```

<br>

결과를 보면 아래와 같이 `Extra = Using Index condition` 이 나왔다. 커버링 인덱스가 적용됐다.<br>

where 절에서 복합키로 사용된 col1 인 email 을 사용했고

 나머지 col2, col3 인 id, age를 group by 를 사용했다. 

복합키를 모두 사용했기에 커버링 인덱스가 적용됐다.

위에서 정리했듯이 a,b,c 방식으로 where , group by 절에 사용되면 커버링인덱스가 적용된다.<br>

도는 a,b 방식으로 where, group by 절에 사용되어도 커버링 인덱스가 적용된다.

![1](./img/COVERING-INDEX/eg_5.png)

<br>

위의 쿼리는 Explain 결과문에 보이듯 40ms 가 소요됐다.<br>

<br>

## where col1 동등비교 아닌 범위 연산 + Group By col2, col3

이번에도 역시 복합키를 순서대로 where 절과 group by 절을 스칠때 커버링인덱스가 적용되는지를 확인한다.<br>

실습에 앞서 아래의 인덱스가 없다면 생성해주자.

```sql
CREATE INDEX index_soccer_player_test1__email_id_age ON soccer_player_test1(email, id, age);
```

<br>

아래와 같은 쿼리를 실행해보자.

```sql
EXPLAIN
SELECT *
FROM soccer_player_test1
WHERE email like 'player-1%'
GROUP BY id, age;

study_mysql> EXPLAIN
             SELECT *
             FROM soccer_player_test1
             WHERE email like 'player-1%'
             GROUP BY id, age
[2022-11-16 20:30:44] [HY000][1003] /* select#1 */ select `study_mysql`.`soccer_player_test1`.`id` AS `id`,`study_mysql`.`soccer_player_test1`.`name` AS `name`,`study_mysql`.`soccer_player_test1`.`age` AS `age`,`study_mysql`.`soccer_player_test1`.`email` AS `email`,`study_mysql`.`soccer_player_test1`.`registered_at` AS `registered_at` from `study_mysql`.`soccer_player_test1` where (`study_mysql`.`soccer_player_test1`.`email` like 'player-1%') group by `study_mysql`.`soccer_player_test1`.`id`,`study_mysql`.`soccer_player_test1`.`age`
[2022-11-16 20:30:44] 1 row retrieved starting from 1 in 29 ms (execution: 7 ms, fetching: 22 ms)
```

<br>

실행결과를 보면 아래와 같이 커버링 인덱스가 적용된 것을 볼 수 있다.

![1](./img/COVERING-INDEX/eg_6.png)

<br>

이번에 사용한 쿼리는 explain 결과문을 보면 29ms 가 소요된것을 볼 수 있다.<br>

<br>

위의 쿼리를 커버링 인덱스를 걸치는 것이 아닌 raw 쿼리로 조회하는 경우를 살펴보자. <br>

테스트를 위해 방금전 생성했던 인덱스를 지워주자.

```sql
DROP INDEX index_soccer_player_test1__email_id_age ON soccer_player_test1;
```

<br>

이제 조회문을 실행해보자<br>

아래 결과에서 보듯 36ms 가 소모됐다.

커버링 인덱스가 적용되도록 작성한 LIKE + GROUP BY 쿼리는 29ms 가 소요됐고

커버링 인덱스 없이 raw sql 을 그대로 LIKE + GROUP BY 쿼리를 실행한 결과는 36ms가 소요됐다.

확실히 7ms 정도의 수행속도가 느려졌다.

```sql
study_mysql> EXPLAIN
             SELECT *
             FROM soccer_player_test1
             WHERE email like 'player-1%'
             GROUP BY id, age
[2022-11-16 20:37:44] [HY000][1003] /* select#1 */ select `study_mysql`.`soccer_player_test1`.`id` AS `id`,`study_mysql`.`soccer_player_test1`.`name` AS `name`,`study_mysql`.`soccer_player_test1`.`age` AS `age`,`study_mysql`.`soccer_player_test1`.`email` AS `email`,`study_mysql`.`soccer_player_test1`.`registered_at` AS `registered_at` from `study_mysql`.`soccer_player_test1` where (`study_mysql`.`soccer_player_test1`.`email` like 'player-1%') group by `study_mysql`.`soccer_player_test1`.`id`,`study_mysql`.`soccer_player_test1`.`age`
[2022-11-16 20:37:44] 1 row retrieved starting from 1 in 36 ms (execution: 11 ms, fetching: 25 ms)
```

<br>

# 요약

나머지는 내일 이후로 정리해야겠다. Database 관련해서 내용 정리하는 것은 왜 이렇게 시간이 불필요하게 많이 드는 작업들이 많은지 모르겠다. 오늘 정리하면서 필요한 요소들이 그나마 갖춰져서 다음부터 정리시에는 그나마 정리하는 데에 금방 정리하지 않을까 싶기는 하다.<bR>

<br>

## 커버링 인덱스가 적용되는 경우, 적용안되는 경우들

인덱스 적용되는 경우

인덱스 적용안되는 경우

<br>

<br>

## 결론

커버링 인덱스가 적용되면 쿼리가 빨라진다. 오늘 테스트에서는 10만건이라는 다소 적은 범위의 데이터로 테스트를 수행했다. 그런데도 커버링 인덱스를 적용되었을때와 적용하지 않았을 때의 where, group by 절의 성능 차이는 7ms 정도 차이가 났다.<br>

ORM을 쓰더라도 쿼리는 작성해야 한다. 조회 쿼리를 작서할 때 가급적이면, 커버링 인덱스에 걸쳐지게끔 컬럼 순서를 잘 맞춰서 쓰는 연습도 해보자.<Br>

<br>

