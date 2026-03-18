---
title: "JPA & MyBatis"
date: 2026-02-01 17:40:00 +0900
categories: [BackEnd, DataBase]
subcategory: JPA, MyBatis
---

![](/assets/img/posts/JPA&MyBatis.png)

## 주제 선정 이유

데이터 접근 계층을 설계하다 보면 하나의 영속성 프레임워크를 선택해 사용하는 것이 일반적이지만, 프로젝트가 
복잡해질수록 단일 방식만으로는 다양한 요구사항을 모두 만족시키기 어렵다는 상황을 마주하게 되며, 이러한 문제를 
어떻게 해결할 수 있을지에 대한 고민이 들게 됩니다.

현대 백엔드 환경에서는 `JPA`와 `MyBatis`를 상황에 맞게 함께 사용하는 방식이 점점 더 많이 활용되고 있지만, 단순히 `ORM`을 사용할지 `SQL Mapper`를 사용할지에 대한 선택을 넘어 객체지향적인 설계의 생산성과 복잡한 쿼리를 직접 
제어하는 유연성 사이에서 어떤 기준으로 기술을 선택해야 하는지에 대해서는 명확하게 이해하지 못한 채 사용하는 
경우가 많으며 또한, 개발 생산성과 성능 최적화 사이에서 균형을 맞추는 것이 중요하지만, 두 기술의 내부 동작 원리와 특징을 제대로 이해하지 못한다면 상황에 맞는 최적의 아키텍처를 설계하기 어렵다는 점을 느끼게 됩니다.

그래서 이번 글에서는 <span style="background-color:#6DB33F">**JPA와 MyBatis의 개념과 차이를 정리해보면서, 두 기술을 어떻게 조합하여 유연하고 확장 가능한 데이터 접근 계층을 설계할 수 있는지에 대한 방향**</span>에 대해 정리해보고자 합니다.

## JPA와 MyBatis를 왜 함께 쓸까?

결국 중요한 것은 특정 프레임워크에 올인하는 것이 아니라, 상황에 맞는 도구를 전략적으로 조합하는 일이라 생각했고 아래 표는 실제 서비스 환경을 가정해 `JPA`와 `MyBatis`를 어떻게 배치할지에 대한 제 기준입니다.

| 고려 항목 | 문제 상황 | 선택 기술 | 해결 전략 |
|:-:|:-:|:-:|:-:|
| 생산성 & 패러다임 | `RDB`와 `OOP` 간 불일치, 반복적인 `SQL` 작성 부담 | `JPA` (`ORM`) | `CRUD` 자동화로 생산성 향상, 비즈니스 로직 집중 |
| 성능 & 복잡한 연산 | 대량 데이터 처리, 다중 조인, `N+1` 문제 | `MyBatis` (`SQL Mapper`) | `SQL` 직접 제어 및 실행 계획 기반 튜닝 |
| 동적 쿼리 & 가독성 | 복잡한 쿼리의 가독성 저하, 부분 수정 불편 | `MyBatis` (`xml`) | 동적 `SQL`로 가독성 확보, 필요한 컬럼만 선택적 업데이트 |

## 그래서 어떻게 쓰라고?

![](/assets/img/posts/muhandojeon.gif)

서비스 레이어에서 비즈니스 로직의 성격에 따라 `JPA`와 `MyBatis`에 각기 다른 책임을 부여했습니다.

### JPA(CRD & Pagination)

엔티티의 생성(`C`), 조회(`R`), 삭제(`D`) 그리고 페이지네이션은 `JPA`가 전담하며 영속성 컨텍스트를 통해 데이터 
정합성을 보장받으면서, 인터페이스 정의만으로 쿼리 구현을 자동화하여 개발 생산성을 극대화했습니다.

#### PetRepository 인터페이스

`JpaRepository`를 상속받아 기본적인 `CRUD`를 확보하고 특히 목록 조회 시 `Pageable` 객체를 파라미터로 넘기도록 
설계하여, 별도의 `SQL` 작성 없이도 `DB` 레벨의 최적화된 페이징(`Limit/Offset`) 처리가 가능하게 했습니다.

```java
/**
 * 반려동물 엔티티에 대한 기본 CRD 및 페이징 처리를 담당하는 리포지토리입니다.
 */
public interface PetRepository extends JpaRepository<Pet, Long> {

    /**
     * 반려동물 등록번호의 중복 여부를 확인합니다.
     *
     * @param registrationNumber 중복 확인할 반려동물 등록번호
     * @return 중복된 경우 true, 그렇지 않은 경우 false
     */
    boolean existsByRegistrationNumber(String registrationNumber);

    /**
     * 종(Species)별 반려동물 목록을 페이지네이션하여 조회합니다.
     *
     * @param species  조회할 반려동물의 종
     * @param pageable 페이징 및 정렬 정보
     * @return 페이징 처리된 반려동물 목록
     */
    Page<Pet> findBySpecies(String species, Pageable pageable);
}
```

메서드 이름만으로 등록번호 중복 확인(`exists`)과 종별 목록 페이징 조회(`Page`) 기능을 구현했습니다.

#### PetService 인터페이스

`IP`를 준수하기 위해 인터페이스를 먼저 정의했으며 대량의 데이터를 다루는 목록 조회 메서드에는 `Spring Data`의 `Pageable`과 `Page` 타입을 적극적으로 활용했습니다.

```java
/**
 * 반려동물 관련 비즈니스 로직을 정의하는 서비스 인터페이스입니다.
 */
public interface PetService {

    /**
     * 새로운 반려동물을 시스템에 등록합니다.
     *
     * @param userId  반려동물을 등록하는 사용자의 고유 식별자
     * @param request 등록할 반려동물의 상세 정보 DTO
     * @return 등록 완료 응답 정보
     */
    PetRegisterResponse registerPet(UUID userId, PetRegisterRequest request);

    /**
     * 특정 반려동물의 상세 정보를 식별자를 통해 조회합니다.
     *
     * @param petId 조회할 반려동물의 고유 식별자
     * @return 반려동물 상세 상세 정보 DTO
     */
    PetDetailResponse getPetDetail(Long petId);

    /**
     * 특정 종에 해당하는 반려동물 목록을 페이징하여 조회합니다.
     *
     * @param species  조회할 종 명칭
     * @param pageable 페이징 정보
     * @return DTO로 변환된 페이징 응답 객체
     */
    Page<PetResponse> getPetsBySpecies(String species, Pageable pageable);

    /**
     * 시스템에서 반려동물 정보를 삭제합니다.
     *
     * @param petId 삭제할 반려동물의 고유 식별자
     */
    void removePet(Long petId);
}
```

#### PetServiceImpl 구현체

`JPA`를 호출하여 실제 비즈니스 로직을 수행하는 단계인데 생성 시에는 `exists` 쿼리로 유효성을 검증하고, 목록 
조회 시에는 `map` 함수를 사용하여 엔티티 객체를 응답용 `DTO`로 깔끔하게 변환하여 반환합니다.

```java
/**
 * JPA를 활용하여 반려동물의 생성, 조회, 삭제 및 페이징 로직을 처리하는 서비스 구현체입니다.
 */
@Service
@RequiredArgsConstructor
public class PetServiceImpl implements PetService {
    private final PetRepository petRepository;

    /**
     * 중복 등록 여부를 확인한 후 새로운 반려동물 엔티티를 저장합니다.
     */
    @Transactional
    @Override
    public PetRegisterResponse registerPet(UUID userId, PetRegisterRequest request) {
        if (petRepository.existsByRegistrationNumber(request.getRegistrationNumber())) {
            throw new PetException(REGISTRATION_NUMBER_DUPLICATED);
        }
        Pet savedPet = petRepository.save(request.toEntity());
        return PetRegisterResponse.toDto(savedPet.getPetId(), "등록 완료");
    }

    /**
     * Pageable을 통해 최적화된 페이징 쿼리를 수행하고 결과를 DTO로 변환합니다.
     */
    @Transactional(readOnly = true)
    @Override
    public Page<PetResponse> getPetsBySpecies(String species, Pageable pageable) {
        return petRepository.findBySpecies(species, pageable)
                .map(PetResponse::fromEntity);
    }

    /**
     * 식별자를 통해 존재 여부를 검증한 후 엔티티를 삭제합니다.
     */
    @Transactional
    @Override
    public void removePet(Long petId) {
        if (!petRepository.existsById(petId)) {
            throw new PetException(PET_NOT_FOUND);
        }
        petRepository.deleteById(petId);
    }
}
```

단순한 데이터 핸들링과 반복적인 페이징 쿼리는 `JPA`에 완전히 맡김으로써, 쿼리 작성 시간을 줄이고 객체 지향적인 
도메인 설계에 더 집중할 수 있는 환경을 만들었습니다.

### MyBatis(U, Search)

수정(`Update`)과 복잡한 조건이 포함되는 동적 조회의 경우 `MyBatis`를 사용하고 필드가 많은 엔티티의 일부만 
수정하는 부분 수정(`Partial Update`)이나, 검색 필터에 따라 `SQL` 구조가 실시간으로 변해야 하는 상황에서 
`MyBatis`의 `xml` 태그는 매우 강력한 가시성과 통제권을 제공하고 특히 자바 코드로 작성 시 파악이 어려운 복잡한 
동적 로직을 `xml`로 분리하여 관리할 수 있다는 점이 큰 장점입니다.

#### PetMapper 인터페이스

수정과 복잡한 검색을 처리하기 위한 매퍼 인터페이스입니다.

```java
/**
 * 복잡한 동적 쿼리 및 부분 수정을 처리하기 위한 MyBatis 매퍼 인터페이스입니다.
 */
@Mapper
public interface PetMapper {

    /**
     * 요청 데이터 중 null이 아닌 필드만을 선택적으로 업데이트합니다.
     *
     * @param request 수정할 필드 정보를 담은 DTO
     * @param petId   수정할 반려동물의 고유 식별자
     */
    void updatePetProfile(@Param("request") PetUpdateRequest request, @Param("petId") Long petId);

    /**
     * 다중 필터 조건(이름, 종 등)에 따라 동적으로 SQL을 생성하여 목록을 검색합니다.
     *
     * @param criteria 검색 조건을 담은 객체
     * @return 검색 결과 리스트
     */
    List<PetResponse> searchPets(@Param("criteria") PetSearchCriteria criteria);
}
```

#### PetService 인터페이스

인터페이스 기반 설계를 유지하며 수정 및 검색 기능을 정의합니다.

```java
/**
 * JPA와 MyBatis 기능을 통합하여 제공하는 반려동물 서비스 인터페이스입니다.
 */
public interface PetService {

    /**
     * 기존 반려동물 정보 중 변경된 사항만 반영하여 수정합니다.
     *
     * @param petId   수정할 반려동물의 고유 식별자
     * @param request 변경할 정보를 담은 DTO
     * @return 수정 결과 응답 정보
     */
    PetUpdateResponse updatePet(Long petId, PetUpdateRequest request);

    /**
     * 검색 필터 조건에 따라 반려동물 목록을 조회합니다.
     *
     * @param criteria 검색 필터 조건 객체
     * @return 조건에 부합하는 반려동물 응답 리스트
     */
    List<PetResponse> searchPets(PetSearchCriteria criteria);
}
```

#### PetServiceImpl 구현체

```java
/**
 * 수정 및 검색 로직에서 MyBatis 매퍼를 호출하여 하이브리드 전략을 완성하는 구현부입니다.
 */
@Service
@RequiredArgsConstructor
public class PetServiceImpl implements PetService {
    private final PetRepository petRepository;
    private final PetMapper petMapper;

    /**
     * JPA로 유효성을 검증한 후 MyBatis를 통해 동적 부분 수정을 수행합니다.
     */
    @Transactional
    @Override
    public PetUpdateResponse updatePet(Long petId, PetUpdateRequest request) {
        if (!petRepository.existsById(petId)) {
            throw new PetException(PET_NOT_FOUND);
        }

        petMapper.updatePetProfile(request, petId);

        return PetUpdateResponse.toDto(petId, "수정 완료");
    }

    /**
     * 복잡한 검색 조건을 MyBatis XML에 정의된 동적 SQL로 처리합니다.
     */
    @Transactional(readOnly = true)
    @Override
    public List<PetResponse> searchPets(PetSearchCriteria criteria) {
        return petMapper.searchPets(criteria);
    }
}
```

#### PetMapper xml

`MyBatis`의 `xml` 기반 동적 `SQL`이며 `<set>`과 `<if>` 태그를 조합하여 파라미터가 존재할 때만 해당 컬럼을 수정하도록 최적화했습니다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.dodo.backend.pet.mapper.PetMapper">

    <update id="updatePetProfile">
        UPDATE pet
        <set>
            <if test="request.petName != null">pet_name = #{request.petName},</if>
            <if test="request.breed != null">breed = #{request.breed},</if>
            <if test="request.age != null">age = #{request.age},</if>
            <if test="request.sex != null">sex = #{request.sex}</if>
        </set>
        WHERE pet_id = #{petId}
    </update>

    <select id="searchPets" resultType="com.dodo.backend.pet.dto.response.PetResponse">
        SELECT * FROM pet
        <where>
            <if test="criteria.name != null and criteria.name != ''">
                AND pet_name LIKE CONCAT('%', #{criteria.name}, '%')
            </if>
            <if test="criteria.species != null">
                AND species = #{criteria.species}
            </if>
        </where>
        ORDER BY created_at DESC
    </select>

</mapper>
```

이렇게 역할을 분담하면 엔티티의 생명주기 관리와 표준 페이징은 `JPA`로 자동화하고, 정교한 필드 수정과 복잡한 
동적 검색은 `MyBatis`로 확실하게 통제하는 효율적인 데이터 레이어를 구축할 수 있습니다.

### JPA와 MyBatis의 혼용 시 주의사항

`JPA`와 `MyBatis`를 한 프로젝트에서 함께 사용할 때 가장 중요한것은 <span style="background-color:#6DB33F">**하나의 트랜잭션 안에서 두 기술이 어떻게 데이터 정합성을 유지하느냐**</span>입니다.

#### JPA로 조회하고 MyBatis로 수정하는 흐름

데이터의 존재 여부나 권한 검증은 객체 지향적인 `JPA`를 통해 수행하고, 실제 동적이고 복잡한 수정은 `MyBatis`에 
맡깁니다.

```java
/**
 * JPA로 데이터의 존재를 검증하고, MyBatis로 부분 수정을 수행하는 전략입니다.
 */
@Transactional
@Override
public PetUpdateResponse updatePet(Long petId, PetUpdateRequest request) {

    if (!petRepository.existsById(petId)) {
        throw new PetException(PET_NOT_FOUND);
    }
    
    petMapper.updatePetProfile(request, petId);

    return PetUpdateResponse.toDto(petId, "수정 완료");
}
```

#### 영속성 컨텍스트 (1차 캐시) 동기화

`JPA`와 `MyBatis`를 혼용할 때 반드시 인지해야 할 점은 `MyBatis`는 영속성 컨텍스트를 모른다는 사실입니다.

```java
/**
 * JPA와 MyBatis 혼용 시 발생하는 영속성 컨텍스트 불일치 문제를 해결한 서비스 구현체입니다.
 */
@Service
@RequiredArgsConstructor
public class PetServiceImpl implements PetService {
    private final PetRepository petRepository;
    private final PetMapper petMapper;
    
    @PersistenceContext
    private final EntityManager em;

    /**
     * MyBatis 업데이트 후 JPA 영속성 컨텍스트를 동기화하여 최신 데이터를 보장합니다.
     * * @param petId   수정할 반려동물 ID
     * @param request 수정 요청 데이터
     * @return 최신 정보가 반영된 수정 결과 DTO
     */
    @Transactional
    @Override
    public PetUpdateResponse updatePet(Long petId, PetUpdateRequest request) {

        Pet pet = petRepository.findById(petId)
                .orElseThrow(() -> new PetException(PET_NOT_FOUND));

        petMapper.updatePetProfile(request, petId);

        em.flush(); // 혹시 남아있을지 모를 JPA의 변경 사항을 DB에 반영
        em.clear(); // 1차 캐시를 완전히 비워 다음 조회 시 DB에서 새 데이터를 가져오도록 강제
        
        Pet updatedPet = petRepository.findById(petId)
                .orElseThrow(() -> new PetException(PET_NOT_FOUND));

        return PetUpdateResponse.toDto(updatedPet.getPetId(), "수정 및 동기화 완료");
    }
}
```

- 문제 상황
  - 만약 `JPA`로 엔티티를 조회하여 1차 캐시에 올린 상태에서 `MyBatis`로 같은 로직 내에서 업데이트를 수행하면, `DB`의 데이터는 바뀌지만 메모리상의 `JPA` 엔티티는 수정 전 데이터를 그대로 들고 있게 됩니다.
- 해결 전략
  - `MyBatis` 업데이트가 발생한 직후에 동일한 엔티티를 `JPA`로 다시 조회해야 한다면, 반드시 `em.clear()` 또는 `em.flush()`를 통해 영속성 컨텍스트를 비우거나 동기화해야 합니다.
  - 가급적 수정 로직과 조회 로직을 분리하여, 수정이 일어난 트랜잭션 내에서 수정된 데이터를 다시 `JPA` 엔티티로 다루지 않도록 설계하는 것이 가장 안전합니다.

#### 동일한 트랜잭션과 커넥션 공유

`Spring` 환경에서 `DataSourceTransactionManager`를 사용하면 `JPA`와 `MyBatis`는 동일한 데이터베이스 커넥션과 
트랜잭션을 공유합니다.
- `MyBatis` 쿼리 실행 도중 예외가 발생하면 `JPA`로 저장했던 내역도 함께 `Rollback`되며, 반대의 경우도 마찬가지며 개발자는 기술의 혼용과 상관없이 `@Transactional` 하나로 데이터의 원자성을 보장받을 수 있습니다.

## 마무리
`JPA`와 `MyBatis`라는 두 기술을 함께 사용해보며 제가 느낀 핵심을 다시 정리해보면 다음과 같습니다.
1. `SQL` 중심의 반복적인 작업을 `JPA`에 맡김으로써 자바의 본질인 객체지향 설계에 집중할 수 있었고, 이는 곧 개발 
생산성의 비약적인 향상으로 이어졌습니다.
2. 1차 캐시, 쓰기 지연, 변경 감지라는 `JPA`의 메커니즘을 이해함으로써 단순한 쿼리 실행기 그 이상의 성능 최적화와 데이터 정합성을 챙길 수 있었고 동시에 `MyBatis`를 통해 `SQL` 통제권을 확보하여 성능의 균형을 맞췄습니다.
3. 테이블 간의 복잡한 관계를 객체 참조로 매핑하고 페치 조인 (`Fetch Join`)을 통해 `N+1` 문제를 해결하되, `JPA`가 
담아내기 어려운 복잡한 통계나 동적 수정 로직은 `MyBatis`로 보완하며 기술적 한계를 극복했습니다.
4. `JPA`와 `MyBatis`는 모두 강력한 도구이지만, 내부 동작 원리를 모른 채 사용하면 언제든 성능 폭탄을 맞을 수 있으니 편리함에 매몰되지 않고 상황에 맞는 최적의 도구를 선택하여 배치하는 기본기를 다지는 것이 무엇보다 중요하다는 것을 깨달았습니다.