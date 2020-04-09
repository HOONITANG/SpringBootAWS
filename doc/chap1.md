# Step-01 스프링 부트와 AWS로 혼자 구현하는 웹 서비스 
찾아보고 싶은 내용을 챕터 마다 정리 

## 목차
* IntellJ 추천 Plugin 
* lombok이란 ?
* Spring boot vs Spring vs gradle
* Spring Initalizr 란?
* mavencenter(), jcenter
* Chap2에서 정리 할 TDD

## IntellJ 추천 Plugin
* Command + Shift + A 로 Action plugins검색  
[https://goddaehee.tistory.com/198](https://goddaehee.tistory.com/198)
1. Grep Console
2. Key Promoter X *
3. Lombok *
4. Rainbow Brackets *
5. CodeGlance : 코드 미니맵
6. Request Mapper * : 엔드 포인트 URL 기반의 검색 및 바로가기 기능 제공 (SHIFT + CTRL + \)
7. iBATIS/MyBatis plugin : 해당하는 자바, XML로 이동하는 플러그인 인 것 같다. 왜 쓰지

## lombok이란?
출처: https://cheese10yun.github.io/lombok/
자바 컴파일 시점에서 특정 어노테이션으로 해당 코드를 추가 할 수 있는 라이브러리 입니다. 이는 가독서 미 유지 보수에 많은 도움이 됩니다.
하지만 편리한 만큼 잘못 사용하기 쉽습니다.

### @DATA는 지양 하자
```java
@Entity
@Table(name = "member")
@Data
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @Column(name = "email", nullable = false)
    private String email;

    @Column(name = "name", nullable = false)
    private String name;

    @CreationTimestamp
    @Column(name = "create_at", nullable = false, updatable = false)
    private LocalDateTime createAt;

    @UpdateTimestamp
    @Column(name = "update_at", nullable = false)
    private LocalDateTime updateAt;
}
```
@Data는 @ToString, @EqualsAndHashCode, @Getter, @Setter, @RequiredArgsConstructor을 한번에 사용하는 강력한 어노테이션 입니다.
강력한 어노테이션인 만큼 그에 따른 부작용도 많다고 생각합니다.

1. 무분별한 `Setter` 남용
**Setter는 그 의도가 분명하지 않고 객체를 언제든지 변경 할 수 있는 상태가 되어서 객체의 안정성이 보장 받기 힙듭니다.**

위 코드에서 email의 변겨 기능이 되지 않는다고 가정하면 email 관련된 setter도 제공되지 않아야 안전합니다.

2. `ToString`으로 인한 양방향 연관관계시 순환 참조 문제

```java
@Entity
@Table(name = "member")
@Data
public class Member {
    ....
    @OneToMany
    @JoinColumn(name = "coupon_id")
    private List<Coupon> coupons = new ArrayList<>();
}

@Entity
@Table(name = "coupon")
@Data
public class Coupon {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @ManyToOne
    private Member member;

    public Coupon(Member member) {
        this.member = member;
    }
}
```
####테스트 코드
```java
public class MemberTest {
  @Test
  public void test_01() {
    final Member member = new Member("asd@asd.com","name");
    final Coupon coupon = new Coupon(member);
    final List<Coupon> coupons = new ArrayList<>();
    coupons.add(coupons);
    member.setCoupons(coupons);
    
    member.toString();
  }
}
```

####정리
->member을 선언함-> 선언한 멤버를 coupon에 넣음 -> coupon객체를 coupon을 담을 수 있는 List를 만들어서 넣음
-> 이 couponsList를 처음 선언한 member에 있는 List coupon 변수에 집어넣음

member에 있는 .toString()을 호출함 -> List coupon안에 있는 coupon이 .toString을 호출함 coupon안에 있는 member가 .toString을 호출
-> member에 있는 .toString()을 호출함 -> 무한 반복

####해결법
toString에서 coupons는 제외 하면 된다.
```java
@ToString(exclude = "coupons")
public class Member {...}
```


### 바람직한 Lombok 사용법

```java
@Entity
@Table(name = "member")
@ToString(exclude = "coupons")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@EqualsAndHashCode(of = {"id", "email"})
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @Column(name = "email", nullable = false)
    private String email;

    @Column(name = "name", nullable = false)
    private String name;

    @CreationTimestamp
    @Column(name = "create_at", nullable = false, updatable = false)
    private LocalDateTime createAt;

    @UpdateTimestamp
    @Column(name = "update_at", nullable = false)
    private LocalDateTime updateAt;

    @OneToMany
    @JoinColumn(name = "coupon_id")
    private List<Coupon> coupons = new ArrayList<>();

    @Builder
    public Member(String email, String name) {
        this.email = email;
        this.name = name;
    }
}
```

* 데이터 안전성
  - 정보 변경 API에서는 firstName, lastName 두 속성만 변경할 수 있다고 했으면 Account 클래스로 RequestBody를 받게 된다면 email, password, Account 클래스의 모든 속성값들을 컨트롤러를 통해서 넘겨받을 수 있게 되고 원치 않은 데이터 변경이 발생할 수 있습니다.
  - firstName, lastName 속성 이외의 값들이 넘어온다면 그것은 잘못된 입력값이고 그런 값들을 넘겼을 경우 Bad Request 처리하는 것이 안전합니다.
   - Response 타입이 Account 클래스일 경우 계정의 모든 정보가 노출 되게 됩니다. JsonIgnore 속성들을 두어 임시로 막는 것은 바람직하지 않습니다.속성들을 두어 임시로 막는 것은 바람직하지 않습니다.
* 명확해지는 요구사항
  - MyAccountReq 클래스는 마이 어카운트 페이지에서 변경할 수 있는 값들로 address1, address2, zip 속성이 있습니다. 요구사항이 이 세 가지 속성에 대한 변경이어서 해당 API가 어떤 값들을 변경할 수가 있는지 명확해집니다.
