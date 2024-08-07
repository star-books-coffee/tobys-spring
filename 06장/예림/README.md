# 06장. AOP
- AOP를 바르게 이용하려면 OOP를 대체하려고 하는 것처럼 보이는 AOP라는 이름 뒤에 감춰진, 그 필연적인 등장배경과 스프링이 그것을 도입한 이유, 그 적용을 통해 얻을 수 있는 장점이 무엇인지에 대한 충분한 이해가 필요하다.

- 서비스 추상화를 통해 많은 근본적인 문제를 해결했던 트랜잭션 경계설정 기능을 AOP를 이용해 더욱 세련되고 깔끔한 방식으로 바꿔보자


## 6.1 트랜잭션 코드의 분리
- 트랜잭션 경계 설정을 위해 넣은 코드 때문에 UserService 코드가 깔끔하지 않다.

### 6.1.1 메서드 분리
```java
// 6-1. 트랜잭션 경계설정과 비즈니스 로직이 공존하는 메서드
public void upgradeLevels() throws Exception {
    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    
    try {
        List<User> users = userDao.getAll();
        for (User user : users) {
            if (canUpgradeLevel(user)) {
                upgradeLevel(user);
            }
        }
        
        this.transactionManager.commit(status);
    } catch (Exception e) {
        this.transactionManager.rollback(status);
        throw e;
    }
}
```
- UserService의 트랜잭션 경계 설정 코드와 비즈니스 로직 코드가 복잡하게 얽혀 있는 듯 보이지만, 두 코드는 성격이 다를 뿐 아니라 서로 주고받는 정보가 없는, 완벽하게 독립적인 코드다.
- 비즈니스 로직을 담당하는 코드를 메서드로 추출해서 독립시켜보자.
```java
private void upgradeLevelsInternal() { 
    List<User> users = userDao.getAll(); 
    for (User user : users) {
        if (canUpgradeLevel(user)) { 
            upgradeLevel(user);
        } 
    }
}
```

### 6.1.2 DI를 이용한 클래스의 분리
- UserService 안의 트랜잭션 코드가 버젓이 UserService 안에 자리잡고 있는 것이 마음에 안든다.
- 간단하게 트랜잭션 코드를 클래스 밖으로 뽑아내면 보이지 않게 할 수 있지 않을까?

#### DI 적용을 이용한 트랜잭션 분리
- 트랜잭션 코드를 UserService 밖으로 빼저리면 UserService의 클라이언트 코드에서는 트랜잭션 기능이 빠진 UserService를 사용하게 된다.
- 구체적인 구현 클래스를 직접 참조하는 경우의 전형적인 단점이다.
- UserService 클래스와 그 사용 클라이언트인 UserServiceTest는 현재 강한 결합도로 고정되어 있다.

<img width="354" alt="스크린샷 2024-06-24 오후 5 06 26" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/9a682bcf-bc12-41ae-942b-68692eab0936">

- 다음과 같이 UserService를 인터페이스로 만들고 UserService의 구현 클래스를 만들어넣도록 하면 클라이언트와 결합이 약해지고, 직접 구현 클래스에 의존하고 있지 않기 때문에 유연한 확장이 가능해진다.
<img width="348" alt="스크린샷 2024-06-24 오후 5 09 45" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/bdf14d95-543c-4bce-a9b0-a003779a14e9">

- 한 번에 두 개의 UserService 인터페이스 구현 클래스를 동시에 이용한다면 어떨까? UserService를 구현한 또 다른 구현 클래스를 만드는 것이다.
- 이 클래스는 단지 트랜잭션 경계 설정이라는 책임을 맡고 있을 뿐이다.
- 그리고 또 다른 UserService 구현 클래스에 실제적인 로직 처리 작업은 위임하는 것이다.

<img width="566" alt="image" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/e5d3202b-ee9f-43be-812c-08e682acc682">

#### UserService 인터페이스 도입
- 기존의 UserService 클래스명을 UserServiceImpl로 변경하고 클라이언트가 사용할 로직을 담은 핵심 메서드만 UserService 인터페이스로 만들기
```java
// 6-3. UserService 인터페이스
public interface UserService {
    void add(User user);
    void upgradeLevels();
}
```
```java
// 7-4. 트랜잭션 코드를 제거한 UserService 구현 클래스
...
public class UserServiceImpl implements UserService {
    UserDao userDao;
    MailSender mailSender;

    public void upgradeLevels() {
        List<User> users = userDao.getAll();
        for(User user : users) {
            if(canUpdateLevel(user)) {
                upgradeLevel(user);
            }
        }
    }
}
```
#### 분리된 트랜잭션 기능
```java
// 6-5. 위임 기능을 가진 UserServiceTx 클래스
...
public UserServiceTx implements UserService {
    UserService userService;

    // UserService를 구현한 다른 오브젝트를 DI 받는다.
    public void setUserService(UserService userService) {
        this.userService = userService;
    }

    // DI 받은 UserService 오브젝트에 모든 기능을 위임한다.
    public void add(User user) {
        userService.add(user);
    }

    // DI 받은 UserService 오브젝트에 모든 기능을 위임한다.
    public void upgradeLevels() {
        userService.upgradeLevels();
    }
}
```
- 이렇게 준비된 UserServiceTex에 트랜잭션 경계 설정 작업을 부여해보자.
```java
// 6-6. 트랜잭션이 적용된 UserServiceTx
public class UserServiceTex implements UserService {
    UserService userService;
    PlatformTransactionManager transactionManager;

    public void setTransactionManager (PlatformTransationManager transactionManaer) {
            this.transactionManager = transactionManager;
    }

    public void setUserService(UserService userService) {
        ...
    }

    ...

    public void upgradeLevels() {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDenition());
        try {
            userService.upgradeLevels();

            this.transactionMananager.commit(status);
        } catch (RuntimeException e) {
            this.transactionManager.rollback(status);
            throw e;
        }
    }
}
```
#### 트랜잭션 적용을 위한 DI 설정
- 클라이언트가 UserService 인터페이스를 통해 사용자 관리 로직을 이용하려고 할 때, 먼저 트랜잭션을 담당하는 오브젝트가 사용돼서 트랜잭션에 관련된 작업을 진행해주고, 사용자 관리 로직을 담은 오브젝트가 이후에 호출돼서 비즈니스 로직에 관련된 작업을 수행하도록 만들어야 한다.

<img width="551" alt="스크린샷 2024-06-27 오후 7 43 19" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/37072770-7a21-453b-baa2-1dabcafaa298">


```xml
// 6-7. 트랜잭션 오브젝트가 추가된 설정 파일
<bean id="userService" class="springbook.user.service.UserServiceTx">
    <property name="transactionManager" ref="transactionManager" />
    <property name="userService" ref="userServiceImpl" />
</bean>

<bean id="userServiceImpl" class="springbook.user.service.UserServiceImpl">
    <property name="userDao" value="userDao" />
    <property name="mailSender" ref="mailSender" />
</bean>
```
- 이제 클라이언트는 UserServiceTx 빈을 호출해서 사용하도록 만들어야 한다.
- 따라서 userService라는 대표적인 빈 아이디는 UserServiceTex 클래스로 정의된 빈에 부여해준다.
#### 트랜잭션 분리에 따른 테스트 수정
- 기존의 UserService 클래스가 인터페이스와 두 개의 클래스로 분리된 만큼 테스트도 변경해야 한다.
- 현재 수정한 스프링 설정 파일에는 UserService라는 인터페이스 타입을 가진 빈이 두 개 존재한다. **UserService 클래스 타입의 빈을 @Autowired로 가져오면 어떤 빈을 가져올까?**
- @Autowired는 타입으로 하나의 빈을 결정할 수 없는 경우에는 필드 이름을 이용해 찾는다. 따라서 다음과 같은 userService 변수를 설정해두면 **아이디가 userService인 빈이 주입**될 것이다.
  ```java
  @Autowired UserService userService;
  ```
- UserServiceTest는 UserServcieImpl 클래스로 정의된 빈도 가져와야 한다.
- 일반적인 UserService 기능 테스트에서는 UserService 인터페이스를 통해 결과를 확인하는 것으로 충분하다. 그러나 목 오브젝트를 이용한 테스트에서는 테스트에서 직접 MailSender를 DI해줘야 할 필요가 있었다.
- MailSender를 DI해줄 대상을 구체적으로 알고 있어야 하기 때문에 UserServiceImpl 클래스의 오브젝트를 가져올 필요가 있다.
- 이렇게 목 오브젝트를 이용해 수동 DI를 적용하는 테스트라면 어떤 클래스의 오브젝트인지 분명하게 알 필요가 있다.
- 그래서 다음과 같이 UserServiceImpl 클래스 타입의 변수를 선언하고 @Autowired를 지정해서 해당 클래스로 만들어진 빈을 주입 받도록 해야 한다.
```java
@Autowired UserServiceImpl userServiceImpl;
```
- upgradeLevels() 테스트 메서드도 다음과 같이 별도로 가져온 userServiceImpl 빈에 MailSender의 목 오브젝트를 설정해줘야 한다.
```java
@Test
public void upgradeLevels() throws Exception {
    ...
    MockMailSender mockMailSender = new MockMailSender();
    userServiceImpl.setMailSender(mockMailSender);
```
- add() 테스트 메서드는 손댈 것이 없지만 upgradeAllNothing() 테스트는 트랜잭션 기술이 바르게 적용되었는지 확인하기 위한 일종의 학습 테스트이기 때문에 변경이 필요하다.
- 기존에는 TestUserService가 오브젝트를 만들어서 의존 오브젝트를 넣어주고서 테스트를 진행했다.
- 이제는 예외를 발생시킬 위치가 UserServiceImpl에 있기 때문에 TestUserService가 트랜잭션 기능은 빠진 UserServiceImpl을 상속하도록 해야 한다.
- 또한 TestUserService 오브젝트를 UserServiceTx 오브젝트에 수동 DI 시킨 후에 트랜잭션 기능까지 포함된 UserServiceTx의 메서드를 호출하면서 테스트를 수행하도록 해야 한다.
```java
// 6-9. 분리된 테스트 기능이 포함되도록 수정한 upgradeAllOrNothing()
@Test
public void upgradeAllOrNothing() throws Exception {
    TestUserService testUserService = new TestUserService(users.get(3).getId());
    testUserService.setUserDao(userDao);
    testUserService.setMailSender(mailSender);

    // 트랜잭션 기능을 분리한 UserServiceTx는 예외 발생용으로 수정할 필요가 없으니 그대로 사용
    UserServiceTx txUserService = new UserServiceTx();
    txUserService.setTransactionManager(transactionManager);
    txUserService.setUserService(testUserService);

    userDao.deleteAll();
    for(User user : users) userDao.add(user);

    try {
        txUserService.upgraeLevels(); // 트랜잭션 기능을 분리한 오브젝트를 통해 예외 발생용 TestUserService가 호출되게 해야 한다.
        fail("TestUserServiceException expected");
    }
    ...
```
- 트랜잭션 테스트용으로 특별히 정의한 TestUserService 클래스는 이제 UserServiceImpl 클래스를 상속하도록 바꿔주면 된다.
```java
static class TestUserService extends UserServiceImpl {
```

#### 트랜잭션 경계 설정 코드 분리의 장점
**1. 비즈니스 로직을 담당하는 코드를 작성할 때 트랜잭션과 같은 기술적인 내용에는 신경쓰지 않아도 된다.**

- 트랜잭션은 DI를 이용해 UserServiceTx와 같은 트랜잭션 기능을 가진 오브젝트가 먼저 실행되도록 만들기만 하면 된다.
- 스프링이나 트랜잭션 같은 로우레벨의 기술적인 지식은 부족한 개발자라고 할지라도 비즈니스 로직을 잘 이해하고 자바 언어의 기초에 충실하면 복잡한 비즈니스 로직을 담은 UserService 클래스 개발 가능
  
**2. 비즈니스 로직에 대한 테스트를 손쉽게 만들어낼 수 있다.**
- 6.2 절에서 좀 더 자세히 알아보자.



## 6.2 고립된 단위 테스트
- 가장 편하고 좋은 테스트 방법은 가능한 한 작은 단위로 쪼개서 테스트하는 것
  - 테스트 단위가 작아야 테스트 의도와 내용이 분명해지고, 만들기도 쉬워진다.
    - 테스트 대상의 단위가 커지면 충분한 테스트를 만들기도 쉽지 않다. 논리적인 오류가 발생해서 결과가 바르게 나오지 않았을 때 그 원인을 찾기도 어려워진다.
- 하지만 작은 단위로 테스트하고 싶어도 그럴 수 없는 경우가 많다.
- 테스트 대상이 다른 오브젝트와 환경에 의존하고 있다면 작은 단위의 테스트가 주는 장점을 얻기 힘들다.

### 6.2.1 복잡한 의존관계 속의 테스트
- UserService의 구현 클래스들이 동작하려면 세 가지 타입의 의존 오브젝트가 필요하다.
  - UserDao 타입의 오브젝트 : DB와 데이터를 주고받는다.
  - MailSender를 구현한 오브젝트 : 메일을 발송한다.
  - PlatformTransactionManger : 트랜잭션 처리를 위해 커뮤니케이션해야 한다.
- 다음은 UserService를 분리하기 전의 테스트가 동작하는 모습이다.
    <img width="576" alt="스크린샷 2024-06-30 오후 2 18 12" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/8e7a360c-76b3-44ac-ab57-458c8287ce8b">

- UserServiceTest의 테스트 단위는 UserService 클래스여야 한다.
- 그러나 UserDao, TransactionManager, MailSender라는 세 가지 의존관계를 갖고 있고, 이들이 테스트가 진행되는 동안에 **같이 실행된다**
- UserService를 테스트하는 것처럼 보이지만 사실은 그 뒤에 존재하는 훨씬 더 많은 오브젝트와 환경, 서비스, 서버, 심지어 네트워크까지 함께 테스트 대상이 되는 것이다.

### 6.2.2 테스트 대상 오브젝트 고립시키기
- 테스트를 의존 대상으로부터 분리해 고립시키는 방법은 MailSender를 적용해봤던 대로 테스트를 위한 대역을 사용하는 것이다. (DummyMailSender, MockMailSender)
#### 테스트를 위한 UserServiceImpl 고립
- UserServiceImpl은 트랜잭션 코드를 독립시켰기 떄문에 더이상 PlatformTransactionManager에 의존하지 않는다. ➡️ PlatformTransactionManager는 고립시키지 않아도 된다.
- 사전에 테스트를 위해 준비된 동작만 하도록 만든 두 개의 목 오브젝트에만 의존하는, 완벽하게 고립된 테스트 대상을 만들어보자.
  <img width="552" alt="스크린샷 2024-06-30 오후 2 26 24" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/64e14efb-daea-4a06-b242-bf30cddba85c">
- UserDaos는 단지 테스트 대상의 코드가 정상적으로 수행되도록 도와주기만 하는 스텁이 아니라, 부가적인 검증 기능까지 가진 목 오브젝트로 만들었다. 그 이유는 고립된 환경에서 동작하는 upgradeLevels()의 테스트 결과를 검증할 방법이 필요하기 때문이다.
- upgradeLevles() 메서드는 void 형 ➡️ 결과를 검증하는 것이 불가능하다.
- 따라서 그 동작이 바르게 됐는지 확인하려면 DB를 직접 확인해야 하는데, 의존 오브젝트나 외부 서비스에 의존하지 않는 고립된 테스트 방식으로 만든 UserServiceImpl은 아무리 기능이 수행돼도 DB 등을 통해 남지 않으니 작업 결과를 검증하기 힘들다.
- 이럴 땐 **테스트 대상인 UserServiceImpl과 그 협력 오브젝트인 UserDao에게 어떤 요청을 했는지 확인하는 작업이 필요하다.**
- UserDao와 같은 역할을 하면서 UserServiceImpl과의 사이에서 주고받은 정보를 저장해뒀다가, 테스트의 검증에 사용할 수 있게 하는 목 오브젝트를 만들 필요가 있다.

#### 고립된 테스트 활용
- 기존 upgradeLevels() 테스트는 다섯 단계의 작업으로 구성된다.
```java
// 6-10. upgradeLevels() 테스트
@Test
public void upgradeLevels() throws Exception {
  // (1) DB 테스트 데이터 준비
  userDao.deleteAll();
  for(User user : users) userDao.add(user);

  // (2) 메일 발송 여부 확인을 위한 목 오브젝트 DI
  MockMailSender mockMailSender = new MockMailSender(); 
  userService.setMailSender(mockMailSender);

  // (3) 테스트 대상 실행
  userService.upgradeLevels();

  // (4) DB에 저장된 결과 확인
  checkLevelUpgraded(users.get(0), false);
  checkLevelUpgraded(users.get(1), true);
  checkLevelUpgraded(users.get(2), false);
  checkLevelUpgraded(users.get(3), true);
  checkLevelUpgraded(users.get(4), false);

  // (5) 목 오브젝트를 이용한 결과 확인
  List<String> request = mockMailSender.getRequests();
  assertThat(request.size(), is(2));
  assertThat(request.get(0), is(users.get(1).getEmail()));
  assertThat(request.get(1), is(users.get(3).getEmail()));
}

private void checkLevelUpgraded(User user, boolean upgraded) {
  User userUpdate = userDao.get(user.getId());
  ...
}
```
- 실제 UserDao와 DB까지 직접 의존하고 있는 (1)과 (4) 테스트 방식도 목 오브젝트를 만들어 적용해보겠다.
- 목 오브젝트는 기본적으로 스텁과 같은 방식으로 테스트 대상을 통해 사용될 때 필요한 기능을 지원해줘야 한다.
- upgradeLevels() 메서드가 실행되는 중에 **UserDao와 어떤 정보를 주고받는지 입출력 내역을 먼저 확인할 필요가 있다.**
- upgradeLevels()와 그 사용 메서드에서 UserDao를 사용하는 경우는 두 가지다.
```java
// 6-1. 사용자 레벨 업그레이드 작업 중에 UserDao를 사용하는 코드
public void upgradeLevels() {
    List<User> users = userDao.getAll(); // (1) 업그레이드 후보 사용자 목록을 가져온다.
    for(User user : users) {
        if(canUpgradeLevel(user)) {
            upgradeLevel(user);
        }
    }
}

protected void upgradeLevel(User user) {
    user.upgradeLevel();
    userDao.upgrade(user); // (2) 수정된 사용자 정보를 DB에 반영한다.
    sendUpgradeEmail();
}
```
- (1) 기능을 테스트용으로 대체 ➡️ 테스트용 UserDao에서는 DB에서 읽어온 것처럼 미리 준비된 사용자 목록을 제공해줘야 한다.
- (2) 기능을 테스트용으로 대체 ➡️ 리턴 값이 없으므로 아무런 내용도 없는 빈 메서드로 만들어도 된다. 하지만 update() 메서드의 내용은 '변경'이 되었는지 검증할 수 있는 중요한 기능이기도 하다.
- 따라서 getAll()에 대해서는 스텁으로서, update()에 대해서는 목 오브젝트로서 동작하는 UserDao로서 동작하는 UserDao 타입의 테스트 대역이 필요하다.
- 이 클래스의 이름을 MockUserDao라고 하자.
```java
// 6-12. UserDao 오브젝트
static class MockUserDao implements UserDao {
    private List<User> users; // 레벨 업그레이드 후보 User 오브젝트 목록
    private List<User> updated = new ArrayList(); // 업그레이드 대상 오브젝트를 저장해둘 목록

    private MockUserDao(List<User> users) {
        this.users = users;
    }

    public List<User> getUpdated() {
        return this.updated;
    }

    // 스텁 기능 제공
    public List<User> getAll(); {
        return this.users;
    }

    // 목 오브젝트 기능 제공
    public void update(User user) {
        updated.add(user);
    }

    // 테스트에 사용되지 않는 메서드
    public void add(User user) { throw new UnsupportedOperationException(); }
    public void deleteAll() { throw new UnsupportedOperationException(); }
    public void get(String id) { throw new UnsupportedOperationException(); }
    public void getCount() { throw new UnsupportedOperationException(); }
}
```
- MockUserDao는 UserDao 구현 클래스를 대신하니 implements 해야 한다.
- 다만 이를 통해 사용하지 않을 메소드도 오버라이드 받게 되는데 이럴때는 UnsupportedOperationException을 던지도록 만드는 편이 좋다.
- 해당 클래스에는 두 개의 User 타입 리스트가 있는데, 변수명 users는 처음 생성자가 생성될 때 전달받은 사용자 목록을 저장한 뒤 getAll() 메소드가 호출되면 전달하는 용도이다.
- 다른 하나는 update() 메소드를 실행하면서 넘겨준 업그레이드 대상 User 오브젝트를 저장해뒀다가 검증을 위해 돌려주기 위한 것이다.이제 upgradeLevels() 테스트가 MockUserDao를 사용하도록 수정해보자.

- 이제 upgradeLevels() 테스트가 MockUserDao를 사용하도록 수정해보자.
```java
// 6-13. MockUserDao를 사용해서 만든 고립된 테스트
@Test
public void upgradeLevels() throws Exception{

    // 고립된 테스트에서는 테스트 대상 오브젝트를 직접 생성하면 된다.
    UserServiceImpl userServiceImpl = new UserServiceImpl();

    // 목 오브젝트로 만든 UserDao를 직접 DI 해준다.
    MockUserDao mockUserDao = new MockUserDao(this.users);
    userServiceImpl.setUserDao(mockUserDao);

    // 메일 발송 여부 확인을 위해 목 오브젝트 DI
    MockMailSender mockMailSender = new MockMailSender();
    userServiceImpl.setMailSender(mockMailSender);

    // 테스트 대상 실행
    userServiceImpl.upgradeLevels();

    List<User> updated = mockUserDao.getUpdated();
    assertThat(updated.size(), is(2));
    chekcUserAndLevel(updated.get(0), "joytouch", Level.SILVER);
    chekcUserAndLevel(updated.get(1), "madnite1", Level.GOLD);


    // 목 오브젝트를 이용한 결과 확인.
    // 목 오브젝트에 저장된 메일 수신자 목록을 가져와 업그레이드 대상과 일치하는지 확인 한다.
    List<String> request = mockMailSender.getRequests();
    assertThat(request.size(), is(2));
    assertThat(request.get(0), is(users.get(1).getEmail()));
    assertThat(request.get(1), is(users.get(3).getEmail()));

}
```
- 고립된 테스트 이전에는 @Autowired를 통해 가져온 UserService 타입의 빈이었다.
- 이제는 완전히 고립되어 테스트만을 위해서 사용할 것이기에 스프링 컨테이너에서 가져올 필요 없이 직접 생성한다.
- 이후 UserServiceImpl 오브젝트의 메소드를 실행시키면 된다.
- 이후 update()를 이용해 몇 명의 사용자 정보를 DB에 수정하려고 했는지, 그 사용자들이 누구인지, 어떤 레벨로 변경됐는지를 확인하면 된다.
- 이렇게 고립된 테스트를 하면 테스트가 다른 의존 대상에 영향을 받을 경우를 대비해 복잡하게 준비할 필요가 없을 뿐만 아니라, 테스트 수행 성능도 크게 향상된다.
- 고립된 테스트를 만들려면 목 오브젝트 작성과 같은 약간의 수고가 더 필요할 지 모르겠지만, 그 보상은 충분히 기대할 만 하다.
### 6.2.3 단위 테스트와 통합 테스트
- upgradeLevels() 테스트처럼 '테스트 대상 클래스를 목 오브젝트 등의 테스트 대역을 이용해 의존 오브젝트나 외부의 리소스를 사용하지 않도록 고립시켜서 테스트하는 것'을 단위테스트, 반면에 두개 이상의, 성격이나 계층이 다른 오브젝트가 연동하도록 만들어 테스트하거나, 또는 외부의 DB 파일, 서비스 등의 리소스가 참여하는 테스트는 통합테스트 이다.

### 6.2.4 목 프레임워크
- 단위 테스트가 많은 장점이 있고 가장 우선시해야 할 테스트 방법인 건 사실이지만 작성이 번거롭다는 것이 문제이다.
- 특히 목 오브젝트를 만드는 일이 가장 큰 점이다. 다행히도, 이런 번거로운 목 오브젝트를 편리하게 작성하도록 도와주는 다양한 목 오브젝트 지원 프레임워크가 있다.

#### Mockito 프레임워크
```java
// 6-14. Mockito를 적용한 테스트 코드
@Test
public void mockUpgradeLevels() throws Exception{
    UserServiceImpl userServiceImpl = new UserServiceImpl();

    // 다이나믹한 목 오브젝트 생성과 메소드의 리턴 값 설정, 그리고 DI까지 세 줄이면 충분하다.
    UserDao mockUserDao = mock(UserDAO.class);
    when(mockUserDao.getAll()).thenReturn(this.users);
    userServiceImpl.setUserDao(mockUserDao);

    // 리턴 값이 없는 메소드를 가진 목 오브젝트는 더욱 간단하게 만들 수 있다.
    MailSender mockMailSender = mock(MailSender.class);
    userServiceImpl.setMailSender(mockMailSender);

    userServiceImpl.upgradeLevels();

    // 목 오브젝트가 제공하는 검증 기능을 통해서 어떤 메소드가 몇 번 호출 됐는지, 파라미터는 무엇인지 확인할 수 있다.
    verify(mockUserDao, times(2)).update(any(User.class));
    verify(mockUserDao).update(users.get(1));
    assertThat(users.get(1).getLevel(), is(Level.SILVER));
    verify(mockUserDao).update(users.get(3));
    assertThat(users.get(3).getLevel(), is(Level.GOLD));

    // Mockito 프레임워크의 툴 사용법으로써 배우지 않은 부분이니 그냥 보내질 메일 주소를 확인한다고 알고 있자.
    ArgumentCaptor<SimpleMailMessage> mailMessageArg = ArgumentCaptor.forClass(SimpleMailMessage.class);
    verify(mockMailSender, times(2)).send(mailMessageArg.capture());
    List<SimpleMailMessage> mailMessages = mailMessageArg.getAllValues();
    assertThat(mailMessages.get(0).getTo()[0], is(users.get(1).getEmail()));
    assertThat(mailMessages.get(1).getTo()[0], is(users.get(3).getEmail()));
}
```
- 다만, 제대로 사용하기 위해서는 Mockito 프레임워크의 툴을 알아두어야 하며 이를 알아둔다면 단위 테스트를 만들때 유용하게 사용될 수 있다는 것을 알아두자.

## 6.3 다이내믹 프록시와 팩토리 빈
### 6.3.1 프록시와 프록시 패턴, 데코레이터 패턴
> 트랜잭션 경계설정 코드를 비즈니스 로직 코드에서 분리해낼 때 적용했던 기법을 다시 검토해보자.

- 다음은 전략 패턴으로 트랜잭션 구현 내용을 분리해냈을 때(트랜잭션과 같은 부가적인 기능을 **위임**을 통해 외부로 분리했을 때)의 결과이다.
- 구체적인 구현 코드는 제거했을지라도 위임을 통해 기능을 사용하는 코드는 핵심 코드와 함께 남아있다.
<img width="544" alt="스크린샷 2024-07-01 오후 11 58 23" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/606a492c-1a95-4e44-9f87-84416e7222c2">

- 트랜잭션 기능은 비즈니스 로직과는 성격이 달라 아예 밖으로 분리할 수 있다. 부가기능 전부를 핵심 코드가 담긴 클래스에서 독립 시킬 수 있다. ➡️ UserServiceTx, 트랜잭션 관련 코드가 하나도 없게 된 UserServiceImpl
<img width="557" alt="스크린샷 2024-07-02 오후 7 53 58" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/3a84b78b-d78c-489b-9883-5bdf7c14a16a">
- 이렇게 **분리된 부가기능**을 담은 클래스는 **나머지 기능은 원래 핵심 기능을 가진 클래스로 위임해줘야 한다**
- 핵심 기능은 부가기능을 가진 클래스의 존재를 모른다. 따라서 부가기능이 핵심 기능을 사용하는 구조가 되는 것이다.
- 문제는 이렇게 구성했더라도 클라이언트가 핵심 기능을 가진 클래스를 직접 사용해버리면 부가기능이 적용될 기회가 없다는 점이다.
- 그래서 **부가기능은 마치 자신이 핵심 기능을 가진 클래스인 것처럼 꾸며서, 클라이언트가 자신을 거쳐서 핵심 기능을 사용하도록 만들어야 한다.**
- 그러기 위해서는 클라이언트는 인터페이스를 통해서만 핵심 기능을 사용하게 해야 하고, 부가기능 자신도 같은 인터페이스를 구현한 뒤에 자신이 그 사이에 끼어들어야 한다.

<img width="551" alt="스크린샷 2024-07-02 오후 8 00 56" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/5504fb75-cc5b-47f7-9bbe-a2b4a030717c">

- 부가기능 코드에서는 핵심 기능으로 요청을 위임해주는 과정에서 자신이 가진 부가기능을 적용해줄 수 있다. (ex. 비즈니스 로직 코드에 트랜잭션 기능 부여)
- 이렇게 마치 자신이 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는 것을 **대리자**, **대리인**과 같은 역할을 한다고 해서 **프록시**라고 부른다.
- 그리고 프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 오브젝트를 **타깃(target)** 또는 **실체(real object)**라고 부른다.
- 다음은 클라이언트가 프록시를 통해 타깃을 사용하는 구조를 보여준다.
  <img width="552" alt="스크린샷 2024-07-02 오후 8 05 18" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/edcdb0d3-4bf3-4719-8878-eb18fd3c3564">

- 프록시의 특징은 **타깃과 같은 인터페이스를 구현**했다는 것과 **프록시가 타깃을 제어할 수 있는 위치에 있다**는 것이다.
- 프록시는 사용 목적에 따라 두 가지로 구분할 수 있다.
  1. 클라이언트가 타깃에 접근하는 방법 제어
  2. 타깃에 부가적인 기능을 부여
- 두 가지 프록시를 두고 사용한다는 점은 동일하지만, 목적에 따라 디자인 패턴에서는 다른 패턴으로 구분한다. 

#### 데코레이터 패턴
- 타깃에 부가적인 기능을 런타임 시 다이내믹하게 부여해주기 위해 프록시를 사용하는 패턴
  - 다이내믹하게 기능을 부가한다 : 컴파일 시점, 즉 코드 상에는 어떤 방법과 순서로 프록시와 타깃에 연결되어 사용되는지 정해져 있지 않다.
- 실제 내용물은 동일하지만 부가적인 효과를 부여해줄 수 있기 때문에 데코레이터 패턴이라고 불린다.
- 프록시가 꼭 한 개로 제한되지 않는다.
- 프록시가 직접 타깃을 사용하도록 고정시킬 필요도 없다.
- 이를 위해 데코레이터 패턴에서는 같은 인터페이스를 구현한 타겟과 여러 개의 프록시를 사용할 수 있다.
- 프록시가 여러 개인만큼 순서를 정해서 단계적으로 위임하는 구조로 만들면 된다.

- 예를 들어 소스코드를 출력하는 기능을 가진 핵심 기능이 있다고 하자. 이 클래스에 데코레이터 개념을 부여해서 타깃과 같은 인터페이스를 구현하는 프록시를 만들 수 있다.
  
  <img width="667" alt="스크린샷 2024-07-03 오후 10 05 50" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/d5c67f4c-dd74-450d-956a-ad0af9ac31dc">
- 프록시로서 동작하는 데코레이터는 위임하는 대상에도 인터페이스로 접근하기 때문에 자신이 최종 타깃으로 위임하는지, 아니면 다음 단계의 데코레이터 프록시로 위임하는지 알지 못한다.
- 그래서 다음 위임 대상은 인터페이스로 선언하고 생성자나 수정자 메서드를 통해 **위임 대상을 외부에서 런타임 시에 주입 받을 수 있도록 만들어야 한다.**
- 데코레이터 패턴의 대표적인 예 : InputStream과 OutputStream
  ```java
  InputStream is = new BufferedInputream(new FileInputStream("a.txt"));
  ```
- UserService 인터페이스를 구현한 타깃인 UserServiceImpl에 트랜잭션 부가 기능을 제공해주는 UserServiceTx를 추가한 것도 데코레이터 패턴을 적용한 것이라 볼 수 있다. (수정자로 UserServiceTx에 위임할 타깃인 UserServiceImpl 주입)
- 인터페이스를 통한 데코레이터 정의와 런타임 시의 다이내믹한 구성 방법은 **스프링의 DI**를 이용하면 편하다. 데코레이터 빈의 프로퍼티로 같은 인퍼에시를 구현한 다른 데코레이터 또는 타깃 빈을 설정하면 된다.

```java
// 6-15. 데코레이터 패턴을 위한 DI 설정
<!-- 데코레이터 -->
<bean id="userService" class="springbook.user.service.UserServiceTx">
    <property name="transactionManager" ref="transactionManager" />
    <property name="userService" ref="userServiceImpl" />
</bean>

<!-- 타깃 -->
<bean id="userServiceImpl" class="springbook.user.service.UserServiceImpl">
    <property name="userDao" ref="userDao" />
    <property name="mailSender" ref="mailSender" />
</bean>
```

- 데코레이터 패턴은 인터페이스를 통해 위임하는 방식 ➡️ 어느 데코레이터에서 타깃으로 연결될지 코드 레벨에서는 미리 알 수 없음.
- 구성하기에 따라 여러 데코레이터 적용 가능
- 데코레이터 패턴은 타깃의 코드를 손대지 않고, 클라이언트가 호출하는 방법도 변경하지 않은 채로 새로운 기능을 추가할 때 유용한 방법이다.

#### 프록시 패턴
- 일반적응로 사용하는 **프록시**라는 용어는 클라이언트와 사용 대상 사이에서 대리 역할을 맡은 오브젝트를 두는 방법을 총칭한다면, 디자인 패턴에서 말하는 프록시 패턴은 프록시를 사용하는 방법 중에서 타깃에 대한 접근 방법을 제어하려는 목적을 가진 경우를 가리킨다.
- 프록시 패턴의 프록시는 타깃의 기능을 확장하거나 추가하지 않는다. 대신 클라이언트가 **타깃에 접근하는 방식**을 변경해준다.
- 클라이언트에게 타깃에 대한 레퍼런스를 넘겨야 하는데, 실제 타깃 오브젝트를 만드는 대신 프록시를 넘겨줄 수 있다.
    - 프록시의 메서드를 통해 타깃을 사용하려고 시도하면, 그 때 프록시가 타깃 오브젝트를 생성하고 요청을 위임해주는 식이다.
    - 만약 레퍼런스는 갖고 있지만 끝까지 사용하지 않거나, 많은 작업이 진행된 후에 사용되는 경우라면, 이렇게 프록시를 통해 **생성을 최대한 늦춤**으로써 얻는 장점이 많다.
- 또는 원격 오브젝트를 이용하는 경우에도 프록시를 사용하면 편하다.
  - RMI나 EJB, 또는 각종 리모팅 기술을 이용해 다른 서버에 존재하는 오브젝트를 사용해야 한다면, 원격 오브젝트에 대한 프록시를 만들어두고, 클라이언트느 마치 로컬에 존재한느 오브젝트를 쓰는 것처럼 프록시를 사용하게 할 수 있다.
  - 프록시는 요청을 받으면 네트워크를 통해 원격 오브젝트를 실행하고 결과를 받아 클라이언트에게 돌려준다.
- 또는 특별한 상황에서 타깃에 대한 접근 권한을 제어하기 위해 프록시 패턴을 사용할 수 있다.
  - 만약 수정 가능한 오브젝트가 있는데, 특정 레이어로 넘어가서는 읽기 전용으로만 동작하게 강제해야 한다고 하자. 이럴 때는 오브젝트의 프록시를 만들어서 사용할 수 있다.
  - 프록시의 특정 메서드를 사용하려고 하면 접근이 불가능하다고 예외를 발생시키면 된다.
  - ex. Collections의 unmodifiableCollection()을 통해 만들어지는 오브젝트
      - 파라미터로 전달된 Collection 오브젝트의 프록시를 만들어서, add()나 remove()와 같이 정보를 수정하는 메서드를 호출할 경우 UnSupportedOperationException 발생
- 구조적으로 프록시와 데코레이터 패턴은 유사하지만, 프록시는 코드에서 자신이 만들거나 접근할 타깃 클래스 정보를 알고 있는 경우가 많다.
- 물론 프록시 패턴이더라도 인터페이스를 통해 위임하도록 만들 수 있다. 인터페이스를 통해 다음 호출 대상으로 접근하게 하면 그 사이에 다른 프록시나 데코레이터가 계속 추가될 수 있기 때문이다.
- 다음은 접근 제어를 위한 프록시를 두는 프록시 패턴과 컬러, 페이징 기능을 추가하기 위한 프록시를 두는 데코레이터 패턴을 함께 적용한 예이다. 두 가지 모두 프록시 기본 원리대로 타깃과 같은 인터페이스의 기본 원리대로 타깃과 같은 인터페이스를 구현해두고 위임하는 방식으로 만들어져 있다.
  <img width="594" alt="스크린샷 2024-07-05 오후 12 32 46" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/52599015-843f-4a2f-9556-f9c795301d5d">

### 6.3.2 다이내믹 프록시
- 프록시는 유용한 방법이지만 그럼에도 불구하고 개발자는 타깃 코드를 직접 고치고 말지 번거롭게 프록시는 만들지 않겠다고 생각한다.
- 목 오브젝트를 만드는 불편함과 목 프레임워크를 사용해 편리하게 바꿨던 것처럼 프록시도 일일히 모든 인터페이스를 구현해서 클래스를 새로 정의하지 않고도 편리하게 만들어서 사용할 방법은 없을까?
- 물론 있다. 자바에는 `java.lang.reflect` 패키지 안에 프록시를 손쉽게 만들 수 있도록 지원해주는 클래스들이 있다. 원리는 목 프레임워크와 비슷하게 몇 가지 API를 이용해 프록시처럼 동작하는 오브젝트를 다이내믹하게 생성하는 것이다.

#### 프록시의 구성과 프록시 작성의 문베점
- 프록시는 다음의 두 가지 기능으로 구성된다.
  - 타깃과 같은 메서드를 구현하고 있다가 메서드가 호출되면 타깃 오브젝트로 **위임**한다.
  - 지정된 요청에 대해서는 **부가기능**을 수행한다.
- `UserServiceTx`는 기능 부가를 위한 프록시다. `UserServiceTx` 코드에서 이 두가지 기능을 구분해보자.

```java
// 6-16. UserServiceTx 프록시의 기능 구분
public class UserServiceTx implements UserService {
    UserService userService; // 타깃 오브젝트
    ...

    // 메서드 구현과 위임
    public void add(User user) {
        this.userService.add(user);
    }

    public void upgradeLevels() { // 메서드 구현
        // 부가 기능 수행
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            // 위임
            userService upgradeLevels(); 

            // 부가 기능 수행 이어서
            this.transactionManager.commit(status);
        } catch(RuntimeException e) {
            this.transactionManager.rollback(status);
            throw e;
            // 부가 기능 수행 끝
        }
    }
}
```


- 그렇다면 프록시를 만들기 번거로운 이유는 무엇일까?

  - 타깃 인터페이스를 구현하고 위임하는 코드를 **작성하기 번거롭다.** 부가기능이 필요 없는 메서드도 구현해서 타깃으로 위임하는 코드를 일일히 만들어줘야 한다. 또 타깃 인터페이스의 메서드가 추가되거나 변경될 때마다 함꼐 수정해줘야 한다는 부담도 있다.

  - 부가 기능 **코드가 중복**될 가능성이 많다.

- 트랜잭션 외의 프록시를 활용할만한 부가기능, 접근제어 기능은 일반적인 성격을 띤 것이 많아 다양한 타깃 클래스와 메서드에 중복돼서 나타날 가능성이 높다.
- 두 번째 문제는 코드를 분리해 어떻게든 해결할 수 있지만 첫 번째 문제는 간단해 보이지 않는다. 바로 이런 문제를 해결하는 데 유용한 것이 JDK의 **다이내믹 프록시**다.

#### 리플렉션
- 다이내믹 프록시는 리플렉션 기능을 이용해 프록시를 만들어준다.
  - 리플랙션 : 자바의 코드 자체를 추상화해서 접근하도록 만든 것
- 리플렉션 API 중에서 메서드에 대한 정의를 담은 Method라는 인터페이스를 이용해 메서드를 호출하는 방법을 알아보자.
  - String 클래스의 정보를 담은 Class 타입의 정보는 `String.class`라고 하면 가져올 수 있다.
  - 또는 스트링 오브젝트 name이 있으면 `name.getClass()`라고 해도 된다.
  - 그리고 이 클래스 정보에서 특정 이름을 가진 메서드 정보를 가져올 수 있다. String의 length() 메서드라면 다음과 같이 하면 된다.
    ```java
    Method lengthMethod = String.class.getMethod("length");
    ```
  - 스트링이 가진 메서드 중에서 "length"의 이름을 갖고 있고, 파라미터는 없는 메서드의 정보를 가져오는 것이다.
- java.lang.reflect.Method 인터페이스는 메서드에 대한 자세한 정보를 담고 있을 뿐만 아니라 이를 이용해 특정 오브젝트의 메서드를 실행시킬 수도 있다. Method 인터페이스에 정의된 `invoke()` 메서드를 사용하면 된다.
- `invoke()` 메서드는 메서드를 실행시킬 대상 오브젝트(`obj`)와 파라미터 목록(`args`)를 받아서 메서드를 호출한 뒤에 그 결과를 `Object` 타입으로 돌려준다.
```java
public Object invoke(Object obj, Object ... args)
```
- 이를 이용해 length() 메서드를 다음과 같이 실행할 수 있다.
```java
int length = lengthMethod.invoke(name); // int length = name.length();
```

```java
// 6-17. 리플렉션 학습 테스트
...
public class ReflectionTest {
    @Test
    public void invokeMethod() throws Exception {
        String name = "Spring";

        // length()
        assertThat(name.length(), is(6));

        Method lengthMethod = String.class.getMethod("length");
        assertThat((Integer)lengthMethod.invoke(name), is(6));

        // charAt()
        assertThat(name.length(0), is('S'));

        Method charAtMethod = String.class.getMethod("charAt", int.class);
        assertThat((Character)charAtMethod.invoke(name, 0), is('S'));
    }
}
```

#### 프록시 클래스
- 다이내믹 프록시를 이용한 프록시를 만들어보자.
- 프록시를 적용할 간단한 타깃 클래스와 인터페이스를 다음과 같이 정의한다.

```java
// 6-18. Hello 인터페이스
interface Hello {
    String sayHello(String name);
    String sayHi(String name);
    String sayThankYou(String name);
}
```

```java
// 6-19. 타깃 클래스
public class HelloTarget implements Hello {
    public String sayHello(String name) {
        return "Hello " + name;
    }

    public String sayHi(String name) {
        return "Hi " + name;
    }

    public String sayThankYou(String name) {
        return "Thank You " +name;
    }
}
```
- 이제 Hello 인터페이스를 통해 `HelloTarget` 오브젝트를 사용하는 **클라이언트 역할을 하는 간단한 테스트**를 다음과 같이 만든다.

```java
// 6-20. 클라이언트 역할의 테스트
@Test
public void simpleProxy() {
    Hello hello = new HelloTarget(); // 타깃은 인터페이스를 통해 접근하는 습관을 들이자.
    assertThat(hello.sayHello("Toby"), is("Hello Toby"));
    assertThat(hello.sayHi("Toby"), is("Hi Toby"));
    assertThat(hello.sayThankYou("Toby"), is("Thank You Toby"));
}
```

- 이제 `Hello` 인터페이스를 구현한 프록시를 만들어보자. 프록시에는 데코레이터 패턴을 적용해서 타깃인 `HelloTarget`에 부가기능을 추가하겠다. 프록시의 이름은 `HelloUppercase`이다.
- 추가하는 기능은 리턴하는 문자를 모두 대문자로 바꿔주는 것이다.
- `SimpleTarget`이라는 원본 클래스는 그대로 두고, 경우에 따라 대문자로 출력이 필요한 경우를 위해서 `HelloUppercase` 프록시를 통해 문자를 바꿔준다. `HelloUppercase` 프록시는 `Hello` 인터페이스를 구현하고, `Hello` 타입의 타깃 오브젝트를 받아서 저장해둔다.
- `Hello` 인터페이스는 구현 메서드에서는 타깃 오브젝트의 메서드를 호출한 뒤에 결과를 대문자로 바꿔주는 부가기능을 적용하고 리턴한다.
- 위임과 기능 부가라는 두 가지 프록시 기능을 모두 처리하는 전형적인 프록시 클래스
```java
// 6-21. 프록시 클래스
public class HelloUpperCase implements Hello {
    Hello hello; // 위임할 타깃 오브젝트. 여기서는 타깃 클래스의 오브젝트인 것은 알지만 다른 프록시를 추가할 수도 있으므로 인터페이스로 접근한다.

    public HelloUppercase(Hello hello) {
        this.hello = hello;
    }

    public String sayHello(String name) {
        return hello.sayHello(name).toUpperCase(); // 위임과 부가기능 적용
    }

    public String sayHi(String name) {
        return hello.sayHi(name).toUpperCase();
    }

    public String sayThankYou(String name) {
        return hello.sayThankYou(name).toUpperCase();
    }

}
```

- 다음과 같이 테스트 코드를 추가해서 프록시가 동작하는지 확인해보자.

```java
// 6-22. HelloUpperCase 프록시 테스트
Hello proxiedHello = new HelloUpperCase(new HelloTarget()); // 프록시를 통해 타깃 오브젝트에 접근하도록 구성한다.
asserThat(proxiedHello.sayHello("Toby"), is("HEELO TOBY"));
asserThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
asserThat(proxiedHello.sayThankYou("Toby"), is("THANK YOU TOBY"));
```
- 이 프록시는 프록시 적용의 일반적인 문제점 두 가지를 갖고 있다.
  - 인터페이스의 모든 메서드를 구현해 위임하도록 코드를 만들어야 한다.
  - 부가기능인 리턴 값을 대문자로 바꾸는 기능이 모든 메서드에 중복돼서 나타난다.

#### 다이내믹 프록시 적용
- 클래스로 만든 프록시인 HelloUpperCase를 **다이내믹 프록시**를 이용해 만들어보자.

<img width="552" alt="스크린샷 2024-07-07 오후 5 04 40" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/6629c78d-e53f-4559-a091-8e4c343ae20e">

- 다이내믹 프록시는 프록시 팩토리에 의해 런타임 시 다이내믹하게 만들어지는 오브젝트다.
- 다이내믹 프록시 오브젝트는 타깃의 인터페이스와 같은 타입으로 만들어진다.
- 클라이언트는 다이내믹 프록시 오브젝트를 타깃 인터페이스를 통해 사용할 수 있다.
- 프록시 팩토리에게 인터페이스 정보만 제공해주면 해당 인터페이스를 구현한 클래스의 오브젝트를 자동으로 만들어주기 때문에 편리하다.
- 그러나 프록시로서 필요한 부가기능 제공 코드는 직접 작성해야 한다. 부가기능은 프록시 오브젝트와 독립적으로 `InvocationHandler`를 구현한 오브젝트에 담는다.

```java
public Object invoke(Object proxy, Method method, Object[] args)
```
- `invoke()` 메서드는 리플렉션의 `Method` 인터페이스를 파라미터로 받는다. 메서드를 호출할 때 전달되는 파라미터도 `args`로 받는다.
- 다이내믹 프록시 오브젝트는 클라이언트의 모든 요청을 리플렉션 정보로 변환해서 `InvocationHandler` 구현 오브젝트의 `invoke()` 메서드로 넘기는 것이다.
- `InvokeHandler` 구현 오브젝트가 타깃 오브젝트 레퍼런스를 갖고 있다면 리플렉션을 이용해 간단한 위임 코드를 만들어낼 수 있다.
- `InvocationHandler` 인터페이스를 구현한 오브젝트를 제공해주면 다이내믹 프록시가 받는 요청을 `InvocationHandler`의 `invoke()` 메서드로 보내준다.
<img width="523" alt="스크린샷 2024-07-07 오후 5 29 04" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/22556acf-1deb-4b80-a6de-cdc6a5a8b180">

```java
// 6-23. InvocationHandler 구현 클래스
public class UppercaseHandler implements InvocationHandler {
    Hello target;

    // 다이내믹 프록시로부터 전달받은 요청을 다시 타깃 오브젝트에 위임해야하기 때문에 타깃 오브젝트를 주입받아 둔다.
    public UppercaseHandler(Hello target) {
        this.target = target;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String ret = (String)method.invoke(target, args); // 타깃으로 위임, 인터페이스의 메서드 호출에 모두 적용한다.
        return ret.toUpperCase(); // 부가기능 제공
    }
}
```

- 다이내믹 프록시를 통해 요청이 전달되면 리플렉션 API를 이용해 타깃 오브젝트의 메서드를 호출한다.
- Hello 인터페이스의 모든 메서드는 결과가 String 타입이므로 메서드 호출의 결과를 String 타입으로 변환해도 안전하다.
- 이제 `InvocationHandler`를 사용하고 Hello 인터페이스를 구현하는 프록시를 만들어보자.
- 다이내믹 프록시의 생성은 Proxy 클래스의 newProxyInstance() 스태틱 팩토리 메서드를 이용하면 된다.

```java
// 6-24. 프록시 생성
Hello proxiedHello = (Hello)Proxy.newProxyInstance( // 생성된 다이내믹 프록시 오브젝트는 Hello 인터페이스를 구현하고 있으므로 Hello 타입으로 캐스팅해도 안전하다.
                getClass.getClassLoader(), // 동적으로 생성되는 다이내믹 프록시 클래스의 로딩에 사용할 클래스로더 
                new Class[] { Hello.class }, // 구현할 인터페이스
                new UpperCaseHandler(new HelloTarget())); // 부가기능과 위임 코드를 담은 invocationHandler
```

- 그런데 원래 만들었던 HelloUpperCase 프록시 클래스보다 그다지 코드 양이 줄어들지 않은 것 같고, 코드 작성도 까다로워진 것 같다.
- 다이내믹 프록시를 적용했을 때 장점은 있는 걸까?

#### 다이내믹 프록시의 확장
- Hello 인터페이스 메서드가 늘어나도 UppercaseHandler와 다이내믹 프록시를 생성해서 사용하는 코드는 전혀 손댈 게 없다.
- UpperCaseHandler에서는 모든 메서드의 리턴 타입이 스트링이라 가정한다. 그런데 스트링 외의 리턴 타입을 갖게 되는 메서드가 추가되면 어떨까?
  - Method를 이용한 타깃 오브젝트의 메서드 호출 후 리턴 타입을 확인해서 스트링인 경우만 대문자로 바꿔주고 나머지는 그대로 넘겨주는 방식으로 수정하는 것이 좋겠다.
- InvocationHandler 방식의 또 한 가지 장점은 타깃의 종류와 상관없이 적용이 가능하다는 점이다.
- 어떤 종류의 인터페이스를 구현한 타깃이든 상관없이 재사용할 수 있고, 메서드의 리턴 타입이 스트링인 경우만 대문자로 결과를 바꿔주도록 UpperCaseHandler를 만들 수 있다.
```java
// 6-25. 확장된 UpperCaseHandler
 public class UppercaseHandler implements InvocationHandler {
    // 어떤 종류의 인터페이스를 구현한 타깃에도 적용 가능하도록 Object 타입으로 수정
    Object target;
    private UppercaseHandler(Object target) {
        this.target = target;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 호출한 메서드의 리턴 타입이 String인 경우만 대문자 변경 기능을 적용하도록 수정
        Object ret = method.invoke(target, args);
        if (ret instanceof String) {
            return ((String ret).toUpperCase();
        }
        else {
            return ret;
        }
    }
}
```
- 리턴 타입 뿐 아니라 메서드의 이름도 조건으로 걸 수 있다.
- 메서드의 이름이 say로 시작하는 경우에만 대문자로 바꾸는 기능을 적용하고 싶다면 다음과 같이 Method 파라미터에서 메서드 이름을 가져와 확인하는 방법을 사용하면 된다.

```java
// 메서드를 선별해서 부가기능을 적용하는 invoke()

public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Object ret = method.invoke(target, args);
    if (ret instanceof String && method.getName().startsWith("say")) { // 리턴 타입과 메서드 이름이 일치하는 경우에만 부가기능 적용
        return ((String ret).toUpperCase();
    }
    else {
        return ret; // 조건이 일치하지 않으면 타깃 오브젝트의 호출결과를 그대로 리턴
    }
}
```

### 6.3.3 다이내믹 프록시를 이용한 트랜잭션 부가기능
- UserServiceTx를 다이내믹 프록시 방식으로 변경해보자.

#### 트랜잭션 InvocationHandler
```java
// 6-27. 다이내믹 프록시를 위한 트랜잭션 부가기능
public class TransactionHandler implements InvocatioinHandler {
    private Object target; // 부가기능을 제공할 타깃 오브젝트. 어떤 타입의 오브젝트에도 적용 가능하다.
    private PlatformTransactionManager transactionManager; // 트랜잭션 기능을 제공하는 데 필요한 트랜잭션 매니저
    private String pattern; // 트랜잭션에 적용할 메서드 이름 패턴

    public void setTarget(Object target) {
        this.target = target;
    }

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        ...
    }

    public void setPattern(String pattern) {
        ...
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 트랜잭션 적용 대상 메서드를 선별해서 트랜잭션 경계설정 기능을 부여해준다.
        if (method.getName().startsWith(pattern)) {
            return invokeTransation(method, args);
        } else {
            return method.invoke(target, args);
        }
    }

    private Object invokeTransation(Method method, Object[] args) throws Throwable {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        
        try {
            // 트랜잭션을 시작하고 타깃 오브젝트의 메서드를 호출한다. 예외가 발생하지 않았다면 커밋한다.
            Object ret = method.invoke(target, args);
            this.transactionManager.commit(status);
            return ret;
        } catch (InvocationTargetException e) { // 예외가 발생하면 트랜잭션을 롤백한다.
            this.transactionManager.rollback(status);
            throw e.getTargetException;
        }
    }
}
```
- 트랜잭션을 적용하면서 타깃 오브젝틩 메서드를 호출하는 것은 UserServiceTx에서와 동일하다.
- 한 가지 차이점은 롤백을 적용하기 위한 예외는 RunTimeException 대신에 InvocationTargetException을 잡도록 해야 한다는 점이다.
  - 리플렉션 메서드인 Method.invoke()를 이용해 타깃 오브젝트의 메서드를 호출할 때는 타깃 오브젝트에서 발생하는 예외가 InvocationTargetException으로 한 번 포장돼서 전달된다.
  - 따라서 일단 InvocationTargetException으로 받은 후 getTargetException() 메서드로 중첩되어 있는 예외를 가져와야 한다.

#### TransactionHandler와 다이내믹 프록시를 이용하는 테스트
```java
// 6-28. 다이내믹 프록시를 이용한 트랜잭션 테스트
@Test
public void upgradeAllOrNothing() throws Exception {
    ...
    // 트랜잭션 헨들러가 필요한 정보와 오브젝트를 DI 해준다.
    TransactionHandler txHandler = new TransactionHander();
    txHandler.setTarget(testUserService);
    txHandler.setTransactionManager(transactionManager);
    txHandler.setPattern("upgradeLevels");

    // UserService 인터페이스 타입의 다이내믹 프록시 생성
    UserService txUserService = (UserService)Proxy.newProxyInstance(getClass.getClassLoader(), new Class[] { UserService.class }, txHandler);
    ...
}
```

### 6.3.4 다이내믹 프록시를 위한 팩토리 빈
- 문제 : DI의 대상이 되는 다이내믹 프록시 오브젝트는 일반적인 스프링 빈으로는 등록할 방법이 없다.
    - 사전에 프록시 오브젝트 클래스 정보를 미리 알아내서 스프링의 빈에 정의할 방법이 없다.
- 다이내믹 프록시는 Proxy 클래스의 newProxyInstance()라는 스태틱 팩토리 메서드를 통해서만 만들 수 있다.


#### 팩토리 빈
- 스프링은 빈을 만들 수 있는 다른 여러가지 방법을 제공한다. 대표적으로 팩토리 빈을 이용한 빈 생성 방법을 들 수 있다.
- 팩토리 빈이란 스프링을 대신해서 오브젝트의 생성로직을 담당하도록 만들어진 특별한 빈을 말한다.
- 팩토리 빈을 만드는 방법에는 여러가지가 있는데, 가장 간단한 방법은 스프링의 FactoryBean이라는 인터페이스를 구현하는 것이다.

```java
// 6-29. FactoryBean 인터페이스
public interface FactoryBean<T> {
    T getObject() throws Exception; // 빈 오브젝트를 생성해서 돌려준다.
    Class<? extends T> getObjectTyoe(); // 생성되는 오브젝트 타입을 알려준다.
    boolean inSingleton(); // getObject()가 돌려주는 오브젝트가 항상 같은 싱글톤 오브젝트인지 알려준다.
}
```
- FactoryBean 인터페이스를 구현한 클래스를 스프링의 빈으로 등록하면 팩토리 빈으로 동작한다.
- 팩토리 빈의 동작 원리를 확인할 수 있도록 만들어진 학습 테스트를 살펴보자.
- 리스트 6-30의 Message 클래스는 생성자를 통해 오브젝트를 만들 수 없다. 오브젝트를 만들려면 반드시 스태틱 메서드를 사용해야 한다. 즉 다음과 같은 방식으로 사용하면 안된다.
  ```xml
  <bean id="m" class="springbook.learningtest.spring.factorybean.Message">
  ...
  ```

```java
public class Message {
    String text;

    // 생성자가 private로 선언되어 있어 외부에서 생성자를 통해 오브젝트를 만들 수 없다.
    private Message(String text) {
        this.text = text;
    }

    public String getText() {
        return text;
    }

    // 생성자 대신 사용할 수 있는 스태틱 팩토리 메서드를 제공한다.
    public static Message newMessage(String text) {
        return new Message(text);
    }
}
```

- 생성자를 private로 만들었다는 것은 스태틱 메서드를 통해 오브젝트가 만들어져야 하는 중요한 이유가 있기 때문이므로 이를 무시하고 오브젝트를 강제로 생성하면 위험하다.
- Message 클래스의 오브젝트를 생성해주는 팩토리 빈 클래스를 만들어보자.
```java
// 6-31. Message의 팩토리 빈 클래스
public class MessageFactoryBean implements FactoryBean<Message> {
    // 오브젝트를 생성할 때 필요한 정보를 팩토리 빈의 프로퍼티로 설정해서 대신 DI 받을 수 있게 한다. 주입된 정보는 오브젝트 생성 중에 사용된다.
    String text;

    public void setText(String text) {
        this.text = text;
    }

    // 실제 빈으로 사용될 오브젝트를 직접 생성한다. 코드를 이용하기 때문에 복잡한 방식의 오브젝트 생성과 초기화 작업도 가능하다.
    public Message getObject() throws Exception {
        return Message.newMessage(this.text);
    }


    public Class<? extends Message> getObjectType() {
        return Message.class;
    }

    // 이 팩토리 빈은 매번 요청할 때마다 새로운 오브젝트를 만들므로 false로 설정한다.
    // 이것은 팩토리 빈의 동작 방식에 관한 설정이고 만들어진 빈 오브젝트는 싱글톤으로 스프링이 관리해줄 수 있다.
    public boolean isSingleton() {
        return false;
    }
}
```

#### 팩토리 빈의 설정 방법

``` xml
// 6-32. 팩토리 빈 설정
// FactoryBeanText-context.xml이라는 이름으로 저장
<bean id="message" class="springbook.learningtest.spring.factorybean.MessageFactoryBean">
    <property name="text" value="Factory Bean" />
</bean>
```
- 여타 빈 설정과 다른 점은 message 빈 오브젝트의 타입이 class 애트리뷰트에 정의된 MessageFactoryBean이 아니라 Message 타입이라는 것이다.
- Message 빈의 타입은 MessageFactoryBean의 getObjectType() 메서드가 돌려주는 타입으로 결정된다.
- 또, getObject() 메서드가 생성해주는 오브젝트가 message 빈의 오브젝트가 된다.

```java
// 6-33. 팩토리 빈 테스트
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration // 설정 파일 이름을 지정하지 않으면 클래스 이름 + "-context.xml"이 디폴트로 사용된다.
public class FactoryBeanTest {
    @Autowired
    ApplicationContext context;

    @Test
    public void getMessageFromFactoryBean() {
        Object message = context.getBean("message");
        asserThat(message, is(Message.class)); // 타입 확인
        asserThat((Message)message.getText(), is("Factory Bean")); // 설정과 기능 확인
    }
    ...
}

- 드물지만 팩토리 빈이 만들어주는 빈 오브젝트가 아니라 팩토리 빈 자체를 가져오고 싶은 경우도 있다.
- 이럴 때는 &을 빈 이름 앞에 붙여주면 된다.
```java
// 6-34. 팩토리 빈을 가져오는 기능 테스트
@Test
public void getFactoryBean() throws Exception {
    Object factory = context.getBean("&message");
    asserThat(factory, is(MessageFactoryBean.class)); 
}
```

#### 다이내믹 프록시를 만들어주는 팩토리 빈
- Proxy의 newProxyInstance() 메서드를 통해서만 생성이 가능한 다이내믹 프록시 오브젝트는 일반적인 방법으로는 스프링의 빈으로 등록할 수 없다.
- 대신 팩토리 빈을 사용하면 다이내믹 프록시 오브젝트를 스프링의 빈으로 만들어줄 수 있다.
- 팩토리 빈의 getObject() 메서드에 다이내믹 프록시 오브젝트를 만들어주는 코드를 넣으면 되기 때문이다.

<img width="542" alt="스크린샷 2024-07-07 오후 7 05 52" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/6692f754-b916-46b5-845e-bc7816cc5098">

- 스프링 빈에는 팩토리 빈과 UserServiceImpl만 빈으로 등록한다.
- 팩토리 빈은 다이내믹 프록시가 위임할 타깃 오브젝트인 UserServiceImpl에 대한 레퍼런스를 프로퍼티를 통해 DI 받아둬야 한다.
- 다이내믹 프록시를 직접 만들어서 UserService에 적용해봤던 upgradeAllOrNothing() 테스트의 코드를 팩토리 빈을 만들어서 getObject() 안에 넣어주기만 하면 된다.

#### 트랜잭션 프록시 팩토리 빈
```java
// 6-35. 트랜잭션 프록시 팩토리 빈
...
public class TxProxyFactoryBean implements FactoryBean<Object> {
    Object target;
    PlatformTransactionManager transactionManager;
    String pattern;
    Class?> serviceInterface; // 다이내믹 프록시를 생성할 때 필요하다. UserService 외의 인터페이스를 가진 타깃에도 적용할 수 있다.

    public void setTarget(Object target) {
        this.target = target;
    }

    public void setTransactionManager (PlatformTransationManager transactionManaer) {
            this.transactionManager = transactionManager;
    }

    public void setPattern(String pattern) {
        ...
    }

    public void setServiceInterface(Class?> serviceInterface) {
        ...
    }

    // FactoryBean 인터페이스 구현 메서드
    public Object getObject() throws Exception { // DI 받은 정보를 이용해서 TransactionHandler를 사용하는 다이내믹 프록시를 생성한다.
        TransactionHandler txHandler = new TransactionHander();
        txHandler.setTarget(testUserService);
        txHandler.setTransactionManager(transactionManager);
        txHandler.setPattern("upgradeLevels");

        return Proxy.newProxyInstance(
            getClass().getClassLoader(), new Class[] { serviceInterface }, txHandler);
    }
    
    public Class<?> getObjectType() {
        return serviceInterface; // 팩토리 빈이 생성하는 오브젝트의 타입은 DI 받은 인터페이스 타입에 따라 달라진다. 따라서 다양한 타입의 프록시 오브젝트 생성에 재사용할 수 있다.
    }

    public booelean isSingleton() {
        return false; // 싱글톤 빈이 아니라는 뜻이 아니라 getObject()가 매번 같은 오브젝트를 리턴하지 않는다는 의미다.
    }
}
```

- 아래와 같이 UserServiceTx 빈 설정을 대신해서 userService라는 이름으로 TxProxyFactoryBean 팩토리 빈을 등록한다. UserServiceTx 클래스는 이제 더 이상 필요 없으니 제거해도 상관 없다.
```xml
// 6-36. UserService에 대한 트랜잭션 프록시 팩토리 빈
<bean id="userService" class="springbook.user.service.TxProxyBean">
    <property name="target" ref="userServiceImpl" />
    <property name="transactionManager" ref="transactionManager" />
    <property name="pattern" value="upgradeLevels" />
    <property name="serviceInterface" value="springbook.user.service.UserService" />
</bean>
```

- serviceInterface는 Class 타입이다. Class 타입은 value를 사용해 클래스 또는 인터페이스의 이름을 넣어주면 된다.

#### 트랜잭션 프록시 빈 테스트
```java
// 6-27. 트랜잭션 프록시 팩토리 빈을 적용한 테스트
public class UserServiceTest {
    ...
    @Autowired ApplicationContext context; // 팩토리 빈을 가져오려면 애플리케이션 컨텍스트가 필요하다. 
    @Test
    @DirtiesContext // 다이내믹 프록시 팩토리 빈을 직접 만들어 사용할 때는 없앴다가 다시 등장한 컨텍스트 무효화 애너테이션
    public void upgradeAllOrNothing() throws Exception {
        TestUserService testUserService = new TestUserService(users.get(3).getId());
        testUserService.setUserDao(userDao);
        testUserService.setMailSender(mailSender);

        // 팩토리 빈 자체를 가져와야 하므로 빈 이름에 & 를 반드시 넣어야 한다.
        TxProxyFactoryBean txProxyFactoryBean = 
            context.getBean("&userService", TxProxyFactoryBean.class); // 테스트용 타깃 주입
        txProxyFactoryBean.setTarget(testUserService);
        UserService txUserService = (UserService) txProxyFactoryBean.getObject(); // 변경된 타깃 설정을 이용해서 트랜잭션 다이내믹 프록시 오브젝트를 다시 생성한다.
                 
        userDao.deleteAll();			  
        for(User user : users) userDao.add(user);
        
        try {
            txUserService.upgradeLevels();   
            fail("TestUserServiceException expected"); 
        }
        catch(TestUserServiceException e) { 
        }
        
        checkLevelUpgraded(users.get(1), false);
    }
}
```

### 6.3.5 프록시 팩토리 빈 방식의 장점과 한계
- 한 번 팩토리 빈을 만들어두면 타깃의 타입과 상관없이 재사용 가능

#### 프록시 팩토리 빈의 재사용
- 타깃 오브젝트에 맞는 프로퍼티 정보를 설정해서 빈으로 등록해주기만 하면 코드 수정 없이 다양한 클래스에 팩토리 빈 적용 가능
  - 하나 이상의 팩토리 빈을 동시에 빈으로 등록해도 상관 없음
- UserService 외에 트랜잭션 경계설정 기능을 부여해줄 필요가 있는 클래스가 있다고 하자. 인터페이스는 CoreService라고 하고, **인터페이스에 정의된 수십 여개의 메서드에 트랜잭션을 모두 적용**해야 한다.
```xml
// 6-38. 트랜잭션 서비스 빈 설정
<bean id="coreService" class="complex.module.CoreServiceImpl">
    <property name="coreDao" ref="coreDao" />
</bean>
```
- coreService 빈에 트랜잭션 기능이 필요해지면 TxProxyFactoryBean을 그대로 적용해주면 된다.
- 일단 기존의 CoreService라는 이름으로 등록했던 빈의 아이디를 다음과 같이 변경한다.
```xml
// 6-39. 아이디를 변경한 CoreService 빈
<bean id="coreServiceTarget" class="complex.module.CoreServiceImpl">
    <property name="coreDao" ref="coreDao" />
</bean>
```
- 이제 coreService 빈은 TxProxyFactoryBean을 이용해 다음과 같이 등록해준다.
```xml
// 6-30. CoreService에 대한 트랜잭션 프록시 팩토리 빈
<bean id="coreService" class="complex.module.TxProxyFactoryBean">
    <property name="target" ref="coreServiceTarget" />
    <property name="transactionManger" ref="transactionManager" />
    <property name="pattern" value="" />
    <property name="serviceInterface" value="complex.module.CoreService" />
</bean>
```
- CoreService 인터페이스에 정의된 모든 메서드에 트랜잭션 기능을 적용하려면 pattern 값을 빈 문자열로 설정해주면 된다.

- 다음 그림은 설정을 변경하기 전과 후의 오브젝트 관계를 나타낸다. 간단히 프록시 팩토리 빈의 설정을 추가해주고 나서는 CoreServiceImpl의 모든 메서드에 트랜잭션 기능이 적용됐다.
  
  <img width="561" alt="스크린샷 2024-07-13 오후 2 12 05" src="https://github.com/user-attachments/assets/928d90cc-6309-48dd-9c9a-671cb2858762">

#### 프록시 팩토리 빈 방식의 장점
- 다이내믹 프록시 장점
    - 타깃 인터페이스를 구현하는 클래스를 일일히 만드는 번거로움 제거
    - 부가기능 중복 문제 해결 : 하나의 헨들러 메서드를 구현하는 것만으로도 수많은 메서드에 부가기능을 부여할 수 있으므로
- 프록시에 팩토리 빈을 이용한 DI까지 더해주면 번거로운 다이내믹 프록시 생성 코드도 제거 가능

#### 프록시 팩토리 빈의 한계
- 프록시를 통해 타깃에 부가기능을 제공하는 것은 메소드 단위로 일어나는 일이다.
- 하나의 클래스 안에 존재하는 여러 개의 메소드에 부가기능을 한 번에 제공하는 건 어렵지 않게 가능했다.
- 하지만 한 번에 여러 개의 클래스에 공통적인 부가기능을 제공하는 일은 지금까지 살펴본 방법으로는 불가능하다.
- 하나의 타깃에 여러 개의 부가기능을 적용하려고 할 때도 문제다. 프록시 팩토리 빈 설정이 부가기능의 개수만큼 따라 붙어야 한다.

## 6.4 스프링의 프록시 팩토리 빈

### 6.4.1 ProxyFactoryBean
- 자바에는 JDK에서 제공하는 다이내믹 프록시 외에도 편리하게 프록시를 만들 수 있도록 지원해주는 다양한 기술이 존재한다.
- 따라서 스프링은 **일관된 방법으로 프록시를 만들 수 있게 도와주는 추상 레이어**를 제공한다.
- 생성된 프록시는 스프링의 빈으로 등록돼야 한다.
- 스프링은 프록시 오브젝트를 생성해주는 기술을 추상화한 팩토리 빈을 제공해준다.

- 스프링의 `ProxyFactoryBean`은 **프록시를 생성해서 빈 오브젝트로 등록하게 해주는 팩토리 빈**이다.
- `ProxyFactoryBean`은 순수하게 프록시를 생성하는 작업만을 담당하고 프록시를 통해 제공해줄 부가기능은 별도의 빈에 둘 수 있다.
- `ProxyFactoryBean`이 생성하는 프록시에서 사용할 부가기능은 `MethodInterceptor` 인터페이스를 구현해서 만든다.
- MethodInterceptor는 InvocationHandler와 비슷하지만 한 가지 다른 점이 있다. InvocationHandler의 invoke() 메소드는 타깃 오브젝트에 대한 정보를 제공하지 않는다.
- 따라서 타깃은 **InvocationHandler를 구현한 클래스가 직접 알고 있어야 한다.**
- 반면에 MethodInterceptor의 `invoke()` 메소드는 ProxyFactoryBean으로부터 타깃 오브젝트에 대한 정보까지도 함께 제공받는다.
- 그 차이 덕분에 MethodInterceptor는 **타깃 오브젝트에 상관없이 독립적으로 만들어질 수 있다.** 따라서 MethodInterceptor 오브젝트는 타깃이 다른 여러 프록시에서 함께 사용할 수 있고, 싱글톤 빈으로 등록 가능하다.

```java
// 6-41. 스프링 ProxyFactoryBean을 이용한 다이내믹 프록시 테스트
public class DynamicProxyTest {
    @Test
    public void simpleProxy() {
        Hello proxiedHello = (Hello)Proxy.newProxyInstance(
            getClass().getClassLoader(),
            new Class[] { Hello.class },
            new UppercaseHandler(new HelloTarget()));
            ...
    }

    @Test
    public void proxyFactoryBean() {
        ProxyFactoryBean pfBean = new ProxyFactoryBean();
        pfBean.setTarget(new HelloTarget()); // 타깃 설정
        pfBean.addAdvice(new UppdercaseAdvice()); // 부가 기능을 담은 어드바이스를 추가한다. 여러 개를 추가할 수도 있다.

        Hello proxiedHello = (Hello) pfBean.getObject(); // FactoryBean이므로 getObject()로 생성된 프록시를 가져온다.

        assertThat(ProxiedHello.sayHello("Toby", is("HELLO TOBY"));
        ....
    }

    static class UppercaseAdvise implements MethodInterceptor {
        public Object invoke(MethodInterceptor invocation) throws Throwable {
            String ret = (String)invocation.proceed(); // 리플렉션의 Method와 달리 메서드 실행 시 타깃 오브젝트를 전달할 필요가 없다. Methodinvocation은 메서드 정보와 함께 타깃 오브젝트를 알고 있기 때문이다.
            return ret.toUpperCase(); // 부가 기능 적용
    }
}
```
#### 어드바이스 : 타깃이 필요 없는 순수한 부가기능
- MethodInvocation은 일종의 콜백 오브젝트로, `proceed()` 메소드를 실행하면 타깃 오브젝트의 메소드를 내부적으로 실행해주는 기능이 있다.
- ProxyFactoryBean은 작은 단위의 템플릿/콜백 구조를 응용해서 적용했기 때문에 템플릿 역할을 하는 MethodInvocation을 싱글톤으로 두고 공유할 수 있다.
- MethodInterceptor처럼 **타깃 오브젝트에 적용하는 부가기능을 담은 오브젝트**를 스프링에서는 **어드바이스(advice)**라고 부른다.
- ProxyFactoryBean은 기본적으로 JDK가 제공하는 다이내믹 프록시를 만들어준다.
- 경우에 따라서는 CGLib이라고 하는 오픈소스 바이트코드 생성 프레임워크를 이용해 프록시를 만들기도 한다.
- 어드바이스는 타깃 오브젝트에 종속되지 않는 순수한 부가기능을 담은 오브젝트라는 사실을 잘 기억해두자.

#### 포인트컷 : 부가기능 적용 대상 메서드 선정 방법
- MethodInterceptor 오브젝트는 여러 프록시가 공유해서 사용할 수 있다.
- 그러기 위해서 MethodInterceptor 오브젝트는 타깃 정보를 갖고 있지 않도록 만들었다.
- 그 덕분에 MethodInterceptor를 스프링의 싱글톤 빈으로 등록할 수 있었다.
- 그런데 여기에다 트랜잭션 적용 대상 메소드 이름 패턴을 넣어주는 것은 곤란하다. 트랜잭션 적용 메소드 패턴은 프록시마다 다를 수 있기 때문에 여러 프록시가 공유하는 MethodInterceptor에 특정 프록시에만 적용되는 패턴을 넣으면 문제가 된다.

<img width="535" alt="스크린샷 2024-07-14 오후 11 29 36" src="https://github.com/user-attachments/assets/100f429a-3c04-42cb-8c80-2137896195e4">

<img width="545" alt="스크린샷 2024-07-14 오후 11 29 49" src="https://github.com/user-attachments/assets/b8b44f94-9d30-450d-99a1-01c8bf287ec8">

- InvocationHandler는 타깃과 메소드 선정 알고리즘 코드에 의존하고 있지만, 스프링의 ProxyFactoryBean 방식은 두 가지 확장 기능인 부가기능(Advice)과 메소드 선정 알고리즘(Pointcut)을 활용하는 유연한 구조를 제공한다.
- 스프링은 부가기능을 제공하는 오브젝트를 **어드바이스**라고 부르고, 메소드 선정 알고리즘을 담은 오브젝트를 **포인트컷**이라고 부른다.
- 어드바이스와 포인트컷은 모두 프록시에 DI로 주입돼서 사용된다.
- 두 가지 모두 여러 프록시에서 공유가 가능하도록 만들어지기 때문에 스프링의 싱글톤 빈으로 등록이 가능하다.
- 프록시는 클라이언트로부터 요청을 받으면 먼저 포인트컷에게 부가기능을 부여할 메소드인지를 확인해달라고 요청한다.
- 포인트컷은 Pointcut 인터페이스를 구현해서 만들면 된다.
- 프록시는 포인트컷으로부터 부가기능을 적용할 대상 메소드인지 확인받으면, MethodInterceptor 타입의 어드바이스를 호출한다.
- 어드바이스는 JDK의 다이내믹 프록시의 InvocationHandler와 달리 직접 타깃을 호출하지 않는다.
- 어드바이스가 일종의 템플릿이 되고 타깃을 호출하는 기능을 갖고 있는 MethodInvocation 오브젝트가 콜백이 되는 것이다.
- 템플릿은 한 번 만들면 재사용이 가능하고 여러 빈이 공유해서 사용할 수 있듯이, 어드바이스도 독립적인 싱글톤 빈으로 등록하고 DI를 주입해서 여러 프록시가 사용하도록 만들 수 있다.
- 프록시로부터 어드바이스와 포인트컷을 독립시키고 DI를 사용하게 한 것은 전형적인 전략 패턴 구조다.


```java
// 6-42. 포인트컷까지 적용한 ProxyFactoryBean
@Test
public void pointcutAdvisor() {
    ProxyFactoryBean pfBean = new ProxyFactoryBean();
    pfBean.setTarget(new HelloTarget());
    
    // 메소드 이름을 비교해서 대상을 선정하는 알고리즘을 제공하는 포인트컷 생성
    NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
    // 이름 비교조건 설정. sayH로 시작하는 모든 메소드를 선택하게 한다.
    pointcut.setMappedName("syaH*");
    
    // 포인트컷과 어드바이스를 advisor로 묶어서 한 번에 추가
    pfBean.addAdvisor(new DefaultPointcutAdvisor(pointcut, new UppercaseAdvice()));
    
    Hello proxiedHello = (Hello) pfBean.getObject();
    
    assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
    assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
	assertThat(proxiedHello.sayThankYou("Toby"), is("Thank You Toby"));
}
```
- ProxyFactoryBean에는 여러 개의 어드바이스와 포인트컷이 추가될 수 있다.
    - 포인트컷과 어드바이스를 따로 등록하면 어떤 어드바이스(부가 기능)에 대해 어떤 포인트컷(메소드 선정)을 적용할지 애매해지기 때문이다.
- 그래서 이 둘을 Advisor 타입의 오브젝트에 담아서 조합을 만들어 등록하는 것이다. 여러 개의 어드바이스가 등록되더라도 각각 다른 포인트컷과 조합될 수 있기 때문에 각기 다른 메소드 선정 방식을 적용할 수 있다.
- 이렇게 **어드바이스와 포인트 컷을 묶은 오브젝트를 인터페이스** 이름을 따서 **어드바이저라**고 부른다.
  ```
  어드바이저 = 포인트컷(메소드 선정 알고리즘) + 어드바이스(부가기능)
  ```

### 6.4.2 ProxyFactoryBean 적용

#### TransactionAdvice

```java
// 6-43. 트랜잭션 어드바이스
public class TransactionAdvice implements MethodInterceptor {
  PlatformTransactionManager transactionManager;

  @Override
  public Object invoke(MethodInvocation invocation) throws Throwable { // 타깃을 호출하는 기능을 가진 콜백 오브젝트를 프록시로부터 받는다. 덕분에 어드바이스는 특정 타깃에 의존하지 않고 재사용이 가능하다.
      TransactionStatus status = this.transactionManager.getTransaction(
                                                                          new DefaultTransactionDefinition());
      try {
          Object ret = invocation.proceed(); // 콜백을 호출해서 타깃의 메서드를 실행한다. 타깃 메서드 호출 전후로 필요한 부가 기능을 넣을 수 있다. 경우에 따라서 타깃이 아예 호출되지 않게 하거나 재시도를 위한 반복적인 호출도 가능하다. 
          this.transactionManager.commit(status);
          return ret;

      } catch (RuntimeException e) { // JDK 다이내믹 프록시가 제공하는 Method와는 달리 스프링의 Methodinvocation을 통한 타깃 호출은 예외가 포장되지 않고 타깃에서 보낸 그대로 전달된다.
          this.transactionManager.rollback(status);
          throw e;
      }
  }
}
```

#### 스프링 XML 설정파일

```xml
// 6-44. 트랜잭션 어드바이스 빈 설정
<bean id="transactionAdvice" class="user.service.TransactionAdvice">
  <property name="transactionManager" ref="transactionManager" />
</bean>
```
```xml
// 6-45. 포인트컷 빈 설정
<bean id="transactionPointCut" class="org.springframework.aop.support.NameMatchMethodPointcut">
  <property name="mappedName" value="upgrade*" />
</bean>
```

```xml
// 6-46. 어드바이저 빈 설정
<bean id="transactionAdvisor" class="org.springframework.aop.support.DefaultPointcutAdvisor">
  <property name="advice" ref="transactionAdvice" />
  <property name="pointcut" ref="transactionPointCut" />
</bean>
```

```xml
// 6-47. ProxyFactoryBean 설정
<bean id="userService" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="target" ref="userServiceImpl" />
    <property name="interceptorNames">
        <list>
            <value>transactionAdvisor</value>
        </list>
    </property>
</bean>
```
- **interceptorNames** : 어드바이스와 어드바이저를 동시에 설정해줄 수 있는 프로퍼티. 리스트에 어드바이스나 어드바이저의 빈 아이디를 값으로 넣어주면 된다. 기존의 ref 애트리뷰트를 사용하는 DI와는 방식이 다름에 주의해야 한다.
- 한 개 이상의 `<value>` 태그를 넣을 수 있다.

#### 어드바이스와 포인트컷의 재사용
- ProxyFactoryBean은 스프링의 DI와 템플릿/콜백 패턴, 서비스 추상화 등의 기법이 모두 적용된 것이다.
- 그 덕분에 독립적이며 여러 프록시가 공유할 수 있는 어드바이스와 포인트컷으로 확장 기능을 분리할 수 있었다.
- 이제 UserService 외에 새로운 비즈니스 로직을 담은 서비스 클래스가 만들어져도 이미 만들어둔 TransactionAdvice를 그대로 재사용할 수 있다.
- 메소드의 선정을 위한 포인트컷이 필요하면 이름 패턴만 지정해서 ProxyFactoryBean에 등록해주면 된다.
- 트랜잭션을 적용할 메소드의 이름을 일관된 명명 규칙을 정해두면 하나의 포인트컷으로도 충분하다.

<img width="540" alt="image" src="https://github.com/user-attachments/assets/2a68d743-7135-40b8-9388-b5d91e86f657">

## 6.5 스프링 AOP
### 6.5.1 자동 프록시 생성
- 프록시 팩토리 빈 방식의 접근 방식의 한계 두 가지가 있었다.
  - 부가기능이 타깃 오브젝트마다 새로 만들어지는 문제 -> `ProxyFactoryBean`의 어드바이스를 통해 해결
  - **부가기능의 적용이 필요한 타깃 오브젝트마다 거의 비슷한 내용의 `ProxyFactoryBean` 빈 설정 정보를 추가해줘야 한다.**
   ➡️ 중복을 제거해주는 방법은 없을까?

#### 중복 문제의 접근 방법
- JDBC API를 사용하는 DAO 코드가 있었다. 메서드마다 JDBC try/catch/finally 블록 코드가 반복해서 나타났다. 이 코드는 **전략패턴**과 **DI**를 적용해 깔끔하게 해결했다.
- 그런데 이와는 다른 방법으로 반복되는 코드의 문제를 해결했던 것이 있다. 바로 반복적인 위임 코드가 필요한 프록시 클래스 코드다. 이는 위임과 부가기능 적용 여부 판단 부분은 코드 생성 기법을 이용하는 **다이내믹 프록시** 기술에 맡기고, 부가기능 코드는 별도로 만들어 **프록시 생성 팩토리에 DI로 제공**하는 방법으로 해결했다.
- 반복적인 프록시의 메서드 구현을 코드 자동생성 기법을 이용해 해결했다면, 반복적인 `ProxyFactoryBean` 설정 문제는 자동 등록 기법으로 해결할 수는 없을까? 또는 실제 빈 오브젝트가 되는 것은 `ProxyFactoryBean`을 통해 생성되는 프록시 그 자체이므로 프록시가 자동으로 빈으로 생성되게 할 수 없을까?

#### 빈 후처리기를 이용한 자동 프록시 생성기
- 스프링은 컨테이너로서 제공하는 기능 중에서 변하지 않는 핵심적인 부분 외에는 대부분 확장할 수 있도록 확장 포인트를 제공해준다. (OCP - 유연한 확장)
- BeanPostProcessor 인터페이스를 구현해서 만든 빈 후처리기는 이름 그대로 빈 오브젝트로 만들어지고 난 후에, 빈 오브젝트를다시 **가공**할 수 있게 해준다.
- 스프링이 제공하는 빈 후처리기 중의 하나인 `DefaultAdvisorAutoProxyCreator`를 살펴보자.
  - 어드바이저를 이용한 자동 프록시 생성기
- 빈 후처리기를 스프링에 적용하는 방법은 **빈 후처리기 자체를 빈으로 등록**하는 것이다.
- 스프링은 빈 후처리기가 빈으로 등록되어 있으면 빈 오브젝트가 생성될 때마다 빈 후처리기에 보내서 후처리 작업을 요청한다.
- 이를 잘 이용하면 스프링이 생성하는 빈 오브젝트의 일부를 프록시로 포장하고, 프록시를 빈으로 대신 등록할 수도 있다. 바로 이것이 자동 프록시 생성 빈 후처리기이다.
- 다음 그림은 빈 후처리기를 이용한 자동 프록시 생성 방법을 설명한다.
  
	<img width="546" alt="스크린샷 2024-07-16 오후 5 05 40" src="https://github.com/user-attachments/assets/ea8e903f-169d-4ddc-a0b6-79426903dd6d">

	- `DefaultAdvisorAutoProxyCreator` 빈 후처리기가 등록되어 있으면 스프링은 빈 오브젝트를 만들 때마다 후처리기에게 빈을 보낸다.
	 - `DefaultAdvisorAutoProxyCreator`으로 등록된 모든 어드바이저 내의 포인트 컷을 이용해 전달받은 빈이 프록시 적용 대상인지 확인한다.
	 - 적용 대상이면 그 때는 내장된 프록시 생성기에게 현재 빈에 대한 프록시를 만들게 하고, 만들어진 프록시에 어드바이저를 연결해준다.
	 - 빈 후처리기는 프록시가 생성되면 원래 컨테이너가 전달해준 빈 오브젝트 대신 프록시 오브젝트를 컨테이너에게 돌려준다.

- 적용할 빈을 선정하는 로직이 추가된 포인트컷이 담긴 어드바이저를 등록하고 빈 후처리기를 사용하면 일일히 `ProxyFactoryBean`을 등록하지 않아도 타깃 오브젝트에 자동으로 프록시가 적용되게 할 수 있다.

#### 확장된 포인트컷
- 지금까지 포인트컷이란, 타깃 오브젝트의 메서드 중 어떤 메서드에 부가기능을 적용할지 선정해주는 역할을 한다고 했다.
- 그런데 여기서는 갑자기 포인트컷이 등록된 빈 중에서 어떤 빈에 프록시를 적용할지를 선택한다는 식으로 설명하고 있다.
- 사실 포인트컷은 두 가지 기능을 모두 갖고 있다.
  - 포인트컷은 클래스 필터와 메서드 매처 두 가지를 돌려주는 메서드를 갖고 있다.
```java
public interface Pointcut {
	ClassFilter getClassFilter(); // 프록시를 적용할 클래스인지 확인해준다.
	MethodMatcher getMethodMatcher(); // 어드바이스를 적용할 메서드인지 확인
}
```

- 만약 Pointcut 선정 기능을 모두 적용한다면 먼저 프록시를 적용할 클래스인지 판단하고 나서, 적용 대상 클래스인 경우에는 어드바이스를 적용할 메서드인지 확인하는 식으로 동작한다.
- `ProxyFactoryBean`에서는 굳이 클래스 레벨의 필터는 필요 없었지만, 모든 빈에 대해 프록시 자동 적용 대상을 선별해야 하는 빈 후처리기인 `DefaultAdvisorAutoProxyCreator`는 클래스와 메서드 선정 알고리즘을 모두 갖고 있는 포인트컷이 필요하다.
- 정확히는 그런 포인트컷과 어드바이스가 결합되어 있는 어드바이저가 등록되어 있어야 한다.

#### 포인트컷 테스트
- 앞에서 사용한 NameMatchMethodPointcut은 클래스 필터 기능이 아예 없다. (사실 있긴 있지만 모든 클래스에 대해 무조건 OK 해버리는, 있으나 마나 한 필터)
- 이번엔 이 클래스를 확장해서 **클래스도 고를 수 있도록** 하겠다.
- 그리고 프록시 적용 후보 클래스를 여러 개 만들어두고 이 포인트컷을 적용한 ProxyFactoryBean으로 프록시를 만들도록 해서 과연 어드바이스가 적용되는지 아닌지를 확인하겠다.

```java
// 6-50. 확장 포인트컷 테스트
@Test
public void classNamePointcutAdvisor() {
	// 포인트컷 준비
	NameMatchMethodPoincut classMethodPointcut = new NameMatchMethodPointcut() {
		public ClassFilter getClassFilter() { // 익명 내부 클래스 방식으로 클래스를 정의한다.
			return new ClassFilter() {
				public boolean matches(Class<?> clazz) {
					return clazz.getSimpleName().startsWith("HelloT"); // 클래스 이름이 HelloT로 시작하는 것만 선정한다.
				}
			};
		}
	}
	classMethodPointcut.setMappedName("sayH*"); // sayH로 시작하는 메서드 이름을 가진 메서드만 선정한다.

	// 테스트
	checkAdviced(new HelloTarget(), classMethodPointcut, true); // HelloTarget은 적용 클래스다.

	class HelloWorld extends HelloTarget {};
	checkAdviced(new HelloWorld(), classMethodPointcut, false); // HelloWorld는 적용 클래스가 아니다.

	class HelloToby extends HelloTarget {};
	checkAdviced(new HelloToby(), classMethodPointcut, true); // HelloToby는 적용 클래스다.
}


private void checkAdviced(Object target, Pointcut pointcut, boolean adviced) { // advised : 적용 대상인가?
	ProxyFactoryBean pfBean = new ProxyFactoryBean();
	pfBean.setType(target);
	pfBean.addAdvisor(new DefaultPointcutAdvisor(pointcut, new UppercaseAdvice()));
	Hello proxiedHello = (Hello) pfBean.getObject();

	if(adviced) {
		// 메서드 선정 방식을 통해 어드바이스 적용
		// ==============================
		assertTrue(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
		asserTrue(proxiedHello.sayHi("Toby"), is("HI TOBY"));
		// ==============================
		asserTrue(proxiedHello.sayThankYou("Toby"), is("Thank You Toby"));
	}
	else {
		// 어드바이스 적용 대상 후보에서 아예 탈락
		// ==============================
		assertTrue(proxiedHello.sayHello("Toby"), is("Hello Toby"));
		asserTrue(proxiedHello.sayHi("Toby"), is("Hi Toby"));
		asserTrue(proxiedHello.sayThankYou("Toby"), is("Thank You Toby"));
		// ==============================
	}
}
```

- 포인트컷이 클래스 필터까지 동작해서 클래스를 걸러버리면 아무리 프록시를 적용했다고 해도 부가기능은 전혀 제공되지 안흔ㄴ다는 점에 주의해야 한다.
- 사실 클래스 필터에서 통과하지 못한 대상은 프록시를 만들 필요조차 없다.

### 6.5.2 DefaultAdvisorAutoProxyCreator의 적용
#### 클래스 필터를 적용한 포인트컷 작성
- 만들어야할 클래스는 하나 뿐. 메서드 이름만 비교하던 포인트컷인 NameMatchMethodPointcut을 **상속**해서 프로퍼티로 주어진 이름 패턴을 가지고 클래스 이름을 비교하는 ClassFilter 추가

```java
// 6-51. 클래스 필터가 포함된 포인트컷

...

public class NameMatchClassMethodPointcut extends NameMatchMethodPointcut {
    public void setMappedClassName(String mappedClassName) {
        this.setClassFilter(new SimpleClassFilter(mappedClassName)); // 모든 클래스를 다 허용하던 디폴트 클래스 필터를 프로퍼티로 받은 클래스 이름을 이용하여 필터를 만들어 덮어 씌운다.
    }

    static class SimpleClassFilter implements ClassFilter {
        private String mappedName;

        private SimpleClassFilter(String mappedName) {
            this.mappedName = mappedName;
        }
        @Override
        public boolean matches(Class<?> clazz) {
            return PatternMatchUtils.simpleMatch(mappedName, clazz.getSimpleName()); // 와일드 카드(*)가 들어간 문자열 비교를 지원하는 스프링의 유틸리티 메서드다. *name, name*, *name* 세 가지 방식을 모두 지원한다.
        }
    }
}
```

#### 어드바이저를 이용하는 자동 프록시 생성기 등록
- 적용할 자동 프록시 생성기인 DefaultAdvisorAutoProxyCreator는 등록된 빈 중에서 Advisor 인터페이스를 구현한 것을 모두 찾는다.
- 그리고 생성되는 모든 빈에 대해 어드바이저 포인트컷을 적용해보면서 프록시 적용 대상을 선정한다.
- 빈 클래스가 프록시 선정 대상이라면 프록시를 만들어 원래 빈 오브젝트와 바꿔치기한다.

- DefaultAdvisorAutoProxyCreator 등록은 다음 한 줄이면 충분하다.
```xml
<bean class+"org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator" />
```

#### 포인트컷 등록
- 다음과 같이 기존 포인트컷 설정을 삭제하고 새로 만든 클래스 필터 지원 포인트컷을 빈으로 등록한다.
- SimpleImpl로 이름이 끝나는 클래스와 upgrade로 시작하는 메서드를 설정해주는 포인트컷이다.
```xml
// 6-52. 포인트컷 빈
<bean id="transactionPointcut"
        class="springbook.service.NameMatchClassMethodPointcut">
	<property name="mappedClassName" value="*ServiceImpl"/> <!-- 클래스 이름 패턴 -->
	<property name="mappedName" value="upgrade*" /> <!-- 메소드 이름 패턴 -->
</bean>
```

#### 어드바이스와 어드바이저
- 이제는 `transactionAdvisor를` 명시적으로 DI하는 빈은 존재하지 않는다.
- 대신 어드바이저를 이용하는 자동 프록시 생성기인 `DefaultAdvisorAutoProxyCreator에` 의해 자동 수집되고, 프록시 대상 선정 과정에 참여하며, 자동 생성된 프록시에 다이내믹하게 DI 돼서 동작하는 어드바이저가 된다.


#### ProxyFactoryBean 제거와 서비스 빈의 원상복구
- userServiceImpl의 빈 아이디를 이제 userService로 되돌려 놓을 수 있다.
- 프록시 팩토리 빈은 제거해도 된다.

#### 자동 프록시 생성기를 사용하는 테스트
- 자동 프록시 생성기를 적용한 후에는 알아서 프록시가 생성되어 프록시 오브젝트만 남아있을 뿐이다.
- 지금까지는 어떻게든 설정파일에는 정상적인 경우의 빈 설정만을 두고 롤백을 일으키는 예외상황에 대한 테스트는 테스트 코드에서 빈을 가져와 수동 DI로 구성을 바꿔서 사용했다.
- 하지만 자동 프록시 생성기라는 스프링 컨테이너에 종속적인 기법을 사용했기 때문에 예외 상황을 위한 테스트 대상도 빈으로 등록해줄 필요가 있다.
- 기존에 만들어서 사용하던 강제 예외 발생용 `TestUserService` 클래스를 이제는 직접 빈으로 등록해보자. 그런데 두 가지 문제가 있다.
  - `TestUserService`가 `UserServiceTest` 클래스의 내부에 정의된 스태틱 클래스다.
  - 트랜잭션 어드바이스를 적용해주는 대상 클래스의 이름 패턴이 *ServiceImpl이라고 되어 있어서 `TestUserService`는 빈으로 등록해도 포인트컷이 프록시 적용 대상으로 선정해주지 않는다.
- `TestUserService`를 조금 수정하자.
```java
// 6-54. 수정한 테스트용 UserService 구현 클래스
static class TestUserServiceImpl extends UserServiceImpl { // 포인트컷 클래스 필터에 선정되도록 이름 변경. 이래서 처음부터 이름을 잘 지어야 한다.
	private String id = "madnite1"; // 테스트 픽스처의 user(3)의 id 값을 고정시켜버렸다.

	protected void upgradeLevel(User user) {
		if(user.getId().equals(this.id)) throw new TestUserServiceException();
		super.upgradeLevel(user);
	}
}
```

```xml
// 6-55. 테스트용 UserService의 등록
<bean id="testUserService" class="springbook.user.service.UserServiceTest$TestUserServiceImpl" parent="userService" />
<!-- 스태틱 멤버 클래스는 $로 지정한다. -->
<!-- 프로퍼티 정의를 포함해서 userService 빈의 설정을 상속 받는다. -->
```

> 특정 테스트 클래스에서만 사용되는 클래스는 스태틱 멤버 클래스로 정의하는 것이 편리하다.

```java
// 6-56. testUserService 빈을 사용하도록 수정된 테스트
public class UserServiceTest {
	@Autowired UserService userService;
	@Autowired UserService testUserService; // 같은 타입의 빈이 두 개 존재하기 때문에 필드 이름을 기준으로 주입될 빈이 결정됨.

	...

	@Test
	public void upgradeAllOrNothing(){
	    	userDao.deleteAll();
		for(User user : users) userDao.add(user);

		try {
			this.testUserService.upgradeLevels();
			fail("TestUserServiceException expeced");
		}
		catch(TestUserServiceException e) {
		}
	
	    	checkLevelUpgraded(users.get(1), false);
	}
}
```
- 예외 상황을 적용하기 위한 DI 작업이 제거되어 코드가 단순해짐.
- upgradeAllOrNothing() 테스트를 통해 자동 프록시 생성기가 userService 빈을 자동으로 트랜잭션 부가기능을 제공해주는 프록시로 대체했는지 확인해보자.

#### 자동생성 프록시 확인
- 최소한 두 가지는 확인해야 한다.

1. **트랜잭션이 필요한 빈에 트랜잭션 부가기능이 적용됐는가**

   - 앞에서 만든 upgradeAllOrNothing() 테스트를 통해 검증했다.

2. **아무 빈에나 트랜잭션 부가기능이 적용된 것은 아닌가**

   - 클래스 이름 패턴을 변경해 `testUserService` 빈에 트랜잭션이 적용되지 않게 해보자.
   - 또는 `getBean("userService")`로 가져온 오브젝트가 JDK의 Proxy 타입인지 확인해보자.
  

### 6.5.3 포인트컷 표현식을 이용한 포인트컷
- 스프링은 아주 간단하고 효과적인 방법으로 포인트컷의 클래스나 메서드를 선정하는 알고리즘을 작성할 수 있는 방법을 제공한다.
- **포인트컷 표현식** : 정규식이나 JSP의 EL과 비슷한 일종의 표현식 언어를 사용해 포인트컷을 작성할 수 있도록 하는 방법

#### 포인트컷 표현식
- `AspectJExpressionPointcut`
  - 포인트컷 표현식을 사용하려면 해당 클래스를 사용
  - AspectJ 프레임워크에서 제공하는 클래스

- `Pointcut` 인터페이스를 구현해야 하는 스프링의 포인트컷은 클래스 선정을 위한 **클래스 필터**와 메소드 선정을 위한 **메소드 매처** 두 가지를 각각 제공해야 한다.

#### 포인트컷 표현식 문법

<img width="609" alt="image" src="https://github.com/user-attachments/assets/a27ccd36-06ce-48b1-ba1e-be300b12fb20">

- 위와 같은 포인트컷 표현식을 통해 기존 클래스, 메서드 별로 따로 프로퍼티를 세팅했던 포인트컷을 하나의 프로퍼티 값으로 명시해줄 수 있다.

- [접근제한자 패턴] : public, private 등이 올 수있다. (생략 가능)
- 타입패턴: 리턴 값의 타입을 나타내는 패턴이다. (필수 항목)
- [타입패턴.]: 패키지와 타입의 이름을 포함한 클래스 타입 패턴이다. (생략 가능, *를 사용할 수 있고 ..를 사용하면 한번에 여러 패키지를 선택할 수 있다.)
- 이름패턴: 메소드 이름 패턴 (필수 항목)
- (타입패턴 | ”..”, ...) : 파라미터의 타입 패턴을 순서대로 넣을 수 있다. 와일드카드를 이용해 파라미터 개수에 상관없는 패턴을 만들 수 있다. (필수 항목)
- [throws 예외 패턴]: 예외 이름 패턴 (생략 가능)


- 주의해야 할 점
  - 표현식이 문자열이기 때문에 컴파일 시점에서는 오류 검증이 불가
  - 다양한 테스트를 미리 만들어서 검증된 표현식을 사용하는 것이 중요

#### 포인트컷 표현식을 이용하는 포인트컷 적용
- 포인트컷 표현식은 메소드의 시그니처를 비교하는 방식인 `execution()` 외에도 몇 가지 표현식 스타일을 갖고 있다.
- 대표적으로 스프링에서 사용될 때 빈의 이름으로 비교하는 `bean()`이 있다.
- 또, 특정 애노테이션이 타입, 메소드, 파라미터에 적용되어 있는 것을 보고 메소드를 선정하게 하는 포인트컷도 만들 수 있다.
- 애노테이션만 부여해놓고, 포인트컷을 통해 자동으로 선정해서, 부가기능을 제공하게 해주는 방식은 스프링 내에서도 애용되는 편리한 방법이다. 아래와 같이 쓰면 @Transaction이라는 애노테이션이 적용된 메소드를 선정하게 해준다.

	```java
	@annotation(org.springframework.transcation.annotation.Transactional)
	```
- 클래스 이름은 ServiceImpl로 끝나고 메소드 이름은 upgrade로 시작하는 모든 클래스에 적용되도록 하는 표현식을 만들고 이를 적용한 빈 설정은 다음과 같다.
```xml
// 6-65. 포인트컷 표현식을 사용한 빈 설정
<bean id="transactionPointcut" class="org.springframework.aop.aspectj.AspectExpressionPoinntcut">
	<property name="expression" value="execution(* *..*ServiceImpl.upgrade*(..))" />
</bean>
```
- 포인트컷 표현식을 사용하면 로직이 짧은 문자열이 담기기 때문에 클래스나 코드를 추가할 필요가 없어서 코드와 설정이 모두 단순해진다.
- 반면에 문자열로 된 표현식이므로 런타임 시점까지 문법 검증이나 기능 확인이 되지 않는다는 단점도 있다.
#### 타입 패턴과 클래스 이름 패턴
- 클래스 이름 패턴과 포인트컷 표현식에서 사용하는 타입 패턴은 중요한 차이점이 있다.
- TetstUserServiceImpl이라고 변경했던 테스트용 클래스의 이름은 다시 TestUserService라고 바꿔보자. 테스트를 실행 해보면 결과는 성공이다.
- 포인트 컷이 `execution(* *..*ServiceImpl.upgrade*(..))` 로 되어 있는데 어떻게 TestUserService 클래스로 등록된 빈이 선정 됐을까?
- 그 이유는 포인트컷 표현식의 클래스 이름에 적용되는 패턴은 클래스 이름 패턴이 아니라 타입 패턴이기 때문이다.
- TestUserService의 클래스 이름은 TestUserService 이지만, 타입을 따져보면 TestUserService 클래스 이고, 슈퍼클래스인 UserServiceImpl, 구현 인터페이스인 UserService 세 가지가 모두 적용된다.

### 6.5.4 AOP란 무엇인가?
비즈니스 로직을 담은 과정을 정리해보자.
- 트랜잭션 서비스 추상화
  - 문제 : 트랜잭션 경계설정 코드를 비즈니스 로직을 담은 코드에 넣으면서 특정 트랜잭션 기술에 종속되는 코드가 되어버림
  - 트랜잭션 적용이라는 추상적인 작업 내용은 유지한 채로 구체적인 구현 방법을 바꿀 수 있도록 서비스 추상화 기법 적용
  - 결국 인터페이스와 DI를 통해 **무엇을 하는지**는 남기고, 그것을 **어떻게 하는지**를 분리
- **프록시와 데코레이터 패턴**
  - 문제 : 여전히 비즈니스 로직 코드에는 트랜잭션을 적용하고 있었음.
  - 트랜잭션을 처리하는 코드는 일종의 데코레이터에 담겨서, 클라이언트와 비즈니스 로직을 담은 타깃 클래스 사이에 존재하도록 만듦
- **다이내믹 프록시와 프록시 팩토리 빈**
  - 문제 : 트랜잭션 기능을 부여하지 않아도 되는 메서드조차 프록시로서의 위임 기능이 필요하기 때문에 일일히 구현해줬어야 했음
  - 그래서 프록시 클래스 없이도 프록시 오브젝트를 런타임 시에 만들어주는 JDK 다이내믹 프록시 기술 적용
  - JDK 다이내믹 프록시와 같은 프록시 기술을 추상화한 스프링의 프록시 팩토리 빈을 이용해 다이내믹 프록시 생성 방법에 DI 도입
- **자동 프록시 생성 방법과 포인트컷**
  - 문제 : 트랜잭션 적용 대상이 되는 빈마다 일일히 프록시 팩토리 빈을 설정해줘야 했음.
  - 스프링 컨테이너의 빈 생성 후처리 기법을 활용해 컨테이너 초기화 시점에서 자동으로 프록시를 만들어주는 방법을 도입
  - 프록시를 적용할 대상을 일일히 지정하지 않고 패턴을 이용해 자동으로 선정할 수 있도록, 클래스를 선정하는 기능을 담은 확장된 포인트컷 이용
  - 최종적으로는 포인트컷 표현식이라는 좀 더 편리하고 깔끔한 방법을 활용해 간단한 설정만으로 적용 대상을 손쉽게 선택할 수 있게 됨

#### 부가기능의 모듈화
- 관심사가 같은 코드를 분리해 한데 모으는 것은 소프트웨어 개발의 가장 기본 원칙
- 트랜잭션 같은 부가기능은 핵심 기능과 다르게 독립적으로 존재하는 것이 불가능
- 그래서 나타난 것이 DI, 데코레이터 패턴, 다이내믹 프록시, 오브젝트 생성 후처리, 자동 프록시 생성, 포인트컷 같은 기법
- 지금까지 해온 모든 작업은 핵심 기능에 부여되는 부가기능을 효과적으로 모듈화하는 방법을 찾는 것
- 어드바이스와 포인트컷을 결합한 어드바이저가 단순하지만 이런 특성을 가진 모듈의 원시적인 형태로 만들어지게 됐다.

#### AOP : 애스펙트 지향 프로그래밍
- 전통적인 객체지향 기술의 설계 방법으로는 독립적인 모듈화가 불가능한 트랜잭션 경계 설정과 같은 부가기능을 어떻게 모듈화할 것인가를 연구해온 사람들은, 이 부가기능 모듈화 작업은 기존의 객체지향 설계 패러다임과는 구분되는 새로운 특성이 있다고 생각했다.
- 이런 부가기능 모듈을 객체지향 기술에서 주로 사용하는 오브젝트와 다르게 **애스펙트(aspect)**라고 부르기 시작
	- 그 자체로 애플리케이션의 핵심 기능을 담고 있지는 않지만, 애플리케이션을 구성하는 중요한 한 가지 요소이고, 핵심 기능에 부가되어 의미를 갖는 특별한 모듈
- 애스펙트는 부가될 기능을 정의한 코드인 어드바이스와, 어드바이스를 어디에 적용할지를 결정하는 포인트컷을 함께 갖고 있다.

<img width="649" alt="스크린샷 2024-07-24 오후 3 43 44" src="https://github.com/user-attachments/assets/e24782a7-622a-4382-bec2-541e70c52a8f">

- 왼쪽은 에스팩트로 부가기능을 분리하기 전의 상태
	- 핵심 기능은 깔끔한 설계를 통해서 모듈화되어 있고, 객체지향적인 장점을 잘 살릴 수 있도록 만들었지만, 부가기능이 핵심 기능의 모듈에 침투 ➡️ 설계와 코드 지저분
- 오른쪽은 침투한 부가기능을 애스펙트로 구분해낸 것
- 2차원 평면구조에서는 해결할 수 없었던 것을 3차원의 다면체 구조로 가져가면서 각각 성격이 다른 부가기능은 다른 면에 존재하도록 함

- 물론 부가기능 에스팩트는 런타임 시 필요한 위치에 다이내믹하게 참여하게 될 것. 하지만 설계와 개발은 애스펙트들을 독립적인 관점으로 작성하게 할 수 있는 것
- 이렇게 애플리케이션의 핵심적인 기능에서 부가적인 기능을 분리해서 애스펙트라는 독특한 모듈로 만들어서 설계하고 개발하는 방법을 **애스펙트 지향 프로그래밍** 또는 약자로 AOP라고 부른다.
- AOP는 OOP를 돕는 보조적인 기술이지 OOP를 완전히 대체하는 새로운 개념은 아니다.
  - 애스펙트를 분리함으로써 핵심 기능을 설계하고 구현할 때 객체지향적인 가치를 지킬 수 있도록 도와주는 것
- 애플리케이션을 특정한 관점을 기준으로 바라볼 수 있게 해준다는 의미에서 관점 지향 프로그래밍이라고도 한다.

### 6.5.5 AOP 적용 기술
#### 프록시를 이용한 AOP
- 스프링은 다양한 기술을 조합해 AOP를 지원하고 있다.
- 그 중 핵심은 프록시
  - 프록시로 만들어 DI로 연결된 빈 사이에 적용해 메서드 호출 과정에 참여해서 부가기능을 제공해주도록 만들었다.
- 스프링 AOP는 기본 JDK와 스프링 컨테이너 외에는 특별한 기술이나 환경을 요구하지 않는다.
- 스프링 AOP의 부가기능을 담은 어드바이스가 적용되는 대상은 오브젝트의 메서드
	- 프록시 방식을 사용했기 때문에 메서드 호출 과정에 참여해서 부가기능을 제공해주게 되어있다.
- 부가기능 모듈을 다양한 타깃 오브젝트 메서드에 다이내믹하게 적용해주기 위해 가장 중요한 역할을 맡고 있는 것이 바로 프록시
- 스프링의 AOP는 프록시 방식의 AOP


#### 바이트 코드 생성과 조작을 위한 AOP
- 프록시 방식이 아닌 AOP? `AspectJ`
- AspectJ는 어떻게 프록시를 사용하지 않고 부가기능을 다이내믹하게 타깃 오브젝트에 적용해줄 수 있을까?

	➡️ 프록시처럼 간접적인 방법이 아니라, **타깃 오브젝트를 뜯어 고쳐서 부가기능을 직접 넣어주는 직접적인 방법 사용**
- 컴파일된 타깃의 클래스 파일 자체를 수정하거나 클래스가 JVM에 로딩되는 시점을 가로채서 바이트 코드를 조작하는 복잡한 방법 사용
- 트랜잭션 코드가 UserService 클래스에 비즈니스 로직과 함께 있었을 때처럼 만들어버리는 것
- 그렇다면 왜 이런 복잡한 방법을 사용할까? 두 가지 이유가 있다.

  (1) 스프링 컨테이너가 사용되지 않는 환경에서도 AOP 적용 가능
  	- 바이트코드를 조작해서 타깃 오브젝트를 직접 수정해버리면 DI 컨테이너의 도움을 받아 자동 프록시 생성 방식을 사용하지 않아도 AOP를 적용할 수 있다.

  (2) 프록시 방식보다 훨씬 강력하고 유연한 AOP 가능
  	- 프록시를 AOP의 핵심 메커니즘으로 사용하면 부가기능을 부여할 대상은 클라이언트가 호출할 때 사용하는 메서드로 제한
  	- 하지만 바이트 코드를 직접 바이트코드를 직접 조작하면 다양한 작업에 부가기능 부여 가능
  	- ex. 타깃 오브젝트가 생성되는 순간 부가기능 부여,  private 메서드 호출, 스태틱 메서드 호출 및 초기화, 필드 입출력

- 대부분의 부가기능은 프록시 방식으로 충분. AspectJ 같은 고급 AOP 기술은 번거로운 작업 필요.

### 6.5.6 AOP의 용어
- **타깃** : 부가기능을 부여할 대상. 핵심 기능을 담은 클래스일 수도 있지만 경우에 따라 부가기능을 제공하는 프록시 오브젝트일 수도 있음
- **어드바이스** : 타깃에게 제공할 부가기능을 담은 모듈. 오브젝트로 정의하기도 하지만 메서드 레벨에서 정의할 수도 있다.
- **조인 포인트** : 어드바이스가 적용될 수 있는 위치. 스프링의 프록시 AOP에서 조인 포인트는 메서드 실행 단계 뿐. 타깃 오브젝트가 구현한 인터페이스의 모든 메서드는 조인 포인트가 된다.
- **포인트컷** : 어드바이스를 적용할 조인 포인트를 선별하는 작업 또는 그 기능을 정의한 모듈
- **프록시** : 클라이언트와 타깃 사이에 투명하게 존재하면서 부가기능을 제공하는 오브젝트
- **어드바이저** : 포인트컷과 어드바이스를 하나씩 갖고 있는 오브젝트. AOP의 가장 기본이 되는 모듈로, 스프링 AOP에서만 사용되는 특별한 용어
- **애스펙트** : AOP의 기본 모듈로 한 개 또는 그 이상의 포인트컷과 어드바이스의 조합으로 만들어지며 보통 싱글톤 형태의 오브젝트로 존재.


### 6.5.7 AOP 네임스페이스
스프링의 프록시 방식의 AOP를 적용하려면 최소한 네 가지 빈 등록 필요
- **자동 프록시 생성기**
  - 스프링의 DefaultAdvisorProxyCreator 클래스를 빈으로 등록
  - DI하지도 DI되지도 않으며 독립적으로 존재
  - 애플리케이션 컨텍스트가 빈 오브젝트를 생성하는 과정에 빈 후처리기로 참여
  - 빈으로 등록된 어드바이저를 이용해서 프록시를 자동으로 생성하는 기능을 담당
- **어드바이스**
  - 부가기능을 구현한 클래스를 빈으로 등록
  - TransactionAdvice는 AOP 관련 빈 중에서 유일하게 직접 구현한 클래스 사용
- **포인트컷**
  - 스프링의 AspectJExpressionPointcut을 빈으로 등록하고 expression 프로퍼티에 포인트컷 표현식을 넣어주면 된다.
- **어드바이저**
  - 스프링의 DefaultPointcutAdvisor 클래스를 빈으로 등록해서 사용한다.


#### AOP 네임 스페이스
- 스프링은 AOP와 관련된 태그를 정의해둔 aop 스키마 제공
- aop 스키마에 정의된 태그는 별도의 네임스페이스를 제정해서 디폴트 네임스페이스의 <bean> 태그와 구분해 사용 가능
- aop 스키마에 정의된 태그를 사용하기 위해 작성해야 하는 설정 파일
```xml
// 6-66. aop 네임스페이스 선언
<?xml version="1.0" encoding="UTF-8"?>          
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"        
    xsi:schemaLocation="http://www.springframework.org/schema/beans          
			http://www.springframework.org/schema/beans/spring-beans-3.0.xsd          
			http://www.springframework.org/schema/aop
			http://www.springframework.org/schema/aop/spring-aop-3.0.xsd"/>
  ...
</beans>
```
- 이제 aop 네임스페이스를 이용해 기존 AOP 관련 빈 설정을 변경해보자.
```xml
// 6-67. aop 네임스페이스를 적용한 AOP 설정 빈
<aop:config>
<aop:pointcut id="transactionPointcut" expression="execution(* *..*ServiceImpl.upgrade*(..))" />
<aop:advisor advice-ref="transactionAdvice" pointcut-ref="transactionPointcut"/>
</aop:config>
```

	- <`aop:config>`, `<aop:pointcut>`, `<aop:advisor>` 세 가지 테그를 정의해두면 그에 따라 세 개의 빈 자동 등록


#### 어드바이저 내장 포인트컷

- 포인트컷 표현식을 직접 `<aop:advisor>` 태그에 담아서 다음과 같이 만들 수도 있다.
```xml
// 6-68. 포인트컷을 내장한 어드바이저 태그
<aop:config>
  <aop:advisor advice-ref="transactionAdvice" pointcut="execution(* *..*ServiceImpl.upgrade*(..))"/>
</aop:config>
```

## 6.6 트랜잭션 속성

- 트랜잭션 추상화를 할 때 그냥 넘어간 것이 한 가지 있음
- 트랜잭션 속성을 담당하는 `DefaultTransactionDefinition` 오브젝트임
- 트랜잭션 경계는 트랜잭션 매니저에서 트랜잭션을 가져오는 것과 `commit()`, `rollback()` 중 하나를 호출하는 것으로 설정됨

```java
// 6-69. 트랜잭션 경계 설정 코드
public Object invoke(MethodInvocation invocation) throws Throwable {
    TransactionStatus status =
            this.transactionManager.getTransaction(
                    new DefaultTransactionDefinition()); 
    try{
        Object ret = invocation.proceed();
        this.transactionManager.commit(status);
        return ret;
    }catch (RuntimeException e){
        this.transactionManager.rollback(status);
        throw e;
    }
}
```

### 6.6.1 트랜잭션 정의
- `DefaultTransactionDefinition`이 구현하고 있는 `TransactionDefinition` 인터페이스는 트랜잭션 동작 방식에 영향을 줄 수 있는 네 가지 속성을 정의하고 있음.


#### 트랜잭션 전파
- **트랜잭션 전파** : 트랜잭션의 경계에서 이미 진행 중인 트랜잭션이 있을 때 또는 없을 때 어떻게 동작할 것인가를 결정하는 방식

<img width="408" alt="image" src="https://github.com/user-attachments/assets/7212d2ee-4e04-4a56-9c63-f926866b33c7">

- B는 A 트랜잭션에 종속돼야 하는가, 독립적으로 작동해야 하는가를 정의하는 것이 트랜잭션 전파 속성
- 대표적으로 다음과 같은 트랜잭션 전파 속성을 줄 수 있다.

- **PROPAGATION_REQUIRED**
  - 가장 많이 사용되는 전파 속성
  - 진행 중인 트랜잭션이 없으면 새로 시작하고, 이미 시작된 트랜잭션이 있으면 참여한다.
- **PROPAGATION_REQUIRES_NEW**
  - 항상 새로운 트랜잭션을 실행한다.
- **PROPAGATION_NOT_SUPPORTED**
  - 진행 중인 트랜잭션이 있어도 무시한다.
  - 포인트컷을 복잡하게 만들지 않고 특정 기능에만 트랜잭션 적용이 안되도록 설정하고자 할 때 사용


#### 격리 수준
- 모든 DB 트랜잭션은 격리수준을 갖고 있어야 한다.
- 기본적으로는 DB에 설정되어 있고 DefaultTransactionDefinition도 `ISOLATION_DEFAULT`이다.
- 특별한 작업을 수행하는 메서드의 경우 독자적인 격리수준을 지정할 필요가 있다.

#### 제한 시간
- 트랜잭션 수행 제한시간을 설정할 수 있다.
- DefaultTransactionDefinition의 기본 설정은 제한시간이 없는 것이다.

#### 읽기 전용
- 읽기전용 트랜잭션은 트랜잭션 내에서 데이터 조작 시도를 막아줄 수 있다.

### 6.6.2 트랜잭션 인터셉터와 트랜잭션 속성
- 메서드 별로 다른 트랜잭션 정의를 적용하려면 어드바이스의 기능을 확장해야 한다.
- 메소드 이름 패턴에 따라 트랜잭션 정의가 적용되도록 만드는 것이다.

#### TransactionIntercepter
- 스프링에서 제공
- TransactionInterceptor는 TransactionAdvise와 비슷하지만, 트랜잭션 정의를 메소드 이름 패턴을 이용해 다르게 지정할 수 있는 방법 추가로 제공
- PlatformTransactionManager와 Properties 타입의 두 가지 프로퍼티를 갖고 있음
- Properties 타입의 프로퍼티 이름은 transactionAttribute로, 트랜잭션 속성을 정의한 프로퍼티임. TransactionAttribute 인터페이스는 TransactionDefinition의 네 가지 기본 항목에 rollbackOn()가 추가로 존재함.
- 우리가 직접 구현했던 TransactionAdvice 코드를 다시 살펴보면 변경될 수 있는 코드가 2군데 존재한다.
  - 하나는 트랙잭션 속성을 설정하는 DefaultTransactionDefinition 객체를 다루는 부분
  - 나머지 하나는 어떤 어떤 예외가 발생하면 롤백을 수행할 지 결정하는 부분
- rollbackOn() 메서드를 사용해 어떤 예외는 롤백을 하거나, 롤백을 하지 않을지 설정할 수 있다.

```java
// 6-70. 트랜잭션 경계 설정 코드의 동작 방식 변경 포인트
@Override
public Object invoke(MethodInvocation invocation) throws Throwable {
    TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition()); // (1) 트랜잭션 정의를 통한 네 가지 조건 (트랜잭션 전파, 격리수준, read-only, 타임아웃)
    try {
        Object ret = invocation.proceed();
        transactionManager.commit(status);
        return ret;
    } catch (RuntimeException e) { // (2) 롤백 대상인 예외 종류
        transactionManager.rollback(status);
        throw e;
    }
}
```

> (1), (2)가 결합해서 트랜잭션 부가기능의 행동을 결정하는 TransactionAttribute 속성이 된다.

- 스프링이 제공하는 TransactionIntercepter에는 기본적으로 두 가지 종류의 예외 처리 방식이 있다.
- 런타임 예외가 발생하면 트랜잭션은 롤백한다.
- 체크 예외가 발생하면 이것을 예외상황으로 해석하지 않고 일종의 비지니스 로직에 따른, 의미가 있는 리턴 방식의 한 가지로 인식해서 트랜잭션을 커밋한다.
- 그러나 이런 예외처리 기본 원칙을 따라지 않는 예외적인 케이스에서는 rollbackOn() 설정을 통해 기본 원칙과 다른 예외처리가 가능하다.

#### 메서드 이름 패턴을 이용한 트랜잭션 속성 지정
- Properties 타입의 transactionAttributes 프로퍼티는 메서드 패턴과 트랜잭션 속성을 키와 값으로 갖는 컬렉션
- 트랜잭션 속성은 다음과 같은 문자열로 정의

  <img width="579" alt="스크린샷 2024-07-28 오후 11 15 09" src="https://github.com/user-attachments/assets/56679330-1405-41bd-9845-6f5942ac3894">

- 트랜잭션 전파 항목만 필수, 나머지는 생략 가능
- 순서 바꿔도 상관 없음

```xml
// 6-71. 트랜잭션 속성 정의 예
<bean id="transactionAdvice" 
    class="org.springframework.transaction.interceptor.TransactionInterceptor">
    <property name="transactionManager" ref="transactionManager"/>
    <property name="transactionAttributes">
        <props>
            <prop key="get*">PROPAGATION_REQUIRED,readOnly,timeout_30</prop>
            <prop key="upgrade*">PROPAGATION_REQUIRES_NEW,ISOLATION_SERIALIZABLE</prop>
            <prop key="*">PROPAGATION_REQUIRED</prop>
        </props>
    </property>
</bean>
```

#### tx 네임스페이스를 이용한 설정 방법
- TransactionInterceptor 타입의 어드바이스 빈과 TransactionAttribute 타입의 속성 정보도 tx 스키마의 전용 태그를 이용해 정의할 수 있다.

```xml
// 6-72. tx 스키마의 적용 태그
<tx:advice id="transactionAdvice" transaction-manager="transactionManager">
    <tx:attributes>
        <tx:method name="get*" propagation="REQUIRED" read-only="true" timeout="30"/>
        <tx:method name="upgrade*" propagation="REQUIRES_NEW" isolation="SERIALIZABLE"/>
        <tx:method name="*" propagation="REQUIRED"/>
    </tx:attributes>
</tx:advice>
```

### 6.6.3 포인트컷과 트랜잭션 속성의 적용 전략
- 트랜잭션 부가기능을 적용할 후보 메서드 선정은 **포인트컷**이 진행
- **어드바이스의 트랜잭션 전파 속성**에 따라서 메서드별로 트랜잭션 적용방식이 결정됨
- 포인트컷 표현식과 트랜잭션 속성을 정의할 때 따르면 좋은 몇 가지 전략을 생각해보자.

#### 트랜잭션 포인트컷 표현식은 타입 패턴이나 빈 이름을 이용한다
- 트랜잭션 타깃 클래스의 모든 메서드가 트랜잭션 적용 후보가 되는 것이 바람직
- 하지만 비즈니스 로직을 담고 있는 클래스라면 메서드 단위까지 세밀하게 포인트컷을 정의해줄 필요는 없다.
- UserService의 add 메서드는 트랜잭션 적용 대상이어야 한다.
  - DB 정보를 다루는 작업이 추가될 가능성이 높다.
- 단순한 조회 작업만 하는 메서드에도 트랜잭션을 적용하는 게 좋다.
  - 조회의 경우에는 읽기 전용으로 트랜잭션 속성을 설정해두면 그만큼의 성능 향상을 가져올 수 있다.
- 따라서 트랜잭션 포인트컷 표션식에서는 메서드나 파라미터, 예외에 대한 패턴을 정의하지 않는 게 바람직하다.
- 트랜잭션의 경계로 삼을 클래스들이 선정됐다면, 패키지를 통째로 선택하거나 클래스 이름에서 일정한 패턴을 찾아서 표현식으로 만들면 된다.
- 메서드 시그니처를 이용한 execution() 방식의 포인트컷 표현식 대신 스프링 빈의 이름을 이용하는 bean() 표현식 사용도 고려해볼만 하다.
  - bean()은 빈 이름을 기준으로 선정하기 때문에 이름에 일정한 규칙을 만들기 어려운 경우 유용

#### 공통된 메서드 이름 규칙을 통해 최소한의 트랜잭션 어드바이스와 속성을 정의한다
- 기준이 되는 몇 가지 트랜잭션 속성을 정의하고, 그에 따라 적절한 메서드 명명 규칙을 만들어두면 하나의 어드바이스만으로도 애플리케이션의 모든 서비스 빈에 트랜잭션 속성을 지정할 수 있다.
- 그런데 가끔 트랜잭션 속성 적용 패턴이 일반적인 경우와 크게 다른 오브젝트도 존재
- 이런 예외적인 경우에는 트랜잭션 어드바이스와 포인트컷을 새롭게 추가해줄 필요가 있다.
- 가장 간단한 트랜잭션 속성 부여방법은 다음과 같이 모든 메서드에 대해 디폴트 속성을 지정하는 것
```xml
// 6-73. 디폴트 트랜잭션 속성 부여
<tx:advice id="transactionAdvice">
	<tx:attributes>
		<tx:method name="*" /> <!-- 모든 타깃 메서드에 기본 트랜잭션 속성 지정 -->
	</tx:attributes>
</tx:attributes>
```
- 한 단계 더 나아간다면, 다음과 같이 간단한 메서드 이름 패턴을 적용해볼 수 있다.
```xml
// 6-74. 읽기 전용 속성 추가
// 6-73. 디폴트 트랜잭션 속성 부여
<tx:advice id="transactionAdvice">
	<tx:attributes>
		<tx:method name="get*" read-only="treu" />
		<tx:method name="*" />
	</tx:attributes>
</tx:attributes>
```
- 일반화하기 적당하지 않은 특별한 트랜잭션 속성이 필요한 타깃 오브젝트에 대해서는 별도의 어드바이스와 포인트컷을 사용하는 편이 좋다.

<img width="606" alt="스크린샷 2024-07-30 오후 10 30 23" src="https://github.com/user-attachments/assets/da4e74fc-e6d3-456f-8ae4-834ba3f91a84">


#### 프록시 방식 AOP는 같은 타깃 오브젝트 내의 메서드를 호출할 때는 적용되지 않는다
- 전략이라기보다는 주의 사항
- 타깃 오브젝트가 자기 자신의 메서드를 호출할 때는 프록시를 통한 부가기능의 적용이 일어나지 않는다.

<img width="546" alt="스크린샷 2024-07-31 오후 10 10 09" src="https://github.com/user-attachments/assets/afd668a4-2f13-471c-b6b9-034f968901bd">

- `delete()`와 `update()`는 모두 트랜잭션 적용 대상인 메서드다.
- 따라서 [1]과 [3]처럼 클라이언트로부터 메서드가 호출되면 트랜잭션 경계 설정 부가기능이 부여될 것이다.
- 그러나 [2]의 경우는 다르다. 프록시를 거치지 않고 직접 타깃의 메서드가 호출된다. [2]를 통해 update() 메서드가 호출될 때는 update() 메서드에 지정된 트랜잭션 속성이 전혀 반영되지 않는다.
- 따라서 **같은 오브젝트 안에서의 호출은 새로운 트랜잭션 속성을 부여하지 못한다는 사실을 의식하고 개발할 필요하 있다.**
- 이 문제를 해결할 수 있는 방법 두 가지
  - 스프링 API를 이용해 프록시 오브젝트에 대한 레퍼런스를 가져온 뒤에 같은 오브젝트의 메서드 호출도 프록시를 이용하도록 강제 (순수 비즈니스 로직을 해치므로 비추천)
  - AspectJ와 같은 타깃의 바이트 코드를 직접 조작하는 방식의 AOP 기술 적용

### 6.6.4 트랜잭션 속성 적용
- 트랜잭션 속성과 그에 따른 트랜잭션 전략을 UserService에 적용해보자.


#### 트랜잭션 경계 설정의 일원화
- 트랜잭션 경계설정의 부가기능을 여러 계층에서 적용하는 것 좋지 않음. 특정 계층의 경계를 트랜잭션 경계와 일치시키는 것이 바람직
- **비즈니스 로직을 담고 있는 서비스 계층의 오브젝트**의 메서드가 트랜잭션 경계를 부여하기에 가장 적절한 대상
- 서비스 계층을 트랜잭션이 시작되고 종료되는 경계로 정했다면 다른 계층이나 모듈에서 DAO에 직접 접근하는 것은 차단해야 한다.
- 트랜잭션은 보통 서비스 계층의 메서드 조합으로 만들어지므로 DAO가 제공하는 주요 기능을 서비스 계층에 위임 메서드를 만들어둘 필요가 있다.
- 가능하면 다른 모듈의 DAO에 접근할 때는 서비스 계층을 거치도록 하는 게 바람직하다.
- 예를 들어 UserService가 아니라면 UserDao에 직접 접근하지 않고 UserService의 메서드를 이용하는 편이 좋다.
- UserDao 인터페이스에 정의된 메서드 중에서 add()를 제외한 5개가 UserService에 새로 추가할 후보 메서드다. getCount()를 제외한 나머지 4개 메서드는 독자적인 트랜잭션을 가지고 사용될 가능성이 높으므로 다음과 같이 UserService에 추가한다.
```java
// 6-76. UserService에 추가된 메서드
public interface UserService {
	void add(User user);
	User get(String id);
	List<User> getAll();
	void deleteAll();
	void update(User user);

	void upgradeLevels();
}
```
```java
// 6-77. 추가 메서드 구현
public class UserServiceImpl implements UserService {
	UserDao userDao;
	...
	public void deleteAll() { UserDao.deleteAll(); }
	public User get(String id) { // 생략 }
	public List<User> getAll() { // 생략 }
	public void update(User user) { // 생략 }
}
```
- 이제 모든 User 관련 데이터 조작은 UserService라는 **트랜젝션 경계**를 통해 진행할 경우 모두 트랜잭션을 적용할 수 있게 됐다.

#### 서비스 빈에 적용되는 포인트컷 표현식 등록
- 모든 비즈니스 로직의 서비스 빈에 트랜잭션이 적용되도록 기존 표인트컷 표현식을 수정한다.
```xml
// 6-78. 빈 이름을 사용한 표현식을 갖는 포인트컷과 어드바이저
<aop:config>
	<aop:advisor advice-ref="transactionAdvice" pointcut="bean(*Service) />
</aop:config>
```
- 이제 아이디가 Service로 끝나는 모든 빈에 transactionAdvice 빈의 부가기능이 적용될 것이다.

#### 트랜잭션 속성을 가진 트랜잭션 어드바이스 등록
- TransactionAdvice 클래스로 정의했던 어드바이스 빈을 **스프링의 TransactionInterceptor**를 이용하도록 변경한다.
- 메서드 패턴과 트랜잭션 속성은 가장 보편적인 방법인 get으로 시작하는 메서드는 읽기 전용, 나머지는 디폴트 트랜잭션 속성을 따르는 것으로 설정

```xml
// 6-79. 트랜잭션 속성을 사용하는 어드바이스
<bean id="transactionAdvice" class="org.springframework.transaction.interceptor.TransactionInterceptor">
    <property name="transactionManager" ref="transactionManager" />
    <property name="transactionAttributes">
        <props>
            <prop key="get*">PROPAGATION_REQUIRED, readOnly</prop>
            <prop key="*">PROPAGATION_REQUIRED</prop>
        </props>
    </property>
</bean>

<tx:advice id="transactionAdvice">
    <tx:attributes>
        <tx:method name="get*" read-only="true"/>
        <tx:method name="*"/>
    </tx:attributes>
</tx:attributes>
```
- 이미 aop 스키마 태그를 적용했으니 어드바이스도 이왕이면 tx 스키마에 정의된 태그를 이용하도록 변경
```xml
// 6-80. tx 스키마의 태그를 이용한 트랜잭션 어드바이스 정의
<tx:advice id="transactionAdvice" transaction-manager="transactionManager">
  <tx:attributes>
    <tx:method name="get*" propagation="REQUIRED" read-only="true" />
    <tx:method name="*" propagation="REQUIRED"/>
  </tx:attributes>
</tx:advice>
```

#### 트랜잭션 속성 테스트
- get으로 시작하는 메서드는 트랜잭션 쓰기 작업이 정말 허용되지 않을까?

```java
// 6-81. 읽기전용 메서드에 쓰기 작업을 추가한 테스트용 클래스
static class TestUserServiceImpl extends UserServiceImpl {
    @Override
    public List<User> getAll() {
        for (User user : super.getAll()) {
            super.update(user); // 강제로 쓰기 시도 -> 읽기 전용 속성으로 인한 예외가 발생해야 한다.
        }
        return null; // 메서드가 끝나기 전에 예외가 발생해야 하니 리턴 값은 별 의미 없음.
    }
}
```
```java
// 6-82. 읽기전용 속성 테스트
@Test // 일단은 어떤 예외가 던져질지 모르기 때문에 expected 없이 작성
public void readOnlyTransactionAttribute() {
    testUserService.getAll(); // 예외가 발생해야 함.
}
```
- 테스트를 실행하면 예외가 발생한다.
- 읽기전용으로 설정된 DB 커넥션에 대해 데이터를 조작하는 작업을 시도했기 때문에 예외 발생
  - TransientDataAccessResourceException이라는 생소한 예외
  - DataAccessException의 한 종류로 일시적인 예외상황을 만났을 때 발생하는 예외

```java
// 6-83. 예외 확인 테스트로 수정
@Test(expected=TransientDataAccessResourceException.class)
public void readOnlyTransactionAttribute() {
    ...
}
```
- 테스트 돌려보면 모두 성공

## 6.7 애노테이션 트랜잭션 속성과 포인트컷
- 가끔은 클래스나 메서드에 따라 제각각 속성이 다른, 세밀하게 튜닝된 트랜잭션 속성이 필요할 때가 있다.
- 이런 경우 스프링이 제공하는 다른 방법이 있다. 직접 타깃에 트랜잭션 속성 정보를 가진 애너테이션을 지정하는 방법이다.

### 6.7.1 트랜잭션 애너테이션

#### @Transactional
```java
// 6-84. @Transaction 애너테이션을 정의한 코드
...
@Target({ElementType.TYPE, ElementType.METHOD}) // 애너테이션을 사용할 대상을 지정한다. 한 개 이상의 대상 지정 가능
@Retention(RetentionPolicy.RUNTIME) // 애너테이션 정보가 언제까지 유지되는지 지정. 이렇게 지정하면 런타임 때도 애너테이션 정보를 리플렉션을 통해 얻을 수 있다.
@Inherited // 상속을 통해서도 애너테이션 정보를 얻을 수 있게 한다.
@Documented
public @interface Transactional {
	// 트랜잭션 속성의 모든 항목을 엘리먼트로 지정할 수 있다.
	// 디폴트 값이 설정되어 있으므로 모두 생략 가능
	@AliasFor("transactionManager")
	String value() default "";
	@AliasFor("value")
	String transactionManager() default "";
	String[] label() default {};
	Propagation propagation() default Propagation.REQUIRED;
	Isolation isolation() default Isolation.DEFAULT;
	int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;
	String timeoutString() default "";
	boolean readOnly() default false;
	Class<? extends Throwable>[] rollbackFor() default {};
	String[] rollbackForClassName() default {};
	Class<? extends Throwable>[] noRollbackFor() default {};
	String[] noRollbackForClassName() default {};

}
```
- `@Transactional` 애너테이션의 타깃은 메서드와 타입이다. ➡️ 메서드, 클래스, 인터페이스에 사용 가능
- 스프링은 `@Transactional`이 부여된 모든 오브젝트를 자동으로 타깃 오브젝트로 인식한다.
- 이 때 사용되는 포인트컷은 TransactionAttribueSourcePointcut이다.
  - 스스로 표현식과 같은 선정 기준을 갖고 있지는 않고, 부여된 빈 오브젝트를 모두 찾아서 포인트컷의 선정 결과로 돌여준다.
- `@Transactional`은 기본적으로 트랜잭션 속성을 정의하는 것이지만, 동시에 포인트컷의 자동등록에도 사용된다.

#### 트랜잭션 속성을 이용하는 포인트컷

- `@Transactional` 애너테이션을 사용했을 때 어드바이저의 동작 방식은 다음과 같다.


<img width="588" alt="스크린샷 2024-08-02 오후 5 03 27" src="https://github.com/user-attachments/assets/fdff02dc-8936-4b4e-ac77-d7447bb3662f">

- TransactionInterceptro는 메서드 이름 패턴을 통해 부여되는 일괄적인 트랜잭션 속성정보 대신 `@Transactional` 애너테이션의 엘리먼트에서 트랜잭션 속성을 가져오는 AnnotationTransactionAttributeSource를 사용한다.

- 이 방식을 이용하면 **포인트컷과 트랜잭션 속성을 애너테이션 하나로 지정**할 수 있다.
- 메서드 단위로 세분화해 트랜잭션 속성을 다르게 지정할 수 있기 때문에 세밀한 트랜잭션 속성 제어가 가능해진다.
- 트랜잭션 부가기능 적용 단위는 메서드다. 따라서 메서드마다 `@Transactional`을 부여하고 속성을 지정할 수 있다.
- 이렇게 하면 유연한 속성 제어는 가능하겠지만 **코드는 지저분해지고, 동일한 속성 정보를 가진 애너테이션을 반복적으로 메서드마다 부여해주는 바람직하지 못한 결과를 가져올 수 있다.**

#### 대체 정책
- 그래서 스프링은 `@Transactional`을 적용할 때 4단계의 대체(fallback) 정책을 이용하게 해준다.
- 메서드의 속성을 확인할 때 **타깃 메서드, 타깃 클래스, 선언 메서드, 선언 타입(클래스, 인터페이스) 순서에 따라서 @Transactional이 적용됐는지 차례로 확인하고, 가장 먼저 발견되는 속성 정보를 사용하게 하는 방법**이다.
- 이런 식으로 메서드가 선언된 타입까지 단계적으로 확인해서 `@Transactional`이 발견되면 적용하고, 끝까지 발견되지 않으면 해당 메서드는 트랜잭션 적용 대상이 아니라고 판단한다.
- 다음과 같이 정의된 인터페이스와 구현 클래스가 있다고 하다. `@Transactional`을 부여할 수 있는 위치는 총 6개다.
```java
// 6-85. @Transactional 대체 정책의 예
[1]
public interface Service {
	[2]
	void method1();
	[3]
	void method2();
}
[4]
public class ServiceImpl implements Service {
	[5]
	public void method1() {
	}
	[6]
	public void method2() {
	}
}
```
- `@Transactional`이 위치할 수 있는 후보 순서
  - [5]와 [6] -> [4] -> [2]와 [3] -> [1]
- `@Transactional`을 사용하면 대체 정책을 잘 활용해서 애너테이션 자체는 최소환으로 사용하면서 세밀한 제어 가능
-  `@Transactional`은 먼저 타입 레벨에 정의되고 공통 속성을 따르지 않는 메서드에 대해서만 메서드 레벨에 다시 `@Transactional`을 부여해주는 식으로 사용해야 한다.
-  기본적으로 `@Transactional` 적용 대상은 클라이언트가 사용하는 인터페이스가 정의한 메서드이므로 `@Transactional`도 타깃 클래스보다는 인터페이스에 두는 게 바람직하다.
-  하지만 인터페이스를 사용하는 프록시 방식의 AOP가 아닌 방식으로 트랜잭션을 적용하면 무시되기 때문에 안전하게 **타깃 클래스에 `@Transactional`에 두는 방법을 권장**

#### 트랜잭션 애너테이션 사용을 위한 설정
- 스프링은 `@Transactional`을 이용한 트랜잭션 속성을 사용하는 데 필요한 모든 설정을 다음 태그 하나에 담아뒀다.
```xml
<tx:annotation-driven/>
```

### 6.7.2 트랜잭션 애너테이션 적용
- `@Transactional`을 UserService에 적용해보자.
- 꼭 세밀한 트랜잭션 설정이 필요할 때만 `@Transactional`을 사용해야 하는 것은 아니다.
- 다만 `@Transactional`을 사용하면 트랜잭션 적용 대상을 손쉽게 파악할 수 없고, 무분별하게 사용되거나 자칫 빼먹을 수도 있기 때문에 주의가 필요하다.
- **`<tx:attributes>` 태그를 이용해 설정했던 트랜잭션 속성을 그대로 애너테이션으로 바꿔보자.**

```xml
// 6-86. tx 스키마의 태그를 이용한 트랜잭션 속성 정의
<tx:attributes>
	<tx:method name="get" read-only="true" />
	<tx:method name="*" />
</tx:attributes>
```
- 애너테이션을 이용할 때는 이 두 가지 속성 중에서 많이 사용되는 한 가지를 타입 레벨에서 공통 속성으로 지정해주고, 나머지 속성은 개별 메서드에 적용해야 한다.
- 메서드 레벨의 속성은 메서드마다 반복돼야 하므로 속성의 종류가 두 가지 이상이고 적용 대상 메서드의 비율이 비슷하다면 메서드에 많은 `@Transactional` 애너테이션이 반복될 수 있다.

```java
// 6-87. @Transactional 애너테이션을 이용한 속성 부여
@Transactional // <tx:method name="*" />과 같은 설정 효과를 가져온다. 메서드 레벨 @Transactional이 없으므로 대체 정책에 따라 타입 레벨에 부여된 디폴트 속성이 적용된다.
public interface UserService {
	void add(User user);
	void deleteAll();
	void update(User user);
	void upgradeLevels();

	@Transactional(readOnly=true) // <tx:method name="get" read-only="true" />를 애너테이션 방식을 변경한 것. 메서드 단위로 부여된 트랜잭션 속성이 타입 레벨에 부여된 것에 우선해서 적용된다. 같은 속성을 가졌어도 메서드 레벨에 부여되는 메서드마다 반복될 수 밖에 없다.
	User get(String id);

	@Transactional(readOnly=true)
	List<User> getAll();
}
```
- **대체 정책 순서는 타깃 클래스가 인터페이스보다 우선하므로 모든 메서드의 트랜잭션은 디폴트 속성을 갖게 된다.**
- UserService 인터페이스의 getAll() 메서드에 부여한 읽기 전용 속성은 무시되고, 읽기전용 속성을 검증하는 `readOnlyTransactionAttribute()` 테스트는 실패할 것이다.


## 6.8 트랜잭션 지원 테스트
### 6.8.1 선언적 트랜잭션과 트랜잭션 전파 속성
- 트랜잭션 전파 속성은 매우 유용한 개념
- 예를 들어 REQUIRED 전파 속성을 가진 메서드를 결합해서 다양한 크기의 트랜잭션 작업을 만들 수 있다. 불필요한 코드 중복을 피하고, 애플리케이션을 작은 단위 기능으로 쪼개서 개발할 수 있다.
- 예를 들어 UserService의 add() 메서드는 트랜잭션 속성이 디폴트로 지정되어 있으므로 트랜잭션 전파 방식은 REQUIRED다. 만약 add() 메서드가 처음 호출되는 서비스 게층의 메서드라면 한 명의 사용자를 등록하는 것이 하나의 비즈니스 작업 단위가 된다. 이때는 add() 메서드가 시작되기 전에 트랜잭션이 시작되고 add() 메서드를 빠져나오면 트랜잭션이 종료되는 것이 맞다.
- 그런데 작업 단위가 다른 비즈니스 로직이 있을 수 있다 : 그날의 이벤트 신청내역을 모아 한 번에 처리하는 기능. 그런데 신청 정보의 회원가입 항목이 체크되어 있는 경우에는 이벤트 참가자를 자동으로 사용자로 등록해줘야 한다.
- 하루치 이벤트 신청 내역을 처리하는 기능은 반드시 하나의 작업 단위로 처리돼야 한다. 이 기능을 EventService 클래스의 processDailyEventRegistration() 메서드로 구현했다고 하면, 이 메서드가 비즈니스 트랜잭션 경계가 됨.
- 그런데 processDailyEventRegistration()는 작업 중간에 사용자 등록을 해야 하므로 add() 메서드는 processDailyEventRegistration()에서 시작된 트랜잭션의 일부로 참여하게 된다.
- 트랜잭션 전파 기법을 사용했기 때문에 add()는 독자적인 트랜잭션 단위가 될 수도 있고, 다른 트랜잭션의 일부로 참여할 수도 있다.
- add() 메서드도 다른 메서드에서 만들어진 트랜잭션 경계에 포함될 수 있다. 이 덕분에 사용자 등록 기능이 다양한 비즈니스 트랜잭션에서 사용되더라도 add() 메서드는 하나만 존재하면 되고 불필요한 코드 중복이 일어나지 않는다.

 <img width="525" alt="스크린샷 2024-08-03 오후 11 12 56" src="https://github.com/user-attachments/assets/6c4b8c4a-d984-48f7-a9c3-aff31f53058b">

 - AOP를 이용해 코드 외부에서 트랜잭션의 기능을 부여해주고 속성을 지정할 수 있게 하는 방법을 **선언적 트랜잭션**이라고 한다.
 - 반대로 TransactionTemplate이나 개별 데이터 기술의 트랜잭션 API를 사용해 직접 코드 안에서 사용하는 방법은 **프로그램에 의한 트랜잭션**이라고 한다.
 - 물론 특별한 경우가 아니라면 선언적 방식을 사용하는 것이 바람직하다.

### 6.8.2 트랜잭션 동기화와 테스트
- AOP와 트랜잭션 추상화 덕분에 트랜잭션의 자유로운 전파와 그로 인한 유연한 개발이 가능했다.

#### 트랜잭션 매니저와 트랜잭션 동기화
- 트랜잭션 추상화 기술의 핵심은 트랜잭션 매니저와 트랜잭션 동기화다.
- `PlatformTransactionManager` 인터페이스를 구현한 트랜잭션 매니저를 통해 트랜잭션 기술의 종류에 상관없이 일관적인 트랜잭션 제어 가능
- 트랜잭션 동기화 기술이 있었기에 트랜잭션 정보를 저장소에 보관해뒀다가 DAO에서 공유할 수 있었다.
- 진행 중인 트랜잭션이 있는지 확인하고, 트랜잭션 전파 속성에 따라서 이에 참여할 수 있게 만들어주는 것도 트랜잭션 동기화 기술 덕분이다.
- 물론 특별한 이유가 없다면 트랜잭션 매니저를 직접 사용하는 코드를 작성할 필요는 없다. 그러나 특별한 이유가 있다면 트랜잭션 매니저를 이용해 트랜젝션에 참여하거나 트랜잭션을 제어하는 방법을 사용할 수도 있다.
- 트랜잭션 매니저 빈도 @Autowired로 가져와 테스트에서 사용할 수 있다.
```java
// 6-90. 트랜잭션 매니저를 참조하는 테스트
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "/test-applicationContext.xml")
public class UserServiceTest {
	@Autowired
	PlatformTransactionManager transactionManage;
```
```java
// 6-91. 간단한 테스트 메서드
@Test
public void transactionSync() {
	userService.deleteAll();

	userService.add(users.get(0));
	userService.add(users.get(1));
}
```
- transactionSync()이 실행되는 동안 몇 개의 트랜잭션이 만들어졌을까? 모든 메서드에는 트랜잭션을 적용했으니 당연히 3개다.

#### 트랜잭션 매니저를 이용한 테스트용 트랜잭션 제어
- 그렇다면 이 테스트 메서드에서 만들어지는 세 개의 트랜잭션을 하나로 통합할 수는 없을까?
- 세 개의 메소드 모두 트랜잭션 전파 속성이 REQUIRED이니 이 메소드들이 호출되기전에 트랜잭션이 시작되게만 한다면 가능하다.
- **테스트 메서드에서 UserService의 메서드를 호출하기 전에 트랜잭션을 미리 시작**해주면 된다.
- 트랜잭션의 전파는 트랜잭션 매니저를 통해 트랜잭션 동기화 방식이 적용되기 때문에 가능하다고 했다. 그렇다면 테스트에서 트랜잭션 매니저를 이용해 트랜잭션을 시작시키고 이를 동기화해주면 된다.

```java
// 6-92. 트랜잭션 매니저를 이용해 트랜잭션을 미리 시작하게 만드는 테스트
@Test
public void transactionSync() {
    DefaultTransactionDefinition txDefinition = new DefaultTransactionDefinition(); // 트랜잭션 정의는 기본값 사용
    TransactionStatus txStatus = transactionManager.getTransaction(txDefinition); // 트랜잭션 매니저에게 트랜잭션 요청

    userService.deleteAll();

    userService.add(users.get(0));
    userService.add(users.get(1));

    transactionManager.commit(status);
}
```
- 테스트 코드에서 트랜잭션 매니저를 이용해서 트랜잭션을 만들고 그 후에 실행되는 UserService의 메소드들이 같은 트랜잭션에 참여하게 만들 수 있다.
- 세 개의 메소드 모두 속성이 REQUIRED이므로 이미 시작된 트랜잭션이 있으면 참여하고 새로운 트랜잭션을 만들지 않는다.

#### 트랜잭션 동기화 검증
- 트랜잭션 속성 중에서 읽기전용과 제한시간 등은 처음 트랜잭션이 시작할 때만 적용되고 그 이후에 참여하는 메소드의 속성은 무시된다.
```java
// 6-93. 트랜잭션 동기화 검증용 테스트
public void transactionSync() {
    DefaultTransactionDefinition txDefinition = new DefaultTransactionDefinition(); 
	txDefinition.setReadOnly(true); // 읽기 전용 트랜잭션으로 정의

    TransactionStatus txStatus = transactionManager.getTransaction(txDefinition);
    

    userService.deleteAll();
    ...
}
```
- 위의 테스트를 실행하면 TransientDataAccessResourceException이 발생한다. 읽기 전용 트랜잭션에서 쓰기를 했기 때문이다.
- 스프링의 트랜잭션 추상화가 제공하는 트랜잭션 동기화 기술과 트랜잭션 전파 속성 덕분에 테스트도 트랙잭션으로 묶을 수 있다.
- JdbcTemplate과 같이 스프링이 제공하는 데이터 액세스 추상화를 적용한 DAO에도 동일한 영향을 미친다. JdbcTemplate은 트랜잭션이 시작된 것이 있으면 그 트랜잭션에 자동으로 참여하고, 없으면 트랜잭션 없이 자동커밋 모드로 JDBC 작업을 수행한다. 개념은 조금 다르지만 JdbcTemplate의 메소드 단위로 마치 트랜잭션 전파 속성이 REQUIRED인것 처럼 동작 한다고 볼 수 있다.

#### 롤백 테스트
- 롤백 테스트는 테스트 내의 모든 DB 작업을 하나의 트랜잭션 안에서 동작하게하고 테스트가 끝나면 무조건 롤백해버리는 테스트를 말한다.
```java
// 6-96. 롤백 테스트
@Test
public void transactionSync() throws InterruptedException {
    DefaultTransactionDefinition txDefinition = new DefaultTransactionDefinition();
    TransactionStatus txStatus = transactionManager.getTransaction(txDefinition);

    try {
        userService.deleteAll();

        userService.add(users.get(0));
        userService.add(users.get(1));
    } finally {
        transactionManager.rollback(txStatus);
    }
}
```
- 롤백 테스트는 DB 작업이 포함된 테스트가 수행돼도 DB에 영향을 주지 않기 때문에 장점이 많다.
  - 테스트용 데이터를 DB에 잘 준비해놓더라도 앞에서 실행된 테스트에서 DB의 데이터를 바꿔버리면 이후에 실행되는 테스트에 영향을 미칠 수 있다.
  - 이런 이유 때문에 롤백 테스트는 매우 유용하다. 롤백 테스트는 테스트를 진행하는 동안에 조작한 데이터를 모두 롤백하고 테스트를 시작하기 전 상태로 만들어주기 때문이다.
- 테스트에서 트랜잭션을 제어할 수 있기 때문에 얻을 수 있는 가장 큰 유익이 있다면 바로 롤백 테스트다.

### 6.8.3 테스트를 위한 트랜잭션 애너테이션
- 테스트 클래스에도 `@Transactional` 애너테이션을 부여해줄 수 있다.
- 그 외에도 스프링 컨텐스트 테스트에서 쓸 수 있는 유용한 애너테이션이 여러 개 있다.

#### @Transactional
- 물론 테스트에서 사용하는 `@Transactional`은 AOP를 위한 것은 아니다. 하지만 기본적인 동작방식과 속성은 UserService 등에 적용한 `@Transactional`과 동일하므로 이해하기 쉽고 편리하다.
- 트랜잭션 적용 여부를 확인해보고 싶다면 트랜잭션을 읽기 전용으로 바꾸고 테스트를 실행해 예외가 발생하는지 확인해보면 된다.
- `@Transactional`은 테스트 클래스 레벨에 부여할 수도 있다.

#### @Rollback
- 테스트용 트랜잭션은 테스트가 끝나면 자동으로 롤백된다.
- 그런데 테스트 메서드 안에서 진행되는 작업을 하나의 트랜잭션으로 묶고 싶기는 하지만 강제 롤백을 원하지 않을 수도 있다. 트랜잭션을 커밋시켜서 테스트에서 진행한 작업을 그대로 DB에 반영하고 싶다면 `@Rollback`을 사용하면 된다.
- `@Rollback`은 롤백 여부를 지정하는 값을 갖고 있다.
- `@Rollback`의 기본값은 true다. 따라서 트랜잭션은 적용되지만 롤백은 원치 않는다면 `@Rollback(false)`라고 해줘야 한다.

#### @TransactionConfiguration
- `@Rollback`은 메서드 레벨에서만 적용 가능
- 테스트 클래스의 모든 메서드에 트랜잭션을 적용하면서 모든 트랜잭션이 롤백되지 않고 커밋되게 하려면 어떻게 해야 할까?
-  `@TransactionConfiguration`을 사용하면 롤백에 대한 공통 속성을 지정할 수 있다.
-  디폴트 롤백 속성은 false로 해두고, 테스트 메서드 중에서 일부만 롤백을 적용하고 싶으면 메서드에 `@Rollback`을 부여해주면 된다.

```java
// 6-100. @TransactionConfiguration의 사용 예
@Runwith(SpringJUnit4Runner.class)
@ContextConfiguration(locations = "/test-applicationContext.xml")
@Transactional
@TransactionConfiguration(defaultRollback=false)
public class UserServiceTest {
	@Test
	@Rollback
	public void add() throws SQLException { ... }
	...
```

#### NonTransactional과 Propagation.NEVER
- 대부분의 메서드에 트랜잭션이 필요하다면 테스트 클래스에 `@Transactional`을 지정하는 것이 편리하다.
- 트랜잭션이 적용되면 안 되는 경우에는 해당 메서드에만 테스트 메서드에 의한 트랜잭션이 시작되지 않도록 만들어줄 수 있다.
- `@NonTransactional`을 테스트 메서드에 부여하면 `@Transactional` 설정을 무시하고 트랜잭션을 시작하지 않은 채로 테스트를 진행한다.
	- 그러나 `@NonTransactional`은 스프링 3.0에서 제거 대상이 됐다.
	- 스프링의 개발자들은 트랜잭션 테스트와 비 트랜잭션 테스트를 아예 클래스를 구분해서 만들도록 권장한다.
- `@NonTransactional` 대신 `@Transactional`의 트랜잭션 전파 속성을 사용하는 방법도 있다.
	- `@Transactional`을 다음과 같이 NEVER 속성으로 지정해주면 된다.
	```java
	@Transactional(propagation=Propagation.NEVER)
	```

#### 효과적인 DB 테스트
- 테스트 내에서 트랜잭션을 제어할 수 있는 네 가지 애너테이션을 잘 활용하면 DB가 사용되는 통합 테스트를 만들 때 매우 편리
- 일반적으로 의존, 협력 오브젝트를 사용하지 않고 고립된 상태에서 테스트를 진행하는 **단위 테스트**와 DB 같은 외부의 리소스나 여러 계층의 클래스가 참여하는 **통합 테스트**는 **아예 클래스를 구분**해서 따로 만드는 게 좋다.
- DB가 사용되는 통합 테스트를 별도의 클래스로 만들어둔다면 클래스 레벨에 `@Transactional`을 부여해준다.
- DB가 사용되는 통합 테스트는 가능한 한 롤백 테스트로 만드는 게 좋다.
- 애플리케이션의 모든 테스트를 한꺼번에 실행하는 빌드 스크립트 등에서 테스트 DB를 셋업해준다.
- 테스트는 어떤 경우에도 서로 의존하면 안된다. 어떤 순서로 진행되더라도 일정한 결과를 내야 한다.

## 6.9 정리
- 트랜잭션 경계설정 코드를 분리해서 별도의 클래스로 만들고 비즈니스 로직 클래스와 동일한 인터페이스를 구현하면 DI 확장 기능을 이용해 깔끔하게 분리된 **트랜잭션 부가기능**을 만들 수 있다.
- **목 오브젝트**를 활용하면 의존관계 속에 있는 오브젝트도 손쉽게 **고립된 테스트**로 만들 수 있다.
- DI를 이용한 트랜잭션 분리는 **데코레이터 패턴**과 **프록시 패턴**으로 이해될 수 있다.
- 번거로운 프록시 클래스 작성은 **JDK의 다이내믹 프록시**를 사용하면 간단하게 만들 수 있다.
- 다이내믹 프록시는 스태틱 팩토리 메서드를 사용하기 때문에 빈으로 등록하기 번거롭다. 따라서 **팩토리 빈**으로 만들어야 한다. 스프링은 **프록시 팩토리 빈**을 제공한다.
- 프록시 팩토리 빈의 설정이 반복되는 문제를 해결하기 위해 **자동 프록시 생성기**와 **포인트컷**을 활용할 수 있다.
- 포인트컷은 **AspectJ 포인트컷 표현식**을 사용해서 작성하면 편리하다.
- AOP는 OOP만으로는 모듈화하기 힘든 부가기능을 효과적으로 모듈화하도록 도와주는 기술이다.
- 스프링은 자주 사용하는 AOP 설정과 트랜잭션 속성을 지정하는 데 사용할 수 있는 **전용 태그**를 제공한다.
- AOP를 이용해 트랜잭션 속성을 지정하는 방법에는 **포인트컷 표현식**과 **메서드 이름 패턴을 이용하는 방법**과 **타깃에 직접 부여**하는 `@Transactional` 애너테이션을 사용하는 방법이 있다.
- `@Transactional`을 이용한 트랜잭션 속성을 테스트에 적용하면 손쉽게 DB를 사용하는 코드의 테스트를 만들 수 있다.
