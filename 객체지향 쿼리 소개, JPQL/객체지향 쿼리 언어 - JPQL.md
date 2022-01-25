- JPQL(Java Persistence Query Language)은 엔티티 객체를 조회하는 객체지향 쿼리다.
- JPQL은 SQL을 추상화하여 특정 데이터베이스에 의존하지 않는다.
- JPQL은 엔티티 직접 조회, 묵시적 조인, 다형성 지원으로 SQL보다 코드가 간결하다.

## 기본 문법과 쿼리 API

### SELECT 문

```
SELECT m FROM Member AS m where m.username = 'Hello'
```

- 대소문자 구분
    - 엔티티와 속성은 대소문자를 구분한다.
    - SELECT, FROM, AS 같은 JPQL 키워드는 대소문자를 구분하지 않는다.
- 엔티티 이름
    - JPQL에서 사용한 Member는 클래스 명이 아니라 엔티티 명이다.
    - 엔티티 명을 지정하지 않으면 클래스 명을 기본값으로 사용한다.
    - 기본값인 클래스 명을 엔티티 명으로 사용하는 것을 추천한다.
- 별칭은 필수
    - Member AS m을 보면 Member에 m이라는 별칭을 주었다. JPQL은 별칭을 필수로 사용해야 한다.

### TypeQuery, Query

작성한 JPQL을 실행하려면 쿼리 객체를 만들어야 한다. 쿼리 객체의 종류는 다음과 같다.

- TypeQuery : 반환 타입을 명확하게 지정할 수 있을 경우 사용한다.

```java
TypedQuery<Member> query = 
		em.createQuery("SELECT m FROM Member m", Member.class);

List<Member> resultList = query.getResultList();
```

- Query : 반환 타입을 명확하게 지정할 수 없을 경우 사용한다.

```java
Query query = 
		em.createQuery("SELECT m FROM Member m");

List resultList2 = query.getResultList();

for (Object o : resultList) {
    Object[] result = (Object[]) o;
}
```

두 코드를 비교해보면 타입을 변환할 필요가 없는 TypeQuery를 사용하는 것이 더 편리한 것을 알 수 있다.

### 결과 조회

다음 메소드들을 호출하면 실제 쿼리를 실행해서 조회한다.

- `query.getResultList()`
    - 결과를 반환한다. 만약 결과가 없으면 빈 컬렉션을 반환한다.
- `query.getSingleResult()`
    - 결과가 정확히 하나일 때 사용한다.
    - 결과가 없거나 1개보다 많으면 예외가 발생한다.

## 파라미터 바인딩

### 이름 기준

- 파라미터를 이름을 구분하는 방법이다.
- 기준 파라미터는 앞에  `:`를 사용해야 한다.

```java
String usernameParam = "User1";

TypeQuery<Member> query =
    em.createQuery("SELECT m FROM Member m WHERE m.username = :username", Member.class);
    .setParameter("username", usernameParam);
```

### 위치 기준 파라미터

- `?` 다음에 위치 값을 주면 된다.
- 위치 값은 1부터 시작한다.

```java
List<member> resultList =
    em.createQuery("SELECT m FROM Member m WHERE m.username = ?1", Member.class)
    .setParameter(1, usernameParam)
    .getResultList();
```

참고로 파라미터 바인딩 방식을 사용하지 않으면 SQL 인젝션, 성능 이슈 등 다양한 문제들이 많으니 꼭 파라미터 바인딩 방식을 사용하자.

## 프로젝션

- SELECT 절에 조회 할 대상을 지정하는 것을 프로젝션이라 한다.
- 프로젝션 대상은 엔티티, 임베디드 타입, 스칼라 타입이 있다.

### 엔티티 프로젝션

```java
SELECT m FROM Member m // 회원
SELECT m.team FROM Member m // 팀
```

참고로 이렇게 조회한 엔티티는 영속성 컨텍스트에서 관리된다.

### 임베디드 타입 프로젝션

임베디드 타입인 Address를 조회하는 예제이다.

```java
SELECT a From Address a //주소
```

참고로 임베디드 타입은 엔티티 타입이 아닌 값 타입이기 때문에, 영속성 컨텍스트에서 관리되지 않는다.

### 스칼라 타입 프로젝션

숫자, 문자, 날짜와 같은 기본 데이터 타입들을 스칼라 타입이라 한다.

```java
List<String> usernames =
    em.createQuery("SELECT username FROM Membe", Member.class)
    .getResultList();
```

### 여러 값 조회

NEW 명령어를 사용해서 `UserDto`처럼 의미 있는 객체로 변환해서 사용할 수 있다.

```java
public class UserDTO {

    private String username;
    private int age;
}

TypedQuery<UserDTO> query = 
    em.createQuery("SELECT new jpabook.jpql.UserDTO(m.username, m.age) FROM Member m", 
    UserDTO.class);

List<UserDTO> resultList = query.getResultList();
```

NEW 명령어를 사용할 때는 2가지를 주의해야 한다.

- 패키지 명을 포함한 전체 클래스 명을 입력해야 한다.
- 순서와 타입이 일치하는 생성자가 필요하다.

## 페이징 API

JPA는 페이징을 다음 두 API로 추상화하였다.

- `setFirstResult(int startPosition)`  : 조회 시작 위치(0부터 시작)
- `setMaxResults(int maxResult)`  : 조회할 데이터 수

```java
query.setFirstResult(10);
query.setMaxResults(20);
query.getResultList();
```

## 집합과 정렬

### 집합 함수

- 집합 함수는 COUNT, MAX, MIN, AVG, SUM이 있다.
- 집합 함수 사용 시 주의할 점은 다음과 같다.
    - NULL 값은 무시한다.
    - 만약 값이 없는데 집합 함수를 사용하면 NULL이 된다. 단, COUNT는 0.
    - DISTINCT를 집합 함수 안에 사용해서 중복된 값을 걸러내고 집합을 구할 수 있다.
    - DISTINCT를 COUNT에서 사용할 때 임베디드 타입은 지원하지 않는다.

### GROUP BY, HAVING

- GROUP BY는 통계 데이터를 구할 때 특정 그룹끼리 묶어 준다.
- HAVING은 그룹별 통계 데이터 중에서 평균 나이가 10살 이상인 그룹을 조회한다.

```java
select t.name, COUNT(m.age), SUM(m.age), AVG(m.age), MAX(m.age), MIN(m.age)
from Member m LEFT JOIN m.team t
GROUP BY t.name
HAVING AVG(m.age) >= 10
```

### 정렬

- ASC : 오름차순(기본값)
- DESC : 내림차순

```java
select t.name, COUNT(m.age) as cnt
from Member m LEFT JOIN m.team t
GROUP BY t.name
ORDER BY cnt
```

## JPQL 조인

### 내부 조인

- 내부 조인은 INNER 조인을 사용하며, INNER은 생략할 수 있다.
- JPQL 조인의 가장 큰 특징은 연관 필드를 사용한다는 것이다.
- 연관 관계 필드는 `m.team`처럼 다른 엔티티와 연관관계를 가지기 위해 사용하는 필드를 말한다.

```java
SELECT m
FROM Member m INNER JOIN m.team t
where t.name = :teamName
```

### 외부 조인

- 외부 조인은 기능상 SQL의 외부 조인과 같다.
- OUTER는 생략 가능해서 보통 LEFT JOIN으로 사용한다.

```java
SELECT m
FROM Member m LEFT OUTER JOIN m.team t
where t.name = :teamName
```

### 컬렉션 조인

- 일대다 관계나 다대다 관계처럼 컬렉션을 사용하는 곳에 조인하는 것을 컬렉션 조인이라 한다.
- [회원 → 팀] 조인은 단일 값 연관 필드(`m.team`)를 사용한다.
- [팀 → 회원] 조인은 컬렉션 값 연관 필드(`m.members`)를 사용한다.

```java
SELECT t, m
FROM Team t LEFT JOIN t.members m
```

### 세타 조인

- WHERE 절을 사용해서 세타 조인을 할 수 있다.
- 세타 조인은 내부 조인만 지원한다.

```java
select count(m) from Member m, Team t
where m.username = t.name
```

## 페치 조인

- 페치 조인은 SQL에서 말하는 조인의 종류는 아니고, JPQL에서 성능 최적화를 위해 제공하는 기능이다.
- `fetch join` 명령어로 연관된 엔티티나 컬렉션을 한 번에 같이 조회할 수 있다.

### 엔티티 페치 조인

- 페치 조인을 사용하면 회원 엔티티를 조회하면서 연관된 팀 엔티티도 함께 조회할 수 있다.
- JPQL 조인과는 다르게 `m.team` 다음에 별칭을 사용할 수 없다.

```java
select m
from Member m join fetch m.team
```

페치 조인을 사용하면 아래 그림처럼 SQL 조인을 시도한다.

![image](https://user-images.githubusercontent.com/55661631/150926056-469c16db-c1aa-4bf2-87b9-3437732e447b.png)

조인의 결과는 다음과 같다.

![image](https://user-images.githubusercontent.com/55661631/150926070-c445a4b6-5108-4a3e-9d6e-fec27e046068.png)

### 컬렉션 페치 조인

이번에는 일대다 관계인 컬렉션을 페치 조인해보자.

```java
select t
from Team t join fetch t.members
where t.name = '팀A'
```

![image](https://user-images.githubusercontent.com/55661631/150926087-f243e034-f2b1-4c00-8317-dae91df52dff.png)

- 위 그림을 보면 TEAM 테이블에 팀A는 하나지만 MEMBER 테이블과 조인하면서 결과가 증가해서 팀A가 2건 조회 되었다.
- 따라서 컬렉션 페치 조인 결과 객체에서 teams 결과 예제를 보면 주소가 0x100으로 같은 팀A를 2건 가지게 된다.

### 페치 조인과 DISTINCT

- SQL의 DISTINCT는 중복된 결과를 제거하는 명령어다.
- JPQL의 DISTINCT 명령어는 SQL에 DISTINCT를 추가하는 것은 물론이고 애플리케이션에서 한 번 더 중복을 제거한다.
- 그러나 각 로우의 데이터가 다르므로 SQL의 DISTINCT는 효과가 없다.

![image](https://user-images.githubusercontent.com/55661631/150926105-d9bba534-595b-46cd-82d9-41cf98b713b3.png)

### 페치 조인과 일반 조인의 차이

- 일반 조인을 사용하면 JPQL은 결과를 반환할 때 연관관계까지 고려하지 않으며, SELECT 절에 지정한 엔티티만 조회할 뿐이다.
- 즉시 로딩으로 설정하고 일반 조인을 사용하면 회원 컬렉션을 즉시 로딩하기 위해 쿼리를 한 번 더 실행한다.
- 반면 페치 조인을 사용하면 연관된 엔티티도 함께 조회한다.

### 페치 조인의 특징과 한계

**특징**

- 페치 조인을 사용하면 SQL 한 번으로 연관된 엔티티들을 함께 조회할 수 있어서 SQL 호출 횟수를 줄여 성능을 최적화할 수 있다.
- 최적화를 위해 글로벌 로딩 전략을 즉시 로딩으로 설정하면 성능에 악영향을 미칠 수 있다. 따라서 글로벌 로딩 전략을 지연 로딩으로 사용하고 최적화가 필요하면 페치 조인을 적용하는 것이 효과적이다.

**한계**

- 페치 조인 대상에는 별칭을 줄 수 없다.
- 둘 이상의 컬렉션을 페치할 수 없다.
- 컬렉션을 페치 조인하면 페이징 API를 사용할 수 없다(단일 값 연관 필드는 가능).

## 서브 쿼리

- JPQL도 SQL처럼 서브 쿼리를 지원한다.
- 서브 쿼리를 WHERE, HAVING 절에만 사용할 수 있고 SELECT, FROM 절에서는 사용할 수 없다.

## 다형성 쿼리

아래와 같이 부모 엔티티를 조회하면 그 자식 엔티티도 함께 조회한다.

```java
select i from Item i
```

### TYPE

TYPE은 엔티티의 상속 구조에서 조회 대상을 특정 자식 타입으로 한정할 때 주로 사용한다.

```java
// Item 중에 Book, Movie를 조회하라
select i from Item i
where type(i) IN (Book, Movie)
```

### TREAT(JPA 2.1)

상속 구조에서 부모 타입을 특정 자식 타입으로 다룰 때 사용한다.

```java
select i from Item i where treat(i as Book).author = 'kim'
```

## 엔티티 직접 사용

### 기본 키 값

- 객체 인스턴스는 참조 값으로 식별하고, 테이블 row는 기본 키 값으로 식별한다.
- 따라서 JPQL에서 엔티티 객체를 직접 사용하면 SQL에서는 해당 엔티티의 기본 값을 사용한다.

```java
select count(m) from Member m //엔티티의 아이드를 사용
select count(m.id) as cnt from Member m //엔티티를 직접 사용
```

### 외래 키 값

```java
Team team = em.find(Team.class, 1L);

String query = "select m from Member m where m.team = :team";
List resultList = em.createQuery(query)
    .setParameter("team", team)
    .getResultList();
```

`m.team`은 `team_id` 외래 키와 매핑되어 있기 때문에 아래와 같은 SQL이 실행된다.

```java
select m.*
from Member m
where m.team_id=? (팀 파라미터의 ID 값)
```

참고로 Member 테이블이 `team_id` 외래 키를 가지고 있으므로 묵시적 조인은 일어나지 않는다. 물론 `m.team.name` 을 호출하면 묵시적 조인이 일어난다.

## 참고

- 자바 ORM 표준 JPA 프로그래밍 - 김영한
