


# Page vs Slice

## 개념

Spring Data JPA에서 페이징 처리를 위해 제공하는 두 가지 반환 타입이다.

**Page**는 전체 데이터 개수를 포함한 완전한 페이징 정보를 제공하고, **Slice**는 다음 페이지 존재 여부만 제공하는 경량 페이징이다.

### 핵심 차이점

- **Page**: 전체 카운트 쿼리를 실행하여 총 페이지 수, 전체 데이터 개수 제공
- **Slice**: 카운트 쿼리를 실행하지 않고, `limit + 1` 방식으로 다음 페이지 존재 여부만 확인

### Pageable

페이지 요청 정보를 담는 인터페이스로, `Page`와 `Slice` 모두 `Pageable`을 파라미터로 받는다.

```kotlin
// Pageable의 주요 속성
interface Pageable {
    val pageNumber: Int    // 페이지 번호 (0부터 시작)
    val pageSize: Int      // 페이지 크기
    val sort: Sort         // 정렬 정보
}
```

---

## 왜 필요한가

### 해결하려는 문제

대량의 데이터를 한 번에 조회하면 메모리 부족과 느린 응답 시간이 발생한다. 또한 무한 스크롤과 페이지 네비게이션은 서로 다른 UX 요구사항을 가진다.

### 기존 방식의 한계

1. **성능 문제**: 전체 데이터를 조회하면 메모리와 네트워크 리소스 낭비
2. **불필요한 카운트 쿼리**: 무한 스크롤에서는 전체 개수가 필요 없음
3. **복잡한 구현**: 페이징 로직을 직접 구현하면 오류 가능성 증가

### 제공하는 가치

- **Page**: 페이지 네비게이션 UI에 필요한 모든 정보 제공 (총 페이지 수, 현재 위치)
- **Slice**: 무한 스크롤에 최적화된 경량 페이징 (불필요한 카운트 쿼리 제거)
- **일관된 API**: Spring Data JPA가 페이징 로직을 표준화하여 제공

---

## 동작 원리

### Page 동작 방식

`Page`는 두 개의 쿼리를 실행한다.

```sql
-- 1. 카운트 쿼리
SELECT COUNT(*)
FROM user
WHERE status = 'ACTIVE'

-- 2. 데이터 조회 쿼리
SELECT *
FROM user
WHERE status = 'ACTIVE'
ORDER BY created_at DESC
LIMIT 10 OFFSET 20
```

**특징:**
- 전체 데이터 개수를 알 수 있어 총 페이지 수 계산 가능
- `hasNext()`, `hasPrevious()` 판단 시 카운트 결과 기반
- 페이지 네비게이션 (1, 2, 3, ... 마지막) UI에 적합

### Slice 동작 방식

`Slice`는 한 개의 쿼리만 실행하되, `limit + 1` 방식을 사용한다.

```sql
-- 데이터 조회 쿼리 (limit + 1)
SELECT *
FROM user
WHERE status = 'ACTIVE'
ORDER BY created_at DESC
LIMIT 11 -- pageSize(10) + 1
```

**동작 과정:**
1. 요청한 페이지 크기보다 1개 더 조회 (`limit + 1`)
2. 결과가 페이지 크기보다 크면 다음 페이지 존재
3. 실제 반환 시에는 요청한 페이지 크기만큼만 반환

**특징:**
- 카운트 쿼리를 실행하지 않아 성능 향상
- `hasNext()` 판단 시 조회된 데이터 개수로 판단
- 무한 스크롤 (더보기, Load More) UI에 적합

### hasNext/hasPrevious 구현 차이

```kotlin
// Page의 hasNext 구현
fun hasNext(): Boolean {
    return getNumber() + 1 < getTotalPages() // 전체 페이지 수 기반
}

// Slice의 hasNext 구현 (SlicedExecution.doExecute())
fun hasNext(): Boolean {
    return content.size > pageSize // limit + 1 결과 크기 비교
}
```

---

## 주의사항

### 1. Page의 성능 비용

`Page`는 항상 카운트 쿼리를 실행하므로 대용량 테이블에서 성능 저하가 발생할 수 있다.

```kotlin
// ❌ 대용량 테이블에서 Page 사용 시 문제
fun findAllUsers(pageable: Pageable): Page<User> {
    // COUNT(*) 쿼리가 매번 실행되어 느림
    return userRepository.findAll(pageable)
}

// ✅ Slice 사용으로 카운트 쿼리 제거
fun findAllUsers(pageable: Pageable): Slice<User> {
    // 카운트 쿼리 없이 limit + 1만 실행
    return userRepository.findAll(pageable)
}
```

### 2. Slice는 전체 페이지 수를 알 수 없음

`Slice`는 총 데이터 개수를 모르므로 페이지 네비게이션 UI를 구현할 수 없다.

```kotlin
val slice: Slice<User> = userRepository.findByStatus("ACTIVE", pageable)

// ✅ 사용 가능
slice.hasNext()       // 다음 페이지 존재 여부
slice.hasPrevious()   // 이전 페이지 존재 여부
slice.content         // 현재 페이지 데이터

// ❌ 사용 불가
slice.getTotalPages()    // 컴파일 에러
slice.getTotalElements() // 컴파일 에러
```

### 3. 복잡한 쿼리의 카운트 최적화 필요

`Page`를 사용할 때 복잡한 JOIN이 포함되면 카운트 쿼리도 동일하게 복잡해진다.

```kotlin
// 복잡한 쿼리의 카운트 최적화
@Query(
    value = "SELECT u FROM User u JOIN FETCH u.orders o WHERE u.status = :status",
    countQuery = "SELECT COUNT(u) FROM User u WHERE u.status = :status" // JOIN 제거
)
fun findUsersWithOrders(status: String, pageable: Pageable): Page<User>
```

---

## 실전 적용

### 1. 기본 사용법

```kotlin
// Repository 정의
interface UserRepository : JpaRepository<User, Long> {
    
    // Page 반환 - 페이지 네비게이션
    fun findByStatus(status: String, pageable: Pageable): Page<User>
    
    // Slice 반환 - 무한 스크롤
    fun findByStatusOrderByCreatedAtDesc(status: String, pageable: Pageable): Slice<User>
}

// Service 사용
@Service
class UserService(private val userRepository: UserRepository) {
    
    fun getUsersForPagination(page: Int, size: Int): Page<User> {
        val pageable = PageRequest.of(page, size, Sort.by("createdAt").descending())
        return userRepository.findByStatus("ACTIVE", pageable)
    }
    
    fun getUsersForInfiniteScroll(page: Int, size: Int): Slice<User> {
        val pageable = PageRequest.of(page, size, Sort.by("createdAt").descending())
        return userRepository.findByStatusOrderByCreatedAtDesc("ACTIVE", pageable)
    }
}
```

### 2. 페이지 네비게이션 API (Page)

```kotlin
@RestController
@RequestMapping("/api/users")
class UserController(private val userService: UserService) {
    
    @GetMapping
    fun getUsers(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "10") size: Int
    ): PageResponse<UserDto> {
        val result = userService.getUsersForPagination(page, size)
        
        return PageResponse(
            content = result.content.map { it.toDto() },
            totalPages = result.totalPages,
            totalElements = result.totalElements,
            currentPage = result.number,
            hasNext = result.hasNext(),
            hasPrevious = result.hasPrevious()
        )
    }
}

data class PageResponse<T>(
    val content: List<T>,
    val totalPages: Int,       // 전체 페이지 수
    val totalElements: Long,   // 전체 데이터 개수
    val currentPage: Int,      // 현재 페이지
    val hasNext: Boolean,
    val hasPrevious: Boolean
)
```

### 3. 무한 스크롤 API (Slice)

```kotlin
@RestController
@RequestMapping("/api/feed")
class FeedController(private val userService: UserService) {
    
    @GetMapping
    fun getFeed(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int
    ): SliceResponse<UserDto> {
        val result = userService.getUsersForInfiniteScroll(page, size)
        
        return SliceResponse(
            content = result.content.map { it.toDto() },
            hasNext = result.hasNext(),
            currentPage = result.number,
            pageSize = result.size
        )
    }
}

data class SliceResponse<T>(
    val content: List<T>,
    val hasNext: Boolean,    // 다음 페이지 존재 여부만 제공
    val currentPage: Int,
    val pageSize: Int
)
```

### 4. 커스텀 쿼리 최적화

```kotlin
interface UserRepository : JpaRepository<User, Long> {
    
    // ✅ 복잡한 JOIN이 있는 경우 카운트 쿼리 최적화
    @Query(
        value = """
            SELECT u FROM User u
            JOIN FETCH u.profile p
            JOIN FETCH u.orders o
            WHERE u.status = :status
            AND o.orderDate >= :startDate
        """,
        countQuery = """
            SELECT COUNT(DISTINCT u)
            FROM User u
            JOIN u.orders o
            WHERE u.status = :status
            AND o.orderDate >= :startDate
        """
    )
    fun findUsersWithDetails(
        status: String,
        startDate: LocalDateTime,
        pageable: Pageable
    ): Page<User>
    
    // ✅ Slice로 변경하여 카운트 쿼리 제거
    @Query("""
        SELECT u FROM User u
        JOIN FETCH u.profile p
        WHERE u.status = :status
    """)
    fun findUsersWithProfile(
        status: String,
        pageable: Pageable
    ): Slice<User>
}
```

### 5. 동적 정렬 적용

```kotlin
@Service
class UserService(private val userRepository: UserRepository) {
    
    fun getUsers(
        page: Int,
        size: Int,
        sortBy: String = "createdAt",
        direction: String = "desc"
    ): Slice<User> {
        val sort = if (direction.lowercase() == "asc") {
            Sort.by(sortBy).ascending()
        } else {
            Sort.by(sortBy).descending()
        }
        
        val pageable = PageRequest.of(page, size, sort)
        return userRepository.findByStatus("ACTIVE", pageable)
    }
}
```

### 6. QueryDSL과 함께 사용

```kotlin
@Repository
class UserRepositoryCustomImpl(
    private val queryFactory: JPAQueryFactory
) : UserRepositoryCustom {
    
    fun searchUsers(
        keyword: String,
        pageable: Pageable
    ): Page<User> {
        val query = queryFactory
            .selectFrom(user)
            .where(
                user.name.contains(keyword)
                    .or(user.email.contains(keyword))
            )
        
        // 데이터 조회
        val results = query
            .offset(pageable.offset)
            .limit(pageable.pageSize.toLong())
            .fetch()
        
        // 카운트 쿼리 (최적화된 별도 쿼리)
        val total = queryFactory
            .select(user.count())
            .from(user)
            .where(
                user.name.contains(keyword)
                    .or(user.email.contains(keyword))
            )
            .fetchOne() ?: 0L
        
        return PageImpl(results, pageable, total)
    }
    
    fun searchUsersSlice(
        keyword: String,
        pageable: Pageable
    ): Slice<User> {
        val results = queryFactory
            .selectFrom(user)
            .where(
                user.name.contains(keyword)
                    .or(user.email.contains(keyword))
            )
            .offset(pageable.offset)
            .limit((pageable.pageSize + 1).toLong()) // limit + 1
            .fetch()
        
        val hasNext = results.size > pageable.pageSize
        val content = if (hasNext) {
            results.dropLast(1) // 마지막 요소 제거
        } else {
            results
        }
        
        return SliceImpl(content, pageable, hasNext)
    }
}
```

---

## 참고 자료
- [Spring Data JPA - Paging and Sorting](https://docs.spring.io/spring-data/jpa/reference/repositories/query-methods-details.html#repositories.special-parameters)
- [Spring Data Commons - Page API](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/Page.html)
- [Spring Data Commons - Slice API](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/Slice.html)
