# 스켈레톤 코드 분석3

날짜: January 18, 2022
유형: 스켈레톤 코드 분석

## SignService.java

```java
package com.ssafy.api.service;

import com.ssafy.core.code.JoinCode;
import com.ssafy.core.code.YNCode;
import com.ssafy.core.entity.User;
import com.ssafy.core.exception.ApiMessageException;
import com.ssafy.core.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class SignService {
    private final UserRepository userRepository;

    /**
     * id로 회원정보 조회
     *
     * @param id
     * @return
     * @throws Exception
     */
    public User findUserById(long id) throws Exception {
        User user = userRepository.findById(id).orElseThrow(() -> new ApiMessageException("존재하지 않는 회원정보입니다."));
        return user;
    }

    /**
     * uid로 user 조회
     *
     * @param uid
     * @return
     * @throws Exception
     */
    public User findByUid(String uid, YNCode isBind) throws Exception {
        return userRepository.findByUid(uid, isBind);
    }

    /**
     * 회원가입 후 userId 리턴
     *
     * @param user
     * @return
     */
    @Transactional(readOnly = false)
    public long userSignUp(User user) {
        User signUpUser = userRepository.save(user);
        return signUpUser.getId();
    }

    /**
     * uid, type으로 회원정보 조회
     *
     * @param uid
     * @param type
     * @return
     */
    public User findUserByUidType(String uid, JoinCode type) {
        return userRepository.findUserLogin(uid, type);
    }

    /**
     * 회원 엔티티 저장
     *
     * @param user
     */
    @Transactional(readOnly = false)
    public void saveUser(User user) {
        userRepository.save(user);
    }

}
```

- saveUser  메소드를 만들면서 의문이 생겼던 곳이 있다.
    - Entity 업데이트는 필요한 데이터만 수정한 후에 Entity 자체를 save() 처리한다. (save()는 *Impl.java 에 별도 작성 없이 사용 가능)
- 여기서 userRepository는 무엇일까?
    - import com.ssafy.core.repository.UserRepository;
- [UserRepository.java](http://UserRepository.java) 를 확인해 보니 인터페이스였다.

```java
package com.ssafy.core.repository;

import com.ssafy.core.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface UserRepository extends JpaRepository<User, Long>, UserRepoCommon{
}
```

- 이는 JpaRepository와 UserRepoCommon을 상속받는다.
    - UserRepoCommon은 사용자가 만든 repository 인터페이스라서 pass
    - **JpaRepository**란 무엇일까?
    

그 전에 Entity부터 살펴보자.

## Entity

---

- 먼저 데이터베이스에 저장하기 위해 유저가 정의한 클래스가 필요한데 그런 클래스를 **Entity**라고 한다. Domain이라고 생각하면 된다. 일반적으로 **RDBMS에서 Table을 객체화** 시킨 것으로 보면 된다. 그래서 Table의 이름이나 컬럼들에 대한 정보를 가진다.

- 스켈레톤 코드의 [User.java](http://User.java) 라는 Entity이다.

```java
@Builder
@Getter
@NoArgsConstructor
@AllArgsConstructor
@Entity
@Table(name = "user",
    uniqueConstraints = {
            @UniqueConstraint(
                    columnNames={"uid"}
            )
    }
)
// 회원 테이블
public class User extends BaseEntity implements UserDetails {

    // User 테이블의 키값 = 회원의 고유 키값
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    // 회원이 가입한 타입 (none:일반, sns:소셜회원가입)
    @Convert(converter = JoinCodeConverter.class)
    @Column(nullable = false, length = 5)
    private JoinCode joinType;

    // 회원아이디(일반:아이디, 소셜회원가입:발급번호)
    @Column(nullable = false, unique = true, length = 120)
    private String uid;

    // 비밀번호
    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    @Column(nullable = false, length = 255)
    private String password;

    // 회원 이름
    @Column(nullable = false, length = 32)
    private String name;

    // 회원 닉네임
    @Column(nullable = false, length = 64)
    private String nickname;

    // 회원 이메일
    @Column(nullable = false, length = 255)
    private String email;

    // 회원 휴대폰
    @Column(nullable = true, length = 32)
    private String phone;

    // 성별 (남자 M, 여자 F)
    @Convert(converter = MFCodeConverter.class)
    @Column(nullable = false, length = 1, columnDefinition = "varchar(1) default ''")
    private MFCode gender;

    // 회원 나이
    @Column(nullable = false, length = 3, columnDefinition = "int(3) default 0")
    private int age;

    // 회원 프로필 이미지
    @Column(nullable = false, length = 255)
    private String img;

    // 주소
    @Column(length = 255)
    private String address;

    // 상세주소
    @Column(length = 255)
    private String addressDetail;

    // 푸쉬알림 설정 (Y:on, N:off)
    @Convert(converter = YNCodeConverter.class)
    @Column(nullable = false, length = 1, columnDefinition = "varchar(1) default 'Y'")
    private YNCode push;

    // 장비 푸시용 토큰
    @Column(length = 255)
    private String token;

    // 사용 여부
    @Convert(converter = YNCodeConverter.class)
    @Column(nullable = false, length = 1, columnDefinition = "varchar(1) default 'Y'")
    private YNCode isBind;
}
```

**@Id**

- primary key를 가지는 변수를 선언하는 것을 뜻한다. @GeneratedValue 어노테이션은 해당 Id 값을 어떻게 자동으로 생성할지 전략을 선택할 수 있다.
    - 여기서 선택한 전략은 “IDENTITY” 이다. (default = “AUTO)
        - **: insert 쿼리가 pk 값 없이 수행된다.**
        - **: 데이터베이스의 auto_increment 동작이 수행된다.**
        - **: ddl-auto: create 을 사용중이라면 pk 옵션이 auto_increment로 생성된다.**

**@Table**

- 별도의 이름을 가진 데이터베이스 테이블과 매핑한다. 기본적으로 @Entity로 선언된 클래스의 이름은 실제 데이터베이스의 테이블 명과 일치하는 것을 매핑한다. 따라서 @Entity의 클래스명과 데이터베이스의 테이블명이 다를 경우에 @Table(name=" ")과 같은 형식을 사용해서 매핑이 가능하다.

**@Column**

- @Column 선언이 꼭 필요한 것은 아니다. 하지만 @Column에서 지정한 변수명과 데이터베이스의 컬럼명을 서로 다르게 주고 싶다면 @Column(name=" ") 같은 형식으로 작성하면 된다. 그렇지 않은 경우에는 **기본적으로 멤버 변수명과 일치하는 데이터베이스 컬럼을 매핑**한다.

## JpaRepository란?

---

> Entity클래스를 작성했다면 이번엔 Repository 인터페이스를 만들어야 한다.
스프링부트에서는 Entity의 **기본적인 CRUD가 가능하도록 JpaRepository 인터페이스**를 제공한다.
> 
- JpaRepository는 인터페이스이다. 인터페이스에 미리 검색 메소드를 정의 해 두는 것으로, 메소드를 호출하는 것만으로 스마트한 데이터 검색을 할 수 있게 됨

```java
@Repository
public interface 이름 extends JpaRepository <엔티티, ID 유형>
```

- JpaRepository 인터페이스는 org.springframework.data.jpa.repository 패키지의 "JpaRepository"라는 인터페이스를 상속
- @Repository : 이 인터페이스가 JpaRepository임을 나타냄. 반드시 붙여 두어야 한다.
- <>안에는 엔티티 클래스 이름과 ID 필드 타입이 지정.  기본형의 경우 래퍼 클래스를 지정한다.

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long>, UserRepoCommon{
}
```

- JpaRepository를 상속하는 것 만으로도 그 인터페이스는 Entity 하나에 대해서 아래와 같은 기능을 제공한다.

| method | 기능 |
| --- | --- |
| save() | 레코드 저장 (insert, update) |
| findOne() | primary key로 레코드 한건 찾기 |
| findAll() | 전체 레코드 불러오기. 정렬(sort), 페이징(pageable) 가능 |
| count() | 레코드 갯수 |
| delete() | 레코드 삭제 |
- 위의 기본기능을 제외한 조회 기능을 추가하고 싶으면 **규칙에 맞는 메서드를 추가**해야한다.
- 스켈레톤 코드의 UserRepoCommon 도 비슷한 방식임.

```java
public interface MemberRepository extends JpaRepository<Member, Long> {

    Member findByName(String name);

    Page<Member> findByName(String name, Pageable pageable);
}
```

- 위와 같이 **Query 메소드를 추가하여 스프링에게 알릴 수 있다.** 그러기위해서는 **규칙에 맞는 메서드를 작성**해야 하는데, 그 규칙은 다음과 같다.

| method | 설명 |
| --- | --- |
| findBy로 시작 | 쿼리를 요청하는 메서드 임을 알림 |
| countBy로 시작 | 쿼리 결과 레코드 수를 요청하는 메서드 임을 알림 |

- 위의 findBy에 이어 해당 Entity 필드 이름을 입력하면 검색 쿼리를 실행한 결과를 전달한다.
- SQL의 where절을 메서드 이름을 통해 전달한다고 생각하면 된다.
- 메서드의 반환형이 Entity 객체이면 하나의 결과만을 전달하고, 반환형이 List라면 쿼리에 해당하는 모든 객체를 전달한다.

- Query 메소드에 **포함할 수 있는 키워드**는 다음과 같다.

| 메서드 이름 키워드 | 샘플 | 설명 |
| --- | --- | --- |
| And | findByEmailAndUserId(String email, String userId) | 여러필드를 and 로 검색 |
| Or | findByEmailOrUserId(String email, String userId) | 여러필드를 or 로 검색 |
| Between | findByCreatedAtBetween(Date fromDate, Date toDate) | 필드의 두 값 사이에 있는 항목 검색 |
| LessThan | findByAgeGraterThanEqual(int age) | 작은 항목 검색 |
| GreaterThanEqual | findByAgeGraterThanEqual(int age) | 크거나 같은 항목 검색 |
| Like | findByNameLike(String name) | like 검색 |
| IsNull | findByJobIsNull() | null 인 항목 검색 |
| In | findByJob(String … jobs) | 여러 값중에 하나인 항목 검색 |
| OrderBy | findByEmailOrderByNameAsc(String email) | 검색 결과를 정렬하여 전달 |

+) 페이징 처리도 이를 이용해 할 수 있다 ⇒ Pageable

참고자료

[JPA 사용법 (JpaRepository)](https://jobc.tistory.com/120)

쿼리 메소드 작성법

[Spring Data JPA - Reference Documentation](https://docs.spring.io/spring-data/jpa/docs/current-SNAPSHOT/reference/html/#jpa.query-methods.query-creation)
