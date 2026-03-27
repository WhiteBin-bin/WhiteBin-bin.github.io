---
title: "TypeScript"
date: 2026-03-25 15:44:20 +0900
categories: [FrontEnd]
subcategory: TypeScript
description: "TypeScript에 대해서 적어봤습니다."
og_image: /assets/img/posts/TypeScript.png
---

![](/assets/img/posts/TypeScript.png)

## 주제 선정 이유  

프론트엔드 개발을 진행하면서 `JavaScript`를 기반으로 기능을 구현하고 화면을 구성하는 데에는 점점 익숙해졌지만 
코드의 규모가 커지고 로직이 복잡해질수록 변수의 타입이 명확하지 않거나, 예상하지 못한 값이 들어오는 문제로 인해 런타임에서 오류가 발생하는 경우를 종종 경험하게 되었고 특히 협업이나 리팩토링 과정에서는 코드의 의도를 파악하기 어렵거나, 수정 이후 기존 기능이 깨지는 상황을 겪으면서 코드의 안정성과 가독성에 대한 고민이 생기게 되었습니다.  

그래서 이번 글에서는 `TypeScript`를 사용할 때 자주 접하게 되는 <span style="background-color:#6DB33F">**타입 시스템을 중심으로, 타입 정의와 추론, <br>인터페이스와 제네릭 같은 핵심 개념들이 어떤 역할을 가지는지 그리고 실제 코드에서 어떻게 활용되는지**</span>를 
정리해보고자 합니다.

## TypeScript가 뭘까?

`TypeScript`는 `JavaScript`에 정적 타입 시스템을 추가한 언어로, 코드 작성 단계에서 변수와 함수의 타입을 명확하게 정의할 수 있도록 도와주는 도구인데 기존 `JavaScript`는 동적 타입 언어이기 때문에 변수에 어떤 값이 들어올지 실행 
시점까지 알 수 없고, 이로 인해 예상하지 못한 오류가 발생할 가능성이 존재합니다.  

```javascript
function add(a, b) {
  return a + b;
}

add(1, "2"); // "12" (의도하지 않은 결과)
```

이처럼 타입이 명확하지 않으면 코드가 정상적으로 실행되더라도 개발자가 의도하지 않은 결과가 나올 수 있는 반면 `TypeScript`에서는 변수와 함수의 타입을 명시할 수 있기 때문에 이러한 문제를 사전에 방지할 수 있습니다.

```typescript
function add(a: number, b: number): number {
  return a + b;
}

add(1, "2"); // ❌ 컴파일 에러
```

위 코드처럼 잘못된 타입이 전달될 경우 실행 전에 에러를 확인할 수 있기 때문에, 런타임 오류를 줄이고 더 안정적인 
코드를 작성할 수 있으며 단순히 타입을 지정하는 것을 넘어서, 타입 추론, 인터페이스, 제네릭 등 다양한 기능을 
제공하며 코드의 구조를 더 명확하게 표현할 수 있도록 도와줍니다.

즉, `TypeScript`는 기존 `JavaScript`의 유연함은 유지하면서도 <span style="background-color:#6DB33F">**코드의 안정성과 가독성을 높여주는 확장된 언어**</span>라고 
볼 수 있습니다.

> `TypeScript`는 브라우저가 직접 실행할 수 없기 때문에, `JavaScript`로 변환된 뒤 실행됩니다.

### 타입 기본 설정

변수와 값에 타입을 명시하여 코드의 형태를 더 명확하게 표현할 수 있으며,  기본적인 타입부터 시작해 점점 더 복잡한 
구조를 정의하는 방식으로 확장해 나갈 수 있습니다.

#### 기본 타입

가장 기본적인 타입으로, 변수에 들어올 값의 형태를 명확하게 정의할 수 있습니다.

```typescript
let name: string = "bin";
let age: number = 20;
let isAdmin: boolean = false;
```

#### 배열 타입

같은 타입의 데이터를 여러 개 다룰 때 사용합니다.

```typescript
let numbers: number[] = [1, 2, 3];
```

#### 튜플 타입

각 요소의 타입이 고정된 배열을 표현할 수 있습니다.

```typescript
let user: [string, number] = ["bin", 20];
```

#### 함수 타입

함수의 매개변수와 반환 값을 타입으로 제한할 수 있습니다.

```typescript
function add(a: number, b: number): number {
  return a + b;
}
```

#### 유니온 타입

하나의 변수에 여러 타입을 허용할 수 있습니다.

```typescript
let value: string | number;

value = "hello";
value = 123;
```

### 타입 심화 설정

기본적인 타입 선언을 넘어, 객체 구조를 정의하고 다양한 상황에 맞게 타입을 유연하게 확장할 수 있습니다.

#### 인터페이스 (Interface)

객체의 구조를 정의할 때 사용되며, 코드의 일관성과 재사용성을 높일 수 있습니다.

```typescript
interface User {
  name: string;
  age: number;
  isAdmin?: boolean;
}

const user: User = {
  name: "bin",
  age: 20,
};
```

#### 타입 별칭

타입에 이름을 부여하여 재사용할 수 있으며, 유니온이나 복잡한 타입을 정의할 때 유용합니다.

```typescript
type ID = string | number;

type User = {
  name: string;
  age: number;
};
```

#### 제네릭

타입을 변수처럼 사용하여 다양한 타입을 유연하게 처리할 수 있습니다.

```typescript
function getValue<T>(value: T): T {
  return value;
}

const numberValue = getValue<number>(10);
const stringValue = getValue<string>("hello");
```

#### 제네릭 인터페이스

공통 구조를 가지면서 내부 타입만 바뀌는 경우에 활용할 수 있습니다.

```typescript
interface ApiResponse<T> {
  data: T;
  status: number;
}

const response: ApiResponse<string> = {
  data: "success",
  status: 200,
};
```

#### Intersection 타입

여러 타입을 결합하여 하나의 타입으로 사용할 수 있습니다.

```typescript
type User = {
  name: string;
};

type Admin = {
  role: string;
};

type SuperUser = User & Admin;

const admin: SuperUser = {
  name: "bin",
  role: "admin",
};
```

#### 타입 가드

조건문을 통해 타입을 좁혀 보다 안전하게 사용할 수 있습니다.

```typescript
function printValue(value: string | number) {
  if (typeof value === "string") {
    console.log(value.toUpperCase());
  } else {
    console.log(value.toFixed(2));
  }
}
```

#### 타입 추론

타입을 명시하지 않아도 값을 기반으로 타입을 자동으로 추론합니다.

```typescript
let count = 10; // number로 추론
let message = "hello"; // string으로 추론
```

명시적으로 타입을 작성하지 않아도 코드의 타입 안정성을 유지할 수 있습니다.

## 그래서 어떻게 쓰라고?

![](/assets/img/posts/pepe.gif)

실제로는 모든 기능을 한 번에 사용하는 것이 아니라 상황에 맞게 필요한 타입을 선택해서 사용하는 것이 중요한데 
초기에는 변수와 함수에 기본 타입을 지정하는 것부터 시작하고, 코드의 규모가 커지면서 동일한 구조가 반복되면 
인터페이스를 통해 구조를 정의하며, 다양한 타입을 처리해야 하는 경우에는 제네릭을 활용하여 재사용성을 높이는 
방식으로 확장해 나갈 수 있습니다.  

```typescript
// 구조 정의
interface User {
  id: number;
  name: string;
}

// 제네릭 응답 타입
interface ApiResponse<T> {
  data: T;
  status: number;
}

// 함수에서 타입 활용
function getUser(response: ApiResponse<User>): string {
  return response.data.name;
}

// 실제 데이터
const response: ApiResponse<User> = {
  data: {
    id: 1,
    name: "bin",
  },
  status: 200,
};

const userName = getUser(response);
console.log(userName); // bin
```

이처럼 `TypeScript`는 단순히 타입을 추가하는 것이 아니라, 코드의 구조를 명확하게 정의하고 재사용 가능한 형태로 
확장해 나가는 방식으로 사용하는 것이 중요하며 기본 타입 → 구조 정의 → 확장이라는 흐름을 기반으로 점진적으로 
적용하는 것이 효과적으로 사용하는 방법이라고 볼 수 있습니다.

## 마무리

`TypeScript`를 정리하면서 느낀 핵심을 다시 정리해보면 다음과 같습니다.

1. `TypeScript`는 `JavaScript`에 정적 타입 시스템을 추가하여 코드의 안정성과 가독성을 높여주는 언어입니다.  
2. 기본 타입부터 시작해 인터페이스와 타입 별칭을 통해 구조를 정의하고, 제네릭을 활용하여 재사용성과 확장성을 높일 수 있습니다.  
3. 타입 추론과 타입 가드 등을 통해 개발 단계에서 오류를 사전에 방지할 수 있으며, 보다 예측 가능한 코드를 작성할 수 있습니다.  
4. `TypeScript`는 단순한 문법이 아니라, 코드의 구조를 명확하게 표현하고 유지보수성을 높이기 위한 도구입니다.