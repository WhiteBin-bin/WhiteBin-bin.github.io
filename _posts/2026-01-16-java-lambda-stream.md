---
title: "Lambda & Stream"
date: 2026-01-16 17:30:00 +0900
categories: [Language]
subcategory: Java
description: "Lamda & Stream에 대해서 적어봤습니다."
og_image: /assets/img/posts/Lambda&Stream.png
---

![](/assets/img/posts/Lambda&Stream.png)

## 주제 선정 이유

프로젝트를 진행하다 보면 리스트 데이터를 가공하는 과정에서 자연스럽게 `for`문과 `if`문을 반복적으로 사용하게 
되며, 코드가 점점 복잡해지고 가독성이 떨어지는 상황을 마주하게 됩니다.

기존의 명령형 방식은 동작 과정을 하나씩 나열하는 데에는 익숙하지만, 필터링이나 변환 로직이 많아질수록 전체 흐름을 한눈에 파악하기 어려워지게 되며 <span style="background-color:#6DB33F">**데이터를 더 직관적으로 처리할 수 있는 방법은 없을까**</span>라는 고민이 자연스럽게 
들게 되고 단순히 반복문을 줄이는 것을 넘어, 어떻게 반복할 것인가보다는 무엇을 할 것인가에 집중하는 방식이 코드의 가독성과 유지보수성을 높이는 데 더 적합하다는 생각이 들게 되며, 이러한 관점에서 람다와 스트림을 이해할 필요성을 느끼게 됩니다.

그래서 이번 글에서는 <span style="background-color:#6DB33F">**Lambda와 Stream을 중심으로 선언형 프로그래밍 방식이 어떻게 코드의 구조를 개선하고 데이터 처리를 효율적으로 만들어주는지**</span>에 대해 정리해보고자 합니다.

## 람다식이 뭘까?

한 마디로 해서 하나의 식 (`expression`)으로 표현한 것이며 함수를 변수처럼 다룰 수 있게 해줘서 코드를 획기적으로 줄여주는 자바 8의 핵심 기능입니다.

> 익명 함수는 이름 없이 독립적으로 사용되며, 일급 객체는 변수에 할당하거나 인자로 전달할 수 있어 코드의 유연성을 높입니다.

### 익명 클래스 vs 람다식

기존 자바에서는 인터페이스의 메서드를 구현하기 위해 복잡한 익명 내부 클래스 형식을 빌려야 했지만 람다를 사용하면 행위 그 자체에만 집중할 수 있습니다.

#### 기존 익명 클래스 방식

```java
Collections.sort(members, new Comparator<Member>() {
    @Override
    public int compare(Member m1, Member m2) {
        return m1.getAge() - m2.getAge();
    }
});
```

#### 람다식 적용 후

```java
Collections.sort(members, (m1, m2) -> m1.getAge() - m2.getAge());
```

#### 람다식의 문법 규칙

람다식은 화살표 `->`를 기준으로 왼쪽에는 매개변수, 오른쪽에는 실행블록이 위치합니다.
- 매개 변수 타입 추론
  - 컴파일러가 문맥을 통해타입을 알 수 있다면 생략이 가능합니다.
- 괄호 생략
  - 매개변수가 딱 하나라면 `()`를 생략할 수 있고 매개변수가 없거나 두 개 이상이면 필수입니다.
- 중괄호 생략
  - 실행 코드가 한 줄이라면 `{}`를 생략할 수 있으며, 이때 `return`문도 생략해야 합니다.

#### 메서드 참조

람다식이 단 하나의 메서드만 호출하는 경우, 불필요한 매개변수 선언을 생략하고 `::` 기호를 사용하여 극단적으로 
코드를 줄일 수 있습니다.

```text
클래스이름::메서드이름 또는 참조변수::메서드이름
```

| 구분 | 람다식 표현 | 메서드 참조 표현 |
| :-: | :-: | :-: |
| 정적 메서드 | `(s) -> Integer.parseInt(s)` | `Integer::parseInt` |
| 인스턴스 메서드 | `(s) -> System.out.println(s)` | `System.out::println` |
| 특정 객체 메서드 | `(m) -> m.getName()` | `Member::getName` |

#### 함수형 인터페이스

람다식은 <span style="background-color:#6DB33F">**아무 인터페이스나 쓸 수 있는것이 아니고 오직 단 하나의 추상 메서드만 가진 인터페이스에서만 사용이 <br>가능하고 이를 함수형 인터페이스**</span>라 합니다.
- `@FunctionalInterface`
  - 해당 인터페이스가 함수형 인터페이스 조건을 만족하는지 컴파일러가 체크해 주는 어노테이션입니다.
- 함수형 프로그래밍의 기반
  - 람다식은 이 인터페이스의 단 하나뿐인 메서드를 구현하는 객체로 취급됩니다.

### 활용 예시

#### 쓰레드 생성

별도의 로직을 비동기로 실행할 때 사용하던 복잡한 선언이 간결해집니다.

- 익명 클래스

```java
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("비동기 실행");
    }
}).start();
```

- 람다식

```java
new Thread(() -> System.out.println("비동기 행")).start();
```

#### 리스트 정렬

객체 리스트를 특정 기준 (나이, 이름 등)으로 정렬할 때 좋습니다.

- 익명 클래스

```java
Collections.sort(names, new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.compareTo(b);
    }
});
```

- 람다식

```java
names.sort((a, b) -> a.compareTo(b));
```

## 스트림이 뭘까?

람다가 엔진이라면, 스트림은 그 엔진을 활용해서 데이터를 운반하고 가공하는 고속도로 파이프라인 같은 느낌입니다.

| 개념 | 설명 |
|:-:|:-:|
| 스트림 | 컬렉션이나 배열의 데이터를 람다식으로 처리하는 흐름 |
| 원본 데이터 보존 | 데이터를 변경하지 않고 읽기만 수행 |
| 일회용성 | 한 번 사용하면 재사용 불가능 |
| 내부 반복 | `for`문 대신 스트림이 내부적으로 반복 처리 |

### 3단계 구조

공장의 컨베이어 벨트처럼 생성 -> 중간연산 -> 최종연산이라는 명확한 흐름을 가집니다.

#### 생성

데이터 소스(리스트, 배열 등)으로부터 스트림을 뽑아내는 단계입니다.

```java
List<String> list = Arrays.asList("apple", "banana", "cherry");
Stream<String> stream = list.stream();
```

#### 중간 연산

데이터를 필터링하거나 변환하는 단계이며 결과가 다시 스트림으로 변환되어 여러 번 연결할 수 있습니다.

- `filter()`, `distinct()`
  - 조건에 맞는 데이터를 걸러내거나 중복을 제거할 때 사용합니다.

  ```java
  List<Integer> numbers = Arrays.asList(1, 2, 2, 3, 4, 4, 5, 6);

  numbers.stream()
      .distinct()                // 중복 제거: [1, 2, 3, 4, 5, 6]
      .filter(n -> n % 2 == 0)   // 짝수만 필터링
      .forEach(System.out::print); 

  // 결과: 246
  ```
- `map()`, `flatMap()`
  - 데이터를 다른 형태로 바꾸거나, 중첩된 구조를 한 줄로 펼칠때 사용합니다.

  ```java
    // map: 이름을 대문자로 변환
  List<String> names = Arrays.asList("apple", "banana", "cherry");
  names.stream()
      .map(String::toUpperCase)
      .forEach(System.out::println); // APPLE, BANANA, CHERRY

  // flatMap: 중첩 리스트를 단일 리스트로 변환
  List<List<String>> complexList = Arrays.asList(
      Arrays.asList("A", "B"),
      Arrays.asList("C", "D")
  );
  complexList.stream()
      .flatMap(Collection::stream)
      .forEach(System.out::print); // ABCD
  ```
- `sorted()`
  - 데이터를 오름차순 또는 내림차순으로 정렬합니다.

  ```java
  List<String> list = Arrays.asList("D", "B", "A", "C");

  list.stream()
      .sorted()                             // 오름차순: A, B, C, D
      .sorted(Comparator.reverseOrder())    // 내림차순: D, C, B, A
      .forEach(System.out::print);
  ``` 
- `limit()`, `skip()`
  - 데이터의 일부분만 가져올 때 유용합니다.

  ```java
  List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

  list.stream()
      .skip(3)   // 처음 3개(1, 2, 3) 건너뛰기
      .limit(5)  // 그 다음부터 5개만 선택
      .forEach(System.out::print);

  // 결과: 45678
  ``` 
- `peek()`
  - 최종 연산 전, 데이터가 어떻게 변하고 있는지 디버깅 용도로 확인하기 좋습니다.

  ```java
  List<Integer> result = IntStream.of(1, 2, 3, 4, 5)
    .filter(n -> n % 2 != 0)
    .peek(n -> System.out.println("필터링된 값: " + n)) // 중간 출력
    .map(n -> n * 10)
    .boxed()
    .collect(Collectors.toList());
  ```

#### 최종 연산

중간 연산을 통해 가공된 스트림을 닫고 최종 결과를 선출하는 단계이며 이 연산이 실행되어야만 앞선 중간 연산들이 
실제로 동작합니다.
- `collect()`
  - 스트림의 요소를 리스트, 셋, 맵 등 컬렉션으로 변환하고 가장 많이 사용되는 연산입니다.

  ```java
  List<String> list = stream.collect(Collectors.toList());
  ```
- `forEach()`
  - 각 요소를 순회하며 특정 작업을 수행하고 주로 결과를 출력할 때 사용합니다.
  
  ```java
  fruits.stream().forEach(System.out::println);
  ```
- `anyMatch()`, `allMatch()`, `noneMatch()`
  - 조건에 맞는 요소가 있는지 확인하여 `boolean`을 반환합니다.
  
  ```java
  boolean hasApple = fruits.stream().anyMatch(f -> f.equals("apple"));
  ```
- `count()`, `max()`, `min()`
  - 요소의 개수나 최댓값, 최솟값을 구합니다.
  
  ```java
  List<Integer> numbers = Arrays.asList(1, 5, 3, 8, 2);

  long count = numbers.stream().count();               // 결과: 5
  Optional<Integer> max = numbers.stream().max(Integer::compare); // 결과: 8
  Optional<Integer> min = numbers.stream().min(Integer::compare); // 결과: 1
  ```
- `reduce()`
  - 모든 요소를 소모하며 누전 연산을 수행합니다.
  
  ```java
  int sum = numbers.stream().reduce(0, (a, b) -> a + b);
  ```

### for문 vs 스트림

리스트에서 `b`를 포함한 과일만 골라 대문자로 변환하여 새로운 리스트 만들기라는 로직을 통해 얼마나 코드가 
직관적으로 변하는지 확인해 볼 수 있습니다.

#### 기존 for문 방식

```java
List<String> result = new ArrayList<>();
for (String fruit : fruits) {
    if (fruit.contains("b")) {
        result.add(fruit.toUpperCase());
    }
}
```

#### 스트림 파이프라인 방식

```java
List<String> result = fruits.stream()
    .filter(f -> f.contains("b"))  // 필터링 (가공)
    .map(String::toUpperCase)      // 변환 (가공)
    .collect(Collectors.toList()); // 수집 (결과)
```

## 그래서 어떻게 쓰라고?

![](/assets/img/posts/hmm.gif)

결론부터 말하자면, 우리는 람다와 스트림을 적극 활용하되, 로직의 복잡도와 팀의 생산성을 고려한 유연한 선택을 해야 하는데 모든 `for`문을 `Lambda`와 스트림으로 억지로 바꾸는 것은 오히려 코드의 복잡성을 높이고 가독성을 해칠 수 
있고 때로는 단순한 반복문이 훨씬 직관적일 수 있기 때문입니다.

### 예시

#### 선언형 스타일의 우선 적용

필터링, 매핑, 정렬이 연속되는 비즈니스 로직에서는 가급적 스트림과 람다를 사용해야 하고 어떻게 반복할까 라는 
물리적 구현보다 무엇을 할까 라는 논리적 의도를 드러내는 것이 훨씬 중요하기 때문입니다.

```java
// 예시: 활성화된 유저 중 나이가 20세 이상인 유저의 이름만 추출
// Before: 어떻게(How) 반복하고 조건문을 걸지 고민함
List<String> userNames = new ArrayList<>();
for (User user : users) {
    if (user.isActive() && user.getAge() >= 20) {
        userNames.add(user.getName());
    }
}

// After: 무엇(What)을 할지 로직의 의도가 드러남
List<String> userNames = users.stream()
    .filter(user -> user.isActive() && user.getAge() >= 20) // 필터링
    .map(User::getName)                                    // 이름 추출
    .collect(Collectors.toList());                         // 수집
```

#### 가독성과 복잡도의 균형

스트림 파이프라인이 너무 길어지거나 람다 내부 로직이 복잡해진다면, 무리하게 한 줄로 짜기보다 메서드 참초(`::`)를 활용하거나 과감히 기준`for`문 으로 회귀하여 직관성을 확보합니다.

```java
// 예시: 가독성이 파괴된 스트림 vs 메서드 참조/for문
// 람다식이 길어지면 읽기가 매우 힘들어짐
list.stream()
    .filter(data -> {
        // 람다 안에 if문, try-catch 등 복잡한 로직이 들어감
        if (data.isValid()) return true;
        else return false;
    })
    .map(data -> data.getValue() * 100 / 2) 
    .collect(Collectors.toList());

// 메서드 참조(::)를 활용하여 로직을 분리
list.stream()
    .filter(Data::isValid)
    .map(this::calculateValue) // 복잡한 로직은 메서드로 추출
    .collect(Collectors.toList());

// 혹은 그냥 for문이 훨씬 직관적일 때 (복잡한 제어 필요 시)
for (Data data : list) {
    if (!data.isValid()) continue;
    // 복잡한 연산 및 외부 변수 수정 등
}
```

#### 상황에 맞는 병렬 처리

대량의 데이터를 처리해야 하는 성능 임계점에서는 `parallelStream()` 도입을 고려하되, 단순한 작업에서는 오히려 
오버헤드가 발생할 수 있음을 인지하고 사용해야 합니다.

```java
// 예시: 병렬 처리가 유리한 경우 vs 불리한 경우

// 병렬 처리가 유리할 때 (대량의 데이터 + 무거운 연산)
long sum = LongStream.rangeClosed(1, 10_000_000)
    .parallel() // 여러 코어를 사용하여 빠르게 합산
    .sum();

// 오히려 손해인 경우 (데이터가 적거나 단순 출력)
// 스레드를 관리하는 비용이 더 커서 일반 스트림보다 느려질 수 있음
List<String> shortList = Arrays.asList("a", "b", "c");
shortList.parallelStream()
    .forEach(System.out::println); // 순서도 보장 안 되고 성능 이점도 없음
```

### 추가

#### Collectors

`collect()` 메서드가 스트림의 요소를 수집하는 최종연산이라면, `Collectors`는 그 요소를 어떻게 수집할지 정의해 
놓은 유틸리티 클래스입니다.

- `toList()`, `toSet()`, `toMap()`
  - 가장 많이 쓰이는 방식으로, 가공된 데이터를 다시 컬렉션으로 변환합니다.

  ```java
  // toList: 짝수만 골라 리스트로 변환
  List<Integer> evenList = numbers.stream()
      .filter(n -> n % 2 == 0)
      .collect(Collectors.toList());

  // toSet: 중복을 제거하고 셋으로 변환
  Set<String> fruitSet = fruits.stream()
      .collect(Collectors.toSet());

  // toMap: 유저 리스트를 ID를 키로 하는 Map으로 변환
  Map<Long, User> userMap = users.stream()
      .collect(Collectors.toMap(User::getId, user -> user));
  ```
- `joining()`
  - 리스트의 문자열 요소들을 하나로 합칠 때 유용하며, 접두사와 접미사도 지정 가능합니다.

  ```java
  List<String> names = Arrays.asList("Kim", "Lee", "Park");

  // 구분자만 넣는 경우
  String result1 = names.stream()
      .collect(Collectors.joining(", ")); // "Kim, Lee, Park"

  // 구분자, 접두사, 접미사 모두 넣는 경우
  String result2 = names.stream()
      .collect(Collectors.joining(", ", "[", "]")); // "[Kim, Lee, Park]"
  ```
- `counting()`, `summingInt()`, `averagingDouble()`
  - 최종 단계에서 숫자 데이터를 집계할 때 사용합니다.
  
  ```java
  // 요소 개수 세기
  long count = users.stream()
      .filter(User::isActive)
      .collect(Collectors.counting());

  // 점수 합계 구하기
  int totalScore = users.stream()
      .collect(Collectors.summingInt(User::getScore));

  // 나이 평균 구하기
  double averageAge = users.stream()
      .collect(Collectors.averagingDouble(User::getAge));
  ```
- `groupingBy()`, `partitioningBy()`
  - 데이터를 특정 기준에 따라 Map 형태로 분류할 때 사용하며, 스트림의 가장 강력한 기능 중 하나입니다.
  
  ```java
  // groupingBy: 부서별로 유저 그룹화 (Map<Department, List<User>>)
  Map<Department, List<User>> userByDept = users.stream()
      .collect(Collectors.groupingBy(User::getDepartment));

  // partitioningBy: 합격/불합격 두 그룹으로 분할 (Map<Boolean, List<User>>)
  Map<Boolean, List<User>> passOrFail = users.stream()
      .collect(Collectors.partitioningBy(user -> user.getScore() >= 80));
  ```

사실 `Collectors`를 쓰지 않고 `forEach`로 하나씩 리스트에 담아도 결과는 같습니다만 `Collectors`를 활용하는 순간, 복잡한 그룹화나 통계 추출 같은 노가다성 로직이 단 한 줄의 선언으로 끝나는 마법을 경험할 수 있습니다.

## 마무리

람다와 스트림을 정리하면서 느낀 핵심을 다시 정리해보면 다음과 같습니다.

1. 익명 클래스의 복잡한 구조에서 벗어나 람다식을 통해 행위를 간결하게 표현할 수 있게 되며, 코드의 본질에 더욱 
집중할 수 있습니다.
2. 반복문 중심의 명령형 방식에서 벗어나 무엇을 할 것인지에 집중하는 선언형 스타일로 데이터를 처리할 수 있게 
되며, 전체 흐름을 더 직관적으로 이해할 수 있습니다. 
3. 스트림의 파이프라인 구조와 `Collectors`를 활용하면 복잡한 데이터 가공 과정도 간결하고 안전하게 표현할 수 
있습니다.  
4. 람다와 스트림은 강력한 도구이지만, 모든 상황에 무조건 적용하기보다는 가독성과 유지보수성을 고려하여 기존 
방식과 적절히 병행하는 것이 중요하다는 점을 느낄 수 있습니다.
