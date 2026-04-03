---
title: "Storybook"
date: 2026-04-03 14:35:40 +0900
categories: [FrontEnd]
subcategory: Vue, Pinia
description: "Storybook에 대해서 적어봤습니다."
og_image: /assets/img/posts/StoryBook.png
---

![](/assets/img/posts/StoryBook.png)

## 주제 선정 이유

프론트엔드 개발을 진행하면서 `Vue`를 기반으로 컴포넌트를 만들고 화면을 구성하는 데에는 점점 익숙해졌지만, 
컴포넌트를 체계적으로 관리하기보다는 필요한 기능이 생길 때마다 그때그때 만들고 사용하는 방식으로 개발을 
해왔는데 이렇게 계속 진행을 하다보니 비슷한 `UI` 컴포넌트가 여러 개 생기기도 하고, 버튼이나 카드 같은 요소들도 
각각 다른 방식으로 구현되면서 일관성이 깨지는 경우가 많았고, 특정 컴포넌트의 상태를 확인하거나 수정하려면 항상 전체 페이지를 실행해야 하는 불편함도 느끼게 되었습니다.  

이러한 과정을 반복하다 보니 컴포넌트를 조금 더 체계적으로 관리하고, 독립적으로 개발할 수 있는 방법이 없을까라는 생각이 들었고, 그 과정에서 `Storybook`이라는 도구를 알게 되었습니다.

그래서 이번 글에서는 `Storybook`을 활용하여 <span style="background-color:#6DB33F">**컴포넌트를 독립된 환경에서 개발하고 테스트하는 방식과, 다양한 상태를 시각적으로 관리하는 방법, 그리고 실제 프로젝트에서 어떻게 활용할 수 있는지**</span>를 정리해보고자 합니다.

## Storybook이 뭘까?

![](/assets/img/posts/StoryBookIcon.png)

`Storybook`은 `UI 컴포넌트`를 독립된 환경에서 개발하고 테스트할 수 있도록 도와주는 도구로,  기존처럼 전체 페이지를 실행하지 않고도 개별 컴포넌트를 따로 띄워서 확인할 수 있는 환경을 제공하는데 일반적으로 프론트엔드 개발에서는 
특정 컴포넌트를 확인하기 위해 해당 컴포넌트가 포함된 화면 전체를 실행해야 하고, 특정 상태를 테스트하려면 
데이터를 직접 조작하거나 조건을 맞춰야 하는 번거로움이 존재합니다.  

하지만 `Storybook`을 사용하면 컴포넌트를 하나의 단위로 분리해서 관리할 수 있고, 각 상태를 `Story`라는 형태로 
정의하여 다양한 `UI` 상태를 한눈에 확인할 수 있는데 이러한 방식은 단순히 개발 편의성을 높이는 것을 넘어, 
컴포넌트를 재사용 가능한 단위로 관리하고 `UI`를 문서화하는 역할까지 함께 수행하게 됩니다.

### Story - 컴포넌트 상태 단위

가장 핵심이 되는 개념은 `Story`로, 컴포넌트의 특정 상태를 정의한 하나의 케이스라고 볼 수 있는데 예를 들어 버튼 
컴포넌트가 있다면 다음과 같이 여러 상태를 각각의 스토리로 정의할 수 있습니다.
- 기본 버튼
- 비활성화 버튼
- 로딩 상태 버튼
- hover 상태 버튼

이처럼 하나의 컴포넌트를 다양한 상태로 나누어 관리할 수 있기 때문에, `UI`의 모든 케이스를 빠르게 확인하고 테스트 할 수 있습니다.

### 사용법

`Storybook`을 사용하기 위해서는 먼저 `Vue` 프로젝트가 필요하고 아직 프로젝트가 없다면 아래 명령어를 통해 생성할 수 있습니다.

```bash
npm create vue@latest
```

![](/assets/img/posts/VueCreate.png)

프로젝트 생성이 완료되었다면, 해당 프로젝트 루트 경로에서 `Storybook`을 설치합니다.

```bash
npx storybook@latest init
```

이 명령어를 실행하면 현재 프로젝트 환경에 맞게 `Storybook`이 자동으로 설정되며, 관련 설정 파일과 기본 예제 코드가 함께 생성됩니다.

![](/assets/img/posts/StoryBookScreen.png)

#### .storybook 폴더

`.storybook` 폴더는 `Storybook`의 전체 동작을 설정하는 공간으로, 주로 다음과 같은 파일들로 구성됩니다.
- `main.js` : `Storybook`의 설정 파일  
- `preview.js` : 전역 설정 및 데코레이터 정의  

```javascript
// .storybook/main.js
export default {
  stories: ['../src/**/*.stories.@(js|ts)'],
  addons: ['@storybook/addon-essentials'],
}
```

이 파일에서는 어떤 스토리 파일을 읽어올지, 어떤 기능을 사용할지 등을 설정하게 됩니다.

#### preview.js

모든 스토리에 공통으로 적용되는 설정을 정의하는 파일입니다.

```javascript
// .storybook/preview.js
export const parameters = {
  actions: { argTypesRegex: '^on.*' },
}
```

예를 들어 전역 스타일을 적용하거나, 공통 레이아웃을 설정할 때 사용됩니다.

#### 파일 구조

각 컴포넌트마다 `*.stories.js` 파일을 만들어 해당 컴포넌트의 상태를 정의합니다.

```javascript
// Button.stories.js
import Button from './Button.vue'

export default {
  title: 'Example/Button',
  component: Button,
}
```

- `title` : 사이드바에 표시되는 이름
- `component` : 연결할 컴포넌트

#### 스토리 정의

각 상태는 아래와 같이 `export`를 통해 정의됩니다.

```javascript
export const Primary = {
  args: {
    label: 'Button',
    primary: true,
  },
}
```

이렇게 정의된 `Primary`는 하나의 `UI` 상태가 되며,각각을 클릭하면서 확인할 수 있습니다.

### 예시

이제 간단한 컴포넌트를 하나 만들고, `Storybook`에 적용하여 정상적으로 동작하는지 확인해보겠습니다.

```vue
<!-- Button.vue -->
<script setup>
defineProps({
  label: String,
  disabled: Boolean,
})
</script>

<template>
  <button :disabled="disabled">
    {{ label }}
  </button>
</template>
```

#### 스토리 파일 작성

같은 경로에 `Button.stories.js` 파일을 생성하고 아래와 같이 작성합니다.

```javascript
// Button.stories.js
import Button from './Button.vue'

export default {
  title: 'Components/Button',
  component: Button,
}

// 기본 상태
export const Default = {
  args: {
    label: 'Button',
    disabled: false,
  },
}

// 비활성화 상태
export const Disabled = {
  args: {
    label: 'Disabled',
    disabled: true,
  },
}
```
이렇게 정의하면 아래와 같이 각 상태가 사이드바에 표시되고, 클릭하면서 컴포넌트의 모습을 바로 확인할 수 있습니다.

![](/assets/img/posts/ButtonExample.png)

## 그래서 어떻게 쓰라고?

![](/assets/img/posts/noharamisae.gif)

실제로는 모든 컴포넌트를 한 번에 `Storybook`으로 관리하려고 하기보다는, 재사용이 많고 상태가 자주 바뀌는 
컴포넌트부터 하나씩 적용해보는 것이 중요한데 예를 들어 버튼, 입력창, 카드, 모달처럼 여러 화면에서 공통으로 
사용되는 `UI` 컴포넌트부터 스토리를 정의해두면 이후에는 해당 컴포넌트의 상태를 훨씬 쉽게 확인하고 관리할 수 
있으며 특히 기본 상태뿐만 아니라 `disabled`, `loading`, `error`처럼 자주 사용되는 다양한 상태를 미리 스토리로 
만들어두면, 전체 페이지를 실행하지 않아도 필요한 `UI`를 바로 확인할 수 있고 컴포넌트가 어떤 모습으로 동작하는지를 한눈에 파악할 수 있게 됩니다.  

```javascript
// Button.stories.js
import Button from './Button.vue'

export default {
  title: 'Components/Button',
  component: Button,
}

export const Default = {
  args: {
    label: 'Button',
    disabled: false,
  },
}

export const Disabled = {
  args: {
    label: 'Disabled',
    disabled: true,
  },
}

export const Loading = {
  args: {
    label: 'Loading',
    disabled: false,
    loading: true,
  },
}
```

```vue
<!-- Button.vue -->
<script setup>
defineProps({
  label: String,
  disabled: Boolean,
  loading: Boolean,
})
</script>

<template>
  <button :disabled="disabled">
    <span v-if="loading">Loading...</span>
    <span v-else>{{ label }}</span>
  </button>
</template>
```

이처럼 `Storybook`은 단순히 컴포넌트를 보여주는 도구가 아니라, 컴포넌트의 상태를 하나씩 분리해서 관리하고 필요한 케이스를 문서처럼 정리할 수 있도록 도와주는 방식으로 사용할 수 있습니다.

## 마무리

`Storybook`을 정리하면서 느낀 핵심을 다시 정리해보면 다음과 같습니다.

1. `Storybook`은 `UI` 컴포넌트를 독립된 환경에서 개발하고 테스트할 수 있도록 도와주는 도구입니다.  
2. `Story`를 통해 컴포넌트의 다양한 상태를 정의하고, 이를 시각적으로 확인할 수 있습니다.  
3. 전체 페이지를 실행하지 않고도 컴포넌트를 확인할 수 있어 개발 효율과 생산성을 크게 높일 수 있습니다.  
4. `Storybook`은 단순한 테스트 도구를 넘어, UI를 문서화하고 컴포넌트 기반 구조를 체계적으로 관리할 수 있도록 
도와주는 역할을 합니다.  