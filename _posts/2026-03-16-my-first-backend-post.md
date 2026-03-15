---
title: "SQL Injection"
date: 2024-03-16 11:13:00 +0900
categories: [BackEnd]
tags: [test, backend]
---

![](https://velog.velcdn.com/images/coding_haehyeon/post/dd148c81-35cb-448b-86ef-b0d1a97af9ae/image.png)

## 주제 선정 이유
Spring을 공부하다가 자주 접하게 된 말이 있는데
> "DB 쿼리를 날릴 때 Statement 대신 PreparedStatement를 사용해야 한다"

이 말이 나온 이유로 가장 많이 들었던 것이 바로 <span style="background-color:#6DB33F">**SQL Injection 방지**</span>였던것 같습니다.

처음에는 단순히 "보안에 좋구나" 하고 넘어갔지만, 문득 "대체 원리가 무엇이길래 쿼리 실행 방식 하나만으로 해킹을 막을 수 있는 걸까?" 하는 호기심이 생겼습니다. 

이번 기회에 SQL Injection의 공격 원리와 이를 방어하는 PreparedStatement의 내부 매커니즘을 깊이 있게 파헤쳐 보고자 이 주제를 선정하게 되었습니다.

## SQL Injection이 뭘까?
일단 SQL Injection이란 웹을 해킹하는데 가장 보편적으로 사용되는 기법 중 하나로, 
웹 페이지의 입력값을 통해 SQL문에 악성 코드를 주입하여 데이터베이스에 접근하는 방식입니다.

웹 애플리케이션은 사용자의 클릭이나 입력에 따라 동적으로 데이터를 가져오기 위해, 사용자가 
입력한 값을 포함한 SQL 쿼리를 생성하는데 이 과정에서 입력값 검증이 제대로 이루어지지 않으면, 공격자가 악의적으로 조작된 쿼리를 주입해 개발자가 의도하지 않은 데이터에 접근하거나 조작할 수 있게 됩니다.

## 공격 유형
데이터베이스의 정보를 탈취하는 방식에 따라 크게 5가지 유형으로 나뉩니다.

### 1. In-band SQL Injection
공격자가 서버와 동일한 통신 경로를 사용하여 공격하고 결과를 직접 확인하면서 참 조건을 이용한 인증 우회 방법입니다.
``` sql
SELECT * FROM user WHERE id='admin' AND password='password';
```
위에처럼 쿼리가 작성되었다고 가정했을때 입력창에 ' OR 1=1 -- 를 입력하면
``` sql
SELECT * FROM user WHERE id='' OR 1=1--' AND password='password';
```
이런식으로 쿼리가 조작되면서 아래와 같이 동작하게 됩니다.
``` sql
SELECT * FROM user WHERE id='' OR 1=1;
```
이 쿼리는 id가 무엇이든 상관없이 <span style="background-color:#6DB33F">**'1=1'이라는 조건이 항상 참(True)**</span>이기 때문에, 데이터베이스는 인증 절차를 무시하고 테이블에 있는 사용자 정보를 반환하게 됩니다. 

결과적으로 공격자는 비밀번호 없이도 로그인을 성공하거나 전체 사용자 데이터를 탈취할 수 있습니다.

### 2. UNION-based SQL Injection
UNION 연산자를 이용해 공격자가 원하는 데이터를 기존 쿼리의 결과와 결합하여 노출시키는 방식입니다.
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
이 쿼리는 원래의 조회 결과 뒤에 공격자가 임의로 만든 <span style="background-color:#6DB33F">**'1, 1'이라는 가상의 행을 강제로 결합**</span>
하라는 의미이며 이걸 통해서 데이터베이스의 컬럼 개수를 파악할 수 있습니다. 

만약 원래 쿼리가 반환하는 컬럼 수와 UNION 뒤에 붙인 컬럼 수가 다르면 에러가 발생하며 <span style="background-color:#6DB33F">**SELECT 1, 1, 1**</span>처럼 숫자를 늘려가며 에러가 나지 않는 지점을 찾으면 해당 테이블의 정확한 구조를 알아낼 수 있습니다.

공격자는 컬럼 개수를 맞춘 뒤, 숫자 1 대신 실제 DB 정보를 출력하는 쿼리를 삽입하여 화면에 나오지 않아야 할 민감한 데이터들을 통째로 가로챌 수 있습니다.
``` sql
admin' UNION SELECT id, password FROM users --
```
위와 같이 컬럼이 2개인 것을 알아낸 후, 아이디와 비밀번호를 직접 조회할 수 있다.

### 3. Blind SQL Injection - Boolean-based
조건의 참 / 거짓 여부를 통해 데이터 존재 여부를 추론하는 방식이다.
``` sql
SELECT * FROM users WHERE username='$input';
```
위에처럼 쿼리가 작성되었다고 가정했을때 입력창에 
`' OR SUBSTRING((SELECT database()),1,1)='t' --`을 입력하면
``` sql
SELECT * FROM users WHERE username = '' OR SUBSTRING((SELECT database()),1,1)='s' -- ';
```
위와 같이 쿼리가 조작되면서 아래와 같이 동작하게 됩니다.
``` sql
SELECT * FROM users WHERE username = '' OR SUBSTRING((SELECT database()),1,1)='s';
```
이 쿼리는 현재 사용 중인 데이터베이스 이름의 첫 글자가 's'인지 확인하는 쿼리문이며, <span style="background-color:#6DB33F">**조건이 일치하여 참(True)이 될 경우**</span> 데이터베이스는 로그인 성공이나 특정 데이터를 포함한 정상 페이지를 반환하게 됩니다.
### 4. Blind SQL Injection - Time-Based
이 방법은 시간 지연을 이용한 조건을 판단하는 방식입니다.
``` sql
SELECT * FROM users WHERE username = '$input';
```
위에처럼 쿼리가 작성되었다고 가정했을때 입력창에 `' OR IF(SUBSTRING((SELECT database()),1,1)='s', SLEEP(5), 0) -- `을 입력하면
``` sql
SELECT * FROM users WHERE username = '' OR IF(SUBSTRING((SELECT database()),1,1)='s', SLEEP(5), 0) -- ';
```
위와 같이 쿼리가 조작되면서 아래와 같이 동작하게 됩니다.
``` sql
SELECT * FROM users WHERE username = '' OR IF(SUBSTRING((SELECT database()),1,1)='s', SLEEP(5), 0);
```
이 쿼리는 데이터베이스 이름의 첫 글자가 's'인지 확인하고, <span style="background-color:#6DB33F">**참(True)일 경우 SLEEP(5) 함수를 실행**</span>하기 때문에 데이터베이스는 즉시 응답하는 대신 의도적으로 5초간 지연된 후 결과를 반환하게 됩니다.
### 5. Mass SQL Injection
자동화된 도구(SQLMap 등)를 사용해 다수의 필드/테이블/레코드에 동시에 공격하는 방식입니다.
``` bash
python sqlmap.py -u "http://example.com/article.php?id=123" --batch
```
이 공격은 특정 데이터 하나를 노리는 것을 넘어, 데이터베이스 내의 <span style="background-color:#6DB33F">**모든 테이블과 레코드를 대상으로 악성 코드를 삽입(Update)하거나 정보를 변조**</span>하라는 의미를 가집니다.
#### SQLMap 공격 예시
**1. 취약점 확인** : `--batch` 옵션으로 해당 URL이 공격 가능한지 자동 진단합니다.
``` bash
python sqlmap.py -u "http://example.com/article.php?id=123" --batch
```
**2. DB 정보 추출** : `--dbs` 옵션을 붙여 현재 서버에 존재하는 모든 데이터베이스 이름을 알아냅니다.
``` bash
python sqlmap.py -u "http://example.com/article.php?id=123" --dbs
```
**3. 테이블 확인** : `-D [DB명] --tables`로 특정 DB 내부의 테이블 목록을 뽑아냅니다.
``` bash
python sqlmap.py -u "http://example.com/article.php?id=123" -D user_db --tables
```
**4. 데이터 덤프** :`--dump` 옵션을 사용하여 테이블에 들어있는 모든 아이디와 비밀번호를 텍스트 파일로 저장합니다.
``` bash
python sqlmap.py -u "http://example.com/article.php?id=123" -D user_db -T users --dump
```
## 그래서 어케 막으라고?
![](https://velog.velcdn.com/images/coding_haehyeon/post/501fbd06-a51e-447e-adcc-1ed2272ce187/image.gif)

앞서 살펴본 무시무시한 SQL Injection 공격들을 코드 한 줄로 막아주는 핵심 도구가 바로 PreparedStatement입니다.
### PreparedStatement 정의
PreparedStatement는 쿼리를 실행하기 위해 데이터베이스와 통신할 때, 쿼리 문장(구조)와 데이터(값)을 분리하여 전달하는 방식입니다. 

일반적인 Statement가 매번 쿼리를 새로 만드는 <span style="background-color:#6DB33F">**수동 방식**</span>이라면, PreparedStatement는 이미 만들어진 틀에 내용물만 갈아 끼우는 <span style="background-color:#6DB33F">**자동화 방식**</span>이라고 이해하면 쉽습니다.
### 작동원리
일반적인 Statement는 실행할 때마다 쿼리 분석 → 최적화 → 컴파일 과정을 거치지만, PreparedStatement는 과정이 다릅니다.

**1. 쿼리 틀 준비**: SQL 쿼리 뼈대`(SELECT * FROM user WHERE id = ?)`를 미리 작성하여 DB에 보냅니다.

**2. DB 측 컴파일**: DB는 입력값이 들어오기 전, 이 쿼리가 어떤 동작을 할지 미리 분석하고 컴파일해둡니다.

**3. 데이터 바인딩**: 사용자가 입력한 값은 나중에 ? 자리에 '단순한 데이터'로만 전달됩니다.
### SQL Injection 막는원리
가장 중요한 포인트는 <span style="background-color:#6DB33F">**DB가 이미 쿼리의 구조를 결정해버렸다**</span>는 점입니다.
- Statement: 사용자가 `' OR 1=1 --`을 입력하면 쿼리 문장 자체가 변해버려 DB가 이를 
명령어로 인식합니다.
- PreparedStatement: 이미 id라는 컬럼에서 값을 찾으라고 컴파일이 끝난 상태입니다. 
공격자가 아무리 SQL 명령어를 넣어보내도, DB는 이를 실행해야 할 명령이 아닌 <span style="background-color:#6DB33F">
  "** `' OR 1=1 -- " 이라는 아주 길고 이상한 이름의 문자열**</span> 데이터로만 취급합니다.
#### 스프링 라이브러리 내부
우리가 흔히 쓰는 스프링의 MyBatis나 Spring Data JPA는 내부적으로 이 과정을 자동화하여 개발자가 실수하지 않도록 도와줍니다.

**1. MyBatis에서의 처리 (# 문법)**
MyBatis는 개발자가 직접 SQL을 작성하지만, 파라미터 바인딩 시 `#{ }` 문법을 사용하면 내부적으로 PreparedStatement를 생성합니다.
``` XML
<select id="findById" resultType="User">
    SELECT * FROM user WHERE id = #{userId}
</select>
```
- 동작 원리: MyBatis 라이브러리는 위 코드를 읽어 `SELECT * FROM user WHERE id = ?`라는 쿼리 틀을 DB에 먼저 전달한 후 사용자가 입력한 userId 값을 JDBC 드라이버의 setObject() 메서드를 통해 안전하게 바인딩합니다.

- 주의: 만약 `${ }` 문법을 사용하면 값이 쿼리에 그대로 합쳐지는 Statement 방식이 되어 SQL Injection에 노출되므로 반드시 `#{ }`를 사용해야 합니다.

**2. Spring Data JPA에서의 처리**
JPA는 메서드 이름만으로 쿼리를 생성하거나 `@Query` 어노테이션을 사용할 때 기본적으로 파라미터 바인딩 방식을 사용합니다.
```java
@Query("SELECT u FROM User u WHERE u.id = :userId")
Optional<User> findById(@Param("userId") String userId);
```
- 동작 원리: JPA 구현체인 Hibernate는 위 JPQL을 파싱하여 실제 DB SQL로 변환할 때 ?가 포함된 PreparedStatement를 생성하는데 이 방법으로 개발자가 직접 쿼리를 조합하지 않아도 라이브러리 레벨에서 데이터와 명령어를 엄격히 분리해줍니다.

**3. 라이브러리 내부의 핵심**
사용자가 입력한 값을 쿼리 문자열에 직접 더하는(+) 것이 아니라, JDBC 인터페이스의 `ps.setXXX()` 메서드를 호출하도록 설계되어 있는데 이 과정에서 입력값에 포함된 위험한 특수문자나 SQL 예약어는 실행 명령이 아닌 단순한 텍스트 데이터로 처리되어 공격을 원천 봉쇄합니다.

## 마무리
SQL Injection의 다양한 공격 유형과 이를 물리적으로 차단하는 PreparedStatement의 메커니즘을 살펴보았는데 핵심을 다시 정리하자면 이렇습니다.

1. SQL Injection은 사용자의 입력값이 SQL 명령어로 해석될 때 발생한다.

2. PreparedStatement는 '쿼리 구조'와 '데이터'를 엄격히 분리하여 컴파일한다.

3. 우리가 사용하는 MyBatis나 JPA는 내부적으로 이 과정을 자동화하여 우리 코드를 보호하고 있다.