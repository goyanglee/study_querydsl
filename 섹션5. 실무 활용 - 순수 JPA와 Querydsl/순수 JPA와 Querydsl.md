# 순수 JPA와 Querydsl

## 순수 JPA 리포지토리와 Querydsl

### 순수 JPA Repository를 만들고 테스트코드 작성하기

```java
@Repository
public class MemberJpaRepository { //엔티티를 조회하기 위해 데이터에 접근하는 계층

    private final EntityManager em; //순수JPA이기 때문에 엔티티매니저필요함
    private final JPAQueryFactory queryFactory; //쿼리DSL 쓰기 위해 필요 : 파람으로 엔티티매니저 필요

    public MemberJpaRepository(EntityManager em) { //스프링 생성자 인젝션
        this.em = em;
        this.queryFactory = new JPAQueryFactory(em);
    }

    public void save(Member member) {
        em.persist(member);
    }

    public Optional<Member> findById(Long id) {
        Member findMember = em.find(Member.class, id);
        return Optional.ofNullable(findMember);
    }

    public List<Member> findAll() {
        return em.createQuery("select m from Member m", Member.class)
                .getResultList();
    }

    public List<Member> findByUsername(String username) {
        return em.createQuery("select m from Member m where m.username = :username", Member.class)
                .setParameter("username", username)
                .getResultList();
    }
}
```

```java
@SpringBootTest
@Transactional
class MemberJpaRepositoryTest {

    @Autowired
    EntityManager em;
    @Autowired
    MemberJpaRepository memberJpaRepository;

    @Test
    public void basicTest() {
        Member member = new Member("member1", 10);
        memberJpaRepository.save(member);

        Member findMember = memberJpaRepository.findById(member.getId()).get();
        Assertions.assertThat(findMember).isEqualTo(member);

        List<Member> result1 = memberJpaRepository.findAll();
        Assertions.assertThat(result1).containsExactly(member);

        List<Member> result2 = memberJpaRepository.findByUsername("member1");
        Assertions.assertThat(result2).containsExactly(member);
    }
}
```

<br/>

### JPQL로 작성된 부분을 Querydsl로 바꾸기 (발생쿼리 동일함)

```java
public List<Member> findAll_Querydsl() {
  return queryFactory
    .selectFrom(QMember.member)
    .fetch();
}

public List<Member> findByUsername_Querydsl(String username) {
  return queryFactory
    .selectFrom(QMember.member)
    .where(QMember.member.username.eq(username))
    .fetch();
}
```

<br/>

#### 장점

- 런타임 오류 방지 (컴파일 시점에 오류를 감지할 수 있음)
- 간단한 코드
- 파라미터 자동 바인딩

<br/>



### 참고

JPAQueryFactory을 빈으로 만들어서 사용할 수도 있다. (위 예제는 생성자 주입 방식을 사용함)

```java
@SpringBootApplication
public class Querydsl1Application {
	public static void main(String[] args) {
		SpringApplication.run(Querydsl1Application.class, args);
	}
  
	@Bean
	JPAQueryFactory jpaQueryFactory(EntityManager em) {
		return new JPAQueryFactory(em);
	}
}
```

```java
@Repository
public class MemberJpaRepository {
    private final EntityManager em; 
    private final JPAQueryFactory queryFactory; 
  
    public MemberJpaRepository(EntityManager em, JPAQueryFactory queryFactory) {
        this.em = em;
        this.queryFactory = queryFactory;
    }
  ...
```

<br/>

#### 장점

- 생성자 롬복(@RequiredArgsConstructor) 사용이 간편하다. 
  - @RequiredArgsConstructor는 final로 선언되어 있는 변수들을 자동 주입해준다. 

#### 단점

- 두 개나 주입해줘야해서 번거롭다. 테스트코드 작성 시 EntityManage와 JpaQueryFactory 모두 선언해줘야 한다. 

<br/>

#### 팩토리를 빈으로 등록했을 때 동시성 문제는 없나?

JPAQueryFactory의 동시성 문제는 EntityManager에 의존한다. EntityManger를 스프링과 함께 사용하게 되면 트랜잭션 단위로 분리되어 동작을 하게된다. 이 때 동작하는 EntityManager는 가짜 프록시이고(영속성 컨텍스트가 아님) 이 프록시가 트랜잭션 단위로 모든 동작을 각각 바인딩되도록 라우팅?해준다. 결론은 스프링과 함께 동작하기 때문에 멀티스레드 환경에서 동시성 이슈를 걱정하지 않아도 된다.

<br/>



## 동적 쿼리와 성능 최적화 조회 - Builder 사용

<내용>

- 리포지토리에서 동작쿼리를 해결하는 방법
- 성능최적화를 위해 빌더를 사용해서 한번에 Dto로 조회하기 (예: 멤버와 팀의 정보를 섞어서 원하는 정보만 한번에 조회)

<br/>

### DTO 프로젝션 준비

```java
@Data
public class MemberTeamDto {
    private Long memberId;
    private String username;
    private int age;
    private Long teamId;
    private String teamName;

    @QueryProjection
    public MemberTeamDto(Long memberId, String username, int age, Long teamId, String teamName) {
        this.memberId = memberId;
        this.username = username;
        this.age = age;
        this.teamId = teamId;
        this.teamName = teamName;
    }
}
```

- @QueryProjection 사용함
  - Gradle > Task > other > compileQuerydsl 해줘야 함 > build/generated에 Q파일 생성됨

<br/>

### 검색조건을 위한 DTO 준비

```java
@Data
public class MemberSearchCondition {
    //회원명, 팀명, 나이(ageGoe, ageLoe)로 필터링
  
    private String username;
    private String teamName;
    private Integer ageGoe; //나이가 크거나같거나
    private Integer ageLoe; //나이가 작거나같거나
}
```

<br/>

### 리포지리에 검색을 위한 메소드 만들기

```java
public List<MemberTeamDto> searchByBuilder(MemberSearchCondition condition) {
  //빌더로 동적쿼리 문제해결
  BooleanBuilder builder = new BooleanBuilder();
  if (StringUtils.hasText(condition.getUsername())) { //null, "" 모두 체크
    builder.and(QMember.member.username.eq(condition.getUsername()));
  }
  if (StringUtils.hasText(condition.getTeamName())) {
    builder.and(QTeam.team.name.eq(condition.getTeamName()));
  }
  if (condition.getAgeGoe() != null) {
    builder.and(QMember.member.age.goe(condition.getAgeGoe()));
  }
  if (condition.getAgeLoe() != null) {
    builder.and(QMember.member.age.loe(condition.getAgeLoe()));
  }

  return queryFactory
    .select(new QMemberTeamDto(
      QMember.member.id.as("memberId"),
      QMember.member.username,
      QMember.member.age,
      QTeam.team.id.as("teamId"),
      QTeam.team.name.as("teamName")
    ))
    .from(QMember.member)
    .leftJoin(QMember.member.team, QTeam.team)
    .where(builder) //빌더를 사용해서 동적쿼리 해결, MemberTeamDto로 한번에 조회해서 성능최적화
    .fetch();
}
```

<br/>

### 테스트코드

```java
@Test
public void searchTest() {
  Team teamA = new Team("teamA");
  Team teamB = new Team("teamB");
  em.persist(teamA);
  em.persist(teamB);

  Member member1 = new Member("member1", 10, teamA);
  Member member2 = new Member("member2", 20, teamA);
  Member member3 = new Member("member3", 30, teamB);
  Member member4 = new Member("member4", 40, teamB);
  em.persist(member1);
  em.persist(member2);
  em.persist(member3);
  em.persist(member4);

  MemberSearchCondition condition = new MemberSearchCondition();
  condition.setAgeGoe(35);
  condition.setAgeLoe(40);
  condition.setTeamName("teamB");

  List<MemberTeamDto> result = memberJpaRepository.searchByBuilder(condition);
  Assertions.assertThat(result).extracting("username").containsExactly("member4");
}

/* select
			member1.id as memberId,
			member1.username,
			member1.age,
			team.id as teamId,
			team.name as teamName
	from
		Member member1
	left join
		member1.team as team
	where
		team.name = ?1
		and member1.age >= ?2
		and member1.age <= ?3 */
```

<br/>

### 실무에서 주의할 내용 (조건이 없는 경우)

```java
MemberSearchCondition condition = new MemberSearchCondition();
//condition.setAgeGoe(35);
//condition.setAgeLoe(40);
//condition.setTeamName("teamB");
List<MemberTeamDto> result = memberJpaRepository.searchByBuilder(condition);

/* select
        member1.id as memberId,
        member1.username,
        member1.age,
        team.id as teamId,
        team.name as teamName 
    from
        Member member1   
    left join
        member1.team as team */
```

- 조건없이 쿼리가 나가면 전체데이터를 긁어오게 된다. 
- 리밋 또는 조건이 있는 것이 좋고, 가급적 페이징 쿼리를 함께 사용해야한다. 

<br/>



## 동적 쿼리와 성능 최적화 조회 - Where절 파라미터 사용

<내용>

- 동적쿼리를 Where절 파라미터 방식을 사용해서 해결하기

```java
public List<MemberTeamDto> search(MemberSearchCondition condition) {
  return queryFactory
    .select(new QMemberTeamDto(
        QMember.member.id.as("memberId"),
        QMember.member.username,
        QMember.member.age,
        QTeam.team.id.as("teamId"),
        QTeam.team.name.as("teamName")
    ))
    .from(QMember.member)
    .leftJoin(QMember.member.team, QTeam.team)
    .where( //빌더에 비해 가독성좋음
        usernameEq(condition.getUsername()),
        teamnameEq(condition.getTeamName()),
        ageGoe(condition.getAgeGoe()),
        ageLoe(condition.getAgeLoe())
  	)
    .fetch();
}

private BooleanExpression usernameEq(String username) {
  return StringUtils.hasText(username) ? QMember.member.username.eq(username) : null;
}
private BooleanExpression teamnameEq(String teamName) {
  return StringUtils.hasText(teamName) ? QTeam.team.name.eq(teamName) : null;
}
private BooleanExpression ageGoe(Integer ageGoe) {
  return ageGoe != null ? QMember.member.age.goe(ageGoe) : null;
}
private BooleanExpression ageLoe(Integer ageLoe) {
  return ageLoe != null ? QMember.member.age.loe(ageLoe) : null;
}
```

<br/>

#### 장점

- 빌더에 비해 가독성이 좋다

- 메소드를 재사용할 수 있다

  ```java
  //엔티티조회 - 셀렉트 프로젝션이 달라져도 메소드들을 재사용할 수 있다. 
  public List<Member> searchMember(MemberSearchCondition condition) {
    return queryFactory
      .selectFrom(QMember.member)
      .leftJoin(QMember.member.team, QTeam.team)
      .where(
          usernameEq(condition.getUsername()),
          teamnameEq(condition.getTeamName()),
          ageGoe(condition.getAgeGoe()),
          ageLoe(condition.getAgeLoe())
    	)
      .fetch();
  }
  ```

- 다양한 함수를 조립해서 조건을 만들 수 있다

  ```java
  public List<Member> searchMember(MemberSearchCondition condition) {
    return queryFactory
      .selectFrom(QMember.member)
      .leftJoin(QMember.member.team, QTeam.team)
      .where(
          usernameEq(condition.getUsername()),
          teamnameEq(condition.getTeamName()),
          ageBetween(condition.getAgeLoe(), condition.getAgeGoe())
      )
      .fetch();
  }
  private BooleanExpression ageBetween(int ageLoe, int ageGoe) {
    return ageLoe(ageLoe).and(ageGoe(ageGoe));
  }
  ```

<br/>



## 조회 API 컨트롤러 개발

<내용>

- 샘플데이터를 넣고
- 조회 API를 만들어서 동적쿼리를 사용해보자. 

<br/>

### 사전세팅하기(서버 구동 시 샘플데이터 넣는 작업)

#### 프로파일 설정

- 테스트코드가 깨지지 않도록 환경을 분리

- 기존 application.yml에 프로파일을 'local'로 설정

  ```properties
  spring:
    profiles:
      active: local
  ```

- src/test/resrouces/application.yml 파일 생성하고 프로파일을 'test'로 설정

  ```properties
  spring:
    profiles:
      active: test
  ```

<br/>

#### 데이터초기화(서버 구동 시 데이터넣는 코드 작성)

- 로컬 프로파일로 실행되면서 해당 코드가 동작하도록 @Profile 어노테이션 달아주고
- 스프링 빈으로 올라갈 수 있도록 @Component 붙이고
- InitMemberService는 생성자 자동주입됨
- 서버 뜨면서 @PostConstruct가 제일 먼저 실행된다. 

```java
@Profile("local")
@Component
@RequiredArgsConstructor
public class InitMember {

    private final InitMemberService initMemberService;
    @PostConstruct
    public void init() {
        initMemberService.init();
    }

    @Component
    static class InitMemberService {
        @PersistenceContext
        private EntityManager em;

        @Transactional
        public void init() {
            Team teamA = new Team("teamA");
            Team teamB = new Team("teamB");
            em.persist(teamA);
            em.persist(teamB);

            for (int i = 0; i < 100; i++) {
                Team selectedTeam = i % 2 == 0 ? teamA : teamB;
                em.persist(new Member("member"+i, i, selectedTeam));
            }
        }
    }
}
```

<br/>

### 컨트롤러 만들기

```java
@RestController
@RequiredArgsConstructor
public class MemberController {

    private final MemberJpaRepository memberJpaRepository;

    @GetMapping("/v1/members")
    public List<MemberTeamDto> searchMemberV1(MemberSearchCondition condition) {
        return memberJpaRepository.search(condition);
    }
}
```

<br/>

#### 실행 결과

http://localhost:8080/v1/members?teamName=teamB&ageGoe=31&ageLoe=35

```
[
    {
        "memberId": 34,
        "username": "member31",
        "age": 31,
        "teamId": 2,
        "teamName": "teamB"
    },
    {
        "memberId": 36,
        "username": "member33",
        "age": 33,
        "teamId": 2,
        "teamName": "teamB"
    },
    {
        "memberId": 38,
        "username": "member35",
        "age": 35,
        "teamId": 2,
        "teamName": "teamB"
    }
]
```
