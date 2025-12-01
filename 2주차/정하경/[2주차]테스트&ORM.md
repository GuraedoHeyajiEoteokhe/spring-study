# 스프링 부트 3 백엔드 개발자 되기 - 자바편

## 04장 스프링 부트 3와 테스트

### 핵심키워드
> Junit / AssertJ / @Test / given-when-then

### 4.1 테스트 코드 개념 익히기
#### 테스트 코드? 작성한 코드가 의도대로 잘 작동 하고 예상치 못한 문제가 없는지 확인할 목적으로 작성하는 코드

 - given-when-test 패턴
 
   1. given : 테스트 실행을 준비하는 단계
   2. when : 테스트를 진행하는 단계
   3. then : 테스트 결과를 검증하는 단계

### 4.2 스프링 부트 3와 테스트
- 스프링 부트 > 테스트하기 위한 도구와 어노테이션 제공 (spring-boot-starter-test 스타터에 도구가 모여있음)
- spring-boot-starter-test : 스프링 부트 스타터 테스트 목록
    - Junit : 자바 프로그래밍 언어용 단위 테스트 프레임워크
    - Spring Test & Spring Boot Test : 스프링 부트 애플리케이션을 위한 통합 테스트 지원
    - AssertJ : 검증문인 어설션을 작성하는 데 사용되는 라이브러리
    - Hamcrest : 표형식을 이해하기 쉽게 만드는 데 사용되는 Matcher 라이브러리
    - Mockito : 테스트에 사용할 가짜 객체인 목 객체를 쉽게 만들고, 관리하고, 검증할 수 있게 지원하는 테스트 프레임워크
    - JSONassert : JSON용 어설션 라이브러리
    - JsonPath : JSON 데이터에서 특정 데이터를 선택하고 검색하기 위한 라이브러리

#### JUnit? 자바 언어를 위한 단위 테스트 프레임워크
 - 단위 테스트 : 작성한 코드가 의도대로 작동하는지 작은 단위로 검증하는 것을 의미 (단위 = 메서드(보통))
 - 특징
   1. 테스트 방식을 구분할 수 있는 어노테이션 제공
   2. @Test어노테이션으로 메서드를 호출 할 때마다 새 인스턴스 생성, 독립 테스트 가능
   3. 예상 결과를 검증하는 어설션 메서드 제공
   4. 사용 방법이 단순, 테스트 코드 작성 시간이 적음
   5. 자동 실행, 자체 결과를 확인하고 즉각적인 피드백 제공 <br>

**Junit으로 단위 테스트 코드 만들기**
```java
@DisplayName("1 + 2는 3이다")
@Test
public void junitTest() {
    int a = 1;
    int b = 2;
    int sum = 3;
    Assertions.assertEquals(sum, a + b);
}
```
- `@DisplayName` : 테스트의 이름 명시
- `@Test` : 해당 어노테이션을 붙인 메서드는 테스트를 수행하는 메서드가 됨

> JUnit은 테스트끼리 영향을 주지 않도록 각 테스트를 실행할 때마다 테스트를 위한 실행 객체를 만들고 테스가 종료되면 실행 객체를 삭제함<br>
> - 위의 테스트는 JUnit에서 제공하는 `assertEquals()`로 a+b와 sum의 값이 같은지 확인
> - 'assertEquals()'메서드의 첫번째 인수에서는 기대하는 값, 두번째 인수에는 실제로 검증할 값을 넣어줌

- 테스트 실패시
    - 테스트가 실패 했다는 표시와 함께 기댓값과 실제로 받은 값을 비교해서 알려줌
    - JUnit은 테스트 케이스가 하나라도 실패하면 전체 테스트를 실패한 것으로 보여줌

- 다양한 Junit 어노테이션
    - @BeforeAll 
      - 전체 테스트를 시작하기전에 처음으로 한번만 실행
      - DB연결을 해야하거나 테스트환경을 초기화할 때 사용됨
      - 전체 테스트 실행 주기에서 한번 만 호출되어야 하므로 static으로 선언
    - @BeforeEach
        - 테스트 케이스 시작 전 매번 실행
        - 객체를 초기화 하거나 테스트에 필요한 값을 미리 넣을 때 사용
        - 각 인스턴스에 대해 메서드를 호출해야 함으로 static이 아니어야함
    - @AfterAll
        - 전체 테스트를 마치고 종료하기 전 한번 만 실행
        - 데이터베이스 연결을 종료할때나 공통적으로 사용하는 자원을 해제할 때 사용
        - 전체 테스트 실행 주기에서 한번 만 호출되어야 하므로 static으로 선언
    - @AfterEach
        - 테스트를 종료하기 전 매번 실행
        - 특정 데이터를 삭제 해야하는 경우 사용
        - 각 인스턴스에 대해 메서드를 호출해야 함으로 static이 아니어야함
    - 생명 주기
        - @BeforeAll > @BeforeEach > @Test > @AfterEach > @AfterAll

#### AssertJ? JUnit과 함께 사용해 검증문의 가독성을 확 높여주는 라이브러리
- 자주 사용하는 메서드

| 메서드 이름 | 설명               |
| --- |------------------|
| isEqualTo(A) | A 값과 같은지 검증      |
| isNotEqualTo(A) | A 값과 다른지 검증      |
| contains(A) | A 값을 포함하는지 검증    |
| doesNotContain(A) | A 값을 포함하지 않는지 검증 |
| startsWith(A) | 접두사가 A인지 검증      |
| endsWith(A) | 점미사가 A인지 검증      |
| isEmpty() | 비어있는 값인지 검증      |
| isNotEmpty() | 비어 있지 않은 값인지 검증  |
| isPositive() | 양수인지 검증          |
| isNegative() | 음수인지 검증          |
| isGreaterThan(A) | A 보다 큰 값인지 검증    |
| isLessThan(A) | A 보다 작은 값인지 검증   |
| isNotNull() | null이 아닌지 확인 |
|isZero() | 0인지 확인 |

``` java
@SpringBootTest        // 테스트용 애플리케이션 컨텍스트 생성
@AutoConfigureMockMvc  // MockMvc 생성 및 자동 구성
class TestControllerTest {

    @Autowired
    protected MockMvc mockMvc;

    @Autowired
    private WebApplicationContext context;

    @Autowired
    private MemberRepository memberRepository;

    @BeforeEach      // 테스트 실행 전 실행하는 메서드
    public void mockMvcSetUp() {
        this.mockMvc = MockMvcBuilders.webAppContextSetup(context)
                .build();
    }

    @AfterEach       // 테스트 실행 후 실행하는 메서드
    public void cleanUp() {
        memberRepository.deleteAll();
    }
}
```
- @SpringBootTest
    - 메인 애플리케이션 클래스에 추가하는 어노테이션인 @SpringBootApplication이 있는 클래스와 찾고 그 클래스에 포함되어 있는 빈을 찾은 다음테스트용 애플리케이션 컨텍스트라는 것을 만드는 것
- @AutoConfigureMockMvc
    - MockMvc를 생성하고 자동으로 구성하는 어노테이션
    - MockMvc는 애플리케이션을 서버에 배포하지 않고도 테스트용 MVC 환경을 만들어 요청 및 전송, 응답 기능을 제공하는 유틸리티 클래스
    - 컨트롤러를 테스트할 때 사용하는 클래스
- @BeforeEach
    - 테스트를 실행하기 전에 실행하는 메서드에 적용하는 어노테이션
    - 여기서는 mockMvcSetUp() 메서드를 실행해 MockMvc를 설정
- @AfterEach
    - 테스트를 실행한 이후에 실행하는 메서드에 적용하는 어노테이션
    - 여기서는 cleanUp() 메서드를 실행해 member 테이블에 있는 데이터를 모두 삭제

```java
@SpringBootTest
@AutoConfigureMockMvc
class TestControllerTest {
    
    // 생략

    @DisplayName("getAllMembers: 아티클 조회에 성공한다.")
    @Test
    public void getAllMembers() throws Exception {
        // given
        final String url = "/test";
        Member savedMember = memberRepository.save(new Member(1L, "홍길동"));

        // when
        final ResultActions result = mockMvc.perform(get(url) // 1️⃣
                .accept(MediaType.APPLICATION_JSON)); // 2️⃣

        // then
        result
                .andExpect(status().isOk()) // 3️⃣
                // 4️⃣ 응답의 0번째 값이 DB에 저장한 값과 같은지 확인
                .andExpect(jsonPath("$[0].id").value(savedMember.getId()))
                .andExpect(jsonPath("$[0].name").value(savedMember.getName()));
    }
}
```
    - given : 멤버를 저장
    - when : 멤버 리스트를 조회하는 API 호출
    - then : 응답 코드가  OK이고, 반환값은 값 중에 0번째 요소인 id와 name이 저장된 값과 같은지 확인
- 1️⃣ perform()메서드를 요청 전송하는 역할
  - ResultActions객체를 받고, ResultActions 객체를 ㅣ반환하고 검증하고 확인하는 andExcept() 메서드 제공
- 2️⃣ accept() 메서드 : 요청을 보낼 때 무슨 타입으로 응답 받을지 결정하는 메소드
- 3️⃣ andExcepts() 메서드 : 응답 검증
- 4️⃣ jsonPath("$[0].$(필드명)") : JSON 응답값의 값을 가져오는 역할을 하는 메서드

## 05장 데이터베이스 조작이 편해지는 ORM

### 핵심키워드
> DBMS / ORM / JPA / 하이버네이트 / 엔티티 / 영속성 컨텍스트 / 스프링 데이터 JPA

### 5.1 데이터베이스? 데이터를 효울적으로 보관하고 꺼내 볼수 있는 곳
<br>

#### 데이터베이스 관리자(DBMS)? 데이터베이스를 관리 하기 위한 소프트웨어
- DB는 많은 사람이 공유할 수 있어야하므로 동시 접근이 가능 해야함
- 이런 요구 사항을 만족하면서도 효율적으로 DB응 관리하고 운영 하는 것 > **DBMS**
- 종류 : MySQL, 오라클 등
- 특징에 따른 분류 : 관계형, 객체-관꼐형, 도튜먼트형, 비관계형 등

#### 관꼐형 DBMS(RDBMS)? 테이블 형태로 이루어진 데이터 저장소
- H2 : 자바로 작성 되어있는 RDBMS
    - 스프링 부트가 지원하는 인메모리 관계형 데이터 베이스
    - 데이터를 다른 공간에 따로 보관하는 것이 아니라 애플리케이션 자체 내부에 데이터를 저장 하는 특징이 있음
    - 애플리케이션을 다시 실행 시 데이터 초기화
    - 테스트 용도로 많이 사용

#### 알아야 하는 DB용어
- 테이블
  - DB에서 데이터를 구성하기 위한 가장 기본적인 단위
  - 행과 열로 구성되며 행은 여러 속성으로 구성 됨
- 행(row)
  - 테이블의 가로로 배열된 데이터의 집합을 의미
  - 행은 반드시 고유한 식별자인 기본키를 가짐
  - 레코드(record)라고 불림
- 열(column)
  - 행에 저장되는 유형의 데이터
  - 각 요소에 대한 속성을 나타내며 데이터에 대한 무결성 보장
- 기본키(primary key)
  - 행을 구별할 수 있는 식별자
  - 테이블에서 유일해야하며 중복값을 가질 수 없음
  - 데이터를 수정하거나 삭제하고, 조회할 때 사용되며 다른 테이블과 관계를 맺어 데이터를 가져 올 수 있음
  - 수정이 안되며, 유효한 값이어야하고 NULL이면 안됨
- 쿼리(query)
  - DB에서 데이터를 조회 하거나 삭제 생성, 수정, 같은 처리를 하기 위해 사용하는 명령문
  - SQL이라는 DB 전용 언어를 사용하여 작성

#### SQL문

- **SELECT** : 테이블 조회
- **WHERE** : 조건
```sql
SELECT <가져올 컬럼 명>
  FROM <가져올 테이블 명>
 WHERE <조건>
```

- **INSERT** : 데이터 추가
```sql
INSERT INTO <테이블명(넣어줄 속성명)>
VALUES <넣어줄 값>
```
- **DELETE** : 데이터 삭제
```sql
DELETE FROM <테이블명>
 WHERE <조건>;
```

- **UPDATE** : 테이블 수정
```sql
UPDATE <테이블명>
   SET <컬럼명> = <값>
 WHERE <조건>;
```

### 5.2 ORM이란?
#### ORM(Object Relational Mapping)? 데이터베이스와 자바의 객체를 연결하는 기법
- ORM을 통해 DB값을 객체처럼 사용할 수 있게 되어 SQL을 몰라도 DB에서 데이터를 꺼내와 사용할 수 있음
- 객체와 DB를 연결해 자바 언어로만 DB를 다룰 수 있게 하는 도구

- 장점
  - SQL을 직접 작성하지 않고 DB에 접근 가능
  - 비지니스 로직에만 집중 가능
  - DB시스템이 추상화 되어 시스템에 대한 종속성이 줄어듦
  - 매핑하는 정보가 명확하기 때문에 ERD에 대한 의존도 낮추고 유지보수 유리

- 단점
  - 프로젝트의 복잡성이 커질수록 난이도 ⬆️
  - 복잡하고 무거운 쿼리는 경우에 따라 해결 불가능 경우 있음

### 5.3 JPA와 하이버네이트?
#### JPA? 자바에서 관계형 DB를 사용하는 방식을 정의한 인터페이스
- 실제 사용을 위해서는 ORM 프레임워크를 추가로 사용 해야함
- 자바 객체와 DB를 연결해 데이터 관리함
- 객체지향 도메인 모델과 DB의 다리역할
#### 하이버네이트? JPA인터페이스를 구현한 구현체이자 자바용 ORM프레임 워크
- 목표 : 자바 객체를 통해 데이터베이스 종류에 상관 없이 데이터베이스를 자유자재로 사용 가능하게 함
- JPA 인테페이스 구현, 내부적으로 JDBC API사용
#### 엔티티? 데이터베이스 테이블과 직접 연결되는 특별한 특징을 가진 객체
#### 엔티티 매니저? 엔티티를 관리해 DB와 애플리케이션 사이에서 객체를 생성, 수정, 삭제하는 등의 역할
- 엔티티 매니저를 만드는 곳 : 매니저 팩토리
#### 영속성 컨텍스트? 영속성 컨텍스트는 JPA의 주요 특징 중 하나로, 엔티티를 관리하는 가상의 공간
- 특징
    - 1차 캐시
        - 영속성 컨텍스트는 캐싱 처리
        - 캐시의 키는 @Id 어노테이션이 달린 기본키 역할을 하는 식별자, 값은 엔티티
        - 엔티티를 조회하면 1차캐시에서 데이터를 조회하고 값이 있음 반환
        - 값이 없으면 DB 조회해 1차캐시에 저장 후 반환
        - 매우 빠르게 데이터 조회 가능
    - 쓰기 지연
        - 트랜잭션을 커밋하기 전까지 DB에 실제로 질의문을 보내지 않고 모았다가 트랜잭션을 커밋하며 모았던 쿼리를 한번에 실행
        - 적당한 묶음으로 쿼리를 요청 가능해 DB시스템의 부담을 줄 일 수 있음
    - 변경 감지
        - 트랜잭션 커밋으로 1차 캐시에 저장되어 있는 엔티티 값과 변경된 엔티티 값을 비교하여 변경 사항을 감지하고 이를 DB에 자동 반영함 
        - 적당한 묶음으로 쿼리를 요청 가능해 DB시스템의 부담을 줄 일 수 있음
    - 지연 로딩
        - 쿼리 요청 데이터를 바로 로딩하지 않고 필요할 때 로딩하는 것을 의미
- 엔티티 상태
    - 분리 상태 : 영속성 컨텍스트가 관리하지 있지 않는 상태
      - datach()
    - 관리 상태 : 영속 컨텍스트가 관리하는 상태
      - persist()
    - 비영속 상태 : 영속 컨텍스트와 전혀 관계없는 상태
        - 엔티티를 처음 만든 상태
    - 삭제 상태 : 삭제된 상태
      - remove()
### 5.4 스프링 데이터와 스프링 데이터 JPA
#### 스프링 데이터? 비즈니스 로직에 더 집중 할 수 있게 DB사용 기능을 클래스 레벨에서 추상화한 것
- 각 데이터베이스의 특징에 맞춰 기능을 확장해 제공하는 기술 제공
#### 스프링 데이터 JPA? JPA를 더 편리하게 사용하는 메서드 제공
- 리포지토리 역할을 하는 인터페이스를 만들어 데이터베이스의 테이블 조회, 수정, 삭제, 생산 같은 작업을 간단히 수행 가능
- 메소드
  - **조회 메소드**
    > findAll() 메소드 사용
  - **쿼리 메소드**
    > id로 조회 : findById() 메소드 사용
    > 특정 컬럼 조회 > 쿼리 메소드 명명 규칙에 맞게 정의후 사용
  - **추가 , 삭제 메소드**
    > 추가 : save() 메소드
    > 여러 엔티티 한꺼번에 추가 : saveAll() 메소드
    > 아이디 사용 삭제 : deleteById() 메소드
    > 모든 데이터 삭제 : deleteAll() 메소드
  - **수정 메소드**
    > 수정 : 조회 후 트랜잭션 범위내에서 필드 값 변경