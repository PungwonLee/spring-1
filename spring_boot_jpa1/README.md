# 출처 
- url: https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-%ED%99%9C%EC%9A%A9-1
- 강사: 김영한님

# [1. 프로젝트 환경설정](./1.project-setting)
<details> <summary> 1. 프로젝트 생성</summary>

</details>

<details> <summary> 2. 라이브러리 살펴보기</summary>

- gradle 의존관계 보기
    - `./gradlew dependencies -configuration compileClasspath`

- spring-boot-starter-web
    - spring-boot-starter-tomcat: 톰캣 (웹서버)
    - spring-webmvc: 스프링 웹 MVC
- spring-boot-starter-thymeleaf: 타임리프 템플릿 엔진(View)
- spring-boot-starter-data-jpa
    - spring-boot-starter-aop
    - spring-boot-starter-jdbc
        - HikariCP 커넥션 풀 (부트 2.0 기본)
    - hibernate + JPA: 하이버네이트 + JPA
    - spring-data-jpa: 스프링 데이터 JPA
- spring-boot-starter(공통): 스프링 부트 + 스프링 코어 + 로깅
    - spring-boot
        - spring-core
    - spring-boot-starter-logging
- logback, slf4j

### 테스트 라이브러리
- spring-boot-starter-test
    - junit: 테스트 프레임워크
    - mockito: 목 라이브러리
    - assertj: 테스트 코드를 좀 더 편하게 작성하게 도와주는 라이브러리
    - spring-test: 스프링 통합 테스트 지원
- 핵심 라이브러리
    - 스프링 MVC
    - 스프링 ORM
    - JPA, 하이버네이트
    - 스프링 데이터 JPA
- 기타 라이브러리
    - H2 데이터베이스 클라이언트
    - 커넥션 풀: 부트 기본은 HikariCP
    - WEB(thymeleaf)
    - 로깅 SLF4J & LogBack
    - 테스트

참고: 스프링 데이터 JPA는 스프링과 JPA을 먼저 이해하고 사용해야 하는 응용기술이다.

</details>

<details> <summary> 3. View 환경설정</summary>

- thymeleaf 템플릿 엔진
    - thymeleaf 공식 사이트: https://www.thymeleaf.org/
    - 스프링 공식 튜토리얼: https://spring.io/guides/gs/serving-web-content/
    - 스프링부트 메뉴얼: https://docs.spring.io/spring-boot/docs/2.1.6.RELEASE/reference/html/
      boot-features-developing-web-applications.html#boot-features-spring-mvc-template-engines
- 스프링 부트 thymeleaf viewName 매핑
    - `resources:templates/` + (ViewName) + `.html`

- 참고: spring-boot-devtools 라이브러리를 추가하면, html 파일을 컴파일만 해주면 서버 재시작 없이
View 파일 변경이 가능하다.
    - `implementation 'org.springframework.boot:spring-boot-devtools'`
- 인텔리J 컴파일 방법: 메뉴 build Recompile

</details>

<details> <summary> 4. H2 데이터베이스 설치 </summary>

- 개발이나 테스트 용도로 가볍고 편리한 DB, 웹 화면 제공
- 주의 Version 1.4.200를 사용
- https://www.h2database.com/html/main.html
- 다운로드 및 설치
- 데이터 베이스 파일 생성 방법
    - `jdbc:h2:~/jpashop` (최소 한번)
    - `~/jpashop.mv.db`파일 생성 확인
    - 이후 부터는 `jdbc:h2:tcp://localhost/~/jpashop`

- 주의: H2 데이터베이스의 MVCC 옵션은 G2 1.4.198 버전부터 제거되었다. 1.4.200 버전에서는 MVCC옵션을 사용하면 오류가 발생한다.

</details>

<details> <summary> 5. JPA와 DB 설정, 동작 확인 </summary>

`main/resources/application.yml`
```
spring:
 datasource:
 url: jdbc:h2:tcp://localhost/~/jpashop
 username: sa
 password:
 driver-class-name: org.h2.Driver
 jpa:
 hibernate:
 ddl-auto: create
 properties:
 hibernate:
# show_sql: true
 format_sql: true
logging.level:
 org.hibernate.SQL: debug
# org.hibernate.type: trace
```

- spring.jpa.hibernate.ddl-auto: create
    - 이 옵션은 애플리케이션 실행 시점에 테이블을 drop 하고, 다시 생성한다.

> 참고: 모든 로그 출력은 가급적 로거를 통해 남겨야 한다
> `show_sql` : 옵션은 `System.out` 에 하이버네이트 실행 SQL을 남긴다.
> `org.hibernate.SQL` : 옵션은 logger를 통해 하이버네이트 실행 SQL을 남긴다

> 주의!`application.yml`같은 `yml`파일은 띄어쓰기(스페이스) 2칸으로 계층을 만든다. 따라서
> 띄어쓰기 2칸을 필수로 적어주어야 한다.
> 예를 들어서 아래의 `datasource`는 `spring:`하위에 있고 앞에 띄어쓰기 2칸이 있으므로
> `spring.datasource`가 된다. 다음 코드에 주석으로 띄어쓰기를 적어두었다.<br>

yml 띄어쓰기 주의
```
spring: #띄어쓰기 없음
 datasource: #띄어쓰기 2칸
 url: jdbc:h2:tcp://localhost/~/jpashop #4칸
 username: sa
 password:
 driver-class-name: org.h2.Driver
 jpa: #띄어쓰기 2칸
 hibernate: #띄어쓰기 4칸
 ddl-auto: create #띄어쓰기 6칸
 properties: #띄어쓰기 4칸
 hibernate: #띄어쓰기 6칸
# show_sql: true #띄어쓰기 8칸
 format_sql: true #띄어쓰기 8칸
logging.level: #띄어쓰기 없음
 org.hibernate.SQL: debug #띄어쓰기 2칸
# org.hibernate.type: trace #띄어쓰기 2칸
```

> 주의! @Test는 JUnit4를 사용하면 org.junit.Test를 사용하셔야 합니다. 만약 JUnit5 버전을 사용하면
> 그것에 맞게 사용하면 된다.
- Entity, Repository 동작 확인
- jar 빌드해서 동작 확인
    - `./gradlew clean build`
    - `cd build/libs/`
    - `java -jar ./jpashop...`

> 오류: 테스트를 실행했는데 다음과 같이 테스트를 찾을 수 없는 오류가 발생하는 경우
> `No tests found for given indlues: [jpabook.jpashop.MemberRepositoryTest]`
> `(filter.includeTestsMatching)`
> 해결: 스프링 부트 2.1.x 버전을 사용하지 않고, 2.2.x 이상 버전을 사용하면 Junit5가 설치된다. 이때는
> `build.gradle` 마지막에 다음 내용을 추가하면 테스트를 인식할 수 있다. Junit5 부터는 `build.gradle`
> 에 다음 내용을 추가 해야 테스트가 인식된다.

`build.gradle`마지막에 추가
```
test {
useJUnitPlatform()
}
```

> 참고: 스프링 부트를 통해 복잡한 설정이 다 자동화 되었다. `persistence.xml`도 없고,
> `LocalContainerEntityManagerFactoryBean`도 없다. 스프링 부트를 통한 추가 설정은
> 스프링 부트 메뉴얼을 참고하고, 스프링 부트를 사용하지 않고 순수 스프링과 JPA 설정 방법은 자바
> ORM표준 JPA 프로그래밍 책을 참고

### 쿼리 파라미터 로그 남기기
- 로그에 다음을 추가하기 `org.hiberrnate.type`: SQL 실행 파라미터를 로그로 남긴다.
- 외부 라이브 러리 사용
    - https://github.com/gavlyukovskiy/spring-boot-data-source-decorator
    - 스프링 부트를 사용하면 이 라이브러리만 추가하면 된다.<br>
    `implementation 'com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.5.6'`

> 참고: 쿼리 파라미터를 로그로 남기는 외부 라이브러리는 시스템 자원을 사용하므로, 개발 단계에서는 편하게
> 사용해도 된다. 하지만 운영시스템에 적용하려면 꼭 성능테스트를 하고 사용하는 것이 좋다.

</details>

# [2. 도메인 분석 설계](./2.domain_analysis_design)

<details> <summary> 1. 요구사항 분석 </summary>

![image](https://user-images.githubusercontent.com/28394879/133268707-02de0e4f-fffb-4e93-a1a6-20d49487e339.png)

### 기능 목록
- 회원 기능
    - 회원 등록
    - 회원 조회
- 상품 기능
    - 상품 등록
    - 상품 수정
    - 상품 조회
- 주문 기능
    - 상품 주문
    - 주문 내역 조회
    - 주문 취소
- 기타 요구사항
    - 상품은 제고 관리가 필요하다.
    - 상품의 종류는 도서,음반,영화가 있다.
    - 상품을 카테고리로 구분할 수 있다.
    - 상품 주문시 배송 정볼르 입력할 수 있다.

</details>

<details> <summary> 2. 도메인 모델과 테이블 설계 </summary>

## 도메인 모델과 테이블 설계
![image](https://user-images.githubusercontent.com/28394879/133434624-67879f8b-c61d-4085-b6e5-acda70cdb6b9.png)
- 회원, 주문, 상품의 관계: 회원은 여러 상품을 주문할 수 있다. 그리고 한 번 주문할 때 여러 상품을 선택할 수 있으므로 주문과 상품은 다대다 관계다.
하지만 이런 다대다 관계는 관계형 데이터베이스는 물론이고 엔티티에서도 거의 사용하지 않는다. 따라서 그림처럼 주문상품이라는 엔티티를 추가해서
다대다 관계를 일대일, 다대일 관계로 풀어냈다.

- 상품 분류: 상품은 도서, 음반, 영화로 구분되는데 상품이라는 공통 속성을 사용하므로 상속 구조로 표현했다.

### 회원 엔티티 분석
![image](https://user-images.githubusercontent.com/28394879/133435599-d393a2c2-0de4-492a-8fa8-4b9885cb8d51.png)
- 회원(Member): 이름과 임베디드 타입인 주소(`Address`), 그리고 주문(`orders`) 리스트를 가진다.
- 주문(Order): 한 번 주문시 여러 상품을 주문할 수 있으므로 주문과 상품주문(`OrderItem`)은 일대다 관계다.
주문은 상품을 주문한 회원과 배송 정보, 주문 날짜, 주문 상태(`status`)를 가지고 있다. 주문 상태는 열거형을 사용했는데 주문(`Order`), 취소(`CANCEL`)을 표현할 수 있다.

- 주문상품(OrderItem): 주문한 상품 정보와 주문 금액(`orderPrice`), 주문 수량(`count`)정보를 가지고 있다. (보통 `OrderLine`, `ListItem`으로 많이 표현한다.)

- 상품(Item): 이름, 가격, 재고수량(`stockQuantity`)을 가지고 있다. 상품을 주문하면 재고수량이 줄어든다. 상품의 종류로는 도서, 음반, 영화가 있는데 각각은 사용하는 속성이 조금씩 다르다.

- 배송(Delivery): 주문시 하나의 배송 정보를 생성한다. 주문과 배송은 일대일 관계다.

- 카테고리(Category): 상품과 다대다 관계를 맺는다. `parent`, `child`로 부모, 자식 카테고리를 연결한다.

- 주소(Address): 값 타입(임베디드 타입)이다. 회원과 배송(Delivery)에서 사용한다.

> 참고: 회원 엔티티 분석 그림에서 Order와 Delivery가 단방향 관계로 잘못 그려져 있다. 양방향 관계가 맞다.

> 참고: 회원이 주문을 하기 때문에, 회원이 주문리스트를 가지는 것은 얼핏 보면 잘 설계한 것 같지만, 객체 세상은
> 실제 세계와는 다르다. 실무에서는 회원이 주문을 참조하지 않고, 주문이 회원을 참조하는 것으로 충분하다.
> 여기서는 일대다, 다대일 양방향 연관관계를 설명하기 위해서 추가했다.

### 회원 테이블 분석
![image](https://user-images.githubusercontent.com/28394879/133438311-c815f9b1-1f81-40ce-8df4-aa9c8400e2cf.png)

- MEMBER: 회원 엔티티의 `Address`임베디드 타입 정보가 회원 테이블에 그대로 들어갔다. 이것은 `DELIVERY`테이블도 마찬가지다.

- ITEM: 앨범, 도서, 영화 타입을 통합해서 하나의 테이블로 만들었다. `DTYPE` 컬럼으로 타입을 구분한다.

> 참고: 테이블명이 `ORDER`가 아니라 `ORDERS`인 것은 데이터베이스가 `order by`때문에 예약어로 잡고 있는 경우가 많다. 그래서 관례상 `ORDERS`를 많이 사용한다.

> 참고: 실제 코드에서는 DB에 소문자 + _(언더스코어) 스타일을 사용한다.
> 데이터베이스 테이블명, 컬럼명에 대한 관례는 회사마다 다르다. 보통은 대문자 + _(언더스코어)나 소문자 + _(언더스코어) 방식 중에 하나를 지정해서 일관성 있게 사용한다.
> 여기에서는 객체와 차이를 나타내기 위해 데이터베이스 테이블, 컬럼명은 대문자를 사용했지만, **실제 코드에서는 소문자 + _(언더스코어) 스타일을 사용하겠다.

### 연관관계 매핑 분석
- 회원과 주문: 일대다, 다대일의 양방향 관계다. 따라서 연관관계의 주인을 정해야 하는데, 외래 키가 있는 주문을 연관관계의 주인으로 정하는 것이 좋다.
그러므로 `Order.member`를 `ORDERS.MEMBER_ID` 외래키와 매핑한다.

- 주문상품과 주문: 다대일 양방향 관계다. 외래 키가 주문상품에 있으므로 주문상품이 연관관계의 주인이다. 그러므로 `OrderItem.order`를 `ORDER_ITEM.ORDER_ID`외래키와 매핑한다.

- 주문상품과 상품: 다대일 단방향 관계다. `OrderItem.item`을 `ORDER_ITEM.ITEM_ID` 외래키와 매핑한다.

- 주문과 배송: 일대일 단방향 관계다. `Order.delivery`를 `ORDERS.DELIVERY_ID` 외래키와 매핑한다.

- 카테고리와 상품: `@ManyToMany`를 사용해서 매핑한다. (실무에서 @ManyToMany는 사용하지 말자. 여기서는 다대다 관계를 예제로 보여주기 위해 추가했을 뿐이다.)

> 참고: 외래 키가 있는 곳을 연관관계의 주인으로 정해라.
> 연관관계의 주인은 단순히 외래 키를 누가 관리하냐의 문제이지 비즈니스상 우위에 있다고 주인으로 정하면
> 안된다.. 예를 들어서 자동차와 바퀴가 있으면, 일대다 관계에서 항상 다쪽에 외래 키가 있으므로 외래 키가
> 있는 바퀴를 연관관계의 주인으로 정하면 된다. 물론 자동차를 연관관계의 주인으로 정하는 것이 불가능 한
> 것은 아니지만, 자동차를 연관관계의 주인으로 정하면 자동차가 관리하지 않는 바퀴 테이블의 외래 키 값이
> 업데이트 되므로 관리와 유지보수가 어렵고, 추가적으로 별도의 업데이트 쿼리가 발생하는 성능 문제도 있
> 다. 자세한 내용은 JPA 기본편을 참고하자.


</details>

<details> <summary> 3. 엔티티 클래스 개발1 </summary>

- 예제에서는 설명을 쉽게하기 위해 엔티티 클래스에 Getter, Setter를 모두 열고, 최대한 단순하게 설계
- 실무에서는 가급적 Getter는 열어두고, Setter는 꼭 필요한 경우에만 사용하는 것을 추천

> 참고: 이론적으로 Getter, Setter 모두 제공하지 않고, 꼭 필요한 별도의 메서드를 제공하는게 가장 이상적이다.
> 하지만 실무에서 엔티티의 데이터는 조회할 일이 너무 많으므로, Getter의 경우 모두 열어두는 것이 편리하다.
> Getter는 아무리 호출해도 호출 하는 것 만으로 어떤 일이 발생하지 않는다. 하지만 Setter는 문제가 다르다.
> Setter를 호출하면 데이터가 변한다. Setter를 막 열어두면 가까운 미래에 엔티티가 도대체 왜 변경되는지
> 추적하기 점점 힘들어진다. 그래서 엔티티를 변경할 때는 Setter 대신에 변경 지점이 명확하도록 변경을 위한
> 비즈니스 메서드를 별도로 제공해야 한다.

### 회원 엔티티
> 참고: 엔티티의 식별자는 id 를 사용하고 PK 컬럼명은 member_id 를 사용했다. 엔티티는 타입(여기서는
> Member)이 있으므로 id 필드만으로 쉽게 구분할 수 있다. 테이블은 타입이 없으므로 구분이 어렵다. 그리
> 고 테이블은 관례상 테이블명 + id 를 많이 사용한다. 참고로 객체에서 id 대신에 memberId 를 사용해도
> 된다. 중요한 것은 일관성이다.

</details>

<details> <summary> 4. 엔티티 클래스 개발2 </summary>

### 카테고리 엔티티
> 참고: 실무에서는 @ManyToMany를 사용하지 말자
> @MnayToMany는 편리한 것 같지만, 중간 테이블(CATEGORY_ITEM)에 컬럼을 추가할 수 없고, 세밀하게
> 쿼리를 실행하기 어렵기 때문에 실무에서 사용하기에는 한계가 있다. 중간 엔티티(CategoryItem)를 만들고
> @ManyToOne, @OneToMany로 매핑해서 사용하자. 정리하면 다대다 매핑을 일대다, 다대일 매핑으로 풀어내서 사용하자.

### 주소 값 타입
> 참고: 값 타입은 변경 불가능하게 설계해야 한다.
> @Setter 를 제거하고, 생성자에서 값을 모두 초기화해서 변경 불가능한 클래스를 만들자. JPA 스펙상 엔티
> 티나 임베디드 타입( @Embeddable )은 자바 기본 생성자(default constructor)를 public 또는
> protected 로 설정해야 한다. public 으로 두는 것 보다는 protected 로 설정하는 것이 그나마 더 안전
> 하다.
> JPA가 이런 제약을 두는 이유는 JPA 구현 라이브러리가 객체를 생성할 때 리플랙션 같은 기술을 사용할 수
> 있도록 지원해야 하기 때문이다.

</details>

<details> <summary> 5. 엔티티 설계시 주의점 </summary>

### 엔티티에는 가급적 Setter를 사용하지 말자
- Setter가 모두 열려있으면, 변경 포인트가 너무 많아서 유지보수가 어렵다.

### 모든 연관관계는 지연로딩으로 설정!
- 즉시로딩(`EAGER`)은 예측이 어렵고, 어떤 SQL이 실행될지 추적하기 어렵다. 특히 JPQL을 실행할 때 N+1 문제가 자주 발생한다.
- 실무에서 모든 연관관계는 지연로딩(`LAZY`)으로 설정해야 한다.
- 연관된 엔티티를 함께 DB에서 조회해야 하면, fetch join 또는 엔티티 그래프 기능을 사용한다.
- @XToOne(OneToOne, ManyToOne)관계는 기본이 즉시로딩이므로 직접 지연로딩으로 설정해야 한다.

### 컬렉션은 필드에서 초기화 하자.
- 컬렉션은 필드에서 바로 초기화 하는 것이 안전하다.
- `null` 문제에서 안전하다.
- 하이버네이트는 엔티티를 영속화 할 때, 컬렉션을 감싸서 하이버네이트가 제공하는 내장 컬렉션으로 변경한다. 만약
`getOrders()`처럼 임의의 메서드에서 컬렉션을 잘못 생성하면 하이버네이트 내부 메커니즘에 문제가 발생할 수 있다.
따라서 필드레벨에서 생성하는 것이 가장 안전하고, 코드도 간결하다.
- ex) `private List<Order> orders = new ArrayList<>();` 얘를 생성자로 orders = new ArrayList<>()하는것보다 낫다.
```java
Member member = new Member();
System.out.println(member.getOrders().getClass());
em.persist(team);
System.out.println(member.getOrders().getClass());

// 출력 결과
class java.util.ArrayList
class org.hibernate.collection.internal.PersistentBag // 똑같은 orders인데 class가 바꼈음.. 근데 이것 이후로 또 ArrayList로 바뀌면 문제 생긴다.
```

### 테이블, 컬럼명 생성 전략
- 스프링 부트에서 하이버네이트 기본 매핑 전략을 변경해서 실제 테이블 필드명은 다름
- https://docs.spring.io/spring-boot/docs/2.1.3.RELEASE/reference/htmlsingle/#howto-configure-hibernate-naming-strategy
- https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#naming
- 하이버네이트 기존 구현: 엔티티의 필드명을 그대로 테이블의 컬럼명으로 사용(`SpringPhysicalNamingStrategy`)
- 스프링 부트 신규 설정(엔티티(필드) -> 테이블(컬럼))
    1. 카멜 케이스 -> 언더스코어(memberPoint -> member_point)
    2. .(점) -> _(언더스코어)
    3. 대문자 -> 소문자

- 적용 2단계
    1. 논리명 생성: 명시적으로 컬럼, 테이블명을 직접 적지 않으면 ImplicitNamingStrategy 사용
    `spring.jpa.hibernate.naming.implicit-strategy`: 테이블이나, 컬럼명을 명시하지 않을 때 논리명 적용
    2. 물리명 적용: `spring.jpa.hibernate.naming.physical-strategy`: 모든 논리명에 적용됨, 실제 테이블에 적용
    (username -> usernm 등으로 회사 룰로 바꿀 수 있음)
    - 스프링 부트 기본 설정
    ```
    spring.jpa.hibernate.naming.implicit-strategy:
    org.springframework.boot.orm.jpa.hibernate.SpringImplicitNamingStrategy

    spring.jpa.hibernate.naming.physical-strategy:
    org.springframework.boot.orm.jpa.hibernate.SpringPhysicalNamingStrategy
    ```


</details>

# [3. 애플리케이션 구현 준비](./3.application_implementation_ready)

<details> <summary> 1. 구현 요구사항 </summary>

### 기능 목록
- 회원 기능
    - 회원 등록
    - 회원 조회
- 상품 기능
    - 상품 등록
    - 상품 수정
    - 상품 조회
- 주문 기능
    - 상품 주문
    - 주문 내역 조회
    - 주문 취소
- 기타 요구사항
    - 상품은 제고 관리가 필요하다.
    - 상품의 종류는 도서,음반,영화가 있다.
    - 상품을 카테고리로 구분할 수 있다.
    - 상품 주문시 배송 정볼르 입력할 수 있다.

### 예제를 단순화 하기 위해 다음 기능은 구현X
- 로그인과 권한 관리X
- 파라미터 검증과 예외 처리 단순화
- 상품은 도서만 사용
- 카테고리는 사용X
- 배송 정보는 사용X



</details>


<details> <summary> 2. 애플리케이션 아키텍처 </summary>

![image](https://user-images.githubusercontent.com/28394879/133625272-6d95d3b0-04f4-413d-acee-37bd5b635d24.png)
### 계층형 구조 사용
- controller, web: 웹 계층
- service: 비즈니스 로직, 트랜잭션 처리
- repository: JPA를 직접 사용하는 계층, 엔티티 매니저 사용
- domain: 엔티티가 모여 있는 게층, 모든 계층에서 사용

### 패키지 구조
- jpabook.jpashop
    - domain
    - exception
    - repository
    - service
    - web

#### 개발순서: 서비스, 리포지트리 계층을 개발하고, 테스트 케이브를 작성해서 검증, 마지막에 웹 계층 적용



</details>

# [4. 회원 도메인 개발](./4.member_domain_develop)

<details> <summary> 1. 회원 리포지토리 개발 </summary>

**구현 기능**
- 회원 등록
- 회원 목록 조회

**순서**
- 회원 엔티티 코드 다시 보기
- 회원 리포지토리 개발
- 회원 서비스 개발
- 회원 기능 테스트

### 회원 리포지 토리 개발

**기술 설명**
- `@Repository`: 스프링 빈으로 등록, JPA 예외를 스프링 기반 예외로 예외 변환
- `@PersistenceContext`: 엔티티 매니저(`EntityManager`)주입
- `@PersistenceUnit`: 엔티티 매니저 팩토리(`EntityManagerFactory`)주입

</details>

<details> <summary> 2. 회원 서비스 개발 </summary>

### 회원 서비스 코드
**기술 설명**
- `@Service`
- `@Transactional`: 트랜잭션, 영속성 컨텍스트
    - `readOnly=true`: 데이터의 변경이 없는 읽기 전용 메서드에 사용, 영속성 컨텍스트를 플러시 하지 않으므로 약간의 성능 향상(읽기 전용에는 다 적용)
    - 데이터베이스 드라이버가 지원하면 DB에서 성능 향상
- `@Autowired`
    - 생성자 Injection 많이 사용, 생성자가 하나면 생략 가능

**기능 설명**
- `join()`
- `findMembers()`
- `findOne()`

> 참고: 실무에서는 검증 로직이 있어도 멀티 쓰레드 상황을 고려해서 회원 테이블의 회원명 컬럼에 유니크 제약 조건을 추가하는 것이
> 안전하다.

> 참고: 스프링 필드 주입 대신에 생성자 주입을 사용하자.

> 참고: 스프링 데이터 JPA를 사용하면 `EntityManager`도 주입 가능

</details>

<details> <summary> 3. 회원 기능 테스트 </summary>

**테스트 요구사항**
- 회원가입을 성공해야 한다.
- 회원가입 할 때 같은 이름이 있으면 예외가 발생해야 한다.

**기술 설명**
- `@SpringBootTest`: 스프링 부트 띄우고 테스트(이게 없으면 `@Autowired`다 실패)
- `@Transactional`: 반복 가능한 테스트 지원, 각각의 테스트를 실행할 때마다 트랜잭션을 시작하고 **테스트가 끝나면
트랜잭션을 강제로 롤백(이 어노테이션이 테스트 케이스에서 사용될 때만 롤백)

**기능 설명**
- 회원가입 테스트
- 중복 회원 예외처리 테스트

> 참고: 테스트 케이스 작성 고수 되는 마법: Given, When, Then
> (http://martinfowler.com/bliki/GivenWhenThen.html)
> 이 방법이 필수는 아니지만 이 방법을 기본으로 해서 다양하게 응용하는 것을 권장

### 테스트 케이스를 위한 설정
- 테스트 케이스 격리된 환경에서 실행하고, 끝나면 데이터를 초기화하는 것이 좋다. 그런 면에서 메모리 DB를 사용하는 것이 가장 이상적이다.
- 추가로 테스트 케이스를 위한 스프링 환경과, 일반적으로 애플리케이션을 실행하는 환경은 보통 다르므로 설정 파일을 다르게 사용하자.
- 다음과 같이 간단하게 테스트용 설정 파일을 추가하면 된다.
    - `test/resources/application.yml`
    ```yml
    spring:
    # datasource:
    # url: jdbc:h2:mem:testdb
    # username: sa
    # password:
    # driver-class-name: org.h2.Driver
    # jpa:
    # hibernate:
    # ddl-auto: create
    # properties:
    # hibernate:
     # show_sql: true
    # format_sql: true
    # open-in-view: false
    logging.level:
     org.hibernate.SQL: debug
    # org.hibernate.type: trace
    ```
    - 이제 테스트에서 스프링을 실행하면 이 위치에 있는 설정 파일을 읽는다. (만약 이 위치에 없으면 `src/resources/application.yml` 을 읽는다.)
- 스프링 부트는 datasource 설정이 없으면, 기본적으로 메모리 DB를 사용하고, driver-class도 현재 등록된 라이브러리를 보고 찾아준다.
- 추가로 `ddl-auto`도 `create-drop`모드로 동작한다. 따라서 데이터소스나, JPA 관련된 별도의 추가 설정을 하지 않아도 된다.


</details>


# [5. 상품 도메인 개발](./5.item_domain_develop)
<details> <summary> 1. 상품 엔티티 개발(비즈니스 로직 추가) </summary>

**구현 기능**
- 상품 등록
- 상품 목록 조회
- 상품 수정

**순서**
- 상품 엔티티 개발(비즈니스 로직 추가)
- 상품 리포지토리 개발
- 상품 서비스 개발

** 비즈니스 로직 분석**
- `addStock()` 메서드는 파라미터로 넘어온 수만큼 재고를 늘린다. 이 메서드는 재고가 증가하거나 상품 주문을 취소해서 재고를 다시 늘려야 할 때 사용한다.
- `remoteStock()` 메서드는 파라미터로 넘어온 수만큼 재고를 줄인다. 만약 재고가 부족하면 예외가 발생한다. 주로 상품을 주문할 떄 사용한다.

</details>

<details> <summary> 2. 상품 리포지토리 개발 </summary>

**기능 설명**
- `save()`
    - `id`가 없으면 신규로 보고 `persist()`실행
    - `id`가 있으면 이미 데이터베이스에 저장된 엔티티를 수정한다고 보고, `merge()`를 실행, 자세한 내용은 뒤에 웹에서 설명(그냥 지금은 저장한다 정도로만 생각하자)

</details>

<details> <summary> 3. 상품 서비스 개발 </summary>

- 상품 서비스는 상품 리포지토리에 단순히 위임한 하는 클래스

- 상품 기능 테스트는 회원 테스트와 비슷하므로 생략 ( 실무에서는 다 작성해야 됨 )

</details>




# [6. 주문 도메인 개발](./6.order-domain-develop)

<details> <summary> 1. 주문, 주문상품 엔티티 개발 </summary>

**구현 기능**
- 상품 주문
- 주문 내역 조회
- 주문 취소

**순서**
- 주문 엔티티, 주문상품 엔티티 개발
- 주문 리포지토리 개발
- 주문 서비스 개발
- 주문 검색 기능 개발
- 주문 기능 테스트

### 주문 엔티티 개발
**기능 설명**
- 생성 메서드(`createOrder()`): 주문 엔티티를 생성할 때 사용한다. 주문 회원, 배송정보, 주문상품의 정보를 받아서 실제 주문 엔티티를 생성한다.
- 주문 취소(`cancel()`): 주문 취소시 사용한다. 주문 상태를 취소로 변경하고 주문상품에 주문 취소를 알린다. 만약 이미 배송을 완료한 상품이면 주문을 취소하지 못하도록 예외를 발생시킨다.
- 전체 주문 가격 조회: 주문 시 사용한 전체 주문 가격을 조회한다. 전체 주문 가격을 알려면 각각의 주문상품 가격을 알아야 한다. 로직을 보면 연관된 주문상품들의 가격을 조회해서 더한 값을 반환 한다.
( 실무에서는 주로 주문에 전체 주문 가격 필드를 두고 역정규화 한다. )

### 주문상품 엔티티 개발
**기능 설명**
- 생성 메서드(`createOrderItem()`): 주문 상품, 가격, 수량 정보를 사용해서 주문상품 엔티티를 생성한다.
그리고 `item.removeStock(count)`를 호출해서 주문한 수량만큼 상품의 재고를 줄인다.
- 주문 취소(`cancel()`): `getItem().addStock(count)`를 호출해서 취소한 주문 수량만큼 상품의 재고를 증가시킨다.
- 주문 가격 조회(`getTotalPrice()`): 주문 가격에 수량을 곱한 값을 반환한다.


</details>

<details> <summary> 2. 주문 리포지토리 개발 </summary>

### 주문 리포지 토리 개발
- 주문 리포지토리에는 주문 엔티티를 저장하고 검색하는 기능이 있다. 마지막의 `findAll(OrderSearch orderSearch)`메서드는 조금 뒤에 있는 주문 검색 기능에서 자세히 알아보자.

</details>

<details> <summary> 3. 주문 서비스 개발 </summary>

### 주문 서비스 개발
- 주문 서비스는 주문 엔티티와 주문 상품 엔티티의 비즈니스 로직을 활용해서 주문, 주문 취소, 주문 내역 검색 기능을 제공한다.

- 참고: 예제를 단순화하려고 한 번에 하나의 상품만 주문할 수 있게 했다.

- 주문(`order()`): 주문하는 회원 식별자, 상품 식별자, 주문 수량 정보를 받아서 실제 주문 엔티티를 생성한 후 저장한다.
- 주문 취소(`cancelOrder()`): 주문 식별자를 받아서 주문 엔티티를 조회한 후 주문 엔티티에 주문 취소를 요청한다.
- 주문 검색(`findOrders()`): `OrderSearch`라는 검색 조건을 가진 객체로 주문 엔티티를 검색한다. 자세한 내용은 다음에 나오는 주문 검색 기능에서 알아보자.

> 참고: 주문 서비스의 주문과 주문 취소 메서드를 보면 비즈니스 로직 대부분이 엔티티에 있다.
> 서비스 계층은 단순히 엔티티에 필요한 요청을 위임하는 역할을 한다.
> 이처럼 엔티티가 비즈니스 로직을 가지고 객체 지향의 특성을 적극 활용하는 것을 **도메인 모델 패턴(DDD)**이라고 한다.
> 반대로 엔티티에는 비즈니스 로직이 거의 없고 서비스 계층에서 대부분의 비즈니스 로직을 처리하는 것을 **트랜잭션 스크립트 패턴** 이라 한다.

</details>

<details> <summary> 4. 주문 기능 테스트 </summary>

**테스트 요구사항**
- 상품 주문이 성공해야 한다.
- 상품을 주문할 때 재고 수량을 초과하면 안 된다.
- 주문 취소가 성공해야 한다.

### 상품 주문 테스트
- 상품주문이 정상 동작하는지 확인하는 테스트다. Given 절에서 테스트를 위한 회원과 상품을 만들고 When 절에서 실제 상품을 주문하고
Then 절에서 주문 가격이 올바른지, 주문 후 재고 수량이 정확히 줄었는지 검증한다.

### 재고 수량 초과 테스트
- 재고 수량을 초과해서 상품을 주문해보자. 이때는 `NotEnoughStockException`예외가 발생해야 한다.

### 주문 취소 테스트
- 주문을 취소하면 그만큼 재고가 증가해야 한다.

</details>

<details> <summary> 5. 주문 검색 기능 개발 </summary>

- JPA에서 **동적 쿼리** 를 어떻게 해결해야 하는가?

**검색을 추가한 주문 리포지토리 코드**

### 검색을 추가한 주문 리포지토리 코드
- `findAll(OrderSearch orderSearch)` 메서드는 검색 조건에 동적으로 쿼리를 생성해서 주문 엔티티를 조회 한다.

### JPQL로 처리
- JPAL 쿼리를 문자로 생성하기는 번거롭고, 실수로 인한 버그가 충분히 발생할 수 있다.

### JPA Criteria로 처리
- JPA Criteria는 JPA 표준 스펙이지만 실무에서 사용하기에 너무 복잡하다. 결국 다른 대안이 필요하다.
- 많은 개발자가 비슷한 고민을 했지만, 가장 멋진 해결책은 Querydsl이 제시했다.
- Querydsl 소개장에서 간단히 언급하겠다. 이것은 이대로 진행하자.

> 참고: JPA Criteria에 대한 자세한 내용은 자바 ORM 표준 JPA 프로그래밍 책을 참고하자


</details>


# [7. 웹 계층 개발](./7.web_hierarchy_develop)

<details> <summary> 1. 홈 화면과 레이아웃 </summary>

### 스프링 부트 타임 리프 기본 설정
```
spring:
 thymeleaf:
 prefix: classpath:/templates/
 suffix: .html
```
- 스프링 부트 타임리프 viewName 매핑
    - `resources:templates/` + (ViewName) + `.html`
    - `resources:templates/home.html`
    - 반환한 문자(`home`)과 스프링부트 설정 `prefix`, `suffix` 정보를 상요해서 렌더링할 뷰(`html`)를 찾는다.


> 참고: Hierarchical-style layouts
> 예제에서는 뷰 템플릿을 최대한 간단하게 설명하려고, header , footer 같은 템플릿 파일을 반복해서 포
> 함한다. 다음 링크의 Hierarchical-style layouts을 참고하면 이런 부분도 중복을 제거할 수 있다.
> https://www.thymeleaf.org/doc/articles/layouts.html

> 참고: 뷰 템플릿 변경사항을 서버 재시작 없이 즉시 반영하기
> 1. spring-boot-devtools 추가
> 2. html 파일 build-> Recompile

### view 리소스 등록
- 이쁜 디자인을 위해 부트스트랩을 사용하겠다. (v4.3.1) (https://getbootstrap.com/)
- `resources/static` 하위에 css , js 추가
- `resources/static/css/jumbotron-narrow.css` 추가



</details>

<details> <summary> 2. 회원 등록 </summary>

- 폼 객체를 사용해서 화면 계층과 서비스 계층을 명확하게 분리한다.


</details>

<details> <summary> 3. 회원 목록 조회 </summary>

### 회원 목록 컨트롤러 추가
- 조회한 상품을 뷰에 전달하기 위해 스프링 MVC가 제공하는 모델(`Model`)객체에 보관
- 실행할 뷰 이름을 반환

### 회원 목록 뷰`(templates/members/memberList.html)`
> 참고: 타임리프에서 `?`를 사용하면 `null`을 무시한다.

> 참고: 폼 객체 vs 엔티티 직접 사용
> 요구사항이 정말 단순할 떄는 폼 객체(`MemberForm`)없이 엔티티(`Member`)를 직접 등록과 수정 화면
> 에서 사용해도 된다. 하지만 화면 요구사항이 복잡해지기 시작하면, 엔티티를 화면을 처리하기 위한 기능이
> 점점 증가한다. 결과적으로 엔티티는 점점 화면에 종속적으로 변하고, 이렇게 화면 기능 떄문에 지저분해진
> 엔티티는 결국 유지보수하기 어려워진다.
> 실무에서는 **엔티티는 핵심 비즈니스 로직만 가지고 있고, 화면을 위한 로직은 없어야 한다**.
> 화면이나 API맞는 폼 객체나 DTO를 사용하자. 그래서 화면이나 API 요구사항을 이것들로 처리하고,
> 엔티니는 최대한 순수하게 유지하자.

</details>

<details> <summary> 4. 상품 등록 </summary>

### 상품 등록
- 상품 등록 폼에서 데이터를 입력하고 Submit 버튼을 클릭하면 `/items/new`를 POST 방식으로 요청
- 상품 저장이 끝나면 상품 목록 화면(`redirect:/items`)으로 리다이렉트

</details>

<details> <summary> 5. 상품 목록 </summary>

### 상품 목록 뷰
- `model`에 담아둔 상품 목록인 `items`를 꺼내서 상품 정보를 출력

</details>

<details> <summary> 6. 상품 수정 </summary>

### 상품 수정 폼 이동
1. 수정 버튼을 선택하면 `/items/{itemId}/edit` URL을 GET 방식으로 요청
2. 그 결과로 `updateItemForm()` 메서드를 실행하는데 이 메서드는 `itemService.findOne(itemId)`를 호출해서 수정할 상품을 조회
3. 조회 결과를 모델 객체에 담아서 뷰(`items/updateItemForm`)에 전달

### 상품 수정 실행
- 상품 수정 폼 HTML에는 상품의 id(hidden), 상품명, 가격, 수량 정보 있음
1. 상품 수정 폼에서 정보를 수정하고 Submit버튼을 선택
2. `/items/{itemId}/edit` URL을 POST 방식으로 요청하고 `updateItem()` 메서드를 실행
3. 이때 컨트롤러에 파라미터로 넘어온 `item` 엔티티 인스턴스는 현재 준영속 상태다. 따라서 영속성 컨텍스트의 지원을 받을 수 없고 데이터를 수정해도
변경 감지 기능은 동작X

</details>

<details> <summary> 7. 변경 감지와 병합(merge) </summary>

> 참고: 정말 중요한 내용이니 꼭! 완벽하게 이해해야 한다.

**준영속 엔티티?**
- 영속성 컨텍스트가 더는 관리하지 않는 엔티티를 말한다.
- (여기서는 `itemService.saveItem(book)`)에서 수정을 시도하는 `Book`객체다. `Book`객체는 이미 DB에 한번 저장되어서 식별자가 존재한다.
이렇게 임의로 만들어낸 엔티티도 기존 식별자를 가지고 있으면 준영속 엔티티로 볼 수 있다.)

**준영속 엔티티를 수정하는 2가지 방법**
- 변경 감지 기능 사용
- 병합(`merge`)사용


### 변경 감지 기능 사용
```java
@Transactional
void update(Item itemParam) { //itemParam: 파리미터로 넘어온 준영속 상태의 엔티티
 Item findItem = em.find(Item.class, itemParam.getId()); //같은 엔티티를 조회한
다.
 findItem.setPrice(itemParam.getPrice()); //데이터를 수정한다.
}
```
- 영속성 컨텍스트에서 엔티티를 다시 조회한 후에 데이터를 수정하는 방법
- 트랜잭션 안에서 엔티티를 다시 조회, 변경할 값 선택 -> 트랜잭션 커밋 시점에 변경 감지(Dirty Checking)이 동작 해서 데이터베이스에 UPDATE SQL 실행

### 병합 사용
- 병합은 준영속 상태의 엔티티를 영속 상태로 변경할 때 사용하는 기능이다.
```java
@Transactional
void update(Item itemParam) { //itemParam: 파리미터로 넘어온 준영속 상태의 엔티티
 Item mergeItem = em.merge(item);
}
```

### 병합: 기존에 있는 엔티티
![image](https://user-images.githubusercontent.com/28394879/133917943-a0c22bdc-64d4-46c4-9695-5b93353a5d0b.png)
**병합 동작 방식**
1. `merge()`를 실행한다.
2. 파라미터로 넘어온 준영속 엔티티의 식별자 값으로 1차 캐시에서 엔티티를 조회한다.
2-1. 만약 1차 캐시에 엔티티가 없으면 데이터베이스에서 엔티티를 조회하고, 1차 캐시에 저장한다.
3. 조회한 영속 엔티티(`mergeMember`)에 `member`엔티티의 값을 채워넣는다.(member 엔티티의 모든 값을
mergeMember에 밀어 넣는다. 이때 mergeMember의 "회원1"이라는 이름이 "회원명변경"으로 바뀐다.)
4. 영속 상태인 mergeMember를 반환한다.

> 참고: 책 자바 ORM 표준 JPA 프로그래밍 3.6.5

**병합시 동작 방식을 간단히 정리**
1. 준영속 엔티티의 식별자 값으로 영속 엔티티를 조회한다.
2. 영속 엔티티의 값을 준영속 엔티티의 값으로 모두 교체한다.(병합한다.)
3. 트랜잭션 커밋 시점에 변경 감지 기능이 동작해서 데이터베이스에 UPDATE SQL이 실행

> 주의: 변경 감지 기능을 사용하면 원하는 속성만 선택해서 변경할 수 있지만, 병합을 사용하면 모든 속성이 변경된다.
> 병합시 값이 없으면 `null`로 업데이트 할 위험도 있다. (병합은 모든 필드를 교체한다.)

### 상품 리포지토리의 저장 메서드 분석 `ItemRepository`
- `save()` 메서드는 식별자 값이 없으면(`null`)새로운 엔티티로 판단해서 영속화(persist)하고 식별자가 있으면 병합(merge)
- 지금처럼 준영속 상태인 상품 엔티티를 수정할 때는 `id`값이 있으므로 병합 수행

**새로운 엔티티 저장과 준영속 엔티티 병합을 편리하게 한번에 처리**
- 상품 리포지토리에선 `save()`메서드를 유심히 봐야 하는데, 이 메서드 하나로 저장과 수정(병합)을 다 처리한다.
- 코드를 보면 식별자 값이 없으면 새로운 엔티티로 판단해서 `persist()`로 영속화하고 만약 식별자 값이 있으면
이미 한번 영속화 되었던 엔티티로 판단해서 `merge()`로 수정(병합)한다.
- 결국 여기서의 저장(save)이라는 의미는 신규 데이터를 저장하는 것뿐만 아니라 변경된 데이터의 저장이라는 의미도 포함한다.
- 이렇게 함으로써 이 메서드를 사용하는 클라이언트는 저장과 수정을 구분하지 않아도 되므로 클라이언트의 로직이 단순해진다.
- 여기서 사용하는 수정(병합)은 준영속 상태의 엔티티를 수정할 때 사용한다. 영속 상태의 엔티티는 변경 감지(dirty checking) 기능이
동작해서 트랜잭션을 커밋할 때 자동으로 수정되므로 별도의 수정 메서드를 호출할 필요가 없고 그런 메서드도 없다.

> 참고: `save()` 메서드는 식별자를 자동 생성해야 정상 동작한다. 여기서 사용한 `Item`엔티티의 식별자는
> 자동으로 생성되도록 `@GeneratedValue`를 선언했다. 따라서 식별자 없이 `save()` 메서드를 호춣하면
> `persist()`가 호출 되면서 식별자 값이 자동으로 할당된다. 반면에 식별자를 직접 할당하도록 `@Id`만 선언
> 했다고 가정하자. 이 경우 식별자를 직접 할당하지 않고, `save()`메서드를 호출하면 식별자가 없는 상태로
> `persist()`를 호출한다. 그러면 식별자가 없다는 예외가 발생한다.

> 참고: 실무에서는 보통 업데이트 기능이 매우 재한적이다. 그런데 병합은 모든 필드를 변경해버리고, 데이터가 없으면
> `null`로 업데이트 해버린다. 병합을 사용하면서 이 문제를 해결하려면, 변경 폼 화면에서 모든 데이터를 항상
> 유지해야 한다. 실무에서는 보통 변경가능한 데이터만 노출하기 때문에, 병합을 사용하는 것이 오히려 번거롭다.

### 가장 좋은 해결 방법
**엔티티를 변경할 때는 항상 변경 감지를 사용하라**
- 컨트롤러에서 어설프게 엔티티를 생성하지 마라.
- 트랜잭션이 있는 서비스 계층에 식별자 (`id`)와 변경할 데이터를 명확하게 전달하라. (파라미터 or dto)
- 트랜잭션이 있는 서비스 계층에서 영속 상태의 엔티티를 조회하고, 엔티티의 데이터를 직접 변경하라.
- 트랜잭션 커밋 시점에 변경 감지가 실행한다.

```java
@Controller
@RequiredArgsConstructor
public class ItemController {
 private final ItemService itemService;
 /**
 * 상품 수정, 권장 코드
 */
 @PostMapping(value = "/items/{itemId}/edit")
 public String updateItem(@ModelAttribute("form") BookForm form) {
 itemService.updateItem(form.getId(), form.getName(), form.getPrice());
 return "redirect:/items";
 }
}

package jpabook.jpashop.service;
@Service
@RequiredArgsConstructor
public class ItemService {
 private final ItemRepository itemRepository;
 /**
 * 영속성 컨텍스트가 자동 변경
 */
 @Transactional
 public void updateItem(Long id, String name, int price) {
 Item item = itemRepository.findOne(id);
 item.setName(name);
 item.setPrice(price);
 }
}
```


</details>

<details> <summary> 8. 상품 주문 </summary>

### 상품 주문 컨트롤러
**주문 폼 이동**
- 메인 화면에서 상품 주문을 선택하면 `/order`를 GET 방식으로 호출
- `OrderController`의 `createForm()` 메서드
- 주문 화면에는 주문할 고객정보와 상품 정보가 필요하므로 `model` 객체에 담아서 뷰에 넘겨줌

**주문 실행**
- 주문할 회원과 상품 그리고 수량을 선택해서 Submit 버튼을 누르면 `/order` URL을 POST방식으로 호출
- 컨트롤러의 `order()` 메서드를 실행
- 이 메서드는 고객 식별자(`memberId`), 주문할 상품 식별자(`itemId`), 수량(`count`)정보를 받아서 주문 서비스에 주문을 요청
- 주문이 끝나면 상품 주문 내역이 있는 `/orders` URL로 리다이렉트

</details>

<details> <summary> 9. 주문 목록 검색, 취소 </summary>



</details>


