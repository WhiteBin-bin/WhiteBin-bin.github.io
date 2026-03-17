---
title: "SQL Injection"
date: 2026-01-04 15:13:00 +0900
categories: [BackEnd, DataBase]
---

![](/assets/img/posts/SqlInjection.png)

## 주제 선정 이유

백엔드를 공부하다 보면 한 번쯤은 **SQL Injection**이라는 용어를 듣게 되며, 특히 데이터베이스와 관련된 보안 취약점을 설명할 때 빠지지 않고 등장하는 개념이며 또한 DB 쿼리를 실행할 때는 `Statement` 대신 `PreparedStatement`를 
<br>사용해야 한다는 이야기를 함께 접하게 되며, 그 이유로 가장 많이 언급되는 것이 바로 <span style="background-color:#6DB33F">**SQL Injection 방지**</span>이기도 합니다.

처음에는 단순히 보안적으로 더 안전한 방식이라는 정도로 이해하고 넘어가기도 하지만, 문득 **단순히 쿼리를 실행하는 방식이 달라지는 것만으로 어떻게 해킹을 막을 수 있는 것일까** 하는 궁금증이 생기기도 하고, 실제로 `SQL Injection`
공격이 어떤 방식으로 이루어지며 `PreparedStatement`는 어떤 원리로 이를 방어할 수 있는지에 대해 정확히 이해해 보고 싶다는 생각이 들어서 이번 글에서는 <span style="background-color:#6DB33F">**SQL Injection이 발생하는 원리와 실제 공격 방식**</span>을 살펴보고, 이어서 `PreparedStatement`가 어떤 방식으로 `SQL Injection`을 방어하는지 그 내부 동작과 메커니즘을 중심으로
정리해보고자 합니다.

## SQL Injection이 뭘까?

웹을 해킹하는데 가장 보편적으로 사용되는 기법 중 하나로, 웹 페이지의 입력값을 통해 `SQL`문에 악성 코드를 주입하여 데이터베이스에 접근하는 방식입니다.

<span style="background-color:#6DB33F">**사용자의 클릭이나 입력에 따라 동적으로 데이터를 가져오기 위해, 사용자가 
입력한 값을 포함한 SQL 쿼리를 생성하며 이 과정에서 입력값 검증이 제대로 이루어지지 않으면, 
공격자가 악의적으로 조작된 쿼리를 주입해 개발자가 의도하지 않은 데이터에 접근하거나 조작**</span>할 수 있게 됩니다.

## 공격 유형

데이터베이스의 정보를 탈취하는 방식에 따라 크게 5가지 유형으로 나뉩니다.

### In-band SQL Injection

공격과 결과 확인이 동일한 통신 채널에서 이루어지는 가장 일반적인 SQL Injection 방식이며, 공격자는 애플리케이션이 반환하는 응답 결과를 통해 직접 공격 성공 여부를 확인할 수 있습니다.

대표적으로 로그인 기능에서 입력값을 조작하여 <span style="background-color:#6DB33F">**참(True) 조건을 만들어 인증을 우회하는 방식**</span>이 자주 사용됩니다.

``` sql
SELECT * FROM user WHERE id='admin' AND password='password';
```

위에처럼 쿼리가 작성되었다고 가정했을때 입력창에 `' OR 1=1 --` 를 입력하면

``` sql
SELECT * FROM user WHERE id='' OR 1=1--' AND password='password';
```

이런식으로 쿼리가 조작되면서 아래와 같이 동작하게 됩니다.

``` sql
SELECT * FROM user WHERE id='' OR 1=1;
```

이 쿼리는 id가 무엇이든 상관없이 `'1=1'`이라는 <span style="background-color:#6DB33F">**조건이 항상 참(True)**</span>이기 때문에, 데이터베이스는 인증 절차를 
무시하고 테이블에 있는 사용자 정보를 반환하게 됩니다. 
### UNION-based SQL Injection

`UNION` 연산자를 이용해 기존 쿼리 결과에 공격자가 원하는 데이터를 결합하여 반환하도록 만드는 공격 방식이며, 
데이터베이스가 반환하는 결과 화면을 통해 추가적인 정보나 민감한 데이터를 함께 노출시키는 방식으로 동작합니다.

즉, 정상적으로 실행되는 쿼리에 `UNION` 구문을 삽입해 <span style="background-color:#6DB33F">**다른 테이블의 
데이터나 시스템 정보를 함께 조회하도록 만들어 결과를 탈취하는 공격 방식**</span>입니다.
``` sql
SELECT * FROM user WHERE id='admin' AND password='password';
```
위에처럼 쿼리가 작성되었다고 가정했을때 입력창에 `admin' UNION SELECT 1,1 -- `를 입력하면
``` sql
SELECT * FROM user WHERE id='admin' UNION SELECT 1,1 --' AND password='password';
```
위와 같이 쿼리가 조작되면서 아래와 같이 동작하게 됩니다.
``` sql
SELECT * FROM user WHERE id='admin' UNION SELECT 1,1;
```
이 쿼리는 원래의 조회 결과 뒤에 공격자가 임의로 만든 `'1, 1'`이라는  <span style="background-color:#6DB33F">**가상의 행을 강제로 결합**</span>하라는 의미이며 <br>이걸 통해서 데이터베이스의 컬럼 개수를 파악할 수 있습니다. 

만약 원래 쿼리가 반환하는 컬럼 수와 UNION 뒤에 붙인 컬럼 수가 다르면 에러가 발생하며 `SELECT 1, 1, 1`처럼 숫자를 늘려가며 에러가 나지 않는 지점을 찾으면 해당 테이블의 정확한 구조를 알아낼 수 있습니다.
``` sql
admin' UNION SELECT id, password FROM users --
```
위와 같이 컬럼이 2개인 것을 알아낸 후, 아이디와 비밀번호를 직접 조회할 수 있습니다.

### Blind SQL Injection - Boolean-based

쿼리 결과가 직접적으로 노출되지 않는 환경에서 조건의 <span style="background-color:#6DB33F">**참(True)과 거짓(False)에 따라 달라지는 응답을 이용해 
<br>데이터를 추론하는 공격 방식**</span>이며, 공격자는 특정 조건을 포함한 쿼리를 반복적으로 보내고 응답 결과의 차이를 통해 
데이터의 존재 여부나 값을 하나씩 추측해 나가게 됩니다.

``` sql
SELECT * FROM users WHERE username='$input';
```

위에처럼 쿼리가 작성되었다고 가정했을때 입력창에 `' OR SUBSTRING((SELECT database()),1,1)='t' --`을 입력하면

``` sql
SELECT * FROM users WHERE username = '' OR SUBSTRING((SELECT database()),1,1)='s' -- ';
```

위와 같이 쿼리가 조작되면서 아래와 같이 동작하게 됩니다.

``` sql
SELECT * FROM users WHERE username = '' OR SUBSTRING((SELECT database()),1,1)='s';
```

이 쿼리는 현재 사용 중인 데이터베이스 이름의 첫 글자가 `'s'`인지 확인하는 쿼리문이며, <span style="background-color:#6DB33F">**조건이 일치하여 참(True)이 
될 경우**</span> 데이터베이스는 로그인 성공이나 특정 데이터를 포함한 정상 페이지를 반환하게 됩니다.

### Blind SQL Injection - Time-Based

데이터베이스의 응답 시간이 달라지는 것을 이용해 조건의 참과 거짓을 판단하는 공격 방식이며, 공격자는 `SLEEP()`과 같은 시간 지연 함수를 쿼리에 삽입하여 <span style="background-color:#6DB33F">**특정 조건이 참일 때만 응답이 지연되도록 만든 뒤 서버의 응답 시간을 
비교하며 데이터를 추론하게 됩니다.**</span>

``` sql
SELECT * FROM users WHERE username = '$input';
```
위에처럼 쿼리가 작성되었다고 가정했을때 입력창에 `' OR IF(SUBSTRING((SELECT database()),1,1)='s', SLEEP(5), 0) -- '`을 입력하면
``` sql
SELECT * FROM users WHERE username = '' OR IF(SUBSTRING((SELECT database()),1,1)='s', SLEEP(5), 0) -- ';
```
위와 같이 쿼리가 조작되면서 아래와 같이 동작하게 됩니다.
``` sql
SELECT * FROM users WHERE username = '' OR IF(SUBSTRING((SELECT database()),1,1)='s', SLEEP(5), 0);
```
이 쿼리는 데이터베이스 이름의 첫 글자가 `'s'`인지 확인하고, 참(True)일 경우 `SLEEP(5)` 함수를 실행하기 때문에 
<span style="background-color:#6DB33F">**데이터베이스는 즉시 응답하는 대신 의도적으로 5초간 지연된 후 결과를 반환**</span>하게 됩니다.

### Mass SQL Injection

자동화된 도구(SQLMap 등)를 이용해 여러 테이블이나 컬럼, 레코드를 대상으로 대량의 `SQL Injection` 공격을 
수행하는 방식이며, 공격자는 스크립트나 자동화 툴을 활용해 <span style="background-color:#6DB33F">**다수의 취약 지점을 동시에 탐색하고 데이터를 
수집하거나 악성 쿼리를 대규모로 실행하는 특징을 가집니다.**</span>

``` bash
python sqlmap.py -u "http://example.com/article.php?id=123" --batch
```

이 공격은 특정 데이터 하나를 노리는 것을 넘어, 데이터베이스 내의 <span style="background-color:#6DB33F">**모든 테이블과 레코드를 대상으로 악성 코드를 삽입(Update)하거나 정보를 변조**</span>하라는 의미를 가집니다.
#### SQLMap 공격 예시

**취약점 확인** 
- `--batch` 옵션으로 해당 URL이 공격 가능한지 자동 진단합니다.
``` bash
python sqlmap.py -u "http://example.com/article.php?id=123" --batch
```

**DB 정보 추출** 
- `--dbs` 옵션을 붙여 현재 서버에 존재하는 모든 데이터베이스 이름을 알아냅니다.
``` bash
python sqlmap.py -u "http://example.com/article.php?id=123" --dbs
```

**테이블 확인**
- `-D [DB명] --tables`로 특정 DB 내부의 테이블 목록을 뽑아냅니다.
``` bash
python sqlmap.py -u "http://example.com/article.php?id=123" -D user_db --tables
```

**데이터 덤프**
- `--dump` 옵션을 사용하여 테이블에 들어있는 모든 아이디와 비밀번호를 텍스트 파일로 저장합니다.
``` bash
python sqlmap.py -u "http://example.com/article.php?id=123" -D user_db -T users --dump
```

## 그래서 어케 쓰라고?
![](/assets/img/posts/tinkletinkle.gif)

앞서 살펴본 `SQL Injection` 공격들을 코드 한 줄로 막아주는 핵심 도구가 바로 `PreparedStatement`입니다.

### PreparedStatement 정의

`PreparedStatement`는 쿼리를 실행하기 위해 데이터베이스와 통신할 때, 쿼리 문장(구조)와 데이터(값)을 분리하여 
전달하는 방식이며 일반적인 `Statement`가 매번 쿼리를 새로 만드는 <span style="background-color:#6DB33F">**수동 방식**</span>이라면, `PreparedStatement`는 
이미 만들어진 틀에 내용물만 갈아 끼우는 <span style="background-color:#6DB33F">**자동화 방식**</span>이라고 이해하면 쉽습니다.

### 작동원리

|   단계     |                               설명                               |
|:--------:|:--------------------------------------------------------------:|
| 쿼리 틀 준비  | SQL 쿼리 뼈대 `(SELECT * FROM user WHERE id = ?)` 를 미리 작성하여 DB에 전송 |
| DB 측 컴파일 |           DB는 입력값이 들어오기 전에 쿼리 구조를 분석하고 실행 계획을 미리 컴파일           |
| 데이터 바인딩  |          사용자가 입력한 값은 이후 `?` 자리에 단순한 데이터 값으로 바인딩되어 전달           |

### SQL Injection 막는원리

가장 중요한 포인트는 <span style="background-color:#6DB33F">**DB가 이미 쿼리의 구조를 결정해버렸다**</span>는 점입니다.
- `Statement`
  - 사용자가 `' OR 1=1 --`을 입력하면 쿼리 문장 자체가 변해버려 DB가 이를 명령어로 인식합니다.
- `PreparedStatement` 
  - 이미 `id`라는 컬럼에서 값을 찾으라고 컴파일이 끝난 상태이며 공격자가 아무리 `SQL` 명령어를 넣어보내도, <br>DB는 이를 실행해야 할 명령이 아닌 `** '' OR 1=1 -- ` 이라는 이상한 이름의 문자열 데이터로만 취급합니다.

#### 스프링 라이브러리 내부
우리가 흔히 쓰는 스프링의 `MyBatis`나 `Spring Data JPA`는 내부적으로 이 과정을 자동화하여 개발자가 실수하지 <br>않도록 도와줍니다.

**MyBatis**
- 개발자가 직접 SQL을 작성하지만, 파라미터 바인딩 시 `#{ }` 문법을 사용하면 내부적으로 `PreparedStatement`를 생성합니다.
```xml
<select id="findById" resultType="User">
    SELECT * FROM user WHERE id = #{userId}
</select>
```
- 동작 원리
  - 위 코드를 읽어 `SELECT * FROM user WHERE id = ?`라는 쿼리 틀을 DB에 먼저 전달한 후 사용자가 입력한 
  `userId` 값을 JDBC 드라이버의 `setObject()` 메서드를 통해 안전하게 바인딩합니다.
- 주의
  - 만약 `${ }` 문법을 사용하면 값이 쿼리에 그대로 합쳐지는 `Statement` 방식이 되어 `SQL Injection`에 <br>노출되므로 반드시 `#{ }`를 사용해야 합니다.

**Spring Data JPA**
- 메서드 이름만으로 쿼리를 생성하거나 `@Query` 어노테이션을 사용할 때 기본적으로 파라미터 바인딩 방식을 
사용합니다.
```java
@Query("SELECT u FROM User u WHERE u.id = :userId")
Optional<User> findById(@Param("userId") String userId);
```
- 동작 원리
  - Hibernate는 위 JPQL을 파싱하여 실제 DB SQL로 변환할 때 `?`가 포함된 `PreparedStatement`를 생성하는데 
  이 방법으로 개발자가 직접 쿼리를 조합하지 않아도 라이브러리 레벨에서 엄격히 분리해줍니다.

**라이브러리 내부**
- 사용자가 입력한 값을 쿼리 문자열에 직접 더하는(`+`) 것이 아니라, JDBC 인터페이스의 `ps.setXXX()` 메서드를 
호출하도록 설계되어 있는데 이 과정에서 입력값에 포함된 위험한 특수문자나 SQL 예약어는 실행 명령이 아닌 
단순한 텍스트 데이터로 처리되어 공격을 원천 봉쇄합니다.

## 마무리

`SQL Injection`의 다양한 공격 유형과 이를 물리적으로 차단하는 `PreparedStatement`의 메커니즘을 살펴보았는데 
<br>핵심을 다시 정리하면 이렇습니다.

1. `SQL Injection`은 사용자의 입력값이 SQL 명령어로 해석될 때 발생한다.

2. `PreparedStatement`는 **쿼리 구조**와 **데이터**를 엄격히 분리하여 컴파일한다.

3. 우리가 사용하는 `MyBatis`나 `JPA`는 내부적으로 이 과정을 자동화하여 우리 코드를 보호하고 있다.
