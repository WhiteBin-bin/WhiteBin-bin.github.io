---
title: "SwaggerUI & ScalarAPI"
date: 2026-02-06 19:00:00 +0900
categories: [BackEnd]
subcategory: SwaggerUI, ScalarAPI
description: "SwaggerUI & ScalarAPI에 대해서 적어봤습니다."
og_image: /assets/img/posts/JPA&MyBatis.png
---

![](/assets/img/posts/ApiReference.png)

## 주제 선정 이유

## 주제 선정 이유

`API`를 개발하다 보면 자연스럽게 `Swagger UI`를 활용해 문서를 생성하고 관리하게 되지만, 프로젝트 규모가 커지고 
협업이 늘어날수록 단순한 명세 제공을 넘어 문서를 얼마나 효율적으로 활용할 수 있는가에 대한 고민을 마주하는데 
기존 `Swagger UI`는 표준화된 `API` 명세를 제공하는 데에는 충분한 역할을 수행하지만, 다양한 언어의 코드 스니펫을 
비교하거나 테스트를 동시에 진행하는 과정에서 인터페이스의 불편함이나 사용자 경험 측면에서 아쉬움을 느끼는 
경우가 많고, 이러한 부분을 어떻게 개선할 수 있을지에 대한 고민이 들게 됩니다.

최근에는 `Scalar`와 같이 개발자 경험 (`DX`)을 개선하기 위한 `API` 문서화 도구들이 등장하고 있지만, 단순히 새로운 `UI`를 사용하는 것을 넘어 실제로 어떤 차이가 있는지, 그리고 기존 방식과 비교했을 때 어떤 상황에서 더 효과적인지에 대해서는 명확하게 이해하지 못한 채 사용하는 경우도 많습니다.

그래서 이번 글에서는 <span style="background-color:#6DB33F">**Open API 스펙을 기반으로 Swagger UI와 Scalar의 차이를 비교해보면서, API 문서를 보다 <br>효율적으로 설계하고 활용할 수 있는 방향**</span>에 대해 정리해보고자 합니다.

## Scalar API가 뭘까?

![](/assets/img/posts/ScalarApi.gif)

`OpenAPI` 명세를 기반으로 하여, 정적인 `API` 문서를 세련되고 인터랙티브한 레퍼런스로 변환해주는 현대적인 문서화 도구인데 단순히 `API`의 엔드포인트를 나열하는 것에 그치지 않고, 문서를 읽는 개발자가 실제 개발 환경에서 느끼는 
불편함을 해결하는 데 초점을 맞춘 오픈소스 프로젝트라고 합니다.

### Scalar API vs Swagger UI

![](/assets/img/posts/ScalarSwagger.png)

두 도구 모두 `OpenAPI` 명세를 기반으로 문서를 생성한다는 본질은 같지만, 정보를 전달하는 방식과 사용자 경험에서 
확연한 차이를 보입니다.

| 비교 항목   | Swagger UI | Scalar API  |
| :-: | :-: | :-: |
| 레이아웃 구조 | 단일 컬럼 | 3컬럼 대시보드 |
| 시각적 가시성 | 정보가 나열되어 있어 탐색이 번거로움 | 계층 구조가 명확하여 `API` 파악이 빠름 |
| 코드 스니펫 | 기본적인 형태만 제공 | `Axios`, `Fetch `등 다양한 라이브러리 지원 |
| 테스트 환경  | `Try it out` 클릭 시 내부 확장 방식 | 우측 패널에 고정된 전용 플레이그라운드       |
| 디자인 | 관습적이고 다소 투박한 디자인 | 세련되고 현대적인 인터페이스 |
| 확장성 | 커스터마이징이 다소 복잡함 | 테마 시스템 및 커스텀 `CSS` 적용 용이 |

## 그래서 어떻게 쓰라고?

![](/assets/img/posts/whattheheck.gif)

단순히 도구를 교체하는 것을 넘어서, <span style="background-color:#6DB33F">**명세 관리는 백엔드가, 시각화(UI)는 Scalar가**</span> 담당하도록 역할을 깔끔하게 분리 하는것이 핵심입니다.

### SpringDoc OpenAPI 설정

백엔드에서는 `API` 구조를 정의하고 `JSON` 형태의 명세서(`v3/api-docs`)를 생성하는 역할을 수행하는데 이를 위해서 
의존성에 `springdoc-openapi` 라이브러리를 활용합니다.

```gradle
dependencies {
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0'
}
```

### OpenApiConfig 설정

보안 설정과 `API` 기본 정보를 포함한 설정을 코드로 작성하는데 이 설정이 완료되면 백엔드는 `Scalar`가 읽어갈 데이터소스를 제공할 준비가 끝납니다.

```java
/**
 * Scalar API Reference 및 OpenAPI 명세를 담당하는 구성 클래스입니다.
 * <p>
 * 이 클래스는 API 문서의 메타데이터 설정 및 JWT Bearer 토큰 인증 체계를 정의합니다.
 */
@Configuration
public class OpenApiConfig {

    /**
     * OpenAPI 설정을 생성하고 Bean으로 등록합니다.
     * <p>
     * 주요 설정 내용:
     * <ul>
     * <li>API 기본 정보 (제목, 설명, 버전)</li>
     * <li>Security Scheme 정의 (JWT Bearer 인증 방식)</li>
     * <li>전역 Security Requirement 적용 (모든 API에 자물쇠 아이콘 표시)</li>
     * </ul>
     *
     * @return 인증 설정이 포함된 OpenAPI 객체
     */
    @Bean
    public OpenAPI openAPI() {

        String securityJwtName = "JWT_Auth";

        SecurityRequirement securityRequirement = new SecurityRequirement().addList(securityJwtName);

        Components components = new Components().addSecuritySchemes(securityJwtName,
                new SecurityScheme()
                        .name(securityJwtName)
                        .type(SecurityScheme.Type.HTTP)
                        .scheme("bearer")
                        .bearerFormat("JWT"));

        Info info = new Info()
                .title("DoDo API Docs")
                .description("DoDo 프로젝트 백엔드 서비스 API 명세서입니다.")
                .version("v0.0.1");

        return new OpenAPI()
                .info(info)
                .addSecurityItem(securityRequirement)
                .components(components);
    }
}
```

### Scalar HTML 구성

이제 백엔드가 생성한 `v3/api-docs` `JSON` 데이터를 `Scalar UI`에 입히면 되는데 별도의 무거운 라이브러리 설치 없이 `HTML` 파일 하나만으로 구성할 수 있다는 점이 강력한 장점입니다.

`src/main/resources/static/scalar/index.html` 경로에 아래 파일을 생성하는데 여기서 `Scalar`는 `CDN`을 통해서 
필요한 리소스를 실시간으로 불러옵니다.

```html
<!DOCTYPE html>
<html>
<head>
    <title>DoDo API Docs</title>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
</head>
<body>
<script
        id="api-reference"
        data-url="/v3/api-docs">
</script>
<script src="https://cdn.jsdelivr.net/npm/@scalar/api-reference"></script>
</body>
</html>
```

그러면 실행시키고 나서 `http://localhost:8080/scalar/index.html`이 링크로 들어가게 되면 다음과 같은 화면이 뜨게되면서 설정은 끝이나게 됩니다.

![](/assets/img/posts/ScalarApiExample.png)

### 테마 및 레이아웃 설정

기본으로 제공되는 테마 외에도 `data-configuration` 속성을 사용하면 다크모드 고정이나 레이아웃 변경이 가능합니다.

```html
<script
  id="api-reference"
  data-url="/v3/api-docs"
  data-configuration='{
    "theme": "purple",
    "layout": "modern",
    "searchHotKey": "k",
    "darkMode": true
  }'>
</script>
```

- `theme`
  - `default`, `alt`, `purple`, `solarized` 등 다양한 분위기를 연출할 수 있습니다.
- `layout`
  - `modern`과`classic`중 선택 가능합니다.
- `darkMode`
  - 사용자의 시스템 설정과 상관없이 다크 모드를 강제하여 일관된 `DX`를 제공할 수 있습니다.

## 마무리

`Swagger UI`에서 `Scalar API`로 문서화 도구를 전환하며 제가 느낀 핵심을 다시 정리해보면 다음과 같습니다.

1. `Scalar`가 제공하는 다양한 언어별 코드 스니펫과 직관적인 테스트 환경은, 문서를 읽는 시간을 줄이고 바로 
구현으로 이어질 수 있는 흐름을 만들어주었습니다.
2. 세개의 컬럼 기반의 레이아웃을 통해 탐색-상세-테스트가 하나의 화면 안에서 자연스럽게 이어지면서, `API`를 
이해하는데 필요한 맥락이 끊기지 않는 경험을 제공했습니다.
3. `OpenAPI` 명세 생성은 `SpringDoc`을 통해 백엔드 코드로 일관되게 관리하고, 시각화와 인터랙션은 `Scalar`에 
위임함으로써 역할과 책임이 명확한 문서화 구조를 구성할 수 있었습니다.
4. 누가 이 문서를 사용하는지를 기준으로 기술을 선택하는 시각이 백엔드 개발자에게 얼마나 중요한지 다시금 깨닫게 되었습니다.