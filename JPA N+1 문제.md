# JPA N+1 문제
- 조회가 한 번만 일어나도록 하였는데 여러 번 일어나는 경우 이를 N+1 문제라고 한다.
- 만약 엔티티에 연관관계가 존재한다면, 연관관계에 의해 한 번 더 쿼리가 발생할 수 있다.
- 예를 들어, 멤버-팀과의 연관관계가 존재한다고 해보자. 그러면 팀에는 멤버들의 리스트가 존재할 수 있다.
- 이 때, 팀을 select 해온다고 해보자. 그러면 팀에 딸린 멤버도 select 될 수 있다.

### 실제 예시
``` java
@Entity
public class Member {
    @Id
    @GeneratedValue
    private Long id;

    private String memberName;

    @OneToMany(mappedBy = "member", fetch = FetchType.LAZY)
    private List<BioInfo> bioInfos = new ArrayList<>();

    public Member() {

    }

    public Member(String memberName) {
        this.memberName = memberName;
    }

    public void addBioInfo(BioInfo bioInfo) {
        bioInfos.add(bioInfo);
    }

    public List<BioInfo> getBioInfos() {
        return bioInfos;
    }

    public String getMemberName() {
        return memberName;
    }
}

@Entity
public class BioInfo {
    @Id
    @GeneratedValue
    private Long id;

    private String bloodPressure;

    private String timestamp;

    @ManyToOne
    @JoinColumn(name = "member_id")
    private Member member;

    public BioInfo(String bloodPressure, String timestamp) {
        this.bloodPressure = bloodPressure;
        this.timestamp = timestamp;
    }

    public BioInfo() {

    }

    public void setMember(Member member) {
        this.member = member;
    }

    public String getTimestamp() {
        return timestamp;
    }
}
```
위와 같은 두 개의 엔티티가 존재한다고 해보자. 하나의 유저는 여러 바이오정보를 가질 수 있다. 이를 통해 N+1 문제를 발생시켜보자.
그를 위해서 다음과 같은 테스트를 진행해보자.

```java
@Test
void test() {
    EntityManagerFactory emf = Persistence.createEntityManagerFactory("jpaproject");
    EntityManager em = emf.createEntityManager();
    EntityTransaction tx = em.getTransaction();

    tx.begin();
    createExample(em);
    em.flush();
    em.clear();

    System.out.println("===before jpql===");
    List<Member> members = em.createQuery("select m from Member m join m.bioInfos", Member.class)
            .getResultList();
    System.out.println("===after jpql===");

    for(Member member: members) {
        System.out.println(member.getMemberName() + " : " + member.getBioInfos().size());
    }

    tx.commit();
    em.close();
    emf.close();
}

private void createExample(EntityManager em) {
    Member member1 = new Member("김철수");
    Member member2 = new Member("박영희");
    Member member3 = new Member("이진수");

    for(int i=0; i<4; i++) {
        BioInfo bioInfo = new BioInfo("120", Instant.now().atZone(ZoneId.of("Asia/Seoul")).toString());
        bioInfo.setMember(member1);
        em.persist(bioInfo);
        member1.addBioInfo(bioInfo);
    }

    for(int i=0; i<4; i++) {
        BioInfo bioInfo = new BioInfo("120", Instant.now().atZone(ZoneId.of("Asia/Seoul")).toString());
        bioInfo.setMember(member2);
        em.persist(bioInfo);
        member2.addBioInfo(bioInfo);
    }

    for(int i=0; i<4; i++) {
        BioInfo bioInfo = new BioInfo("120", Instant.now().atZone(ZoneId.of("Asia/Seoul")).toString());
        bioInfo.setMember(member3);
        em.persist(bioInfo);
        member3.addBioInfo(bioInfo);
    }

    em.persist(member1);
    em.persist(member2);
    em.persist(member3);
}
```

중요하게 봐야할 코드는 다음 줄이다.
```java
List<Member> members = em.createQuery("select m from Member m join m.bioInfos", Member.class)
            .getResultList();
```

이 코드는 그냥 join을 사용하여 bioInfo가 존재하는 member들을 가져온다. 개발자는 select 문 한 번으로 다음과 같은 효과를 기대할 것이다.
```SQL
SELECT m.id, m.member_name, b.id, b.bloodPressure, b.timestamp, b.member_id
WHERE Member m join BioInfo b
```

하지만 실제로는 다음과 같은 query가 나가는 것을 알 수 있다.
```SQL
Hibernate: 
    select
        m1_0.id,
        m1_0.memberName 
    from
        Member m1_0 
    join
        BioInfo bi1_0 
            on m1_0.id=bi1_0.member_id
```

## 해결 방법

### Fetch Join
이 것을 해결하는 방법 중 하나는 Fetch Join을 사용하는 것이다.
```java
List<Member> members = em.createQuery("select m from Member m join fetch m.bioInfos", Member.class)
                .getResultList();
```
그러면 다음과 같은 결과를 얻을 수 있다.
```SQL
Hibernate: 
    select
        m1_0.id,
        bi1_0.member_id,
        bi1_0.id,
        bi1_0.bloodPressure,
        bi1_0.timestamp,
        m1_0.memberName 
    from
        Member m1_0 
    join
        BioInfo bi1_0 
            on m1_0.id=bi1_0.member_id
```

하지만 Fetch Join의 단점도 존재한다.
- Pagination이 사실상 불가능하다.
  - Pagination이 인메모리를 이용하여 fetch join을 한 결과를 retrieve하기 때문에 문제가 생길 수 있다는 것이다. 이는 OOM을 일으킬 수 있다.
  - 이는 일대다 관계에서만 생기는 문제이다.
    - 예를 들어, BioInfo가 약 천만건이 있다고 해보자. 그러면 페이지네이션을 사용하여 가져오게 되면, 천만건의 Fetch join 결과를 인메모리에 가지고 있다가 페이지만큼 돌려준다.
    - 일대일, 다대일의 경우는 이런 경우가 없을 것이다. 왜냐하면 결국에 fetch join을 해서 가져온 결과가 한 건에 불과할 것이기 때문이다.
이런 문제를 해결하기 위해 @BatchSize 등을 사용할 수 있다.
