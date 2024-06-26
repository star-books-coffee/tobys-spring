# 01장. 오브젝트와 의존 관계

## 들어가며
- 스프링의 핵심철학은 자바 엔터프라이즈 기술의 혼란 속에서 잃어버렸던 객체지향 기술의 진정한 가치를 회복시키고, 그로부터 객체지향 프로그래밍이 제공하는 폭넓은 혜택을 누릴 수 있도록 기본으로 돌아가자는 것이다.
- 그래서 스프링이 가장 관심을 많이 두는 대상이 오브젝트이며, 스프링을 이해하려면 먼저 오브젝트에 깊은 관심을 가져야 한다.
- 1장에서는 스프링이 관심을 갖는 대상인 오브젝트의 설계와 구현, 동작원리에 더 집중하기를 바란다.

## 1.1 초난감 DAO
사용자 정보를 JDBC API를 통해 DB에 저장하고 조회할 수 있는 간단한 DAO를 하나 만들어보자.
> DAO : DB를 사용해 데이터를 조회하거나 조작하는 기능을 전담하도록 만든 오브젝트

### 1.1.1 User
사용자 정보를 저장할 때는 자바빈 규약을 따르는 오브젝트를 이용하면 편하다.


> **자바빈**
> 
> 원래 비주얼 툴에서 조작 가능한 컴포넌트. 자바의 주력 개발 플랫폼이 웹 기반의 엔터프라이즈 방식으로 바뀌면서 비주얼 컴포넌트로서 자바빈은 인기를 잃어핬지만, 자바빈의 몇 가지 코딩 관례는 JSP 빈, ELB와 같은 표준 기술과 자바빈 스타일의 오브젝트를 사용하는 오픈소스 기술을 통해 계속 이어져 왔다. 이제는 자바빈이라고 말하면 비주얼 컴포넌트라기보다는 다음 두 가지 관례를 따라 만들어진 오브젝트를 가리킨다. 간단히 빈이라고 부르기도 한다.
>
> - 디폴트 생성자 : 자바빈은 파라미터가 없는 디폴트 생성자를 갖고 있어야 한다.
> - 프로퍼티 : 자바빈이 노출하는 이름을 가진 속성. setter와 getter를 통해 수정 또는 조회할 수 있다.

```java
// 1-1. 사용자 정보 저장용 자바빈 User 클래스
package springbook.user.domain;

public class User {
  String id;
  String name;
  String password;

  public String getId() {
    return id;
  }
  public setId() {
    this.id = id;
  }
  public String getName() {
    return name;
  }
  public String getPassword() {
    return password;
  }
  public void setPassword(String password) {
    this.password = password;
  }
}

```

이제 User 오브젝트가 담긴 정보가 실제로 보관될 DB의 테이블을 하나 만들어보자. 테이블 이름은 USER로 User 클래스의 프로퍼티와 동일하게 구성한다.


- 테이블 필드 내역
  | 필드명 | 타입 | 설정 |
  |------|-----|-----|
  | id | VARCHAR(10) | Primary Key |
  | Name | VARCHAR(20) | Not Null |
  | Password | VARCHAR(20) | Not Null |
- SQL문
  ```sql
  create table users (
    id varchar(10) primary key,
    name varchar(20) not null,
    password varchar(10) not null
  )
  ```

### 1.1.2 UserDao
사용자 정보를 DB에 넣고 관리할 수 있는 DAO 클래스를 만들어보자.
JDBC를 이용하는 작업의 일반적인 순서는 다음과 같다.
- DB 연결을 위한 Connection을 가져온다.
- SQL을 담은 Statement(또는 PreparedStatement)를 만든다.
- 만들어진 Statement를 실행한다.
- 조회의 경우 SQL 쿼리의 실행결과를 ResultSet으로 받아서 정보를 저장할 오브젝트(여기는 User)에 옮겨준다.
- 작업 중에 생성된 Connection, Statement, ResultSet 같은 리소스는 작업을 마친 후 반드시 닫아준다.
- JDBC API가 만드는 예외를 잡아서 직접 처리하거나, 메소드에 throws를 선언해서 예외가 발생하면 메소드 밖으로 던지게 한다.
```java
// 1-2. JDBC를 이용한 등록과 조회 기능이 있는 UserDao 클래스
package springbook.user.dao;
...
public class UserDao {
  public void add(User user) throws ClassNotFoundException, SQLException {
    Class.forName("com.mysql.jdbc.Driver");
    Connection c = DriverManager.getConnection(
      "jdbc:mysql://localhost/springbook", "spring", "book");

    PreparedStatement ps = c.preparedStatement(
      "insert into users(id, name, password) values(?,?,?)");
    ps.setString(1, user.getId());
    ps.setString(2, user.getName());
    ps.setString(3, user.getPassword());

    ps.executeUpdate();

    ps.close();
    c.close();
  }

  public User get(String id) throws ClassNotFoundException, SQLException {
    Class.forName("com.mysql.jdbc.Driver");
    Connection c = DriverManager.getConnection(
      "jdbc:mysql://localhost/springbook", "spring", "book");

    PreparedStatement ps = c.preparedStatement(
      "select * from users where id = ?");

    ps.setString(1, id);

    ResultSet rs = ps.executeQuery();
    rs.next();
    User user = new User();
    user.setId(rs.getString("id"));
    user.setId(rs.getString("name"));
    user.setId(rs.getString("password"));

    rs.close();
    ps.close();
    c.close();

    return user;
  }
}
```

이 클래스가 제대로 동작하는지 어떻게 확인할 수 있을까? 웹 어플리케이션을 만들어 서버에 배치하고, 웹 브라우저를 통해 DAO 기능을 사용하기에는 배보다 배꼽이 더 크다.

### 1.1.3 main()을 이용한 DAO 테스트 코드
```java
// 1-3. 테스트용 main() 메서드
public static void main(String[] args) thorws ClassNotFoundException, SQLException {
  UserDao dao = new UserDao();

  User user = new User();
  user.setId("whiteship");
  user.setName("백기선");
  user.setPassword("married");

  dao.add(user);

  System.out.println(user.getId() + " 등록 성공");

  User user2 = dao.get(user.getId());
  System.out.println(user2.getName());
  System.out.println(user2.getPassword());

  System.out.println(user2.getId() + " 조회 성공");
}
```
main() 클래스를 실행하면 다음과 같은 테스트 성공 메시지를 얻을 수 있다.
```
whiteship 등록 성공
백기선
married
whiteship 조회 성공
```

그런데 UserDao 클래스 코드에는 사실 여러가지 문제가 있다. 정말 한심한 코드다.

## 1.2 DAO의 분리
### 1.2.1 관심사의 분리
- 객체지향의 세계에서는 모든 것이 변한다(사용자의 비즈니스 프로세스와 그에 따른 요구사항, 기술, 운영환경 등).
- 그래서 개발자가 객체를 설계할 때 가장 염두에 둬야할 사항은 바로 미래의 변화를 어떻게 대비할 것인가다.
- 객체지향 설계와 프로그래밍이 절차적 프로그래밍 패러다임에 비해 초기에 좀 더 번거로운 작업을 요구하는 이유는 변화에 효과적으로 대처할 수 있다는 기술적인 특징 때문이다.
- **변경에 필요한 작업을 최소화하고, 그 변경이 다른 곳에 문제를 일으키지 않게 하기 위해서는 `분리와 확장`을 고려한 설계가 필요하다.**

- 먼저 분리에 대해서 생각해보자.
- 모든 변경과 발전은 한 번에 한 가지 관심사항에 집중해서 일어난다. 문제는, 그에 따른 작업은 한 곳에 집중되지 않는 경우가 많다는 점이다.
- 프로그래밍의 기초 개념 중 하나인 `관심사의 분리`를 객체지향에 적용해보면 다음과 같다.
  - 관심이 같은 것끼리는 하나의 객체 안으로 또는 친한 객체로 모이게 한다.
  - 관심이 다른 것은 가능한 따로 떨어져서 서로 영향을 주지 않도록 분리한다.
 

### 1.2.2 커넥션 만들기의 추출
UserDao에 구현된 add() 메소드에서 적어도 세 가지의 관심사항을 발견할 수 있다.
#### UserDao의 관심사항
1. DB와 연결을 위한 커넥션 가져오기
2. 사용자 등록을 위해 DB에 보낼 SQL 문장을 담을 Statement를 만들고 실행하기
3. 작업이 끝나면 사용한 리소스인 Statement와 Connection 오브젝트를 닫아주기

#### 중복 코드의 메서드 추출
커넥션을 가져오는 중복된 DB 연결 코드를 getConnection()이라는 이름의 독립적인 메서드로 만든다.
```java
// 1-4. getConnection() 메서드를 추출해서 중복을 제거한 UserDao
public void add(User user) throws ClassNotFoundException, SQLException {
  Connection c = getConnection();
  ...
}

public User get(String id) throws ClassNotFoundException, SQLException {
  Connection = getConnection();
  ...
}

private Connection getConnection() throws ClassNotFoundException, SQLException {
  Class.forName("com.mysql.jdbc.Driver");
  Connection c = DriverManager.getConnection(
    "jdbc:mysql://localhost/springbook", "spring", "book");
  return c;
}
```
- 관심의 종류에 따라 코드를 구분해놓았기 때문에 한 가지 관심에 대한 변경이 일어날 경우 그 관심이 집중되는 부분의 코드만 수정하면 된다.

#### 변경사항에 대한 검증 : 리팩토링과 테스트
- main() 메서드를 여러 번 실행하면 id 값이 중복되기 때문에 두 번째부터는 무조건 예외가 발생한다.
  => main() 메서드 재실행 전 User 테이블의 사용자 정보를 모두 삭제해주어야 한다.
- 기능에는 영향을 주지 않으면서 코드의 구조만 변경하는 작업을 `리팩토링`이라고 한다
- 공통의 기능을 담당하는 메서드로 중복된 코드를 뽑아내는 것을 `메소드 추출(extract method)`기법이라고 한다.

### 1.2.3 DB 커넥션 만들기의 독립
- 만약 발전을 거듭한 UserDao가 인기를 끌어 N사와 D사에서 UserDao를 구매하겠다는 주문이 들어왔다고 하자.
- 문제
  - N 사와 D 사가 각기 다른 종류의 DB를 사용하고 DB 커넥션을 가져오는 데 있어 독자적으로 만든 방법을 적용하고 싶어한다.
  - 고객에게 소스를 직접 공개하고 싶지는 않다. 미리 컴파일된 클래스 바이너리 파일만 제공하고 싶다.
- 어떻게 소스코드를 N사와 D사에 제공해주지 않고도 고객 스스로 원하는 DB 커넥션 생성 방식을 적용해가면서 UserDao를 사용하게 할 수 있을까?

#### 상속을 통한 확장
- getConnection()을 추상 메서드로 만들고 추상 클래스인 UserDao를 N사와 D사에 판매한다.
- UserDao 클래스를 상속해서 NUserDao와 DUserDao라는 서브 클래스를 만든다.
- 기존에는 같은 클래스에서 다른 메서드로 분리됐던 DB 커넥션 연결이라는 관심을 **상속을 통해 서브클래스로 분리**해버리는 것이다.
<img width="533" alt="스크린샷 2024-03-23 오후 8 11 35" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/51d8038f-6dcc-4756-898d-c0b4bb134e6c">

```java
// 1-5. 상속을 통한 확장 방법이 제공되는 UserDao
public abstract class UserDao {
  public void add(User user) throws ClassNotFoundException, SQLException {
    Connection c = getConnection();
    ...
  }
  
  public User get(String id) throws ClassNotFoundException, SQLException {
    Connection = getConnection();
    ...
  }
  
  private abstract Connection getConnection() throws ClassNotFoundException, SQLException;
}

public class NUserDao extends UserDao {
  public Connection getConnection() throws ClassNotFoundException, SQLException {
    // N사 DB Connection 생성 코드
  }
}

public class DUserDao extends UserDao {
  public Connection getConnection() throws ClassNotFoundException, SQLException {
    // D사 DB Connection 생성 코드
  }
}
```
- UserDao가 담당하는 기능 : DAO의 핵심 기능인 어떻게 데이터를 등록하고 가져올 것인가라는 관심을 담당
- NUserDao, DUserDao가 담당하는 기능 : DB 연결 방법은 어떻게 할 것인가라는 관심을 담당
- 이렇게 슈퍼클래스에 기본적인 로직의 흐름을 만들고, 그 기능의 일부를 추상 메서드나 오버라이딩이 가능한 `protected` 메서드 등으로 만든 뒤 서브클래스에서 이런 메서드를 필요에 맞게 구현해서 사용하도록 하는 방법을 `템플릿 메서드 패턴(template method pattern)`이라고 한다.
- UserDao의 서브클래스의 getConnection() 메서드처럼 서브 클래스에서 구체적인 오브젝트 생성 방법을 결정하게 하는 것을 `팩토리 메서드 페턴(factory method pattern)`이라고 부르기도 한다.
  
  <img width="605" alt="스크린샷 2024-03-24 오전 10 25 47" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/aa2eba6e-4164-40c4-8b9e-577ec847a068">
  UserDao에 적용된 팩토리 

> **템플릿 메서드 패턴**
>
> 변하지 않는 기능은 슈퍼 클래스에 만들어두고 자주 변경되어 확장할 기능은 서브 클래스에서 만들도록 한다.
> - 슈퍼 클래스에서는 미리 추상 메서드 또는 오버라이드 가능한 메서드를 정의해두고 이를 활용해 코드의 기본 알고리즘을 담고 있는 템플릿 메서드를 만든다.
> - 슈퍼 클래스에서 디폴트 기능을 정의해두거나 비워뒀다가 서브클래스에서 선택적으로 오버라이드할 수 있도록 만들어둔 메서드를 `훅(hook) 메서드`라고 한다.
> - 서브 클래스에서는 추상 메서드를 구현하거나, 훅 메서드를 오버라이딩하는 방법을 활용해 기능의 일부를 확장한다.
> ```java
> public abstract class Super {
>   public void templateMethod() {
>     // 기본 알고리즘 코드
>     hookMethod();
>     abstractMethod();
>     ...
>   }
>   protected void hookMethod() { } // 선택적으로 오버라이드 가능한 훅 메서드
>   public abstract void abstractMethod(); // 서브 클래스에서 반드시 구현해야 하는 추상 메서드
> }
>
> public class Sub1 extends Super {
>   protected void hookMethod() {
>     ...
>   }
>   public void abstractMethod() {
>     ...
>   }
> }


> **팩토리 메서드 패턴**
>
> - 슈퍼 클래스 코드에서는 서브 클래스에서 구현할 메서드를 호출해서 필요한 타입의 오브젝트를 가져와 사용한다.
> - 이 메서드는 주로 인터페이스 타입으로 오브젝트를 리턴하므로 서브 클래스에서 정확히 어떤 클래스의 오브젝트를 만들어 리턴할지는 슈퍼 클래스에서는 알지 못한다.

- 하지만 상속을 이용한 방법은 많은 한계점이 있다.
  - 자바는 클래스의 다중상속을 허용하지 않는다. 단지, 커넥션 객체를 가져오는 방법을 분리하기 위해 상속 구조로 만들어버리면, 후에 다른 목적으로 UserDao에 상속을 적용하기 힘들다.
  - 상속을 통한 상하위 클래스의 관계는 생각보다 밀접하다. 슈퍼 클래스 내부의 변경이 있을 때 모든 서브클래스를 함께 수정하거나 다시 개발해야할 수도 있다.
  - 확장된 기능인 DB 커넥션을 생성하는 코드를 다른 DAO 클래스에 적용할 수 없다. 만약 UserDao 외의 DAO 클래스들이 계속 만들어진다면 상속을 통해 만들어진 getConnection() 구현 코드가 매 DAO 클래스마다 중복돼서 심각한 문제가 발생할 것이다.

## 1.3 DAO의 확장
- 추상 클래스를 만들고 이를 상속한 서브 클래스에서 변화가 필요한 부분을 바꿔서 쓸 수 있게 만든 이유는 이렇게 변화의 성격이 다른 것을 분리해서, 서로 영향을 주지 않은 채로 각각 필요한 시점에 독립적으로 변경할 수 있게 하기 위해서다.
- 그러나 여러가지 단점이 많은, 상속이라는 방법을 사용했다는 사실이 불편하게 느껴진다.
### 1.3.1 클래스의 분리
- 이번에는 아예 상속관계도 아닌 완전히 독립적인 클래스로 만들어보겠다.
- DB 커넥션과 관련된 부분을 서브클래스가 아니라, 아예 별도의 클래스에 담는다. 그리고 이렇게 만든 클래스를 UserDao가 이용하게 하면 된다.
  <img width="587" alt="스크린샷 2024-03-24 오전 11 37 14" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/aabb5f20-1a47-476c-8466-e6a8971aa57f">
```java
// 1-6. 독립된 SimpleConnectioinMaker를 사용하게 만든 UserDao
public class UserDao {
  private SimpleConnectionMaker simpleConnectionMaker;

  public UserDao() {
    // 상태를 관리하는 것도 아니니 한 번만 만들어 인스턴스 변수에 저장해두고 메서드에서 사용하게 한다.
    simpleConnectionMaker = new SimpleConnectionMaker();
  }

  public void add(User user) throws ClassNotFoundException, SQLException {
    Connection c = simpleConnectionMaker.makeNewConnection();
    ...
  }

  public User get(String id) throws ClassNotFoundException, SQLException {
    Connection c = simpleConnectionMaker.makeNewConnection();
    ...
  }
}
```
DB 커넥션 생성 기능을 독립시킨 SimpleConnectionMaker는 리스트 1-7과 같이 만든다.
```java
// 1-7. 독립시킨 DB 연결 기능인 SimpleConnectionMaker
package springbook.user.dao;
...
public class SimpleConnectionMaker {
  public Connection makeNewConnection() throws ClassNotFoundException, SQLException {
    Class.forName("com.mysql.jdbc.Driver");
    Connection c = DriverManager.getConnection(
      "jdbc:mysql://localhost/springbook", "spring", "book");
    return c;
  }
}
```
- 이번에는 다른 문제가 발생했다. N 사와 D 사에 UserDao 클래스만 공급하고 상속을 통해 DB 커넥션 기능을 확장해서 사용하게 했던 게 다시 불가능해졌다.
  - `UserDao`의 코드가 `SimpleConnectionMaker`라는 특정 클래스에 종속되어 있기 때문에 상속을 사용했을 때처럼 UserDao 코드의 수정 없이 DB 커넥션 생성 기능을 변경할 방법이 없다.
- 다른 방식으로 DB 커넥션을 제공하는 클래스를 사용하기 위해서는 UserDao 소스코드의 다음 줄을 직접 수정해야 한다.
  ```java
  simpleConnectionMaker = new SimpleConnectionMaker();
  ```
- 이렇게 클래스를 분리한 경우에도 상속을 이용했을 때와 마찬가지로 자유로운 확장이 가능하게 하려면 두 가지 문제를 해결해야 한다.
  1. SimpleConnectionMaker의 메서드 문제
     만약 D 사에서 만든 DB 커넥션 제공 클래스는 openConnection()이라는 메서드 이름을 사용했다면 UserDao에 있는 add(), get() 메서드의 커넥션을 가져오는 코드를 다음과 같이 일일히 변경해야 한다.
     ```java
     Connection c = simpleConnectionMaker.openConnection();
     ```
 2. DB 커넥션을 제공하는 클래스가 어떤 것인지 UserDao가 구체적으로 알고 있어야 한다
    UserDao에서 SimpleConnectionMaker라는 클래스 타입의 인스턴스 변수까지 정의해놓고 있으니, N 사에서 다른 클래스를 구현하면 어쩔 수 없이 UserDao 자체를 다시 수정해야 한다.

- 이런 문제의 근본적인 원인은 UserDao가 바뀔 수 있는 정보, 즉 DB 커넥션을 가져오는 클래스에 대해 너무 많이 알고 있기 때문이다.

### 1.3.2 인터페이스의 도입
- 이런 문제를 해결할 가장 좋은 해결책은 두 개의 클래스가 서로 긴밀하게 연결되어 있지 않도록 중간에 추상적인 느슨한 연결고리를 만들어주는 것이다.
- 인터페이스를 도입하면 UserDao는 자신이 사용할 클래스가 어떤 것인지 몰라도 된다. 단지 인터페이스를 통해 원하는 기능을 사용하면 된다.
  <img width="552" alt="스크린샷 2024-03-24 오전 11 59 50" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/74273027-5137-486a-b1f6-61cac910bf92">

```java
// 1-8. ConnectionMaker 인터페이스
package springbook.user.dao;
...
public interface ConnectionMaker {
  public Connection makeConnection() throws ClassNotFoundException, SQLException;
}
```
- 고객에게 납품을 할 떄는 UserDao 클래스와 함께 ConnectionMaker 인터페이스도 전달한다.
- 그리고 D 사의 개발자라면 다음과 같이 ConnectionMaker를 구현한 클래스를 만들고, 자신들의 DB 연결 기술을 이용해 DB 커넥션을 가져오도록 메서드를 작성해주면 된다.
```java
// 1-9. ConnectionMaker 구현 클래스
package springbook.user.dao;
...
public class DConnectionMaker implements ConnectionMaker {
  ...
  public Connection makeConnection() throws ClassNotFoundException, SQLException {
    // D 사의 독자적인 방법으로 Connection을 생성하는 코드
  }
}
```
- 리스트 1-10은 특정 클래스 대신 인터페이스를 사용해서 DB 커넥션을 가져와 사용하도록 수정한 UserDao 코드다.
```java
// 1-10. ConnectionMaker 인터페이스를 사용하도록 개선한 UserDao
public class UserDao {

  // 인터페이스를 통해 오브젝트에 접근하므로 구체적인 클래스 정보를 알 필요가 없다.
  private ConnectionMaker connectionMaker;

  public UserDao() {
    // 앗! 그런데 여기에는 클래스 이름이 나오네!
    connectionMaker = new DConnectionMaker();
  }

  public void add(User user) throws ClassNotFoundException, SQLException {
    // 인터페이스에 정의된 메서드를 사용하므로 클래스가 바뀐다고 해도 메서드 이름이 변경될 걱정은 없다.
    Connection c = connectionMaker.makeConnection();
    ...
  }

  public User get(String id) throws ClassNotFoundException, SQLException {
    Connection c = connectionMaker.makeConnection();
    ...
  }
}
```
- DB 커넥션을 제공하는 클래스에 대한 구체적인 정보는 모두 제거가 가능했지만, 초기에 한 번 어떤 클래스의 오브젝트를 사용할지 결정하는 생성자의 코드는 제거되지 않고 남아있다.

### 1.3.3 관계설정 책임의 분리
- 인터페이스를 이용한 분리에도 불구하고 여전히 UserDao 변경 없이는 DB 커넥션 기능의 확장이 자유롭지 못하다.
- 그 이유는 UserDao 안에 분리되지 않은, **또 다른 관심사항이 존재하고 있기 때문이다.**
- `new DConnectionMaker()`라는 코드는 그 자체로 충분히 독립적인 관심사를 담고 있다.
  - UserDao가 어떤 ConnectionMaker 구현 클래스의 오브젝트를 이용하게 할지를 결정하는 것이다.
  - UserDao와 UserDao가 사용할 ConnectionMaker의 특정 구현 클래스 사이의 관계를 설정해주는 것에 관한 관심이다.

- 두 개의 오브젝트가 있고 한 오브젝트가 다른 오브젝트의 기능을 사용한다면, 사용되는 오브젝트를 서비스, 사용하는 오브젝트를 클라이언트라고 부를 수 있다. (UserDao - 서비스 / UserDao를 사용하는 오브젝트 - 클라이언트)
- 바로 이 UserDao의 클라이언트 오브젝트가 제3의 관심사항인 UserDao와 ConnectionMaker 구현 클래스의 관계를 결정해주는 기능을 분리해서 두기에 적절한 곳이다.

- 먼저 UserDao가 어떤 ConnectionMaker의 구현 클래스를 사용할지 결정하도록 만들어보자. 즉 UserDao 오브젝트와 특정 클래스로부터 만들어진 ConnectionMaker 오브젝트 사이에 관계를 설정해주는 것이다.
- 오브젝트 사이의 관계는 런타임시에 한쪽이 다른 오브젝트의 레퍼런스를 갖고 있는 방식으로 만들어진다.
  - 이는 클래스 사이의 관계를 설정해주는 건 아니다. 클래스 사이의 관계가 만들어진다는 것은 한 클래스가 인터페이스 없이 다른 클래스를 직접 사용한다는 뜻이다.
  - 예를 들면, 다음 코드는 DConnectionMaker의 오브젝트 레퍼런스를 UserDao의 connectionMaker 변수에 넣어서 사용하게 함으로써, 두 개의 오브젝트가 '사용'이라는 관계를 맺게 해준다.
  ```
  connectionMaker = new DConnectionMaker();
  ```
- 오브젝트 사이의 관계가 만들어지려면 일단 오브젝트가 있어야 하직접 생성자를 호출해서 직접 오브젝트를 만드는 방법도 있지만 외부에서 준 것을 가져오는 방법도 있다.
-   외부에서 만든 오브젝트를 전달받으려면 메서드 파라미터나 생성자 파라미터를 이용하면 된다.
- UserDao의 모든 코드는 ConnectionMaker 인터페이스 외에는 어떤 클래스와도 관계를 가져서는 안되게 해야 한다.
- UserDao 오브젝트가 DConnectionManager 오브젝트르 사용하게 하려면 두 클래스의 오브젝트 사이에 런타임 사용관계, 링크, 또는 의존관계라고 불리는 관계를 맺어주면 된다. 아래 그림은 런타임 시점의 오브젝트 간 관계를 나타내는 오브젝트 다이어그램이다. 모델링 싱[는 없었던, 그래서 코드에는 보이지 않던 관계가 오브젝트로 만들어진 후에 생성되는 것이다.
  <img width="415" alt="스크린샷 2024-03-24 오후 12 40 40" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/d76f06a2-d5e5-4715-bc91-adb05685f468">

- UserDao의 클라이언트의 책임은 아래와 같은 구조의 클래스들을 이용해 위와 같이 런타임 오브젝트 관계를 갖는 구조로 만들어주는 것이다.
  <img width="552" alt="스크린샷 2024-03-24 오전 11 59 50" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/74273027-5137-486a-b1f6-61cac910bf92">
- 책임을 떠넘겨보자.
  - 현재는 UserDao()의 main() 메서드가 UserDao의 클라이언트라고 볼 수 있다. 아예 UserDaoTest라는 이름의 클래스를 만들어 main() 메서드를 UserDaoTest로 옮겨보자.
  - 그리고 리스트 1-11과 같이 생성자를 수정해주면 UserDao에는 DConnectionMaker가 사라진다.
    ```java
    // 1-11. 수정한 생성자
    public UserDao(ConnectionMaker connectionMaker) {
      this.connectionMaker = connectionMaker;
    }
    ```
    - UserDaoTest는 UserDao와 ConnectionMaker 구현 클래스와의 런타임 오브젝트 관계를 설정하는 책임을 담당한다.
    ```java
    public class UserDaoTest {
      public static void main(String[] args) throws ClassNotFoundExceptoin, SQLException {
      ConnectionMaker connectionMaker = new DConnectionMaker();

      UserDao dao = new UserDao(connectionMaker);
      ...
      }
    }
- 인터페이스를 도입하는 방법은 다른 DAO 클래스에서도 ConnectionMaker의 구현 클래스들은 그대로 적용할 수 있기 때문에 상속에 비해 훨씬 더 유연한다.
- 아래 그림은 UserDao가 사용할 ConnectionMaker 클래스를 선정하는 책임을 UserDaoTest가 담당하고 있는 구조를 보여준다.
  <img width="541" alt="스크린샷 2024-03-24 오후 12 58 45" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/e6fe221f-a7d9-4081-a5f4-7aa2bc518e28">

### 1.3.4 원칙과 패턴
지금까지 초난감 DAO 코드를 개선해온 결과를 객체지향 기술의 여러가지 이론을 통해 설명하려고 한다.
#### 개방 폐쇄 원칙
깔끔한 설계를 위해 적용 가능한 객체지향 설계 원칙 중 하나다.
> '클래스의 모듈은 확장에는 열려 있어야 하고 변경에는 닫혀 있어야 한다'
- UserDao는 DB 연결 방법이라는 기능을 확장하는 데는 열려 있다. UserDao에 전혀 영향을 주지 않고도 얼마든지 기능 확장 가능하다.
- UserDao 자신의 핵심 기능을 구현한 코드는 그에 변화에 받지 않고 유지할 수 있으므로 변경에는 닫혀 있다.

#### 높은 응집도와 낮은 결합도
개방 폐쇄 원칙은 높은 응집도와 낮은 결합도(high coherence and low coupling)라는 소프트웨어 개발의 고전적인 원리로도 설명이 가능하다.
- 높은 응집도
  - 응집도가 높다는 것은 변화가 일어날 때 해당 모듈에서 변하는 부분이 크다는 것으로 설명할 수 있다.
  - 즉, 변경이 일어날 때 모듈의 많은 부분이 함께 바뀐다면 응집도가 높다고 말할 수 있다.
- 낮은 결합도
  - 높은 응집도보다 더 민감한 원칙
  - 낮은 결합도, 즉 느슨한 연결은 관계를 유지하는 데 꼭 필요한 최소한의 방법만 간접적인 형태로 제공하고, 나머지는 서로 독립적이고 알 필요도 없게 만들어주는 것이다.
  - 결합도가 낮아지면 변화에 대응하는 속도도 높아지고, 구성이 깔끔해진다. 또한 확장하기에도 매우 편리하다.
  - 여기서 결합도란 '하나의 오브젝트가 변경이 일어날 때에 관계를 맺고 있는 다른 오브젝트에게 변화를 요구하는 정도'라고 설명할 수 있다.

#### 전략 패턴
- 개선한 UserDaoTest-UserDao-ConnectionMaker 구조를 디자인 패턴의 시각으로 보면 전략 패턴(Strategy Pattern)에 해당한다고 볼 수 있다.
- 자신의 기능 맥락(context)에서, 필요에 따라 변경이 필요한 알고리즘을 인터페이스를 통해 통째로 외부로 분리시키고, 이를 구현한 구체적인 알고리즘 클래스를 바꿔서 사용할 수 있게 하는 디자인 패턴이다.
- UserDao는 전략 패턴의 컨텍스트에 해당한다.
  - 컨텍스트는 자신의 기능을 수행하는 데 필요한 기능 중에서 변경 가능한, DB 연결 방식이라는 알고리즘을 ConnectionMaker라는 인터페이스로 정의하고, 이를 구현한 클래스, 즉 번략을 바꿔가면서 사용할 수 있게 분리했다.
 
## 1.4 제어의 역전

### 1.4.1 오브젝트 팩토리
- 현재 UserDaoTest는 UserDao의 기능이 잘 작동하는지 테스트하려고 만들었지만 엉겹결에 어떤 ConnectionMaker 구현 클래스를 사용할지 결정하는 기능도 떠맡게 되었다.
- 관심사를 분리하자.
  - UserDao와 ConnectionMaker 구현 클래스의 오브젝트를 만드는 역할
  - 그렇게 만들어진 두 개의 오브젝트가 연결돼서 사용할 수 있도록 관계를 맺어주는 역할
#### 팩토리
- 객체의 생성 방법을 결정하고 그렇게 만들어진 오브젝트를 오브젝트를 돌려주는 클래스를 만들어보겠다.
- 이런 일을 하는 오브젝트를 `팩토리(factory)`라고 부른다.
```java
// 1-14. UserDao의 생성 책임을 맡은 팩토리 클래스
package springbook.user.dao;
...
public class DaoFactory {
  public UserDao userDao() {
    ConnectionMaker connectionMaker = new DConnectionMaker();
    UserDao userDao = new UserDao(connectionMaker);
    return userDao;
  }
}
```
```java
// 1-15. 팩토리를 사용하도록 수정한 UserDaoTest
public class UserDaoTest {
  public static void main(String[] args) throws ClassNotFoundException, SQLException {
    UserDao dao = new DaoFactory.userDao();
    ...
  }
}
```

#### 설계도로서의 팩토리
- UserDao와 ConnetionMaker가 실질적인 로직을 담당하는 컴포넌트라면, DaoFactory는 컴포넌트의 구조와 관계를 정의한 설계도와 같은 역할을 한다.
- 이런 작업이
 애플리케이션 전체에 걸쳐서 일어난다면 컴포넌트의 의존관계에 대한 설계도와 같은 역할을 하게될 것이다.
 <img width="554" alt="스크린샷 2024-03-24 오후 11 38 47" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/5cf03ae7-4ee7-44be-8396-41f03f3b743e">
- 이제 N사와 D사에 UserDao를 공급할 때, DaoFactory도 같이 제공하되, ConnetionMaker의 구현 클래스를 변경할 수 있도록 소스코드 형태로 제공한다. 
- 팩토리를 통해 애플리케이션의 컴포넌트 역할을 하는 오브젝트와 애플리케이션의 구조를 결정하는 오브젝트를 분리할 수 있다.

### 1.4.2 오브젝트 팩토리의 이용
- DaoFactory에 UserDao가 아닌 다른 DAO의 생성 기능을 넣게 되면 ConnectionMaker 구현 클래스의 오브젝트를 선정하고 생성하는 코드가 메서드마다 반복된다.
```java
// 1-16 DAO 생성 메서드 추가로 인해 발생하는 중복
public class DaoFactory {
  public UserDao userDao() {
    return new UserDao(new DConnectionMaker());
  }

  public AccountDao accountDao() {
    return new AccountDao(new DConnectionMaker());
  }

  public MessageDao messageDao() {
    return new MessageDao(new DConnectionMaker());
  }
}
```

- ConnectionMaker의 구현 클래스를 결정하고 오브젝트를 만드는 코드를 별도의 메서드 뽑아내자.
반복된다.
```java
// 1-17. 생성 오브젝트 코드 수정
public class DaoFactory {
  public UserDao userDao() {
    return new UserDao(new ConnectionMaker());
  }

  public AccountDao accountDao() {
    return new AccountDao(new ConnectionMaker());
  }

  public MessageDao messageDao() {
    return new MessageDao(new ConnectionMaker());
  }
  
  public ConnectionMaker connectionMaker() {
    // 분리해서 중복을 제거한 ConnectionMaker 타입 오브젝트 생성 코드
    return new DConnectionMaker();
  }
}
```

### 제어권 이전을 통한 제어관계 역전
- 제어의 역전이란, 간단히 프로그램의 제어 구조가 뒤바뀌는 것
- 제어의 역전에서는 오브젝트가 자신이 사용할 오브젝트를 스스로 선택하지도 생성하지도 않는다. 또 자신이 어떻게 만들어지고 어디서 사용되는지를 알 수 없다.
- 모든 제어 권한을 자신이 아닌 다른 대상에게 위임하기 때문이다.
- main()과 같은 엔트리 포인트를 제외하면 모든 오브젝트는 이렇게 위임받은 제어권한을 갖는 특별한 오브젝트에 의해 결정되고 만들어진다.
- 서블릿이나 JSP, EJB처럼 컨테이너 안에서 동작하는 구조는 간단한 방식이긴 하지만 제어의 역전 개념이 적용되어 있다.
- 템플릿 메서드 패턴에서도, 추상 메서드를 상속한 서브 클래스는 자신의 메서드가 언제 어떻게 사용될지는 모른다. 
  - 즉, 제어권을 상위 템플릿 메서드에 넘기고 자신은 필요할 때 호출되어 사용하게 하도록 한다는 제어의 역전의 개념을 발견할 수 있다.
- 프레임워크도 제어의 역전이 적용된 대표적인 기술이다. 
- 자연스럽게 관심을 분리하고 책임을 나누고 유연하게 확장 가능한 구조로 만들기 위해 DaoFactory를 도입했던 과정이 IoC를 적용하는 과정이었다.
- IoC를 애플리케이션 전반에 걸쳐 본격적으로 사용하려면 스프링 같은 IoC 프레임워크의 도움을 받는 편이 유리하다.
- 스프링은 IoC를 모든 기능의 기초가 되는 기반기술로 삼고 있으며, IoC를 극한으로 적용하고 있는 프레임워크다.

## 1.5 스프링의 IoC
- 스프링의 핵심을 담당하는 건 바로 빈 팩토리 또는 애플리케이션 컨텍스트라고 불리는 것이다.
- 이 두가지는 DaoFactory가 하는 일을 좀 더 일반화한 것이라 설명할 수 있다.
### 1.5.1 오브젝트 팩토리를 이용한 스프링 IoC
#### 애플리케이션 컨텍스트와 설정 정보
- 스프링에서는 스프링이 제어권을 가지고 직접 만들고 관계를 부여하는 오브젝트를 빈(bean)이라고 부른다.
  - 자바빈 또는 엔터프라이즈 자바빈(EJB)에서 말하는 빈과 비슷한 오브젝트 단위의 애플리케이션 컴포넌트
  - 스프링 컨테이너가 생성과 관계 설정, 사용 등을 제어해주는 제어의 역전이 적용된 오브젝트를 가리킨다.
- 스프링애서는 빈의 설정과 관계 설정 같은 제어를 담당하는 IoC 오브젝트를 빈 팩토리(bean factory)라고 부른다.
- 보통 빈 팩토라보다는 이를 확장한 애플리케이션 컨텍스트(application context)를 더 많이 사용한다.
  - IoC 방식을 따라 만들어진 일종의 빈 팩토리다.
  - 별도의 정보를 참고해서 빈(오브젝트)의 생성, 관계 설정 등의 제어 작업을 총괄한다.
  > 빈 팩토리와 애플리케이션 컨텍스트는 동일하다고 생각하면 된다. 빈 팩토리라고 말할 때는 빈을 생성하고 관계를 설정하는 IoC의 기본 기능에 초점을 맞춘 것이고, 애플리케이션 컨텍스트라고 말할 때에는 애플리케이션 전반에 걸쳐 모든 구성요소의 제어 작업을 담당하는 IoC 엔진이라는 의미가 좀 더 부각된다고 보면 된다.
- 기존 DaoFactory 코드에는 설정 정보, 예를 들어 어떤 클래스의 오브젝트를 생성하고 어디에서 사용하도록 연결해줄 것인가에 관한 정보가 평범한 자바 코드로 만들어져있다.
- 애플리케이션 컨텍스트는 직접 이런 정보를 담고 있진 않다. 대신 별도의 설정정보를 담고 있는 무엇인가를 가져와 이를 활용하는 범용적인 IoC 엔진 같은 것이라고 볼 수 있다.
- 앞서 말한 설계도라는 게 바로 이런 애플리케이션 컨텍스트와 그 설정 정보를 말한다고 보면 된다.
- 그 자체로는 애플리케이션 로직을 담당하지는 않지만 IoC 방식을 이용해 애플리케이션 컴포넌트를 생성하고, 사용할 관계를 맺어주는 등의 책임을 담당하는 것이다.

#### DaoFactory를 사용하는 애플리케이션 컨텍스트
- `@Configration` 애노테이션을 사용하여 설정정보 추가
- `@Bean` 애노테이션을 사용하여 오브젝트를 만들 수 있게 함
- 두 가지 애노테이션만으로 스프링 프레임워크의 빈 팩토리 또는 애플리케이션 컨텍스트가 IoC 방식의 기능을 제공할 설정정보가 됨

```java
// 1-18. 스프링 빈 팩토리가 사용할 설정 정보를 담은 DaoFactory 클래스
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
...
@Configuration // 애플리케이션 컨텍스트 또는 빈 팩토리가 사용할 설정정보라는 표시
public class DaoFactory {
  @Bean // 오브젝트 생성을 담당하는 IoC용 메서드라는 표시
  public UserDao userDao() {
    return new UserDao(connectionMaker());
  }

  @Bean
  public ConnectionMaker connectionMaker() {
    return new DConnectionMaker();
  }
}
```
- 이제 DaoFactory를 설정 정보로 사용하는 애플리케이션 컨텍스트를 만들어보자.
- 애플리케이션 컨텍스트는 `ApplicationContext` 타입의 오브젝트다.
- A`pplicationContext`를 구현한 클래스는 여러 가지가 있다.
  -  DaoFactory처럼 @Configuration이 붙은 자바코드를 설정 정보로 사용하려면 `AnnotationConfigApplicationContext`를 이용하면 된다.
```java
public class UserDaoTest {
  public static void main(String[] args) throws ClassNotFoundException, SQLException {
  ApplicationContext context =
      new AnnotationConfigApplicationContext(Daofactory.class);
  UserDao dao = context.getBean("userDao", UserDao.class);
}
```
- getBean() 메서드는 ApplicationContext가 관리하는 오브젝트를 요청하는 메서드다.

- 스프링의 기능을 사용했으니 필요한 라이브러리를 추가해줘야 한다.
- 스프링 배포판을 다운로드하거나 lib 폴더에서 필요한 파일을 가져다 프로젝트에 추가하고 클래스패스에 포함시켜줘야 한다.
### 1.5.2 애플리케이션 컨텍스트의 동작방식
- 기존의 오브젝트 팩토리에 대응되는 것이 스프링의 애플리케이션 컨텍스트다. 스프링에서는 이것을 IoC 컨테이너라고 하기도 하고, 스프링 컨테이너 혹은 빈 팩토리 라고 부르기도 한다.
- DaoFactory가 UserDao를 비롯한 DAO 오브젝트를 생성하고 DB 생성 오브젝트와 관계를 맺어주는 제한적인 역할을 하는 데 반해, 애플리케이션 컨텍스트는 애플리케이션에서 IoC를 적용해서 관리할 모든 오브젝트에 대한 생성과 관계 설정을 담당한다.


<img width="595" alt="스크린샷 2024-03-24 오후 11 53 44" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/6f898944-83ac-4d41-a599-494b17033957">


- 애플리케이션 컨텍스트는 DaoFactory 클래스를 설정 정보로 등록해두고 @Bean이 붙은 메서드 이름을 가져와 빈 목록을 만들어둔다.
- 클라이언트가 애플리케이션 컨텍스트의 getBean() 메서드를 호출하면 자신의 빈 목록에서 요청한 이름이 있는지 찾고, 있다면 빈을 생성하는 메서드를 호출해서 오브젝트를 생성시킨 후 클라이언트에 돌려준다.
- DaoFactory를 오브젝트 팩토리로 직접 사용했을 때와 비교해서 애플리케이션 컨텍스트를 사용했을 때 얻을 수 있는 장점은 다음과 같다.


  - **클라이언트는 구체적인 팩토리 클래스를 알 필요가 없다.**
    - 기존 팩토리 방식은 어떤 팩토리 클래스를 사용하는지 알아야 하고, 필요할 때마다 팩토리 오브젝트를 생성해야 함
  애플리케이션 컨텍스트를 이용하면 일관된 방식으로 원하는 오브젝트를 가져올 수 있음
    - XML처럼 단순한 방법을 사용하여 IoC 설정정보를 만들 수 있음
  - **애플리케이션 컨텍스트는 종합 IoC 서비스를 제공해준다.**
    - 오브젝트 생성 방식, 시점, 전략 등을 다르게 가져갈 수 있음
    - 자동생성, 후처리, 정보의 조합, 설정 방식의 다변화, 인터셉팅 등 오브젝트를 효과적으로 활용할 수 있는 다양한 기능 제공
    - 기반기술 서비스나 외부 시스템과의 연동을 컨테이너 차원에서 제공
  - **애플리케이션 컨텍스트는 빈을 검색하는 다양한 방법을 제공한다.**
    - getBean() 메소드를 사용하여 빈을 찾을 수 있음
    - 타입만으로 빈을 검색하거나 특별한 애노테이션 설정이 되어잇는 빈을 찾을 수 있음
### 1.5.3 스프링 IoC의 용어 정리
- **빈(Bean)**
  - 스프링이 IoC 방식으로 관리하는 오브젝트
  - 스프링을 사용하는 애플리케이션에서 만들어지는 모든 오브젝트가 전부 빈은 아님
  - 스프링이 직접 생성과 제어를 담당하는 오브젝트만이 빈임
- **빈 팩토리(Bean Factory)**
  - 스프링의 IoC를 담당하는 핵심 컨테이너
  - 빈을 등록, 생성, 조회, 관리 등의 역할을 함
  - 보통 빈 팩토리를 사용하지 않고 빈 팩토리를 확장한 애플리케이션 컨텍스트를 이용
  - `BeanFactory`는 빈 팩토리가 구현하고 있는 가장 일반적인 인터페이스
- **애플리케이션 컨텍스트(Application Context)**
  - 빈 팩토리를 확장한 IoC 컨테이너
  - 기본적 기능은 빈 팩토리와 동일
  - 추가로 스프링이 제공하는 각종 부가 서비스 제공
  - 애플리케이션 컨텍스트라고 할 때는 스프링이 제공하는 애플리케이션 지원 기능을 모두 포함한 것
  - ApplicationContext는 애플리케이션 컨텍스트가 구현해야 하는 기본적인 인터페이스
  - ApplicationContext는 BeanFactory를 상속받고 있음
  - 애플리케이션 컨텍스트 오브젝트는 보통 하나의 애플리케이션에 여러 개가 존재
    -> 이를 통틀어서 스프링 컨테이나라고도 함
- **설정정보/설정 메타정보(Configration metadata)**
  - 애플리케이션 컨텍스트 또는 빈 팩토리가 IoC를 적용하기 위해 사용하는 메타정보
  - 형상정보, 청사진(blue print) 라고도 함
  - 주로 IoC 컨테이너에 의해 관리되는 애플리케이션 오브젝트를 생성하고 구성할 때 사용 됨
- **컨테이너(container) 또는 IoC 컨테이너**
  - 애플리케이션 컨텍스트나 빈 팩토리를 컨테이너 또는 IoC 컨테이너 라고도 함
  - 컨테이너 말 자체가 IoC 개념을 담고 있음
- **스프링 프레임워크**
  - IoC 컨테이너, 애플리케이션 컨텍스트를 포함해서 스프링이 제공하는 모든 기능을 통틀어 말할 때 스프링 프레임워크라 함
  - 줄여서 스프링이라 함

## 1.6 싱글톤 레지스트리와 오브젝트 스코프
- 스프링의 애플리케이션 컨텍스트는 기존에 직접 만들었던 오브젝트 팩토리와는 중요한 차이점이 있다.
- 먼저 DaoFactory의 userDao() 메서드를 두 번 호출해서 리턴되는 UserDao 오브젝트를 비교해보자. 이 두 개는 같은 오브젝트일까?
- 코드를 보면 매번 userDao 메서드를 호출할 때마다 new 연산자에 의해 새로운 오브젝트가 만들어지게 되어있다. 생성된 오브젝트를 콘솔에 출력해보자. 오브젝트를 직접 출력하면 오브젝트별로 할당되는 고유한 값이 출력된다. 이 값이 같으면 동일한 오브젝트임을 알 수 있다.
```java
  // 1-20 직접 생성한 DaoFactory 오브젝트 출력 코드
  DaoFactory factory = new DaoFactory();
  UserDao dao1 = factory.userDao();
  UserDao dao2 = factory.userDao();

  System.out.println(dao1);
  System.out.println(dao2);
```
```
// 출력 결과
springbook.dao.UserDao@118f375
springbook.dao.UserDao@117a8bd
```
  - 출력 결과에서 알 수 있듯이, 두 개는 각기 다른 값을 가진 동일하지 않은 오브젝트다. 즉 오브젝트가 두 개 생겼다는 사실을 알 수 있다.
- 이번엔 스프링의 애플리케이션 컨텍스트에 DaoFactory를 설정정보로 등록하고 getBean() 메서드를 이용해 userDao라는 이름으로 등록된 오브젝트를 가져와보자.

```java
// 1-21. 스프링 컨텍스트로부터 가져온 오브젝트 출력 코드
ApplicationContext context =
      new AnnotationConfigApplicationContext(Daofactory.class);

UserDao dao3 = context.getBean("userDao", UserDao.class);
UserDao dao4 = context.getBean("userDao", UserDao.class);

System.out.println(dao3);
System.out.println(dao4);
```
- 다음 코드를 실행한 결과는 다음과 같다.
```
springbook.dao.UserDao@ee22f7
springbook.dao.UserDao@ee22f7
```
- 스프링은 여러 번에 걸쳐 빈을 요청하더라도 매번 동일한 오브젝트를 돌려준다. 왜 그럴까?
### 1.6.1 싱글톤 레지스트리로서의 애플리케이션 컨텍스트
- 애플리케이션 컨텍스트는 오브젝트 팩토리와 비슷하게 동작하는 IoC 컨테이너다. 그러면서 동시에 싱글톤을 저장하고 관리하는 `싱글톤 레지스트리(singleton registry)`이기도 하다.
- 스프링은 별다른 설정을 하지 않으면 내부에서 생성하는 빈 오브젝트를 모두 싱글톤으로 만든다.
#### 서버 애플리케이션과 싱글톤
- 스프링이 싱글톤으로 빈을 만드는 이유는 스프링이 주로 적용되는 대상이 자바 엔터프리이즈 기술을 사용하는 서버 환경이기 때문이다.
- 매번 클라이언트에서 요청이 올 때마다 각 로직을 담당하는 오브젝트를 만들어서 사용한다고 생각해보자.
  - 아무리 자바의 오브젝트 생성과 가비지 컬렉션 성능이 좋아졌다고 한들 이렇게 부하가 걸리면 서버가 감당하기 힘들다.
- 그래서 엔터프라이즈 분야에서는 서비스 오브젝트 라는 개념을 일찍부터 사용해왔다.
- 서블릿은 자바 엔터프라이즈 기술의 가장 기본이 되는 서비스 오브젝트라고 할 수 있다. 스펙에서 강제하지는 않지만, 서블릿은 대부분 멀티스레드 환경에서 싱글톤으로 동작한다.
  - 서블릿 클래스당 하나의 오브젝트만 만들어두고, 사용자의 요청을 담당하는 여러 스레드에서 하나의 오브젝트를 공유해 동시에 사용한다.
- 이렇게 애플리케이션 안에 제한된 수, 대개 한 개의 오브젝트만 만들어서 사용하는 것이 싱글톤 패턴의 원리다. 따라서 서버환경에서는 서비스 싱글톤의 사용이 권장된다.
- 하지만 디자인 패턴에 소개된 싱글톤 패턴은 사용하기가 까다롭고 여러가지 문제점이 있다. 심지어 이런 싱글톤 패턴을 안티패턴(anti pattern)이라고 부르는 사람도 있다.

#### 싱글톤 패턴의 한계
자바에서 싱글톤을 구현하는 방법은 보통 이렇다.
- 클래스 밖에서는 오브젝트를 생성하지 못하도록 생성자를 `private`로 만든다.
- 생성된 싱글톤 오브젝트를 저장할 수 있는 자신과 같은 타입의 스태틱 필드를 정의한다.
- 스태틱 팩토리 메서드인 `getInstance()`를 만들고 이 메서드가 최초로 호출되는 시점에서 한 번만 오브젝트가 만들어지게 한다. 생성된 오브젝트는 스태틱 필드에 저장된다. 또는 스태틱 필드의 초기값으로 오브젝트를 미리 만들어둘 수도 있다.
- 한 번 오브젝트(싱글톤)가 만들어지고 난 후에는 `getInstance()` 메서드를 통해 이미 만들어져 스태틱 필드에 저장해둔 오브젝트를 넘겨준다.
```java
// 1-22. 싱글톤 패턴을 적용한 UserDao
public class UserDao {
  private static UserDao INSTANCE;
  ...
  private UserDao(ConnectionMaker connectionMaker) {
    this.connectionMaker = connectionMaker;
  }

  public static synchronized UserDao getInstance() {
    if(INSTANCE == null) INSTANCE = new UserDao(???);
    return INSTANCE;
  }
}
```
- 싱글톤을 위한 코드가 추가되고 나니 코드가 상당히 지저분해졌다는 느낌이 든다.
- private로 바뀐 생성자는 외부에서 호출할 수 없기 때문에 DaoFactory에서 UserDao를 생성하며 ConnectionMaker 오브젝트를 넣어주는 게 이제는 불가능해졌다.
- 일반적으로 싱글톤 패턴 구현 방식에는 다음과 같은 문제가 있다.
  **- private 생성자를 갖고 있기 떄문에 상속할 수 없다.**
    - 싱글톤 패턴은 오직 싱글톤 클래스 자신만이 자기 오브젝트를 만들도록 제한한다.
    - private 생성자를 가진 크래스는 상속이 불가능하다. 따라서 객체지향의 장점인 상속과 이를 이용한 다형성을 적용할 수 없다.
    - 상속과 다형성 같은 객체지향의 특징이 적용되지 않는 스태틱 필드와 메서들르 사용한다.
  **- 싱글톤은 테스트하기 힘들다**
    - 싱글톤은 만들어지는 방식이 제한적이기 때문에 테스트에서 사용될 때 목 오브젝트 등으로 대체하기 힘들다.
    - 싱글톤은 초기화 과정에서 생성자 등을 통해 사용할 오브젝트를 다이내믹하게 주입하기도 힘들기 때문에 필요한 오브젝트는 직접 오브젝트를 만들어 사용할 수 밖에 없다. 이런 경우 테스트용 오브젝트로 대체하기가 힘들다.
      - 테스트는 엔터프라이즈 개발의 핵심인데 태스트를 만드는 데 지장이 있다는 건 큰 단점이다.
**- 서버 환경에서는 싱글톤이 하나만 만들어지는 것을 보장하지 못한다**
    - 서버에서 클래스 로더를 어떻게 구성하고 있느냐에 따라 싱글톤 클래스임에도 하나 이상의 오브젝트가 만들어질 수 있다.
    - 여러 개의 JVM에 분산돼서 설치가 되는 경우에도 각각 독립적으로 오브젝트가 생긴다.
 **- 싱글톤의 사용은 전역 상태를 만들 수 있기 때문에 바람직하지 못하다**
    - 싱글톤은 사용하는 클라이언트가 정해져 있지 않다.
    - 싱글톤의 스태틱 메서드를 통해 언제든지 접근 가능 -> 자연스럽게 전역 상태(global state)로 사용되기 쉽다.
    - 아무 객체나 자유롭게 접근하고 수정하고 공유할 수 있는 전역 상태를 갖는 것은 객체지향 프로그래밍에서는 권장되지 않는다.
#### 싱글톤 레지스트리
- 스프링은 서버 환경에서 싱글톤이 만들어져서 서비스 오브젝트 방식으로 사용되는 것은 적극 지지한다.
- 하지만 자바의 기본적인 싱글톤 패턴의 구현 방식은 여러가지 단점이 있기 때문에, 스프링은 직접 싱글톤 형태의 오브젝트를 만들고 관리하는 기능을 제공한다. => `싱글톤 레지스트리(singleton registry)`
- 스프링 컨테이너는 싱글톤 생성, 관리, 공급하는 싱글톤 관리 컨테이너이기도 하다.
- 싱글톤 레지스트리는 스태틱 메서드와 private 생성자를 사용해야 하는 평범한 자바 클래스를 싱글톤으로 활용하게 해준다.
  - 평범한 자바 클래스라도 IoC 방식의 컨테이너를 사용해서 제어권을 컨테이너에게 넘기면 싱글톤 방식으로 만들어져 관리되게 할 수 있다.
  - 스프링의 싱글톤 레지스트리 덕분에 싱글톤 방식으로 사용될 애플리케이션 클래스라도 public 생성자를 가질 수 있다.
  - 싱글톤으로 사용돼야 하는 환경이 아니라면 간단히 오브젝틀르 생성해서 사용할 수 있다.
  - 생성자 파라미터를 이용해서 사용할 오브젝트를 넣어주게 할 수도 있다.
- 가장 중요한 것은 싱글톤 패턴과 달리 스프링이 지지하는 객체지향적인 설계 방식과 원칙, 디자인 패턴 등을 적용하는 데 아무런 제약이 없다는 점이다.
- 스프링이 빈을 싱글톤으로 만드는 것은 결국 오브젝트의 생성 방법을 제어하는 IoC 컨테이너로서의 역할이다.

### 1.6.2 싱글톤과 오브젝트의 상태
- 싱글톤은 멀티스레드 환경이라면 여러 스레드가 동시에 접근할 수 있기 때문에 상태 관리에 주의를 기울여야 한다.
- 싱글톤이 멀티스레드 환경에서 서비스 형태의 오브젝트로 사용되는 경우에는 상태정보를 내부에 갖고 있지 않은 `무상태(stateless)` 방식으로 만들어야 한다.
- 다중 사용자 요청을 한꺼번에 처리하는 스레드들이 동시에 싱글톤 오브젝트 인스턴스 변수를 수정하는 것은 매우 위험하다.
  - 저장할 공간이 하나 뿐이니 서로 값을 덮어쓰고 자신이 저장하지 않은 값을 읽어올 수 있기 때문이다.
- 따라서 싱글톤은 기본적으로 인스턴스 필드의 값을 변경하고 유지하는 `상태유지(stateful)` 방식으로 만들지 않는다.
  - 물론 읽기전용의 값이라면 초기화 시점에 인스턴스 변수에 저장해두고 공유하는 것은 아무 문제가 없다.
- 상태가 없는 방식으로 클래스를 만드는 경우에는 정보를 어떻게 다뤄야할까?
  -> 파라미터와 로컬 변수, 리턴 값 등을 이용하면 된다.
  - 메서드 파라미터나, 메서드 안에서 사용되는 로컬 변수는 매번 새로운 값을 저장할 독립적인 공간이 만들어지기 때문이다.
```java
// 1-23. 인스턴스 변수를 사용하도록 수정한 UserDao
public class UserDao {
  private ConnectionMaker connectionMaker; // 초기에 설정하면 사용 중에는 바뀌지 않는 읽기 전용 인스턴스 변수
  private Connection c;  // 매번 새로운 값으로 바뀌는 정보를 담은 인스턴스 변수. 심각한 문제가 발생한다.
  private User user;  // 매번 새로운 값으로 바뀌는 정보를 담은 인스턴스 변수. 심각한 문제가 발생한다.

  public User get(String id) throws ClassNotFoundException, SQLException {
    this.c = connectionMaker.makeConnection();
    ...
    this.user = new User();
    this.user.setId(rs.getString("id"));
    this.user.setName(rs.getString("name"));
    this.user.setPassword(rs.getString("password"));
    ...
    return this.user;
  }
}
```
- 읽기전용의 속성을 가진 정보라면 싱글톤에서 인스턴스 변수로 사용해도 좋다.
- 물론 단순한 읽기 전용 값이라면 static final이나 final로 선언하는 편이 나을 것이다.

### 1.6.3 스프링 빈의 스코프
- 스프링이 관리하는 오브젝트, 즉 빈이 생성되고, 존재하고, 적용되는 범위 => 빈의 `스코프(scope)`
- 스프링 빈의 기본 스코프는 싱글톤이다.
  - 싱글톤 스코프튼 컨테이너 내에 한 개의 오브젝트만 만들어져서, 강제로 제거되지 않는 한 스프링 컨테이너가 존재하는 동안 계속 유지된다.
- 경우에 따라서는 싱글톤 외의 스코프를 가질 수 있다.
  - 프로토타입(prototype) 스코프
    - 컨테이너에 빈을 요청할 때마다 매번 새로운 오브젝트를 생성
  - 요청(request) 스코프
    - 웹을 통해 새로운 HTTP 요청이 생길 때마다 생성
  - 세션(session) 스코프
    - 웹의 세션과 스코프가 유사

## 1.7 의존 관계 주입(DI)
- 스프링을 ioC 컨테이너라고만 해서는 스프링이 제공하는 기능의 특징을 명확하게 설명하지 못한다.
  - 스프링이 제공하는 IoC 방식의 핵심을 짚어주는 의존관계 주입(Dependency Injection)이라는 좀 더 의도가 명확히 드러나느 이름을 사용하기 시작했다.
- 스프링이 여타 프레임워크와 차별화돼서 제공해주는 기능은 의존관계 주입이라는 새로운 용어를 사용할 때 분명하게 드러난다.
- 주로 IoC 컨테이너라고 불리던 스프링이 지금은 의존관계 주입 컨테이너 또는 DI 컨테이너라고 더 많이 불리고 있다.
> **의존관계 주입, 의존성 주입, 의존 오브젝트 주입?**
>
> - 의존성이라는 말은 DI의 의미가 무엇인지 잘 드러내지 못한다.
> - 엄밀히 말해서 오브젝트는 다른 오브젝트에 주입할 수 있는 게 아니다. 오브젝트의 레퍼런스가 전달될 뿐이다.
> - DI는 오브젝트 레퍼런스를 외부로부터 제공(주입)받고 이를 통해 여타 오브젝트와 다이내믹하게 의존관계가 만들어지는 것이 핵심이다.
> - 이 책에서는 의존관계 주입이라고 번역하여 사용한다.

### 1.7.2 런타임 의존관계 설정
#### 의존관계
- 두 개의 클래스 또는 모듈이 의존관계에 있다고 말할 때는 항상 방향성을 부여해줘야 한다.
- UML 모델에서는 두 클래스의 의존관계(dependency relationship)를 다음과 같이 점선으로 된 화살표로 표현한다. 다음 그림은 A가 B에 의존하고 있음을 나타낸다.
  <img width="451" alt="스크린샷 2024-03-29 오후 6 05 22" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/1c737501-7d06-4e91-ba28-fa99696ba734">
- 의존하고 있다는 것의 의미는 의존 대상, 여기서는 B가 변하면 그것이 A에 영향을 미친다는 뜻이다.
- A가 B에 의존하고 있지만, 반대로 B는 A에 의존하지 않는다. 의존하지 않는다는 말은 B는 A의 변화에 영향을 받지 않는다는 뜻이다.

#### UserDao의 의존관계
- UserDao는 ConnectionMaker 인터페이스를 사용하고 있다. -> UserDao는 ConnectionMaker에 의존하고 있다.
- 하지만 ConnectionMaker 인터페이스를 구현한 클래스, 즉 DConnectionMaker 등이 다른 것으로 바뀌거나 메서드에 변화가 생겨도 UserDao에 영향을 주지 않는다.
- 이렇게 인터페이스에 대해서만 의존관계를 만들어두면 인터페이스 구현 클래스와는 관계는 느슨해지면서 변화에 영향을 덜 받는 상태가 된다. => 결합도가 낮다.
  <img width="386" alt="스크린샷 2024-03-29 오후 6 12 31" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/6687aa0d-7166-4fa2-9c9b-5f2e5c568c0e">

- 그런데 모델이나 코드에서 클래스와 인터페이스를 통해 드러나는 의존관계 말고, 런타임 시에 오브젝트 사이에서 만들어지는 의존관계도 있다. => **런타임 의존관계 또는 오브젝트 의존관계**
  - 설계 시점의 의존관계가 실체화된 것
- 느슨한 의존관계를 갖는 경우에는 오브젝트가 런타임 시에 사용할 오브젝트가 어떤 클래스로 만든 것인지 미리 알 수가 없다.
- 프로그램이 시작되고 UserDao 오브젝트가 만들어지고 나서 런타임 시에 의존관계를 맺는 대상, 즉 실제 사용대상인 오브젝트를 `의존 오브젝트(dependent object)`라고 말한다.
- 의존관계 주입은 이렇게 구체적인 의존 오브젝트와 그것을 사용할 주체, 보통 클라이언트라고 부르는 오브젝틀르 런타임시에 연결해주는 작업을 말한다.
- **의존관계 주입이란 다음과 같은 세 가지 조건을 충족하는 작업이다.**
  - 클래스 모델이나 코드에는 런타임 시점의 의존관계가 드러나지 않는다. (인터페이스에만 의존)
  - 런타임 시점의 의존관게는 컨테이너나 팩토리 같은 제3의 존재가 결졍한다.
  - 의존관계는 사용할 오브젝트에 대한 레퍼런스를 외부에서 제공(주입)해줌으로써 만들어진다.

### UserDao의 의존관계 주입
```java
// 1-24. 관계 설정 책임 분리 전의 생성자
public UserDao() {
  connectionMaker = new DConnectionMaker();
}
```
- 이 코드에 따르면 UserDao는 이미 설계 시점에서 DConnectionMaker라는 구체적인 클래스의 존재를 알고 있다.
- 이 코드의 문제는 이미 런타임 시 의존관계가 코드 속에 다 미리 결정되어있다는 점이다.
- 그래서 제3의 존재에 런타임 의존관계 결정 권한을 위임해 최종적으로 만들어진 것이 DaoFactory다. DaoFactory는 의존관계 주입의 세 가지 조건을 모두 충족하고, 이미 DaoFactory를 만든 시점에서 DI를 이용한 셈이다.
- 다음과 같이 UserDao의 의존관계는 ConnectionMaker 인터페이스 뿐이다.
  <img width="339" alt="스크린샷 2024-03-31 오후 11 26 47" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/572a66fa-d88d-49e4-afdf-2198983cf6eb">
- 런타임 시점의 의존관계를 결정하고 만드는 제3의 존재의 역할을 DaoFactory가 담당한다고 해보자. DaoFactory는 의존관계 주입을 담당하는 컨테이너라고 볼 수 있다.(DI 컨테이너)
  - 보통 DI는 그 근간이 되는 IoC와 함께 사용해서 IoC/DI 컨테이너라는 식으로 함께 사용되기도 한다.
- DI 컨테이너는 UserDao를 만드는 시점에서 생성자의 파라미터로 이미 만들어진 DConnectionMaker의 오브젝트를 전달하는데, 정확히는 DConnectionMaker 오브젝트의 **레퍼런스**가 전달되는 것이다.
  ```java
  // 1-25. 의존관계 주입을 위한 코드
  public class UserDao {
    private ConnectionMaker connectionMaker;

    public UserDao(ConnectionMaker connectionMaker) {
      this.connectionMaker = connectionMaker;
    }
    ...
  }
  ```
- 이렇게 DI 컨테이너에 의해 런타임 시에 의존 오브젝트를 사용할 수 있도록 그 레퍼런스를 전달받는 과정이 마치 메서드(생성자)를 통해 DI컨테이너가 UserDao에게 주입해주는 것과 같다고 해서 이를 의존관계 주입이라고 부른다.
- 다음은 이런 객체에 대한 런타임 의존관계 주입과 그것으로 발생하는 런타임 사용 의존관계의 모습을 보여준다.
  <img width="448" alt="스크린샷 2024-03-31 오후 11 32 07" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/20bf14d9-d9c8-44b6-aecf-2e3938157aed">

- 스프링의 IoC는 주로 의존관계 주입 또는 DI라는 데 초점이 맞춰져 있다.

## 1.7.3 의존관계 검색과 주입
- 스프링에는 의존관계를 맺는 방법이 외부로부터의 주입이 아니라 스스로 검색을 이용하기 때문에 `의존관계 검색(dependency lookup)`이라고 불리는 것도 있다.
- 의존관계 검색은 자신이 필요로 하는 의존 오브젝트를 능동적으로 찾는다. 물론 자신이 어떤 클래스의 오브젝트를 이용할지 결정하지는 않는다. 그러면 IoC라고 할 수는 없을 것이다.
- 런타임 시 의존관계를 맺을 오브젝트를 결정하는 것과 오브젝트 생성 작업은 외부 컨테이너에게 IoC로 맡기지만, 이를 가져올 떄는 메서드나 생성자를 통한 주입 대신 스스로 컨테이너에게 요청하는 방법을 사용한다.
```java
// 1-26. DaoFactory를 이용하는 생성자
public UserDao() {
  DaoFactory daoFactory = new DaoFactory();
  this.connectionMaker = daoFactory.connectionMaker();
}
```
- UserDao는 여전히 어떤 ConnectionMaker 오브젝트를 사용할지 미리 알지 못하고 런타임 시에 DaoFactory가 만들어서 돌려주는 오브젝트와 다이내믹하게 런타임 의존관계를 맺는다.
- 따라서 IoC 개념을 잘 따르고 있으며, 그 혜택을 받고 있는 코드다. 하지만 적용 방법은 외부로부터의 주입이 아니라 스스로 IoC 컨테이너인 DaoFactory에게 요청하고 있는 것이다.
- 이런 작업을 일반화한 스프링의 애플리케이션 컨텍스트라면 미리 정해놓은 이름을 전달해서 그 이름에 해당하는 오브젝트를 찾게 된다. 따라서 이를 일종의 검색이라고 볼 수 있다.
- 애플리케이션 컨텍스트는 getBean()이라는 메서드를 제공한다. 바로 이 메서드가 의존관계 검색에 사용된다.
```java
// 1-27. 의존관계 검색을 이용하는 UserDao 생성자
public UserDao() {
  AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(DaoFactory.class);
  this.connectionMaker = context.getBean("connectionMaker", ConnectionMaker.class);
```

- 의존관계 검색은 의존관계 주입의 거의 모든 장점을 갖고 있고, IoC 원칙에도 잘 들어 맞는다.
- 둘 중 어떤 것이 나을까? => 의존관계 주입
  - 의존관계 주입은 훨씬 단순하고 깔끔하다.
  - 의존관계 검색은 코드 안에 오브젝트 팩토리 클래스나 스프링 API가 나타난다. 애플리케이션 컴포넌트가 컨테이너와 같이 성격이 다른 오브젝트에 의존하게 되는 것이므로 바람직하지 않다.
- 그런데 의존관계 검색 방식을 사용해야 할 때가 있다.
  - 애플리케이션의 기동 시점에서 적어도 한 번은 의존관계 검색 방식을 사용해 오브젝트를 가져와야 한다.
  - 서블릿에서 스프링 커네티어네 담긴 오브젝트를 사용하려면 한 번은 사용해야 한다.
- 의존관계 검색과 의존관계 주입의 중요한 차이점
  - 의존관계 검색 방식에서 검색하는 오브젝트는 **스프링 빈일 필요가 없다**
  - 의존관계 주입에서는 UserDao와 ConnectionMaker 사이에서 DI가 적용되려면 UserDao도 반드시 컨테이너가 만드는 빈 오브젝트여야 한다.
- DI를 원하는 오브젝트는 자기 자신이 컨테이너가 관리되는 빈이 돼야 한다.

### 1.7.4 의존관계 주입의 응용
- DI 기술의 장점
  - 객체지향 설계와 프로그래밍 원칙을 따랐을 때 얻을 수 있는 장점과 같음
- UserDao가 ConnectionMaker 인터페이스에 의존하고 있다는 건, ConnectionMaker를 구현하기만 한다면 어떤 오브젝든 사용할 수 있다는 뜻

#### 기능 구현의 교환
- 만약 로컬DB와 운영DB를 번갈아 사용해야 하는 상황이라고 가정해보자.
- DI 방식을 적용하지 않는다면 로컬 DB로 개발중에는 로컬 구체 클래스에 의존하고, 운영DB를 사용할때는 운영 구체 클래스에 의존하므로 번갈아 사용할때마다 DAO를 무진장 고쳐야 한다. 

- DI방식을 사용한다면 모든 DAO는 생성 시점에 ConnectionMaker 타입의 오브젝트를 컨테이너로부터 제공받는다. 
```java
// 1-28. 개발용 ConnectionMaker 생성 코드
@Bean
public ConnectionMaker connectionMaker(){
	return new LocalDBConnectionMaker();
}
```
```java
// 1-29. 운영용 ConnectionMaker 생성 코드
@Bean
public ConnectionMaker connectionMaker(){
	return new ProductionDbConnectionMaker();
}
```
- 로컬 DB에서 운영DB로 옮겨도 DAO 코드를 단 한줄도 수정할 필요가 없다.
- 단지 서버에서 사용할 DaoFactory를 1-29와 같이 수정해주면 된다.
#### 부가기능 추가
- 다음과 같은 상황이 존재한다고 해보자.
- DAO가 DB를 얼마나 많이 연결해서 사용하는지 파악하고 싶어서 
모든 makeConnection() 메소드를 호출하는 부분에 카운터를 증가시키는 코드를 넣으려고 한다. 
- 다행히 DI 컨테이너라면 아주 간단한 방법으로 카운팅이 가능하다. DAO와 DB 커넥션을 만드는 오브젝트 사이에 연결횟수를 카운팅하는 오브젝트를 추가하는 것이다. 아래 코드를 보자.
```java
// 1-31. CountingConnectionMaker 의존관계가 추가된 DI 설정용 클래스
public class CountingConnectionMaker implements ConnectionMaker{
	int count = 0;
    private ConnectionMaker realConnectionMaker;
	
	public CountingConnectionMaker(ConnectionMaker realConnectionMaker){
    	this.realConnectionMaker = realConnectionMaker;
    }    
    public Connection makeConnection() throws ClassNotFoundException, SQLException{
    	this.counter++;
        return realConnectionMaker.makeConnection();
    }
    
    public int getCounter(){
    	return this.counter;
    }
}
```
- CountingConnectionMaker라는 클래스는 ConnectionMaker를 구현해서 만든다.
- 하지만 내부에서 직접 커넥션을 만들지 않는다. 대신 DAO가 DB 커넥션을 가져올 때마다 호출하는 makeConnection()에서 카운터를 증가시킨다.
 
- 카운팅 증가 작업이 끝나면, 실제 DB 커넥션을 만들어주는 realConnectionMaker에 저장된  ConnectionMaker타입 오브젝트의 makeConnection()을 호출해 그 결과를 DAO에 돌려준다.

- 그럼 설정정보는 아래와 같이 변경된다.

```java
@Configuration
public class DaoFactory{
	...
    ...
    @Bean
    public UserDao userDao(){
    	return new UserDao(connectionMaker());
    }
    
	@Bean
    public ConnectionMaker connectionMaker(){
    	return new CountingConnectionMaker(realConnectionMaker());
    }
    
    @Bean
    public ConnectionMaker realConnectionMaker(){
    	return new DConnectionMaker();
    }
}
```

### 1.7.5 메소드를 이용한 의존관계 주입
- 의존관계 주입 시 생성자만 사용해야 하는 것은 아니다. 다른 방법들을 소개한다.

- **수정자 메소드를 통한 의존관계 주입**
  - setter 메소드를 활용한다. setter는 IDE의 자동생성을 사용하도록 하자.
  ```java
  public class UserDao{
      private ConnectionMaker connectionMaker;
  
      public void setConnectionMaker(ConnectionMaker connectionMaker){
          this.connectionMaker = connectionMaker;
      }
  }
  ```
- **일반 메소드를 통한 의존관계 주입**
  - 수정자 메소드와 달리 한번에 여러 개의 파라미터를 받을 수 있다는 장점이 있다(생성자 방식도 가능하다). 

## 1.8 XML을 이용한 설정
- 본격적인 DI 컨테이너를 사용하면서 오브젝트 사이의 의존 정보는 일일히 자바 코드로 만들어주려면 번거롭다.
- 스프링은 자바 클래스를 이용하는 것 외에도 다양한 방법을 통해 DI 의존관계 설정정보를 만들 수 있다. -> 가장 대표적인 것이 XML
- XML은 단순 텍스트 파일이라 다루기 쉽고 이해하기 쉬우며 별도의 빌드 작업이 없다. 또한 환경이 달라져서 오브젝트 관계가 바뀌는 경우에도 빠르게 변경사항을 확인할 수 있다.

### 1.8.1 XML 설정
- 스프링의 애플리케이션 컨텍스트는 XML에 담긴 DI 정보를 활용할 수 있다.
- DI 정보가 담긴 XML 파일은 `<beans>`를 루트 앨리멘트로 사용한다.
  - `<beans>` 안에는 여러 개의 `<bean>`을 정의할 수 있다.
- 하나의 빈을 통해 얻을 수 있는 빈의 DI 정보는 다음 세가지다.
  - 빈의 이름 : @Bean 메서드 이름이 빈의 이름이다. 이 이름은 getBean()에서 사용된다.
  - 빈의 클래스 : 빈 오브젝트를 어떤 클래스를 이용해서 만들지를 정의한다.
  - 빈의 의존 오브젝트 : 빈의 생성자나 수정자 메서드를 통해 의존 오브젝트를 넣어준다. 의존 오브젝트는 하나 이상일 수도 있다.
#### connectionMaker() 전환
- connectionMaker()로 정의되는 빈은 의존하는 다른 오브젝트는 없으니 DI 정보 세 가지 중 두 가지만 있으면 된다.
- DI 정의 세 가지 중에서 빈의 이름과 빈 클래스 두가지를 <bean> 태그와 id와 class 애트리뷰트를 이용해 정의할 수 있다.

|        | 자바 코드 설정 정보 | XML 설정 정보 |
|--------| --------------- | ---------- |
| 빈 설정 파일 | @Configuration | <beans> |
| 빈의 이름 | @Bean methodName() | <bean id="methodName" |
| 빈의 클래스 | return new BeanClass(); | class="a,b,c... BeanClass">|

- <bean> 태그의 class 애트리뷰트에 지정하는 것은 메서드에서 오브젝트를 만들 때 사용하는 클래스
  -> 메서드의 리턴 타입을 class 애트리뷰트에 사용하지 않도록하자.
- XML에서는 리턴하는 타입을 지정하지 않아도 된다. class 애트리뷰트에 넣을 클래스 이름은 패키지까지 모두 포함해야 한다.
```java
// 1-35. connectionMaker() 메서드의 <bean> 태그 전환
@Bean // -> <bean
public ConncectionMaker
connectionMaker() { // -> id="connectionMaker"
  return new DConnectionMaker(); // -> class "springbook...DConnectionMaker" />
}
```

#### userDao() 전환
- userDao의 메서드를 XML로 변환해보자.
  - 수정자 메서드를 사용해 의존관계를 주입하는 부분에 주목
- 자바빈의 관계를 따라서 수정자 메서드는 프로퍼티가 된다. 프로퍼티 이름은 메서드 이름에서 set을 제외한 나머지 부분을 사용한다.
- `<property>` 태그를 사용해 의존 오브젝트와의 관계를 정의한다.
  - name과 ref라는 두 개의 애트리뷰트를 갖는다.
    - name : DI에 사용할 수정자 메서드의 프로퍼티 이름 
    - ref : 수정자 메서드를 통해 주입해줄 오브젝트 빈 이름
- @Bean 메서드에서라면 다음과 같이 @Bean 메서드를 호출해서 주입할 오브젝트를 가져온다.
    ```java
    userDao.setConnectionMaker(connectionMaker());
    ```
  - `userDao.setConnectionMaker` : userDao 빈의 connectionMaker 프로퍼티를 이용해 의존관계를 주입한다.
  - `connectionMaker()` : connectionMaker() 메서드를 호출해서 리턴하는 오브젝트를 주입하라.
- 각 정보를 <property> 태그에 대응하면 다음과 같이 전환이 가능하다.
  ```xml
  <property name="connectionMaker" ref="connectionMaker" />
  ```
- 이렇게 완성된 userDao 빈을 위한 XML 정보다.
  ```java
  <bean id="userDao" class="springbook.dao.UserDao">
    <property name="connectionMaker" ref="connectionMaekr" >
  </bean>
  ```
#### XML 의존관계 주입 정보
```xml
// 1-37. 완성된 XML 설정 정보
<beans>
  <bean id="connectionMaker" class="springbook.user.dao.DConnectionMaker" />
  <bean id="userDao" class="springbook.user.dao.UserDao">
    <property name="connectionMaker" ref="connectionMaker" />
  </bean>
</beans>
```
- 보통 프로퍼티의 이름과 DI되는 빈의 이름이 같은 경우가 많다. 둘 다 주입할 빈 오브젝트의 인터페이스 이름을 따르는 경우가 많기 때문이다.
- 하지만 프로퍼티 이름이나 빈의 이름은 인터페이스 이름과 다르게 정해도 상관없다.
- 빈의 이름을 바꾸는 경우 그 이름을 참조하는 다른 빈의 <property> ref 애트리뷰트의 값도 함꼐 변경해줘야 한다.
```xml
// 1-38. 빈의 이름과 참조 ref의 변경
<beans>
  <bean id="myConnectionMaker" class="springbook.user.dao.DConnectionMaker" />

  <bean id="userDao" class="springbook.user.dao.UserDao">
    <property name="connectionMaker" ref="myConnectionMaker" />
  </bean>
</beans>
```
- 때로는 같은 인터페이스를 구현한 의존 오브젝트를 여러 개 정의해두고 그 중에서 원하는 걸 골라 DI하는 경우도 있다.
- 이때는 각 빈의 이름을 독립적으로 만들어두고 ref 애트리뷰트를 이용해 DI 받을 빈을 지정해주면 된다.
```xml
// 1-39. 같은 인터페이스 타입의 빈을 여러 개 정의한 경우
<beans>
  <bean id="localDBConnectionMaker" class="...LocalDBConnectionMaker" />
  <bean id="testDBConnectionMaker" class="...TestDBConnectionMaker" />
  <bean id='productionDBConnectionMaker" class="...ProductionDBConnectionMaker" />

  <bean id="userDao" class="springbook.user.dao.UserDao">
    <property name="connectionMaker" ref="localDBConnectionMaker" />
  </bean>
</beans>
```
- 환경에 따라 XML 파일을 다르게 사용하면 된다.

> **DTD와 스키마**
>
> - XML 문서는 미리 정해진 구조를 따라서 작성됐는지 검사할 수 있다. XML 문서의 구조를 정의하는 방법에는 DTD와 스키마(schema)가 있다.
> - 특별한 이유가 없다면 DTD보다는 스키마를 사용하는 편이 바람직하다.

### 1.8.2 XML을 이용하는 애플리케이션 컨텍스트
- 애플리케이션 컨텍스트가 DaoFactory 대신 XML 설정정보를 활용하도록 만들어보자.
- XML에서 빈의 의존관계 정보를 이용하는 IoC/DI 작업에는 GeneralXmlApplicationContext를 사용한다.
  - GeneralXmlApplicationContext의 생성자 파라미터로 XML 파일의 클래스패스를 지정해주면 된다.
- 애플리케이션 컨텍스트가 사용하는 XML 설정파일의 이름은 과례를 따라 applicationContext.xml이라고 만든다.
```xml
// 1-40. XML 설정 정보를 담은 applicationContext.xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans /spring-beans=3.0.xsd">
  <bean id="connectionMaker" class="springbook.user.dao.DConnectionMaker" />
  <bean id="userDao"  class="springbook.user.dao.UserDao">
    <property name="connectionMaker" ref="localDBConnectionMaker" />
  </bean>
</beans>
```

- 다음은 UserDaoTest의 애플리케이션 컨텍스트 부분을 수정한다. 이 때 GenericXmlApplicationContext을 사용한다.

```java
ApplicationContext context = GenericXmlApplicationContext("applcationContext.xml");
```

- GenericXmlApplicationContext 외에도 ClassPathXmlApplicationContext 사용 가능
- GenericXmlApplicationContext는 클래스 패스 뿐만 아니라 다양한 소스로부터 설정 파일을 읽어올 수 있다.
- ClassPathXmlApplicationContext은 클래스패스의 경로정보를 클래스에서 가져오는 기능이 있다. XML 파일과 같은 클래스 패스에 있는 클래스 오브젝트를 넘겨서 클래스패스에 대한 힌트 제공 가능
- 이 방법으로 클래스패스를 지정하는 경우가 아니라면 GenericXmlApplicationContext이 무난하다.

### 1.8.3 DataSource 인터페이스로 변환
#### DataSource 인터페이스 적용
- 사실 자바에서는 DB 커넥션을 가져오는 오브젝트의 기능을 추상화해서 비슷한 용도로 사용할 수 있게 만들어진 DataSource라는 인터페이스 존재 -> 실전에서는 ConnectionMaker를 만들어서 사용하지 않아도 됨.
- DataSource를 직접 구현해서 클래스를 만들 일은 거의 없고, DB 연결과 같은 pooling 기능을 갖춘 많은 구현 클래스를 가져다 사용하면 충분하다.
```java
// 1-41. DataSource 인터페이스
package javax.sql

public interface DataSource extends CommonDataSource, Wrapper {
  Connection getConnection() throws SQLException; // ConnectionMaker의 makeConnection()과 목적 동일
}
```

- DataSource의 인터페이스와 다양한 구현 클래스를 사용할 수 있도록 UserDao를 리팩토링 해보자.
  - ConnectionMaker -> DataSource / makeConnection() -> getConnection()
```java
// 1-42. DataSource를 사용하는 UserDao
import javax.sql.DataSource;

public class UserDao {
  private DataSource dataSource;

  public void setDataSource(DataSource dataSource) {
    this.dataSource = dataSource;
  }

  public void add(User user) throws SQLException {
    Connection c = dataSource.getConnection();
    ...
  }
  ...
}
```
  
- 다음은 DataSource의 구현 클래스가 필요하다. DriverManager를 사용하는 SimpleConnectionMaker처럼 DataSource의 구현 클래스 중에서 테스트 환경에서 간단히 사용할 수 있는 SimpleDriverDataSource라는 것이 있다.
- DB 연결에 필요한 필수 정보를 제공받을 수 있도록 여러 개의 수정자 메서드를 갖고 있다.

#### 자바 코드 설정 방식
```java
// 1-43. DataSource 타입의 dataSource 빈 정의 메서드
@Bean
public DataSource dataSource() {
  SimpleDriverDataSource dataSource = new SimpleDriverDataSource();

  // DB 연결정보를 수정자 메서드를 통해 넣어준다. 이렇게 하면 오브젝트 레벨에서 DB 연결 방식을 변경할 수 있다.
  dataSource.setDriverClass(com.mysql.jdbc.Driver.class);
  dataSource.setUrl("jdbc:mysql://localhost/springbook");
  dataSource.setUsername("spring");
  dataSource.setPassword("book");

  return dataSource;
}
```java
// 1-44. DataSource 타입의 빈을 DI 받은 userDao() 빈 정의 메서드
@Bean
public UserDao userDao() {
  UserDao userDao = new UserDao();
  userDao.setDataSource(dataSource());
  return userDao;
}
```
#### XML 설정 방식
```xml
// 1-45. dataSource 빈
<bean id="dataSource"
  class="org.springframework.jdbc.datasource.SimpleDriverDataSource" />
```

그런데 DB 접속 정보는 나타나 있지 않다. SimpleDriverDataSource는 다른 빈에 의존하지 않는 단순 Class 타입의 오브젝트나 텍스트 값이다. 그렇다면 XML에서는 어떻게 해서 dataSource() 메서드에서처럼 DB 연결 정보를 넣도록 설정을 만들 수 있을까?

### 1.8.4 프로퍼티 애트리뷰트

#### 값 주입

- DaoFactory의 dataSource는 수정자 메소드에 다른 빈, 오브젝트 뿐 만이 아니라 String 같은 단순한 값을 넣을 수 있음(dataSource.setUrl)
- dataSource.setDriverClass()는 Class 타입의 오브젝트이나, 다른 빈 오브젝트를 DI 방식으로 가져와서 넣는 것은 아님
- 이렇게 다른 빈 오브젝트의 레퍼런스가 아닌 단순 정보도 오브젝트를 초기화하는 과정에서 수정자 메소드에 넣을 수 있음
- 이때는 DI에서처럼 오브젝트의 구현 크래스를 동적으로 바꿀수 있게 하는 목적이 아님
- 단순히 변경 가능한 정보(DB 접속 아이디, 비밀번호와 같은)를 설정하도록 위한 것
- 텍스트나 단순 오브젝트 등을 수정자 메서드에 넣어주는 것 ‘값을 주입’하는 것 이므로 DI의 일종임
- `<property ref=””>`의 일종이나 단순 값을 주입하는 것이기 때문에` value` 애트리뷰트 사용
```java
// 1-46. 코드를 통한 DB 연결 정보 주입
dataSource.setDriverClass(com.mysql.cj.jdbc.Driver.class);
dataSource.setUrl("jdbc:mysql://localhost/toby?serverTimezone=UTC");
dataSource.setUsername("root");
dataSource.setPassword("1234");
```
```java
// 1-47. XML을 이용한 DB 연결 정보 설정
<property name="driverClass" value="com.mysql.cj.jdbc.Driver"/>
<property name="url" value="jdbc:mysql://localhost/toby?serverTimezone=UTC"/>
<property name="username" value="root"/>
<property name="password" value="1234"/>
```

#### value 값의 자동 변환
- DriverClass는 String이 아닌 java.lang.Class 타입
- 스트링 값을 Class 타입 변수에 넣을 수는 없음
- String이 아닌 클래스임에도 value를 통해 넣을 수 있는 까닭은, 스프링이 프로퍼티 값을, 수정자 메서드의 파라미터 타입을 참고로 해서 적절한 형태로 변환해주기 때문
- 내부적으로 다음과 같은 변환 작업이 일어남
  ```java
  Class driverClass = Class.forName("com.mysql.jdbc.Driver");
  dataSource.setDriverClass(driverClass);
  ```
- 값이 여러개라면 List, Map, 배열 등을 사용해 값 주입 가능
- 최종적으로 다음과 같은 XML 설정 파일이 만들어짐
  ```xml
  // 1-48. DataSource를 적용 완료한 applicationContext.xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://www.springframework.org/schema/beans
                              http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">
      <bean id="dataSource" class="org.springframework.jdbc.datasource.SimpleDriverDataSource">
          <property name="driverClass" value="com.mysql.cj.jdbc.Driver"/>
          <property name="url" value="jdbc:mysql://localhost/toby?serverTimezone=UTC"/>
          <property name="username" value="root"/>
          <property name="password" value="1234"/>
      </bean>
      <bean id="userDao" class="com.ksb.spring.UserDao">
          <property name="dataSource" ref="dataSource"/>
      </bean>
  </beans>
  ```
## 1.9 정리
- 책임이 다른 코드를 분리하여 두 개의 클래스로 만들었다. (`관심사의 분리`, `리팩토링`)
- 추후에 변경이 일어날 수 있는 부분은 인터페이스(`ConnectionMaker`)를 만들어 구현하도록 하고, 다른 클래스에서 인터페이스를 통해서만 접근하도록 만들었다. 인터페이스를 정의한 쪽의 구현 방법이 달라져 클래스가 바뀌더라도, 그 기능을 사용하는 클래스의 코드는 같이 수정할 필요가 없게 만들었다. (`전략 패턴`)
- 자신의 책임 자체가 변경되는 경우 외에는 불필요한 변화가 발생하지 않도록 막아주고, UserDao가 사용하는 외부 오브젝트의 기능은 자유롭게 확장하거나 변경할 수 있게 만들었다. (개방 폐쇄의 원칙)
- 한쪽의 기능 변화가 다른쪽의 변경을 요구하지 않아도 되도록 만들고(낮은 결합도), 자신의 책임과 관심사에만 순수하게 집중하는(높은 응집도) 깔끔한 코드를 만들 수 있었다.
- 오브젝트가 생성되고 타 오브젝트와 관계를 맺는 작업의 제어권을 별도의 오브젝트(Factory)를 만들어 넘겼다. 이후 Factory의 기능을 일반화한 IoC 컨테이너로 넘겨 오브젝트 자신이 사용할 대상의 생성이나 선택에 관한 책임으로부터 자유롭게 만들었다. (`제어의 역전/IoC`)
- 전통적 싱글톤 구현 방식의 단점을 개선한, 서버 서비스 오브젝트로서의 장점을 살릴 수 있는 싱글톤 레지스트리 를 이용하여 컨테이너를 활용하는 방법에 대해 알아봤다. (`싱글톤 레지스트리`)
- 설계 시점에서는 클레스와 인터페이스 사이의 느슨한 의존 관계만 만들어놓고, 런타임 시에 실제 의존 오브젝트를 제 3자(DI 컨테이너)의 도움을 받아 주입받아서 동적인 의존관계를 갖게 해주는 IoC의 특별한 케이스를 알아봤다.(`의존관계 주입/DI`)
- 의존 오브젝트를 주입할 때 생성자를 이용하는 방법과 수정자 메서드를 이용하는 방법을 알아봤다. (`생성자 주입과 수정자 주입`)
- XML을 이용해 DI 설정정보를 만드는 방법과 의존 오브젝트가 아닌 일반 값을 외부에서 설정하여 런타임 시에 주입하는 방법을 알아봤다. (`XML 설정`)
