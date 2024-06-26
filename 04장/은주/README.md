# 4장. 예외
## 4.1. 사라진 SQL EXCEPTION
### 4.1.1. 초난감 예외처리
#### 예외 블랙홀
```java
try {
    ...
} catch(SQLException e) {

}
```
- 예외를 잡고 아무것도 하지 않는 코드로, 굉장히 위험하다.
```java
} catch (Exception e) {
  System.out.println(e);
}

} catch (Exception e) {
  e.printStackTrace();
}
```
- 예외를 단순히 출력만 하는 것도 안된다. 
- 모든 예외는 적절히 `복구` 되든지, 작업을 `중단` 시키고 운영자/개발자에게 분명히 통보되어야 한다
- 굳이 예외를 잡아서 뭔가 조치를 취할 방법이 없다면 잡지 말아야 한다.
- 메소드에 throws SQLException 을 선언해서 메소드 밖으로 던지고, 자신을 호출한 코드에 **예외처리 책임을 전가해버리자**
  
#### 무의미하고 무책임한 throws
```java
public void method1() throws Exception {
  method2();
  ...
}

public void method2() throws Exception {
  method3();
  ...
}

public void method3() throws Exception ...
```
- 예외블랙홀 보단 낫지만 메소드 선언에 throws Exception을 기계적으로 붙이는 게 되면, 결과적으로 **적절한 처리를 통해 복구될 수 있는 예외상황도 제대로 다룰 수 있는 기회를 박탈당한다.**

### 4.1.2. 예외의 종류와 특징
- Error
  - 시스템에 비정상적인 상황이 발생했을 경우
  - 주로 자바 VM 에서 발생시키는 것이고, 애플리케이션 코드에서 잡으려고 하면 안된다.
  - OutOfMemoryError 나 ThreadDeath 같은 에러는 catch 로 잡아봤자 대응 방법이 없다.
- Exception 과 체크 예외
  ![alt text](image.png)
  - 애플리케이션 코드 작업 중에 예외상황 발생했을 경우에 사용된다
  - `체크 예외` : Exception 클래스의 서브 클래스, **RuntimeException 클래스 상속하지 않은 것**
  - `언체크 예외` : Exception 클래스의 서브 클래스, **RuntimeException 클래스 상속한 것**
  - 일반적으로 예외라 함은, 체크 예외라고 생각해도 된다
  - 체크 예외가 발생할 수 있는 메소드 사용 시 **반드시 예외를 처리하는 코드를 함께 작성**해야 한다
    - 그렇지 않으면 **컴파일 에러** 가 발생한다
- RuntimeException 과 언체크/런타임 예외
  - 런타임예외는 catch 문으로 잡거나 throws 로 선언하지 않아도 된다
  - 주로 프로그램의 오류가 있을 때 발생하도록 의도된 것으로, **피할 수 있지만 개발자가 부주의해서 발생할 수 있는 경우**에 발생하도록 만든 것이다.
  - 따라서 예상치 못한 예외상황에서 발생하는 게 아니기 때문에 굳이 catch 나 throws 를 사용하지 않아도 되도록 만든 것이다.
### 4.1.3. 예외처리 방법
#### 예외 복구
- 예외 상황을 파악하고, 문제를 해결해서 `정상 상태`로 돌려놓는 것이다
- 예외로 인해 기본 작업 흐름이 불가능하면 다른 작업 흐름으로 자연스럽게 유도해주면 예외를 복구했다고 볼 수 있다
- 예외처리 코드를 강제하는 체크 예외들은, **예외를 어떤 식으로든 복구할 가능성이 있는 경우에 사용** 한다
```java
int maxRetry = MAX_RETRY;

while(maxRetry --> 0) {
  try {
    ... // 예외가 발생할 수 있는 시도
    return; // 작업 성공
  }
  catch(SomeException e) {
    // 로그 출력, 정해진 시간만큼 대기
  }
  finally {
    // 리소스 반납, 정리 작업
  }
}
throw new RetryFailedException(); // 최대 재시도 횟수를 넘기면 직접 예외 발생
```
- 예외가 발생할 경우 최대횟수만큼 반복적으로 시도함으로써 예외상황에서 복구되게 할 수 있다.

#### 예외처리 회피
- 예외처리를 **자신이 담당하지 않고 자신을 호출한 쪽으로 던져버리는 것**
  - catch 문으로 예외를 잡은 후 로그를 남기고 throw (rethrow)
  - throws 문으로 선언
- 예외를 자신이 처리하지 않고 회피하는 방법이다
- 예외처리를 회피하려면 반드시 다른 오브젝트나 메소드가 예외를 대신 처리할 수 있게 던져주어야 한다
```java
public void add() throws SQLException {
  // JDBC API
}

public void add() throws SQLException {
  try {
    // JDBC API
  catch(SQLException e) {
    // 로그 출력
    throw e;
  }
}
```
- 콜백 오브젝트의 메소드는 SQLException 에 대한 예외를 회피하고 템플릿 레벨에서 처리하도록 던져준다.
- 하지만 콜백과 템플릿처럼 긴밀하게 역할을 분담하고 있는 관계가 아니라면 자신의 코드에서 발생하는 예외를 그냥 던지는 것은 무책임한 책임회피일 수 있다.
- 예외를 회피하는 것은 예외를 복구하는 것처럼 의도가 분명해야 한다.
  - 콜백/템플릿처럼 긴밀한 관계에 있는 다른 오브젝트에게 예외처리 책임을 분명히 지게 하거나,
  - **자신을 사용하는 쪽에서 예외를 다루는 게 최선의 방법**이라는 분명한 확신이 있어야 한다 

#### 예외 전환
- 예외 회피처럼 예외를 밖으로 던지지만, **발생한 예외를 그대로 넘기는 게 아니라 적절한 예외로 전환해서 던진다**
- 주로 2가지 목적으로 사용
  - 내부에서 발생한 예외를 그대로 던지는 것이 **예외상황에 대한 적절한 의미를 부여하지 못할 경우**에 의미를 분명하게 해줄 수 있는 예외로 변경하기 위해서이다.
    - SQLException은 DB 연결 실패, 쿼리의 실수, 중복된 아이디 존재 등 다양한 이유로 발생
    - 추가하려는 아이디가 이미 존재할 때 발생하는 예외는 DuplicateUserIdException 같은 예외로 바꿔서 던져주는 게 좋다
  - 의미가 분명한 예외가 던져지면, 서비스 계층 오브젝트에는 **적절한 복구 작업을 시도**할 수 있다
  ```java
  public void add(User user) throws DuplicatedUserIdException, SQLException{
      try{
          ...
      }catch(SQLException e){
          if(e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
              throw DuplicatedUserIdException();
          else 
              throw e;
      }
  }
  ```
  - 보통 전환하는 예외에 원래 발생한 예외를 담아서 `중첩 예외` 로 만드는 것이 좋다
    - 생성자 또는 initCause() 메소드를 통해 원인이 되는 예외를 내부에 담아서 던진다. 
    - 주로 **예외처리를 강제하는 체크 예외를 언체크 예외인 런타임 예외로 바꾸는 경우에 사용** 한다
    ```java
    catch(SQLException e){
        ...
        throw DuplicatedUserIdException(e);
        throw DuplicatedUserIdException().initCause(e);
    }
    ```
    - 대표적으로 EJBException 이 있는데, EJB 컴포넌트 코드에서 발생하는 대부분의 체크 예외는 비즈니스 로직상 의미있는 예외이거나 복구 가능한 예외가 아니다
      - 따라서 어차피 복구 불가능한 예외라면 **가능한 빨리 런타임 예외로 포장해 던져서 다른 계층 메소드 작성 시 불필요한 throws 선언이 들어가지 않도록** 해야 한다
      - 애플리케이션 로직 상에서 예외조건이나 예외 상황 발생할 수도 있는데, 이 때는 `체크예외` 사용하는 것이 적절하다.
        - 왜냐하면 비즈니스적 의미가 있는 예외는 이에 대한 **적절한 대응이나 복구작업이 필요**하기 때문이다.
      - 어차피 복구못할 예외라면 애플리케이션 코드에서 포장해서 던지고, 예외 처리 서비스 등을 이용하여 로그 기록 및 운영진에게 알리는 것이 바람직 함
    ```java
    try{
        OrderHome orderHome = EJBHomeFactory.getInstance().getOrderHome();
        Order order = orderhome.findByPrimaryKey(Integer id);
    } catch (NamingException ne){
        throw new EJBException(ne); //체크 예외 -> 런타임 예외
    }
    ```
### 4.1.4. 예외처리 전략 
#### 런타임 예외의 보편화
- 일반적으로 체크 예외가 `일반적인 예외`를 다루고, 언체크 예외는 `시스템 장애나 프로그램 상의 오류` 에 사용한다
  - 문제는 체크예외는 **복구할 가능성이 조금이라도 있는, 말 그대로 예외적 상황이기에** 자바는 catch 블록이나 throws 선언을 강제하고 있다
- 자바가 처음 만들어질 때 사용되던 애플릿, AWT, 스윙을 사용한 독립형 애플리케이션은 통제 불가능한 시스템 예외더라도, 상황을 복구해야 했다
- 하지만 자바 엔터프라이즈 서버환경은 다르다.
  - 독립형 애플리케이션과 달리 서버에서 예외 발생 시 작업을 일시 중지하고 사용자와 커뮤니케이션하며 예외상황을 복구할 수 있는 방법이 없다
  - 차라리 예외가 발생하지 않도록 차단하거나, 해당 요청 작업을 취소하고 서버 관리자에게 통보하는 게 낫다
- 자바의 환경이 서버로 이동하며 **체크 예외의 활용도와 가치가 점점 떨어지고 있다**
  - throws Exception 이 달린 아무런 의미없는 메소드들만 낳을 뿐이다
  - **대응 불가능한 체크 예외라면 빨리 런타임 예외로 전환해서 던지는 게 낫다**
- 예전에는 복구 가능성이 조금이라도 있다면 체크 예외로 만들었는데, 지금은 **항상 복구할 수 있는 예외가 아니라면 일단 언체크 예외로 만드는** 경향이 있다

#### add() 메소드의 예외처리
```java
public void add(User user) throws DuplicatedUserIdException, SQLException{
    try{
      ...
    } catch(SQLException e){
      if(e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
        throw DuplicatedUserIdException();
      else 
        throw e;
    }
}
```
- DuplicatedUserIdException 은 충분히 복구 가능한 예외니까 add() 사용부에서 잡아서 대응할 수 있다.
  - 하지만 SQLException 의 경우 대부분 복구 불가능한 예외이므로, throws 를 타고 전달되다가 애플리케이션 밖으로 던져질 것이다
  - 그럴 바에는 `런타임 예외` 로 포장해 던져서 그밖의 메소드가 신경쓰지 않게 해주는 게 낫다
- DuplicatedUserIdException 도 굳이 체크예외로 두지 않아도 된다.
  - add() 메소드 호출부보다 더 앞단의 오브젝트에서 다룰 수도 있고, **어디에서든 잡아서 처리할 수만 있다면 굳이 체크예외로 안만들고 런타임예외로 만드는 게 낫다**
  - 대신 add() 메소드는 명시적으로 DuplicatedUserIdException 를 던진다고 선언해야 개발자에게 의미있는 정보 전달이 가능하다

```java
public class DuplicateUserldException extends RuntimeException {
  public DuplicateUserldException(Throwable cause) {
    super(cause);
  }
}

public void add() throws DuplicatedUserIdException {
    try{
      ...
    } catch(SQLException e){
      if(e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
        throw DuplicatedUserIdException(e); // 예외 전환
      else 
        throw new RuntimeException(e); // 예외 포장
    }
}
```
- DuplicatedUserIdException 도 필요 없다면 신경쓰지 않도록 RuntimeException 을 상속한 `런타임 예외` 로 만들어 준다
- 이제 DuplicatedUserIdException 외에 시스템 예외에 해당하는 SQLException 은 `언체크 예외` 가 되었다
- 런타임 예외로 수정하여 언체크 예외로 만들기는 했지만 add() 사용부에서 아이디 중복 예외 처리하고 싶을 경우 활용할 수 있음을 알리기 위해 throws 선언에 포함한다
- **런타임 예외를 일반화해서 사용하면, 컴파일러가 예외처리를 강제하지 않으므로 사용에 더 주의를 기울일 필요도 없다**

#### 애플리케이션 예외
- 런타임 예외 중심의 전략은 `낙관적` 예외처리 기법이라고 할 수 있다
  - 복구할 수 있는 예외가 없다고 가정하고, 어차피 런타임 예외니까 시스템 레벨에서 처리할 것이며 꼭 필요한 경우 잡아서 복구/대응할 수 있으니 문제가 없다는 태도 기반
  - 직접 처리할 수 없는 예외가 대부분이어도 일단 잡고 보는 체크 예외의 `비관적` 접근 방법과 대비 된다
- `애플리케이션 예외` : 시스템 또는 외부 예외 상황이 원인이 아니라, **애플리케이션 자체 로직에 의해 의도적으로 발생시키고, 반드시 catch 해서 조치를 취하도록 요구하는 예외**
  - 출금 기능 메소드를 예외 상황에 대해 **비즈니스적 의미를 띤 예외를 던지도록** 만든다

### 4.1.5. SQLException 은 어떻게 됐나?
- SQLException 은 과연 복구 가능한 예외인가?
  - 코드레벨에서는 복구 방법이 없다.
  - 프로그램 오류 or 개발자의 부주의 or 통제불가능한 외부상황 때문에 발생
  - ex) SQL 문법 오류, 제약조건 위반, DB 서버 다운, 네트워크 불안정, DB 커넥션 풀 가득찬 경우
- 시스템 예외일 경우 애플리케이션 레벨에서 복구할 방법이 없고, 관리자/개발자에게 예외 발생 사실을 빠르게 알리는 방법 뿐이다
- 불필요한 기계적 throws 선언이 등장하도록 방치하지 않고, 가능한 빨리 언체크/런타임 예외로 전환해야 한다
- **스프링의 JdbcTemplate 은 이 예외처리 전략을 따르면서, JdbcTemplate 템플릿과 콜백 내에서 발생하는 모든 SQLException 을 런타임 예외인 DataAccessException 으로 포장해서 던져준다**
- 스프링 API 메소드에 정의된 대부분의 예외는 런타임 예외이다

## 4.2. 예외 전환
- 예외 전환의 2가지 목적
  - 런타임 예외로 포장해서 굳이 필요하지 않은 catch/throws 를 줄이는 것
  - 로우레벨 예외를 좀 더 의미 있고 추상화된 예외로 바꿔서 던지는 것

### 4.2.1. JDBC 의 한계
- 내부 구현은 DB마다 다르겠지만, JDBC 의 Connection, Statement, ResultSet 등의 `표준 인터페이스`를 통해 그 기능을 제공해주기 때문에 DB 종류와 관계없이 **일관된 방법** 으로 프로그램을 개발할 수 있다.
- JDBC API 가 DB 프로그램 개발 방법을 학습하는 부담은 줄여주지만, DB 를 자유롭게 변경해서 사용할 수 있는 유연한 코드를 보장해주진 못한다

#### 비표준 SQL
- SQL 은 어느정도 표준화된 언어이지만, 대부분의 DB는 표준을 따르지 않는 비표준 문법과 기능도 제공한다.
  - 특별한 기능을 제공하는 함수를 SQL 에 사용하려면 대부분 **비표준 SQL 문장** 이 만들어지고 DAO 코드에 들어가며, 해당 DAO는 특정 DB 에 대해 `종속적`인 코드가 되고 만다
- 이 문제의 해결책은 **호환 가능한 표준 SQL 만 사용하거나**, **DB별로 별도의 DAO 를 만들거나**, **SQL 을 외부에 독립시켜 DB에 따라 변경하며 사용** 하는 방법이 있다.
  - 하지만 1번째 경우는 현실성이 없고 2,3번 케이스가 적용하능하다

#### 호환성 없는 SQLException 의 DB 에러정보
- DB마다 SQL 이 다른 것뿐만 아니라 에러 종류와 원인도 제각각인데, JDBC 는 데이터처리 중 발생하는 예외를 그냥 SQLException 하나에 모두 담아버린다
  - 예외가 발생한 원인은 SQLException 내의 에러 코드와 SQL 상태정보를 참조해야 한다.
  - SQLException 의 getErrorCode() 로 가져올 수 있는 에러코드는 DB 벤더가 정의한 고유 에러코드를 사용하므로 DB 별로 모두 다르다
  - SQLException 은 **예외 발생 시 DB 상태를 담은 SQL 상태정보를 부가적으로 제공하기 위해** getSQLState() 메소드를 제공한다.
    - DB 별로 달라지는 에러코드를 대신하여 DB 에 독립적인 에러정보를 얻게 해준다
    - 하지만 DB의 JDBC 드라이버에서 SQLException 을 담을 상태코드를 **정확하게 만들어주지 않는다**는 문제가 있다.

### 4.2.2. DB 에러 코드 매핑을 통한 전환
- DB 종류가 바뀌더라도 DAO 를 수정하지 않으려면 위 2가지 문제를 해결해야 한다.
- SQLException 에 담긴 SQL 상태코드는 신뢰할 수 없으므로, DB에서 직접 제공해주는 **DB 에러코드를 참고해서 발생한 예외의 원인이 무엇인지 해석해주는 기능을 만들어야 한다**
- 키 값이 중복돼서 중복 오류가 발생하는 경우, MySQL (1062), Oracle (1), DB2 (-803) 이라는 에러코드를 받게 되는데, 키 중복으로 인한 SQLException 을 DuplicateKeyException 이라는 예외로 전환할 수 있다.
  - 문제는 DB 마다 에러코드가 제각각인데, DAO 나 JdbcTemplate 에서 일일이 DB 별 에러코드 종류를 확인하는 것은 번거롭다
- 스프링은 DataAccessException 이라는 SQLException 을 대체할 수 있는 런타임 예외를 정의할 뿐만 아니라 DataAccessException 의 서브클래스로 세분화된 예외클래스들을 정의하고 있다
- 스프링은 **DB별 에러코드를 분류해서 스프링이 정의한 예외클래스와 매핑해놓은 에러코드 매핑정보 테이블을 만들어두고 이를 이용**한다.
```xml
<bean id="Oracle" class="org.springframework.jdbc.support.SQLErrorCodes">
		<property name="badSqlGrammarCodes">
			<value>900,903,904,917,936,942,17006,6550</value>
		</property>
		<property name="invalidResultSetAccessCodes">
			<value>17003</value>
		</property>
		<property name="duplicateKeyCodes">
			<value>1</value>
		</property>
		<property name="dataIntegrityViolationCodes">
			<value>1400,1722,2291,2292</value>
		</property>
		<property name="dataAccessResourceFailureCodes">
			<value>17002,17447</value>
		</property>
		<property name="cannotAcquireLockCodes">
			<value>54,30006</value>
		</property>
		<property name="cannotSerializeTransactionCodes">
			<value>8177</value>
		</property>
		<property name="deadlockLoserCodes">
			<value>60</value>
		</property>
</bean>
```
- JdbcTemplate 은 SQLException 을 단지 런타임 예외인 DataAccessException 으로 포장하는게 아니라, DB 에러코드를 DataAccessException 계층구조 클래스 중 하나로 매핑해준다.
  - **드라이버나 DB 메타정보** 를 참고해서 DB 종류 확인하고, DB별로 준비된 위와 같은 매핑정보를 참고해서 적절한 예외클래스를 선택하기에, DB 가 달라져도 같은 종류 에러라면 동일한 예외를 받을 수 있는 것이다
  ```java
  public class DuplicateUserldException extends RuntimeException {
    public DuplicateUserldException(Throwable cause) {
      super(cause);
    }
  }

  public void add() throws DuplicatedUserIdException {
      try{
        ...
      } catch(SQLException e){
        if(e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
          throw DuplicatedUserIdException(e); // 예외 전환
        else 
          throw new RuntimeException(e); // 예외 포장
      }
  }
  ```
  - 위와 같았던 코드가 아래와 같이 간단해지며, **DB를 변경하더라도 동일한 예외가 던져지는 것이 보장된다**
    - JdbcTemplate 안에서 DB 별로 미리 준비된 에러코드와 비교해서 적절한 예외를 던지기 때문이다.
  ```java
  public void add() throws DuplicateKeyException {...}
  ```

### 4.2.3. DAO 인터페이스와 DataAccessException 계층구조
- DataAccessException 은 **의미가 같은 예외라면 데이터 액세스 기술 종류와 관계없이 `일관된` 예외가 발생하도록 만들어준다**
  - 데이터 액세스 기술에 **독립적인 추상화된 예외를 제공하는 것**

#### DAO 인터페이스와 구현의 분리
- DAO 를 따로 만들어서 사용하는 이유는, 데이터 액세스 로직을 담은 코드를 **성격이 다른 코드에서 분리**해놓기 위함이다
  - 분리된 DAO 는 `전략패턴`을 활용하여 **구현방법을 변경해서 사용**할 수 있게 만들기 위함이다.
  - 따라서 DAO 사용부는 DAO 가 내부에서 어떤 데이터 액세스 기술을 사용하는지 신경쓰지 않아도 된다
- 이런면에서 DAO 는 `인터페이스` 를 사용해 **구체적인 클래스 정보와 구현 방법을 감추고, DI 를 통해 제공되도록 만드는 것이 바람직** 하다
```java
public interface UserDao {
  public void add(User user); // JdbcTemplate에서 런타임 예외로 감싸준 덕에 throws가 없다.
}
```
- 기술에 독립적인 이상적인 DAO 인터페이스는 위와 같지만, 실제 위와 같은 메소드 선언은 사용할 수 없다.
  - DAO 에서 데이터 액세스 기술의 API 가 예외를 던지기 때문이다.
- 자바에서 사용하는 데이터 접근 API가 바뀌면 인터페이스마저 바꿔주어야 하는 일이 생길 수 있다.
```java
public interface UserDao {
  public void add(User user) throws SQLException; // JDBC
  public void add(User user) throws PersistentException; // JPA
  public void add(User user) throws HibernateException; // Hibernate
  public void add(User user) throws JdoException; // JDO
}
```
- 결국 인터페이스로 메소드 구현은 추상화하더라도, **구현 기술마다 던지는 예외가 다르기에 메소드 선언이 달라진다는 문제가 발생** 한다.
- 위 예시 중 JDBC 를 제외하고는 SQLException 같은 체크 예외 대신 런타임 예외를 사용하므로 throws 선언이 불필요하다
  - JDBC API 도 DAO 메소드 내에서 런타임 예외로 포장해서 던져주면 처음 의도대로 `public void add(User user);` 선언이 가능해진다 
- 하지만 문제는 데이터 액세스 기술이 달라지면 **같은 상황에서도 다른 종류의 예외가 던져진다는 점** 이다.
  - 따라서 DAO 사용하는 클라이언트 입장에서 DAO 기술에 따라 예외처리 방법이 달라지며, 결국 DAO 기술에 의존적이 될 수밖에 없다
  - 단지 인터페이스로 추상화하고, 일부 기술에서 발생하는 체크 예외를 런타임 예외로 전환하는 것만으로는 불충분하다

#### 데이터 액세스 예외 추상화와 DataAccessException 계층 구조
- 스프링은 자바의 다양한 데이터 액세스 기술을 사용할 때 발생하는 예외들을 추상화해서 DataAccessException 계층구조 내에 정리해놓았다.
  - DataAccessException 는 자바의 주요 데이터 액세스 기술에서 발생할 수 있는 **대부분의 예외를 추상화하고 있다**
- `인터페이스` 사용, `런타임 예외전환` 과 함께 **DataAccessException 예외 추상화** 를 적용하면 데이터 액세스 기술과 구현 방법에 독립적인 이상적인 DAO 를 만들 수 있다.

### 4.2.4. 기술에 독립적인 UserDao 만들기
#### DataAccessException 활용 시 주의 사항
- 스프링을 활용하면 DB 종류나 데이터 액세스 기술에 상관없이 키 값이 중복된 경우 동일한 예외가 발생하리라 기대한다.
- 하지만 DuplicateKeyException 은 아직까지 JDBC 를 이용하는 경우에만 발생하며, Hibernate 나 JPA 사용 시에도 동일한 예외가 발생할 거라 기대하지만 실제로 다른 예외가 던져진다.
  - 이유는 SQLException 에 담긴 DB 에러코드를 바로 해석하는 JDBC 와 달리, JPA / Hibernate / JDO 등에서는 **각 기술이 재정의한 예외를 가져와 스프링이 최종적으로 DataAccessException 으로 변환** 하는데, DB 에러코드와 달리 이런 예외들은 세분화 되어 있지 않기 때문이다.
- DataAccessException 이 기술에 상관없이 어느정도 추상화된 공통 예외로 변환해주지만, 근본적 한계로 인해 완벽함을 기대할 수는 없다.
  - DataAccessException 을 잡아서 처리하는 코드를 만들 때는 미리 학습테스트로 실제 전환되는 예외 종류를 확인할 필요가 있다.
  - 만약 DAO 에서 사용하는 기술의 종류와 상관없이 동일한 예외를 얻고 싶다면, 직접 예외를 정의해두고 각 DAO 의 메소드에서 좀 더 상세한 예외전환을 해줄 필요가 있다.

### 4.3. 정리
- 예외는 복구하거나, 예외처리 오브젝트로 의도적으로 전달하거나, 적절한 예외로 전환해야 한다.
- **복구할 수 없는 예외는 가능한 빨리 런타임 예외로 전환**하는 게 바람직하다
- 애플리케이션 로직을 담기 위한 예외는 체크 예외로 만든다.
- JDBC 의 SQLException 은 대부분 복구 불가능한 예외이므로 런타임 예외로 포장해야 한다
- SQLException 의 에러코드는 DB에 종속되기 때문에, DB에 독립적인 예외로 전환될 필요가 있다
- 스프링은 DataAccessException을 통해 **DB에 독립적으로 적용 가능한 추상화된 런타임 예외 계층을 제공** 한다
- DAO 를 데이터 액세스 기술에서 독립시키려면, 인터페이스 도입 / 런타임 예외 전환 / 기술에 독립적인 추상화된 예외로 전환이 필요하다