## 스프링 데이터 JPA 리파지토리로 변경

1. 인터페이스 생성 

```kotlin
interface MemberRepository : JpaRepository<Member, Long> {
    fun findByUsername(username: String): List<Member>
}
```

1. 테스트 작성 

```kotlin
@SpringBootTest
@Transactional
internal class MemberRepositoryTest {

    @Autowired
    lateinit var repository: MemberRepository

    //런타임 시점 오류 가능
    @Test
    fun saveTest() {
        val member = Member("member1", 10)
        repository.save(member)
        //저장
        val oneSelected = repository.findById(member.memberId).get()
        assertThat(member).isEqualTo(oneSelected)

        //전체 셀렉 그리고 셀렉 여부
        val members = repository.findAll()
        assertThat(members).containsExactly(member)

        //유저네임 셀렉
        val selected = repository.findByUsername("member1").get(0)
        assertThat(selected).isEqualTo(member)
    }

}
```

findById같은 함수들은 JpaRepository에서 제공을 해주는데 QueryDsl 을 같이 쓰고 싶으면 어떻게 할까?

## 사용자 정의 리포지토리

1. 리포지토리 생성 

```kotlin
interface MemberRepository : JpaRepository<Member, Long>, MemberRepositoryCustom {
    fun findByUsername(username: String): List<Member>
}
```

1. Custom 리포지토리 생성 

```kotlin
interface MemberRepositoryCustom {
    fun search(condition: MemberTeamCondition): List<MemberTeamDto>
}
```

1. Impl 생성

```kotlin
class MemberRepositoryImpl(
    private val queryFactory: JPAQueryFactory
) : MemberRepositoryCustom {

    override fun search(condition: MemberTeamCondition): List<MemberTeamDto> {
        return queryFactory
            .select(
                QMemberTeamDto(
                    QMember.member.memberId,
                    QMember.member.username,
                    QMember.member.age,
                    QTeam.team.teamId,
                    QTeam.team.name
                )
            )
            .from(QMember.member)
            .leftJoin(QMember.member.team, QTeam.team)
            .where(usernameEq(condition.username),
                teamNameEq(condition.teamName),
                ageGoe(condition.ageGoe),
                ageLod(condition.ageLoe))
            .fetch()
    }

    private fun ageLod(ageLoe: Int?): BooleanExpression? {
        return ageLoe?.let { QMember.member.age.loe(it)}
    }

    private fun ageGoe(ageGoe: Int?): BooleanExpression? {
        return ageGoe?.let { QMember.member.age.goe(it) }
    }

    private fun teamNameEq(teamName: String?): BooleanExpression? {
        return teamName?.let { QTeam.team.name.eq(it) }
    }

    private fun usernameEq(username: String?): BooleanExpression? {
        return username?.let { QMember.member.username.eq(it) }
    }
}
```

너무 특화된 조회 기능은 별도의 @Repository 를 붙인 클래스를 생성해서 사용하는 것을 추천한다. 

## 페이징

1. 페이지를 사용하는 함수 생성 

```kotlin
fun searchPageSimple(condition: MemberTeamCondition, pageable: Pageable): Page<MemberTeamDto>
```

1. Impl 생성

```kotlin
override fun searchPageSimple(condition: MemberTeamCondition, pageable: Pageable): Page<MemberTeamDto> {
        val result =  queryFactory
            .select(
                QMemberTeamDto(
                    QMember.member.memberId,
                    QMember.member.username,
                    QMember.member.age,
                    QTeam.team.teamId,
                    QTeam.team.name
                )
            )
            .from(QMember.member)
            .leftJoin(QMember.member.team, QTeam.team)
            .where(usernameEq(condition.username),
                teamNameEq(condition.teamName),
                ageGoe(condition.ageGoe),
                ageLod(condition.ageLoe))
            .offset(pageable.offset)
            .limit(pageable.pageSize.toLong())
            .fetchResults() //fetch query, count query 해서 2번 날림

        val count = result.total
        val results = result.results

        return PageImpl(results, pageable, count)

    }
```

1. 요청해보기

```kotlin
@Test
    fun searchPageSimple() {
        val teamA = Team("teamA")
        val teamB = Team("teamB")
        em.persist(teamA)
        em.persist(teamB)
        val member1 = Member("member1", 10, teamA)
        val member2 = Member("member2", 20, teamA)
        val member3 = Member("member3", 30, teamB)
        val member4 = Member("member4", 40, teamB)
        em.persist(member1)
        em.persist(member2)
        em.persist(member3)
        em.persist(member4)
        var condition = MemberTeamCondition(null, null, null, null)

        val page = PageRequest.of(0, 3) //0페이지의 3개를 가져오겠다.
        val result  = repository.searchPageSimple(condition, page)

        assertThat(result.size).isEqualTo(3)
        assertThat(result.content).extracting("username").containsExactly("member1", "member2", "member3")
    }

//실행 쿼리 
//select
//            member0_.member_id as col_0_0_,
//            member0_.username as col_1_0_,
//            member0_.age as col_2_0_,
//            team1_.team_id as col_3_0_,
//            team1_.name as col_4_0_ 
//        from
//            member member0_ 
//        left outer join
//            team team1_ 
//                on member0_.team_id=team1_.team_id limit ?
```

### 최적화

- 단, totalCount 쿼리를 날리는 경우 order by 를 무시하기 때문에 위의 예제에서는 fetchResults를 함으로서 카운트 쿼리를 날려서 order by 가 소용이 없다.

### 카운트와 컨텐츠 별도 조회

```kotlin
override fun searchPageComplex(condition: MemberTeamCondition, pageable: Pageable): Page<MemberTeamDto> {
        val result =  queryFactory
            .select(
                QMemberTeamDto(
                    QMember.member.memberId,
                    QMember.member.username,
                    QMember.member.age,
                    QTeam.team.teamId,
                    QTeam.team.name
                )
            )
            .from(QMember.member)
            .leftJoin(QMember.member.team, QTeam.team)
            .where(usernameEq(condition.username),
                teamNameEq(condition.teamName),
                ageGoe(condition.ageGoe),
                ageLod(condition.ageLoe))
            .offset(pageable.offset)
            .limit(pageable.pageSize.toLong())
            .fetch()

        val totalCount = queryFactory.selectFrom(member)
            //.leftJoin(member.team, team)
            .where(usernameEq(condition.username),
                teamNameEq(condition.teamName),
                ageGoe(condition.ageGoe),
                ageLod(condition.ageLoe))
            .fetchCount()

        return PageImpl(result, pageable, totalCount)
    }
```

 조인이 필요없다. 조인을 했을떄와 안했을때 카운트가 같을 경우가 있는데 별도로 조회하는 것도 방법이다. 혹은 카운트를 먼저 조회해서 없으면 데이터를 조회안한다거나 하는 경우도 있다.

## 카운트 쿼리 최적화

1. 카운트 쿼리를 생략 가능한 경우 생략 
    1. 페이지 시작이면서 컨텐츠 사이즈가 페이지 사이즈보다 작을 때 
    2. 마지막 페이지일때 offset + 컨텐츠사이즈 해서 구할 수 있으니까 이런 경우 

```kotlin
override fun searchPageComplex(condition: MemberTeamCondition, pageable: Pageable): Page<MemberTeamDto> {
        val result =  queryFactory
            .select(
                QMemberTeamDto(
                    QMember.member.memberId,
                    QMember.member.username,
                    QMember.member.age,
                    QTeam.team.teamId,
                    QTeam.team.name
                )
            )
            .from(QMember.member)
            .leftJoin(QMember.member.team, QTeam.team)
            .where(usernameEq(condition.username),
                teamNameEq(condition.teamName),
                ageGoe(condition.ageGoe),
                ageLod(condition.ageLoe))
            .offset(pageable.offset)
            .limit(pageable.pageSize.toLong())
            .fetch()

        val countQuery = queryFactory.selectFrom(member)
            .where(usernameEq(condition.username),
                teamNameEq(condition.teamName),
                ageGoe(condition.ageGoe),
                ageLod(condition.ageLoe))
            

        
        return PageableExecutionUtils.getPage(result, pageable) {
            countQuery.fetchCount()
        }
        
    }
```

## 페이징 활용해서 컨트롤러 개발

localhost:8080/v2/members?page=1&size=5

```kotlin
@GetMapping("/v2/members")
    fun searchMemberV2(condition: MemberTeamCondition, pageable: Pageable): Page<MemberTeamDto> {
        return memberRepository.searchPageSimple(condition, pageable)
    }
```

```kotlin
select
            member0_.member_id as col_0_0_,
            member0_.username as col_1_0_,
            member0_.age as col_2_0_,
            team1_.team_id as col_3_0_,
            team1_.name as col_4_0_ 
        from
            member member0_ 
        left outer join
            team team1_ 
                on member0_.team_id=team1_.team_id limit ?

select
            count(member0_.member_id) as col_0_0_ 
        from
            member member0_ 
        left outer join
            team team1_ 
                on member0_.team_id=team1_.team_id
```

[http://localhost:8080/v3/members?page=0&size=100](http://localhost:8080/v3/members?page=0&size=100) 

```kotlin
@GetMapping("/v3/members")
    fun searchMemberV3(condition: MemberTeamCondition, pageable: Pageable): Page<MemberTeamDto> {
        return memberRepository.searchPageComplex(condition, pageable)
    }
```

```kotlin
select
            member0_.member_id as col_0_0_,
            member0_.username as col_1_0_,
            member0_.age as col_2_0_,
            team1_.team_id as col_3_0_,
            team1_.name as col_4_0_ 
        from
            member member0_ 
        left outer join
            team team1_ 
                on member0_.team_id=team1_.team_id limit ?
```

[http://localhost:8080/v3/members?page=0&size=5](http://localhost:8080/v3/members?page=0&size=5)

```kotlin
select
            member0_.member_id as col_0_0_,
            member0_.username as col_1_0_,
            member0_.age as col_2_0_,
            team1_.team_id as col_3_0_,
            team1_.name as col_4_0_ 
        from
            member member0_ 
        left outer join
            team team1_ 
                on member0_.team_id=team1_.team_id limit ?

select
            count(member0_.member_id) as col_0_0_ 
        from
            member member0_
```

스프링 데이터에서 소트를 QueryDsl(OrderSpecifier)로 변경하는 기능을 제공한다. 조인과 같이 복잡해지면 잘 동작하지 않기 때문에 직접 받아서 우리가 해주는걸 추천한다.

## 내가 몰랐던 것들..

```kotlin
@RequestParam 은 생략 가능하다.
스프링은 String, int와 같은 단순 타입은 RequestParam 이 생략된 것으로 보고 자동으로 바인딩해준다.
```
