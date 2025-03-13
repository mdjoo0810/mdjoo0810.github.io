---
title: QueryDSL ZeroOffsetPagingItemReader 으로 성능 개선
author: 코드돈
date: 2025-02-15 10:00:00 +0900
categories: [개발, SpringBatch]
tags: [springbatch, querydsl, itemreader]
---
## 배경

회사에서 암호화 솔루션이 변경되면서 기존 컬럼 암호화가 적용된 수십 개 테이블의 데이터를 새로운 암호화 방식으로 마이그레이션해야 했습니다.

이 작업을 수동으로 처리하는 것은 비효율적이었기 때문에 Spring Batch를 활용하여 마이그레이션 배치 작업을 구성했습니다. 처음에는 큰 문제 없이 실행되는 듯했지만, 수만 건 이하의 데이터를 가진 테이블은 정상적으로 완료된 반면, 수백만 건 이상의 데이터를 가진 테이블에서는 배치 실행 시간이 예상보다 훨씬 오래 걸리는 문제가 발생했습니다.

단순히 데이터 양이 많기 때문이라고 생각했지만, 실행 시간이 데이터 수에 따라 선형적으로 증가하는 것이 아니라 시간이 지날수록 점점 더 느려지는 현상이 나타났습니다.

원인을 파악하기 위해 **Spring Batch의 메타데이터 테이블** 을 확인했으며, `BATCH_STEP_EXECUTION` 테이블의 `read_count` 값을 모니터링했습니다.
초기 실행 시점에는 1초에 수천 개의 레코드가 처리되었지만, 1시간 정도가 지나자 초당 1000개도 읽지 못하는 현상을 확인할 수 있었습니다. 배치가 진행될수록 성능이 점점 더 저하되는 문제를 해결해야 했습니다.

## JpaPagingItemReader 문제

팀 내에서는 데이터 접근을 할 때 JPA를 적극적으로 사용하고 있었고, 신규 암호화 컬럼 적용 시에도 기존 데이터는 특정 범위만 변환하면 되는 상황이었습니다. 그래서 큰 고민 없이 **Spring Batch에서 기본적으로 제공하는 JpaPagingItemReader** 를 활용하여 데이터를 읽도록 구성했습니다.

하지만 실행 후 데이터를 조회하는 속도가 점점 느려지는 것을 확인하게 되었고, JpaPagingItemReader의 동작 방식을 분석하게 되었습니다.

### JpaPagingItemReader의 내부 동작 방식

Spring Batch의 JpaPagingItemReader는 내부적으로 아래와 같은 방식으로 데이터를 조회합니다.

```java
// JpaPagingItemReader 내부 코드 (Spring Batch 5 기준)
// https://github.com/spring-projects/spring-batch/blob/main/spring-batch-infrastructure/src/main/java/org/springframework/batch/item/database/JpaPagingItemReader.java#L211
Query query = createQuery()
		.setFirstResult(getPage() * getPageSize())  // Offset 설정
```

이 코드를 보면 JPQL 기반으로 setFirstResult(offset)와 setMaxResults(limit)를 사용하여 페이징을 구현하고 있음을 확인할 수 있습니다.

- setFirstResult(n): 조회를 시작할 위치(= Offset)를 지정합니다.
- setMaxResults(m): 한 번의 조회에서 가져올 최대 결과 개수(= Limit)를 설정합니다.

즉, JpaPagingItemReader는 Offset 기반 페이징을 사용하는데, 이 방식은 데이터가 많아질수록 성능 저하가 심해지는 단점이 있습니다.

### Offset 기반 페이징이 성능을 저하시킨 이유
Offset 기반 페이징은 다음과 같은 문제를 가지고 있습니다.
1. Offset이 커질수록 조회 시간이 증가
    - 데이터베이스는 OFFSET N을 사용할 경우, N개의 행을 먼저 스캔한 후 버리고, 그다음 데이터를 반환합니다.
    - 예를 들어, OFFSET 1,000,000이라면 100만 개의 데이터를 먼저 스캔한 후, 원하는 데이터만 반환하게 되므로 불필요한 연산 비용이 증가합니다.
    - 실행 초반에는 빠르게 동작하지만, 시간이 지날수록 점점 느려지는 현상이 발생합니다.
2. 인덱스를 사용하지 못하는 경우 발생
    - 일부 DBMS에서는 Offset을 사용할 때 인덱스를 비효율적으로 사용하게 되고, 쿼리가 풀스캔으로 변할 가능성도 있습니다.
    - 특히 정렬 기준이 없는 경우, 매번 전체 테이블을 조회한 후 결과를 잘라서 반환하기 때문에 배치 실행 시간이 비정상적으로 길어질 수 있습니다.
3. 대량 데이터 배치에는 적합하지 않음
    - JpaPagingItemReader는 기본적으로 페이징 처리된 데이터를 한 번에 로드한 후, 이를 배치 단위로 처리하는 방식이기 때문에 처리해야 할 데이터가 많을수록 비효율적입니다.
    - 테이블의 데이터가 100만 건을 넘어가면, 후반부 데이터를 조회하는 속도가 초반보다 훨씬 느려지는 현상이 발생합니다.

위와 같은 문제를 해결하기 위해 Offset을 사용하지 않는 방식으로 Reader를 변경하기로 결정했습니다.

## CursorItemReader

Offset 기반 페이징의 성능 문제를 해결하기 위해 Cursor 기반 페이징(Cursor Based Pagination) 도 고려했습니다. Spring Batch에서는 HibernateCursorItemReader 또는 JdbcCursorItemReader 를 활용하면 Cursor 기반으로 데이터를 읽을 수 있습니다.

### CursorItemReader의 동작 방식

Cursor 기반 페이징은 데이터베이스와 하나의 Connection을 유지하면서, 데이터를 스트림으로 가져오는 방식입니다.

1. 배치 실행 시 하나의 DB Connection을 맺고 유지
2. 특정 위치에서 Stop Point(커서 위치)를 기록
3. 이후 다시 데이터를 읽을 때 해당 위치부터 이어서 조회

이 방식은 Offset 기반보다 성능이 향상될 가능성이 있지만, 운영 환경에서는 몇 가지 단점이 있었습니다.

### CursorItemReader를 선택하지 않은 이유

1. 장시간 실행 시 Connection Timeout 문제 발생
    - Cursor 방식은 실행되는 동안 하나의 Connection을 계속 유지하기 때문에, 배치 실행 시간이 길어질 경우 DB 커넥션 타임아웃(Connection Timeout) 이 발생할 가능성이 높습니다.
    - 실제로 JpaPagingItemReader를 사용했을 때 500만 건의 데이터를 처리하는 데 70시간 이상 걸렸기 때문에, Cursor 방식으로 개선하더라도 최소 1시간 이상 실행될 것으로 예상되었습니다.
    - 이 경우 DB Connection을 1시간 이상 유지해야 하는데, 운영 환경에서는 너무 긴 시간 동안 Connection을 유지하는 것이 안정적인 선택이 아니었습니다.

2. 멀티 잡 실행 시 커넥션 과부하 발생 가능
    - 마이그레이션 배치는 여러 개의 잡(Job)이 동시에 실행될 가능성이 있었습니다.
    - Cursor 방식은 각 Job이 개별적인 Connection을 유지해야 하기 때문에, 동시 실행 시 DB에 과부하를 줄 가능성이 있었습니다.
    - 또한, API 서비스도 같은 데이터베이스를 사용하고 있었기 때문에, Batch 실행 중 API 서비스의 DB Connection Pool이 부족해지는 문제가 발생할 수도 있는 상황이었습니다.

결과적으로, Cursor 방식이 아닌 Offset 기반을 유지하면서 성능을 개선할 수 있는 방법을 찾기로 했습니다.

- Offset을 0으로 고정하는 방식으로 불필요한 데이터 스캔을 최소화하는 ZeroOffset Pagination을 적용하기로 결정했습니다.
- 이 방식을 적용하기 위해 커스텀 ItemReader를 직접 구현하여 문제를 해결하였습니다.

## QueryDslZeroOffsetItemReader

팀 내에서는 데이터 조회 시 복잡한 쿼리를 직접 JPQL 문자열로 관리하는 것보다 Type-Safe한 QueryDSL을 활용하는 것이 일반적이었습니다.

- JPQL을 문자열로 관리하면 런타임 에러가 발생할 가능성이 높고, 쿼리 작성 및 유지보수가 어렵습니다.
- 반면, QueryDSL은 컴파일 타임에 문법 검사를 수행하고, 가독성이 높아 유지보수가 용이합니다.

따라서 기존 JpaPagingItemReader의 문제를 해결하기 위해, QueryDSL을 활용한 ZeroOffsetPagingItemReader를 직접 구현하기로 결정했습니다.

### ZeroOffset Paging 방식의 차이점

기존 JpaPagingItemReader 와 ZeroOffset 방식의 가장 큰 차이점은 Offset을 0으로 고정하고, 대신 WHERE 조건을 활용하여 페이징을 수행하는 것입니다.

#### 기존 JpaPagingItemReader (Offset 기반 페이징)
```java
Query query = createQuery()
    .setFirstResult(getPage() * getPageSize())  // OFFSET 적용
    .setMaxResults(getPageSize()); // LIMIT 적용
```

#### ZeroOffset 방식 (WHERE 조건 기반 페이징)
```java
query.where(entity.id.gt(lastProcessedId)) // WHERE 조건 사용
    .limit(pageSize);
```

### JpaPagingItemReader 소스 분석 및 수정

ZeroOffset 방식의 Reader를 구현하기 위해, 기존 JpaPagingItemReader 의 소스를 가져와 베이스로 활용했습니다.

기본적으로 Spring Batch의 모든 Paging 기반 ItemReader는 AbstractPagingItemReader를 상속하여 구현되므로, 먼저 AbstractPagingItemReader의 동작 방식을 이해해야 했습니다.

### AbstractPagingItemReader의 동작 방식

Spring Batch에서 페이징 기반의 Reader는 AbstractPagingItemReader 를 상속하여 구현됩니다.
기본적인 동작 방식은 다음과 같습니다.

1. doRead()
    - 한 개의 아이템을 읽어옴 (current 인덱스를 기반으로 result 리스트에서 가져옴)
    - 만약 더 이상 읽을 값이 없다면 doReadPage()를 호출하여 새로운 페이지를 로드
2. doReadPage()
    - 새로운 데이터를 조회하고, result 리스트에 원하는 크기만큼 채움
    - 페이징 처리 로직이 이곳에서 수행됨
    - 여기에서 QueryDSL을 활용한 ZeroOffset 로직을 구현하면 됨
3. 데이터가 더 이상 존재하지 않는 경우
    - doReadPage()에서도 가져올 데이터가 없다면 null을 반환하여 더 이상 읽을 아이템이 없음을 알림

즉, 커스텀 Reader를 구현할 때 핵심은 doReadPage()의 페이징 로직을 수정하는 것이었습니다.

### 어떻게 동작하는가?

실제로 구현한 코드는 아래와 같습니다.

```kotlin
open class QuerydslZeroOffsetItemReader<T>(
		// 어떤 컬럼을 통해 정렬할 것인가, 오름차순 or 내림차순인가? 에 대한 옵션
    private val option: QuerydslZeroOffsetOption<T>,
    private val entityManager: EntityManager,
    // querydsl 을 통한 쿼리
    private val queryCreator: (JPAQueryFactory) -> JPAQuery<T>
) : AbstractPagingItemReader<T>() {

    override fun doReadPage() {
        clear()

        val query = createQuery().limit(pageSize.toLong())

        initResults()
        fetchQuery(query)
        resetStartIdNotLastPage()
    }

    private fun clear() {
        entityManager.clear()
    }

    private fun createQuery(): JPAQuery<T> {
        val query = queryCreator(JPAQueryFactory(entityManager))
        return option.createQuery(query)
    }

    private fun initResults() {
				if (CollectionUtils.isEmpty(results)) {
					results = CopyOnWriteArrayList();
				} else {
					results.clear();
				}
    }

    private fun fetchQuery(query: JPQLQuery<T>) {
        results.addAll(query.fetch())
    }

    private fun resetStartIdNotLastPage() {
        if (!isNotEmptyResults()) return
        option.resetStartId(getLastItem())
    }

    private fun getLastItem(): T {
        return results.get(results.size - 1)
    }

    private fun isNotEmptyResults(): Boolean {
        return !CollectionUtils.isEmpty(results) && results.get(0) != null
    }

    override fun doJumpToPage(itemIndex: Int) {}

    override fun doClose() {
        entityManager.close()
        super.doClose()
    }
}

/**
 * ID type 이 Long 인 경우만 커버함
 */
class QuerydslZeroOffsetOption<T>(
    private var startId: Long,
    private val field: NumberPath<Long>,
    private val fieldName: String,
    private val sort: Sort.Direction = Sort.Direction.DESC
) {

    fun resetStartId(item: T) {
        startId = getFieldValue(item) as Long

        if (startId <= 0) return

        if (sort == Sort.Direction.DESC) {
            startId--
        } else {
            startId++
        }
    }

    fun createQuery(query: JPAQuery<T>): JPAQuery<T> {
        return if (sort == Sort.Direction.DESC) {
            query.where(field.loe(startId))
                .orderBy(field.desc())
        } else {
            query.where(field.goe(startId))
                .orderBy(field.asc())
        }
    }

    private fun getFieldValue(item: T): Any? {
        return try {
            val field: Field = item!!::class.java.getDeclaredField(fieldName)
            field.isAccessible = true
            field.get(item)
        } catch (e: NoSuchFieldException) {
            throw IllegalArgumentException("Not Found or Not Access Field")
        } catch (e: IllegalAccessException) {
            throw IllegalArgumentException("Not Found or Not Access Field")
        }
    }
}
```

기존 JpaPagingItemReader와 비교했을 때, 가장 큰 차이점은 페이징 방식에서 Offset을 제거하고 WHERE 조건을 활용하여 데이터를 가져온다는 점입니다.

doReadPage()는 Spring Batch의 AbstractPagingItemReader를 상속받아 페이징 처리를 수행하는 메서드입니다.
ZeroOffset 방식으로 개선된 doReadPage()의 주요 동작을 하나씩 살펴보겠습니다.

#### doReadPage() 코드 분석
```kotlin
override fun doReadPage() {
    // 새로운 트랜잭션으로 판단하고 entityManager에서 기존 관리하던 영속성 객체를 초기화한다.
    clear()
		
    // ItemReader 생성 시 주입받은 Option을 통해 쿼리를 생성
    // 이는 하단에서 자세히 다룰 예정
    val query = createQuery().limit(pageSize.toLong())

    // 쿼리 결과를 저장하기 위해 result를 초기화한다.
    // 여기서 result는 AbstractPagingItemReader에 존재하며, 페이징 되어 가져온 데이터가 저장되는 변수
    initResults()

    // 실제로 쿼리를 실행하여 데이터를 가져온다.
    // 이후 result에 데이터를 addAll로 할당한다.
    fetchQuery(query)

    // 결과가 비어있지 않다면 다음 페이지 시작 지점을 지정한다.
    resetStartIdNotLastPage()
}
```

1. clear()
    - EntityManager가 관리하는 영속성 컨텍스트를 초기화하여, 이전 트랜잭션에서 관리되던 엔티티 객체를 제거합니다.
    - 이렇게 하면 새로운 트랜잭션에서 조회하는 데이터가 이전 상태에 영향을 받지 않고 독립적으로 관리될 수 있습니다.
2. createQuery().limit(pageSize.toLong())
    - QuerydslZeroOffsetOption에서 정의된 createQuery() 함수를 호출하여 페이징 쿼리를 생성합니다.
    - 여기서 중요한 점은 Offset을 사용하지 않고 WHERE 조건을 추가하여 페이징을 수행한다는 점입니다.
3. initResults()
    - AbstractPagingItemReader의 result 리스트를 초기화하여, 새로운 페이징 결과를 저장할 준비를 합니다.
4. fetchQuery(query)
    - query를 실행하여 데이터를 가져오고, result 리스트에 추가합니다.
    - 여기서 조회한 데이터 개수(pageSize)만큼 result에 저장됩니다.
5. resetStartIdNotLastPage()
    - 마지막 페이지가 아닐 경우, 다음 페이지를 가져올 기준(startId)을 업데이트합니다.
    - 즉, 현재 가져온 데이터 중 가장 마지막 데이터를 기준으로 startId를 설정하여, 다음 페이지에서 해당 ID 이후의 데이터를 가져오도록 합니다.

#### createQuery()의 동작 방식

이제 createQuery() 함수에서 어떻게 WHERE 조건이 추가되는지 살펴보겠습니다.


```kotlin
// QuerydslZeroOffsetItemReader.kt
private fun createQuery(): JPAQuery<T> {
    val query = queryCreator(JPAQueryFactory(entityManager))
    return option.createQuery(query)
}

// QuerydslZeroOffsetOption.kt
fun createQuery(query: JPAQuery<T>): JPAQuery<T> {
    return if (sort == Sort.Direction.DESC) {
        query.where(field.loe(startId))  // ID가 startId보다 작거나 같은 데이터 조회
            .orderBy(field.desc())  // 내림차순 정렬
    } else {
        query.where(field.goe(startId))  // ID가 startId보다 크거나 같은 데이터 조회
            .orderBy(field.asc())  // 오름차순 정렬
    }
}
```

1. QueryDSL의 JPAQueryFactory를 사용하여 기본 쿼리를 생성합니다.
2. 이후, option.createQuery(query)를 호출하여 WHERE 조건이 적용된 최종 쿼리를 만듭니다.
	- option 의 sort == DESC (내림차순 조회)
    	- WHERE id <= {startId} ORDER BY id DESC
    	- 즉, 이전 페이지의 마지막 ID보다 작거나 같은 데이터를 가져와서 내림차순으로 정렬
    	- 최신 데이터부터 순차적으로 가져올 때 사용
	- option 의 sort == ASC (오름차순 조회)
    	- WHERE id >= {startId} ORDER BY id ASC
    	- 즉, 이전 페이지의 마지막 ID보다 크거나 같은 데이터를 가져와서 오름차순으로 정렬
    	- 과거 데이터부터 순차적으로 가져올 때 사용


## 마무리

이번에 QuerydslZeroOffsetItemReader를 직접 구현하여 적용했는데요.
이를 실제로 사용할 때는 다음과 같이 설정할 수 있습니다. (예제 코드입니다)

```kotlin
@Bean("${JOB_NAME}_reader")
@StepScope
fun reader(
    @Value("#{jobParameters['startId']}") startId: Long?,
    @Value("#{jobParameters['chunkSize']}") chunkSize: Int?
): QuerydslZeroOffsetItemReader<User> {
    return QuerydslZeroOffsetItemReader(
        option = QuerydslZeroOffsetOption(
            startId = startId!!,
            field = user.userId,
            fieldName = "userId"
        ),
        entityManager = entityManager
    ) {
        it.selectFrom(user)
    }
}
```

### ZeroOffset 방식 적용 후 성능 개선 효과

이렇게 QuerydslZeroOffsetItemReader를 적용한 결과, 기존 JpaPagingItemReader를 사용했을 때 500만 건 데이터 처리에 약 70시간이 걸리던 작업이 1시간 이내로 단축되는 성능 개선을 이루었습니다.

이 덕분에 총 2주 정도 소요될 것으로 예상했던 마이그레이션 작업을 단 하루 만에 완료할 수 있었습니다.

또한, 성능을 더 높이기 위해 JpaItemWriter 대신
- Batch Insert를 지원하는 JdbcItemWriter
- 별도의 ItemWriter를 직접 구현하여 EntityManager를 활용한 벌크 처리

등을 고려할 수도 있었습니다. 하지만
- 이번 배치는 1회성 마이그레이션 작업이었고,
- 이미 절반정도 마이그레이션을 완료한 상태였기 때문에, (남은 시간 1일)
- 추가적인 개선 작업 없이 현재 방식으로 마무리하기로 결정했습니다.

이번 작업을 통해 단순히 Spring Batch에서 제공하는 기본 기능을 그대로 사용하는 것이 아니라, 내부 동작 방식을 이해하고 필요에 따라 최적화하는 것이 중요하다는 점을 다시 한번 깨달았습니다.

앞으로도 대량 데이터를 다뤄야 하는 상황에서 페이징 성능 이슈를 피하기 위해 ZeroOffset 방식의 페이징을 적극 활용할 예정입니다.