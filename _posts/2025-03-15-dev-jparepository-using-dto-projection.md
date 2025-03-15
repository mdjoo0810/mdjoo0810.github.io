---
title: JPA Repository 도 DTO Projection 이 가능하다.
author: 코드돈
date: 2025-03-15 10:00:00 +0900
categories: [개발, JPA]
tags: [spring, jpa]
---

## 궁금증

QueryDSL을 쓰다보니 좋았던 기능 중 하나가, 단순 조회 시 DTO Projection을 사용해서 필요한 컬럼만 조회할 수 있다는 점이었는데요. 이렇게 하면 성능상 유리하고, 영속성 컨텍스트도 필요 없어서 가볍게 조회할 때 매우 좋았습니다.

그런데 생각해보니 JPA Repository에서도 이런 DTO Projection 기능을 제공하지 않을까? 하는 궁금증이 들어서 찾아보게 되었습니다.

## 가능하다

일반적으로 JPA에서 엔티티를 조회할 때는 필요하지 않은 컬럼들까지 모두 조회하게 되어 성능상 손해가 있습니다. 이런 문제를 해결하려면 필요한 필드만 선별적으로 조회하는 DTO Projection을 사용하는데요.

보통 DTO Projection을 구현할 때는 JPQL을 쓰거나 QueryDSL을 많이 사용하지만, 사실 Spring Data JPA의 Repository 메서드에서도 DTO Projection을 손쉽게 사용할 수 있습니다.

이번 포스팅에서는 Spring Data JPA에서 지원하는 다양한 DTO Projection 방법과 성능 차이를 함께 살펴보겠습니다.

## 사용법

Spring Data JPA에서 DTO Projection을 구현하는 방법은 크게 세 가지가 있습니다.

1. Interface 기반 Projection
2. Class 기반 Projection
3. Dynamic Projection

처음에는 특별히 생각하지 않고 편하게 Interface 기반 Projection을 사용해봤는데, 오히려 전체 Entity를 조회하는 것보다 속도가 느리게 나와서 매우 당황했습니다. 그래서 이 참에 각 방식이 어떤 차이가 있는지 깊게 확인하게 되었습니다.

### Interface 기반 Projection이 왜 느릴까?

Interface 기반의 Projection이 성능 저하를 일으키는 가장 큰 원인은 내부적으로 Proxy 객체를 생성하고 Reflection을 사용하기 때문입니다.

Spring Data JPA에서 Interface Projection을 사용하면, 아래와 같은 과정이 이루어집니다.

- 내부적으로 JPA는 Interface를 구현한 Proxy 객체를 생성
- 조회된 각 필드의 값을 Proxy의 메서드가 호출될 때마다 Reflection을 통해 반환

이 과정에서 Reflection을 매번 호출하기 때문에 오버헤드가 크고, 다량의 데이터를 처리할 때 급격히 성능이 떨어지는 것입니다.

### Class 기반 Projection은 왜 빠를까?

Class 기반 Projection은 다음과 같이 정의할 수 있는데요, JPQL의 new 키워드를 사용하지 않고도 Kotlin의 data class로 매우 쉽게 사용할 수 있습니다.

```kotlin
// 기존의 Interface Projection 방식
interface ContentListItemInterface {
    val slug: String
    val title: String
    val updatedAt: LocalDateTime
}

// Class 기반의 Projection 방식
data class ContentListItemDto(
    val slug: String,
    var title: String,
    var updatedAt: LocalDateTime,
)

internal interface ContentEntityJpaRepository : JpaRepository<ContentEntity, Long> {
    fun findAllByUserId(userId: Long): List<ContentListItemDto>
}
```

Spring Data JPA가 내부적으로 DTO 클래스의 생성자를 바로 호출하여 인스턴스를 생성합니다. 인터페이스 방식과 다르게 내부적으로 생성자를 직접 호출하기 때문에 리플렉션과 Proxy 객체 생성의 오버헤드가 없어서 성능이 훨씬 좋습니다.

### 실제 성능 비교 예시

실제로 성능을 측정해 보면 다음과 같은 결과를 얻을 수 있습니다.

- Interface Projection: 1006 ms
  - ![alt text](/assets/img/posts/2025-03-15-dev-jparepository-using-dto-projection/image.png)
- Entity 전체 조회: 348 ms
  - ![alt text](/assets/img/posts/2025-03-15-dev-jparepository-using-dto-projection/image1.png)
- Class Projection: 144 ms
  - ![alt text](/assets/img/posts/2025-03-15-dev-jparepository-using-dto-projection/image2.png)

## Dynamic Projection 방식

추가로, 상황에 따라 원하는 필드만 동적으로 선택하고 싶다면 Dynamic Projection 을 사용할 수 있습니다.

```kotlin
internal interface ContentEntityJpaRepository : JpaRepository<ContentEntity, Long> {
    fun <T> findBySlug(slug: String, type: Class<T>): T
}

// 사용할 때
val content = contentEntityJpaRepository.findBySlug("slug", ContentListItemDto::class.java)
```

다만 Dynamic Projection 또한 인터페이스 기반으로 동작하기 때문에, 성능적으로 Class Projection보다는 다소 떨어질 수 있습니다. 그래도 상황에 따라 필드가 유동적이라면 활용하기 좋은 옵션입니다.

## 마무리 

위 결과처럼 Class Projection을 사용하면, 전체 엔티티 조회보다 성능이 더 빠른 것을 확인할 수 있었습니다. 필요한 컬럼만 조회해오기 때문에 DB 성능뿐만 아니라, 메모리 사용량과 네트워크 부하까지 줄일 수 있습니다.

실제로 필드 수가 많아질수록 성능 개선 효과는 더 커지므로, 복잡한 Entity에서는 적극적으로 사용을 고려해볼만 합니다.
