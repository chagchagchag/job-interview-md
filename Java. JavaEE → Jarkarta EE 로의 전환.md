### Java. JavaEE → Jarkarta EE 로의 전환

참고자료

- [Java EE 에서 Jakarta EE 로의 전환](https://www.samsungsds.com/kr/insights/java_jakarta.html)

<br>



오라클은 JavaEE 8 릴리즈를 마지막으로 오픈소스 SW 를 지원하는 비영리 단체인 이클립스 재단에 자바EE프로젝트를 이관했다. (수익화 실패 등과 같은 이슈가 있을 것으로 추측)

이클립스 재단으로 이관된 JAVA EE 의 공식 명칭은 자카르타EE, 프로젝트 명은 EE4J (Eclipse Enterprise for Java) 로 변경됐다. JCP 정책 역시 기존정책에서 탈피해 오픈소스 기반의 자카르타 EE 사양 프로세스 (Jakarta EE Specification Process, JESP) 라는 정책으로 전환했는데 이 정책은 개방적이고 중립적인 정책이다.<br>

자바 네임스페이스와 API 패키지명 역시 변경되었다.

- 자바 네임스페이스는 Jakarta로 변경되고
- API 패키지명은 Jakarta.\* 로 변경됐다. (javax.\* → jakarta.\*)

> 이와 같은 자카르타의 네임스페이스 변화는 기존 개발자들에게 다소 혼란을 줄 것으로 예상됩니다.

<br>

자카르타 EE는 자바 EE를 대체하지 않는다. JAVAEE 에서 하드포크된 새로운 플랫폼이다.  JAVA EE 는 계속 유지되지만 8 버전을 마지막으로 더 이상 릴리즈, 추가 기능은 제공되지 않는다. 

|        Java EE 용어        |  Jakarta EE 용어  |                          |                     |
| :------------------------: | :---------------: | ------------------------ | ------------------- |
|        Java Servlet        |   javax.servlet   | Jakarta Servlet          | jakarta.servlet     |
|   JavaServer Pages (JSP)   | javax.servlet.jsp | Jakarta Server Pages     | jakarta.servlet.jsp |
|   JavaServer Faces (JSF)   |    javax.faces    | Jakarta Server Faces     | jakarta.faces       |
| Java Message Service (JMS) |     javax.jms     | Jakarta Messaging        | jakarta.jms         |
| Java Persistence API (JPA) | javax.persistence | Jakarta Persistence      | jakarta.persistence |
| Java Transaction API (JTA) | javax.transaction | Jakarta Transaction      | jakarta.transaction |
| Enterprise JavaBeans (EJB) |     javax.ejb     | Jakarta Enterprise Beans | jakarta.ejb         |
|         Java Mail          |    javax.mail     | Jakarta Mail             | Jakarta.mail        |