---
title: "Vue Composition API"
date: 2026-03-25 15:44:20 +0900
categories: [FrontEnd]
subcategory: Vue
---

![](/assets/img/posts/VueCompositionApi.png)

## 주제 선정 이유  

프론트엔드 개발을 진행하면서 `Vue`를 사용해 화면을 구성하고 상태를 관리하는 데에는 점점 익숙해졌고, 
최근에는 `Composition API`를 중심으로 컴포넌트를 작성하며 개발을 진행하게 되었습니다.  

처음에는 `setup()` 안에서 상태와 함수를 선언하는 방식 정도로만 이해하고 사용했지만, 실제로 코드를 
작성하다 보니 `ref`, `reactive`, `computed`, `watch` 같은 문법들이 각각 어떤 역할을 가지는지, 언제 어떤 방식을 선택해서 써야 하는지 헷갈리는 순간이 많았는데 특히 화면은 잘 동작하더라도 단순히 따라 쓰는 수준에 머무르면 문법의 차이만 아는 상태가 되기 쉽고, 구조를 스스로 설계하거나 상황에 맞게 적절한 방법을 선택하기는 
어렵다는 생각이 들었습니다.  

그래서 이번 글에서는 `Vue Composition API`를 사용할 때 자주 접하게 되는 <span style="background-color:#6DB33F">**ref, reactive, computed, watch를 중심으로 각각의 역할과 사용 방법, 그리고 어떤 상황에서 활용되는지**</span>를 정리해보고자 합니다.

## ref, reactive, computed, watch, watchEffect가 뭘까?

`Composition API`에서는 상태를 선언하고, 해당 상태를 기반으로 값을 계산하거나, 변화를 감지해 특정 로직을 수행하는 등 컴포넌트의 동작을 구성하기 위한 다양한 기능들을 제공합니다.  

그 중에서도 `ref`, `reactive`, `computed`, `watch`, `watchEffect`는 가장 기본이 되는 핵심 문법으로, 각각의 
역할을 이해하는 것이 `Composition API`를 제대로 사용하는 데 있어 중요한 시작점이라고 할 수 있습니다.  

### ref - 상태 선언

단일 값을 반응형 상태로 만들 때 사용하는 방법이며 주로 숫자, 문자열, 불리언과 같은 기본 타입 데이터를 
다룰 때 사용되고 값이 변경되면 해당 값을 사용하는 템플릿이 자동으로 업데이트되며 `ref`로 선언된 값은 `.value`를 통해 접근하고 수정해야 한다는 특징이 있습니다.  

```vue
<template>
  <div>
    <p>count: {{ count }}</p>
    <button @click="increase">증가</button>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const count = ref(0)

const increase = () => {
  count.value++
}
</script>
```

### reactive - 상태 선언

객체 형태의 데이터를 반응형으로 만들 때 사용하는 방법이며 여러 개의 상태를 하나의 객체로 묶어서 관리할 수 있으며,객체 내부의 값이 변경되면 해당 값을 사용하는 템플릿이 자동으로 업데이트되고 `ref`와 달리 
`.value` 없이 속성에 직접 접근할 수 있다는 특징이 있습니다.

```vue
<template>
  <div>
    <p>name: {{ user.name }}</p>
    <p>age: {{ user.age }}</p>
    <button @click="increaseAge">나이 증가</button>
  </div>
</template>

<script setup>
import { reactive } from 'vue'

const user = reactive({
  name: '홍길동',
  age: 20
})

const increaseAge = () => {
  user.age++
}
</script>
```

### computed - 계산된 값

기존 상태를 기반으로 새로운 값을 계산할 때 사용하는 기능이며 의존하고 있는 값이 변경될 때만 다시 계산되며, 불필요한 연산을 줄여 성능을 최적화할 수 있다는 특징이 있어서 주로 화면에 보여줄 값을 가공하거나, 여러 
상태를 조합해 새로운 값을 만들어낼 때 사용됩니다.

```vue
<template>
  <div>
    <p>이름: {{ firstName }} {{ lastName }}</p>
    <p>전체 이름: {{ fullName }}</p>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const firstName = ref('길동')
const lastName = ref('홍')

const fullName = computed(() => {
  return `${lastName.value} ${firstName.value}`
})
</script>
```

### watch - 상태 변화 감지

특정 상태의 변화를 감지하여 특정 로직을 실행할 때 사용하는 기능이며 값이 변경될 때마다 콜백 함수가 
실행되며, 이전 값과 현재 값을 함께 확인할 수 있어서 주로 `API` 호출, 로그 처리, 조건에 따른 추가 작업 등 
사이드 이펙트를 처리할 때 사용됩니다.

```vue
<template>
  <div>
    <input v-model="keyword" placeholder="검색어 입력" />
  </div>
</template>

<script setup>
import { ref, watch } from 'vue'

const keyword = ref('')

watch(keyword, (newValue, oldValue) => {
  console.log('이전 값:', oldValue)
  console.log('현재 값:', newValue)
})
</script>
```

### watchEffect - 상태 변화 감지

`watch`와 비슷하게 상태 변화를 감지하지만, 의존성을 자동으로 추적한다는 차이가 있어서 특정 값을 명시하지 않아도 콜백 내부에서 사용된 반응형 값들을 자동으로 추적하여 실행됩니다.

```vue
<script setup>
import { ref, watchEffect } from 'vue'

const count = ref(0)

watchEffect(() => {
  console.log('count:', count.value)
})
</script>
```

### ref vs reactive

`ref`와 `reactive`는 모두 상태를 선언하는 방법이지만, 사용하는 방식과 특징에는 차이가 있습니다.  
- `ref`는 <span style="background-color:#6DB33F">**단일 값**</span>을 다룰 때 사용  
- `reactive`는 <span style="background-color:#6DB33F">**객체 형태의 데이터**</span>를 다룰 때 사용  

```vue
<script setup>
import { ref, reactive } from 'vue'

const count = ref(0)
count.value++

const user = reactive({ name: '홍길동' })
user.name = '김철수'
</script>
```

### computed vs watch

`computed`와 `watch`는 모두 상태를 기반으로 동작하지만, 목적이 다릅니다.  
- `computed` → 값을 <span style="background-color:#6DB33F">**계산**</span>하기 위한 용도  
- `watch` → 값의 <span style="background-color:#6DB33F">**변화를 감지**</span>해서 로직 실행  

```vue
<script setup>
import { ref, computed, watch } from 'vue'

const count = ref(0)

const double = computed(() => count.value * 2)

watch(count, (newValue, oldValue) => {
  console.log('변경됨:', oldValue, '->', newValue)
})
</script>
```

### watch vs watchEffect

`watch`와 `watchEffect`는 모두 상태 변화를 감지한다는 공통점이 있지만, 사용 방식에는 차이가 있습니다.  

- `watch` → 특정 값을 <span style="background-color:#6DB33F">**명시적으로 지정**</span>하여 감지  
- `watchEffect` → 내부에서 사용된 값을 <span style="background-color:#6DB33F">**자동으로 추적**</span>  

```vue
<script setup>
import { ref, watch, watchEffect } from 'vue'

const count = ref(0)

watch(count, (newValue, oldValue) => {
  console.log('watch:', oldValue, '->', newValue)
})

watchEffect(() => {
  console.log('watchEffect:', count.value)
})
</script>
```

## 그래서 어떻게 쓰라고?

![](/assets/img/posts/minionsfunny.gif)

지금까지 `ref`, `reactive`, `computed`, `watch`, `watchEffect`를 각각 살펴봤다면,  실제 코드에서는 이 문법들을 따로 사용하는 것이 아니라 함께 조합해서 사용하게 됩니다.

```vue
<template>
  <div>
    <p>이름: {{ user.name }}</p>
    <p>나이: {{ user.age }}</p>
    <p>성인 여부: {{ isAdult }}</p>

    <button @click="increaseAge">나이 증가</button>
  </div>
</template>

<script setup>
import { reactive, computed, watch, watchEffect } from 'vue'

// 상태 선언
const user = reactive({
  name: '홍길동',
  age: 19
})

// 계산된 값
const isAdult = computed(() => user.age >= 20)

// watch - 특정 값 감지
watch(() => user.age, (newValue, oldValue) => {
  console.log(`나이 변경: ${oldValue} -> ${newValue}`)
})

// watchEffect - 자동 추적
watchEffect(() => {
  console.log(`현재 상태: ${user.name}, ${user.age}`)
})

// 상태 변경 함수
const increaseAge = () => {
  user.age++
}
</script>
```

이처럼 `reactive`로 상태를 선언하고, `computed`로 값을 가공하며, `watch`와 `watchEffect`를 통해 상태 변화를 감지하는 방식으로 각 문법을 함께 조합하여 하나의 로직을 구성하게 됩니다.  

즉, `Composition API`는 개별 문법을 따로 사용하는 것이 아니라 <span style="background-color:#6DB33F">**상태, 계산, 변화 감지를 하나의 흐름으로 묶어서 사용하는 것이 핵심**</span>이라고 볼 수 있습니다.

## 마무리

`Composition API`의 문법을 정리하면서 느낀 핵심을 다시 정리해보면 다음과 같습니다.
1. `ref`와 `reactive`를 통해 상태를 선언하고, 이를 기반으로 컴포넌트의 데이터를 구성할 수 있습니다.  
2. `computed`는 상태를 기반으로 새로운 값을 효율적으로 계산할 수 있도록 해주며, `watch`와 `watchEffect`는 상태 변화에 따라 필요한 로직을 실행할 수 있도록 해줍니다.  
3. 각 문법은 개별적으로 사용하는 것이 아니라 함께 조합되면서 하나의 컴포넌트 로직을 구성하게 됩니다.  
4. `Composition API`는 단순한 문법의 변화가 아니라, 상태와 로직을 함수 기반으로 구성하여 더 유연하고 
확장성 있는 구조를 만들 수 있도록 도와주는 방식입니다.  