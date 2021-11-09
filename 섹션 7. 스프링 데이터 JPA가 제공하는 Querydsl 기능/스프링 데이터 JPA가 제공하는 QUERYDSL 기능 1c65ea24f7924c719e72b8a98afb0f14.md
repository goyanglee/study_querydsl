# 스프링 데이터 JPA가 제공하는 QUERYDSL 기능

스프링 데이터 JPA가 제공하는 Querydsl 기능

<br><br>

: 스프링 데이터가 querydsl을 위해 몇가지 기능을 제공하는데, 여기서 소개하는 기능은 제약이 커서 복잡한 실무 환경에서 사용하기에는 많이 부족하다. 왜 부족한지 배워보자! 

<br><br>

## 인터페이스 지원 - QuerydslPredicateExecutor

공식 URL: [https://docs.spring.io/spring-data/jpa/docs/2.2.3.RELEASE/reference/html/#core.extensions.querydsl](https://docs.spring.io/spring-data/jpa/docs/2.2.3.RELEASE/reference/html/#core.extensions.querydsl)

```java
QuerydslPredicateExecutor 인터페이스
public interface QuerydslPredicateExecutor {
 Optional findById(Predicate predicate);
 Iterable findAll(Predicate predicate);
 long count(Predicate predicate);
 boolean exists(Predicate predicate);
 // … more functionality omitted.
}
```

memberRepository에 추가하기.

```java
interface MemberRepository extends JpaRepository,
QuerydslPredicateExecutor { //기능을 다 사용하고 parameter로 querydsl조건을 넣을 수 있다.
}
```

test

```java
@Test
    public void querydslPredicateExcutorTest(){
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

        Iterable<Member> result = memberRepository.findAll(
                member.age.between(10, 40)
                        .and(member.username.eq("member1"))
        );

        for(Member findMember:result){
            System.out.println("findMember = " + findMember);
        }
    }

/* select
        member1 
    from
        Member member1 
    where
        member1.age between ?1 and ?2 
        and member1.username = ?3 */
```

### * 한계점

- 조인X (묵시적 조인은 가능하지만 left join이 불가능하다.)
- 클라이언트가 Querydsl에 의존해야 한다. 서비스 클래스가 Querydsl이라는 구현 기술에 의존해야 한다. : querydsl 객체를 넘겨야하기에.
- 복잡한 실무환경에서 사용하기에는 한계가 명확하다.

> 참고: QuerydslPredicateExecutor 는 Pagable, Sort를 모두 지원하고 정상 동작한다.

<br><br>

### Querydsl Web 지원(권장하지 않음)

공식 URL: [https://docs.spring.io/spring-data/jpa/docs/2.2.3.RELEASE/reference/html/
#core.web.type-safe](https://docs.spring.io/spring-data/jpa/docs/2.2.3.RELEASE/reference/html/)

```java
@Controller
class UserController {

  @Autowired UserRepository repository;

  @RequestMapping(value = "/", method = RequestMethod.GET)
  String index(Model model, @QuerydslPredicate(root = User.class) Predicate predicate,    
          Pageable pageable, @RequestParam MultiValueMap<String, String> parameters) {

    model.addAttribute("users", repository.findAll(predicate, pageable));

    return "index";
  }
} //이렇게하면, predicate를 만들어서, querydsl에 파라미터 바인딩을 딱해준다.
```

### *한계점

- 단순한 조건만 가능(거의 eq,contains,in 정도만 된다고보면됨)
- 조건을 커스텀하는 기능이 복잡하고 명시적이지 않음(querydslBinderCustomizer)
- 컨트롤러가 Querydsl에 의존
- 복잡한 실무환경에서 사용하기에는 한계가 명확

<br><br>

### 리포지토리 지원 - QuerydslRepositorySupport

querydsl 라이브러리를 쓰기위해서 레파지토리 구현체가 받으면 편리한 추상클래스

```java
public class MemberRepositoryImpl extends QuerydslRepositorySupport implements MemberRepositoryCustom
```

: `EntityManager`, `JPAQueryFactory` 도 제공해준다. 

```java
public List<MemberTeamDto> search(MemberSearchCondition condition) {
        return from(member)
                .leftJoin(member.team, team)
                .where(usernameEq(condition.getUsername()),
                        teamNameEq(condition.getTeamName()),
                        ageGoe(condition.getAgeGoe()),
                        ageLoe(condition.getAgeLoe()))
                .select(new QMemberTeamDto(
                        member.id,
                        member.username,
                        member.age,
                        team.id,
                        team.name))
                .fetch();
    }
```

: from 부터 시작, querydsl 3버전까지는 from부터 였는데 3버전을 따름

<br><br>

query 이용하는 방법

```java
JPQLQuery<MemberTeamDto> jpaQuery = from(member)
                .leftJoin(member.team, team)
                .where(usernameEq(condition.getUsername()),
                        teamNameEq(condition.getTeamName()),
                        ageGoe(condition.getAgeGoe()),
                        ageLoe(condition.getAgeLoe()))
                .select(new QMemberTeamDto(
                        member.id.as("memberId"),
                        member.username,
                        member.age,
                        team.id.as("teamId"),
                        team.name.as("teamName")));

        JPQLQuery<MemberTeamDto> query =  getQuerydsl().applyPagination(pageable,jpaQuery);
        query.fetch();
```

### **장점**

- getQuerydsl().applyPagination() 스프링 데이터가 제공하는 페이징을 Querydsl로 편리하게 변환
가능(단! Sort는 오류발생)
- from() 으로 시작 가능(최근에는 QueryFactory를 사용해서 select() 로 시작하는 것이 더 명시적)
EntityManager 제공

### 한계

- Querydsl 3.x 버전을 대상으로 만듬
- Querydsl 4.x에 나온 JPAQueryFactory로 시작할 수 없음
    - select로 시작할 수 없음 (from으로 시작해야함)
- QueryFactory 를 제공하지 않음(그냥 주입받으면 됨)
- 스프링 데이터 Sort 기능이 정상 동작하지 않음 (스프링데이터의 Qsort 이용해야함)

<br><br><br><br>

### Querydsl 지원 클래스 직접 만들기

스프링 데이터가 제공하는 QuerydslRepositorySupport 가 지닌 한계를 극복하기 위해 직접 Querydsl
지원 클래스를 만들어보자.

### 장점

- 스프링 데이터가 제공하는 페이징을 편리하게 변환
- 페이징과 카운트 쿼리 분리 가능
- 스프링 데이터 Sort 지원
- select() , selectFrom() 으로 시작 가능
- EntityManager , QueryFactory 제공

### Querydsl4RepositorySupport

```java
package study.querydsl.repository.support;

import com.querydsl.core.types.EntityPath;
import com.querydsl.core.types.Expression;
import com.querydsl.core.types.dsl.PathBuilder;
import com.querydsl.jpa.impl.JPAQuery;
import com.querydsl.jpa.impl.JPAQueryFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.support.JpaEntityInformation;
import
        org.springframework.data.jpa.repository.support.JpaEntityInformationSupport;
import org.springframework.data.jpa.repository.support.Querydsl;
import org.springframework.data.querydsl.SimpleEntityPathResolver;
import org.springframework.data.repository.support.PageableExecutionUtils;
import org.springframework.stereotype.Repository;
import org.springframework.util.Assert;

import javax.annotation.PostConstruct;
import javax.persistence.EntityManager;
import java.util.List;
import java.util.function.Function;

/**
 * Querydsl 4.x 버전에 맞춘 Querydsl 지원 라이브러리
 *
 * @author Younghan Kim
 * @see org.springframework.data.jpa.repository.support.QuerydslRepositorySupport
 */

@Repository
public abstract class Querydsl4RepositorySupport {
    private final Class domainClass;
    private Querydsl querydsl;
    private EntityManager entityManager;
    private JPAQueryFactory queryFactory;

    public Querydsl4RepositorySupport(Class<?> domainClass) {
        Assert.notNull(domainClass, "Domain class must not be null!");
        this.domainClass = domainClass;
    }

    @Autowired
    public void setEntityManager(EntityManager entityManager) {
        Assert.notNull(entityManager, "EntityManager must not be null!");
        JpaEntityInformation entityInformation =
                JpaEntityInformationSupport.getEntityInformation(domainClass, entityManager);
        SimpleEntityPathResolver resolver = SimpleEntityPathResolver.INSTANCE;
        EntityPath path = resolver.createPath(entityInformation.getJavaType());
        this.entityManager = entityManager;
        this.querydsl = new Querydsl(entityManager, new
                PathBuilder<>(path.getType(), path.getMetadata()));
        this.queryFactory = new JPAQueryFactory(entityManager);
    }

    @PostConstruct
    public void validate() {
        Assert.notNull(entityManager, "EntityManager must not be null!");
        Assert.notNull(querydsl, "Querydsl must not be null!");
        Assert.notNull(queryFactory, "QueryFactory must not be null!");
    }

    protected JPAQueryFactory getQueryFactory() {
        return queryFactory;
    }

    protected Querydsl getQuerydsl() {
        return querydsl;
    }

    protected EntityManager getEntityManager() {
        return entityManager;
    }

    protected<T> JPAQuery<T> select(Expression<T> expr) {
        return getQueryFactory().select(expr);
    }

    protected<T> JPAQuery<T> selectFrom(EntityPath<T> from) {
        return getQueryFactory().selectFrom(from);
    }

    protected<T> Page<T> applyPagination(Pageable pageable,
                                   Function<JPAQueryFactory, JPAQuery> contentQuery) {
        JPAQuery jpaQuery = contentQuery.apply(getQueryFactory());
        List<T> content = getQuerydsl().applyPagination(pageable,
                jpaQuery).fetch();
        return PageableExecutionUtils.getPage(content, pageable,
                jpaQuery::fetchCount);
    }

    protected<T> Page<T> applyPagination(Pageable pageable,
                                   Function<JPAQueryFactory, JPAQuery> contentQuery, Function<JPAQueryFactory, JPAQuery> countQuery) {
        JPAQuery jpaContentQuery = contentQuery.apply(getQueryFactory());
        List content = getQuerydsl().applyPagination(pageable,
                jpaContentQuery).fetch();
        JPAQuery countResult = countQuery.apply(getQueryFactory());
        return PageableExecutionUtils.getPage(content, pageable,
                countResult::fetchCount);
    }
}
```

### Querydsl4RepositorySupport 사용 코드 : MemberTestRepository

```java
package study.querydsl.repository;

import com.querydsl.core.types.dsl.BooleanExpression;
import com.querydsl.jpa.impl.JPAQuery;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.repository.support.PageableExecutionUtils;
import org.springframework.stereotype.Repository;
import study.querydsl.dto.MemberDto;
import study.querydsl.dto.MemberSearchCondition;
import study.querydsl.entity.Member;
import study.querydsl.repository.support.Querydsl4RepositorySupport;

import java.util.List;

import static org.springframework.util.StringUtils.isEmpty;
import static study.querydsl.entity.QMember.member;
import static study.querydsl.entity.QTeam.team;

@Repository
public class MemberTestRepository extends Querydsl4RepositorySupport {
    public MemberTestRepository() {
        super(Member.class);
    }

    public List<Member> basicSelect() {
        return select(member)
                .from(member)
                .fetch();
    }

    public List<Member> basicSelectFrom() {
        return selectFrom(member)
                .fetch();
    }

    public Page searchPageByApplyPage(MemberSearchCondition condition,
                                      Pageable pageable) {
        JPAQuery query = selectFrom(member)
                .leftJoin(member.team, team)
                .where(usernameEq(condition.getUsername()),
                        teamNameEq(condition.getTeamName()),
                        ageGoe(condition.getAgeGoe()),
                        ageLoe(condition.getAgeLoe()));
        List content = getQuerydsl().applyPagination(pageable, query)
                .fetch();
        return PageableExecutionUtils.getPage(content, pageable,
                query::fetchCount);
    }

    public Page applyPagination(MemberSearchCondition condition,
                                Pageable pageable) {
        return applyPagination(pageable, contentQuery -> contentQuery
                .selectFrom(member)
                .leftJoin(member.team, team)
                .where(usernameEq(condition.getUsername()),
                        teamNameEq(condition.getTeamName()),
                        ageGoe(condition.getAgeGoe()),
                        ageLoe(condition.getAgeLoe())));
    }

    public Page applyPagination2(MemberSearchCondition condition,
                                 Pageable pageable) {
        return applyPagination(pageable, contentQuery -> contentQuery
                        .selectFrom(member)
                        .leftJoin(member.team, team)
                        .where(usernameEq(condition.getUsername()),
                                teamNameEq(condition.getTeamName()),
                                ageGoe(condition.getAgeGoe()),
                                ageLoe(condition.getAgeLoe())),
                countQuery -> countQuery
                        .selectFrom(member)
                        .leftJoin(member.team, team)
                        .where(usernameEq(condition.getUsername()),
                                teamNameEq(condition.getTeamName()),
                                ageGoe(condition.getAgeGoe()),
                                ageLoe(condition.getAgeLoe()))
        );
    }

    private BooleanExpression usernameEq(String username) {
        return isEmpty(username) ? null : member.username.eq(username);
    }

    private BooleanExpression teamNameEq(String teamName) {
        return isEmpty(teamName) ? null : team.name.eq(teamName);
    }

    private BooleanExpression ageGoe(Integer ageGoe) {
        return ageGoe == null ? null : member.age.goe(ageGoe);
    }

    private BooleanExpression ageLoe(Integer ageLoe) {
        return ageLoe == null ? null : member.age.loe(ageLoe);
    }
}
```

