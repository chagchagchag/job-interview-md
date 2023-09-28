### Database. (MySQL) Clustered Index, Non Clustered Index



### 참고자료

https://velog.io/@gillog/SQL-Clustered-Index-Non-Clustered-Index

<br>



### Index, Clustered Index, Non Clustered Index

Index 는 열(컬럼)단위로 생성된다. 하나의 열(컬럼)에 Index를 생성할수도 있고, 여러 열에도 Index를 생성할 수 있다.<br>

제약 조건 없이 테이블 생성시 Index를 만들수 없다.<br>

인덱스가 자동으로 하나 이상 생성되게끔 하려면, Primary Key 또는 Unique Key 제약 조건을 추가해줘야 한다.<br>

Index에는 Clustered Index, Non Clustered Index 이렇게 두 종류가 존재한다.<br>

<br>



### Primary Key 또는 Unique Key 생성시 Clustered Index, Non Clustered Index 생성 규칙

Clustered Index

- 테이블 생성시 하나의 열에 Primary Key 를 지정하면 자동으로 Clustered Index 가 생성된다.
- Primary Key 에 대해 강제로 Non-Clustered Index로 지정하는 것 역시 가능하다.

<br>

Non Clustered Index

- 테이블을 Unique 제약 조건과 함께 생성하면, 데이터베이스 엔진은 자동으로 Non Clustered Index를 만든다.
- 이미 존재하는 테이블에 Primary Key 제약 조건을 적용하려 하면 Non Clustered Index 가 적용된다.
- 테이블에 이미 Clustered Index가 있는 상태에서 Primary Key 제약 조건을 적용하려 하면 Non Clustered Index 가 적용된 기본키가 적용된다.

<br>



### 클러스터드 인덱스, Primary Index, non 클러스터드 인덱스, Secondary Index

**클러스터드 인덱스, Primary Key(프라이머리 키)**

클러스터(Cluster)라는 단어는 여러 개를 하나로 묶는다는 의미로 주로 사용된다.<br>

클러스터 인덱스는 비슷한 것들을 묶어서 저장하는 형태의 인덱스를 의미한다. <br>

클러스터드 인덱스는 주로 비슷한 값들을 동시에 조회하는 경우가 많다는 점에 착안하여 고안된 개념이다.<br>

**비슷한 값** 이라는 의미는 **물리적으로 인접한 장소에 저장되어 있는 데이터들**을 의미한다.<br>

클러스터드 인덱스는 **테이블의 프라이머리 키에 대해서만 적용**된다.<br>

따라서 **테이블 당 한 개만 생성할 수 있다.**<br>

**프라이머리 키 값이 비슷한 레코드 끼리 묶어서 저장하는 것**을 <u>클러스터드 인덱스</u>라고 표현한다.<br>

<br>



**프라이머리 키와 클러스터드 인덱스**<br>

클러스터드 인덱스에서는 프라이머리 키 값에 의해 레코드의 저장위치가 결정된다.<br>

프라이머리 키 값이 변경되면, 그 레코드의 물리적인 저장 위치 또한 변경되어야 한다.<br>

따라서 프라이머리 키를 신중하게 결정하고 클러스터드 인덱스를 사용해야 한다.<br>

<br>



**non 클러스터드 인덱스, Secondary Key(세컨더리 키)**<br>

non 클러스터드 인덱스는 테이블 당 여러개 생성할 수 있다.





