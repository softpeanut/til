# Page vs Slice

## Concept

Two return types provided by Spring Data JPA for pagination handling.

**Page** provides complete pagination information including total data count, while **Slice** provides lightweight pagination with only next page existence information.

### Key Differences

- **Page**: Executes total count query to provide total pages and total element count
- **Slice**: Doesn't execute count query, only checks next page existence using `limit + 1` approach

### Pageable

An interface that contains page request information, used as a parameter by both `Page` and `Slice`.

```kotlin
// Key properties of Pageable
interface Pageable {
    val pageNumber: Int    // Page number (starts from 0)
    val pageSize: Int      // Page size
    val sort: Sort         // Sort information
}
```

---

## Why Needed

### Problems to Solve

Querying large amounts of data at once causes memory shortage and slow response times. Additionally, infinite scroll and page navigation have different UX requirements.

### Limitations of Existing Approach

1. **Performance Issues**: Querying all data wastes memory and network resources
2. **Unnecessary Count Queries**: Total count is not needed for infinite scroll
3. **Complex Implementation**: Implementing pagination logic manually increases error probability

### Value Provided

- **Page**: Provides all information needed for page navigation UI (total pages, current position)
- **Slice**: Optimized lightweight pagination for infinite scroll (eliminates unnecessary count queries)
- **Consistent API**: Spring Data JPA provides standardized pagination logic

---

## How It Works

### Page Operation

`Page` executes two queries.

```sql
-- 1. Count query
SELECT COUNT(*)
FROM user
WHERE status = 'ACTIVE'

-- 2. Data retrieval query
SELECT *
FROM user
WHERE status = 'ACTIVE'
ORDER BY created_at DESC
LIMIT 10 OFFSET 20
```

**Characteristics:**
- Can calculate total pages by knowing total data count
- `hasNext()`, `hasPrevious()` determined based on count result
- Suitable for page navigation (1, 2, 3, ... last) UI

### Slice Operation

`Slice` executes only one query using the `limit + 1` approach.

```sql
-- Data retrieval query (limit + 1)
SELECT *
FROM user
WHERE status = 'ACTIVE'
ORDER BY created_at DESC
LIMIT 11 -- pageSize(10) + 1
```

**Operation Process:**
1. Query one more than requested page size (`limit + 1`)
2. If result size is larger than page size, next page exists
3. Returns only requested page size when returning actual result

**Characteristics:**
- Performance improvement by not executing count query
- `hasNext()` determined by number of queried data
- Suitable for infinite scroll (load more, Load More) UI

### hasNext/hasPrevious Implementation Difference

```kotlin
// Page's hasNext implementation
fun hasNext(): Boolean {
    return getNumber() + 1 < getTotalPages() // Based on total pages
}

// Slice's hasNext implementation (SlicedExecution.doExecute())
fun hasNext(): Boolean {
    return content.size > pageSize // Compare limit + 1 result size
}
```

---

## Pitfalls

### 1. Performance Cost of Page

`Page` always executes count query, which can cause performance degradation on large tables.

```kotlin
// ❌ Problem when using Page on large tables
fun findAllUsers(pageable: Pageable): Page<User> {
    // COUNT(*) query executes every time, slow
    return userRepository.findAll(pageable)
}

// ✅ Remove count query by using Slice
fun findAllUsers(pageable: Pageable): Slice<User> {
    // Execute only limit + 1 without count query
    return userRepository.findAll(pageable)
}
```

### 2. Slice Cannot Know Total Pages

`Slice` doesn't know total data count, so cannot implement page navigation UI.

```kotlin
val slice: Slice<User> = userRepository.findByStatus("ACTIVE", pageable)

// ✅ Available
slice.hasNext()       // Whether next page exists
slice.hasPrevious()   // Whether previous page exists
slice.content         // Current page data

// ❌ Not available
slice.getTotalPages()    // Compile error
slice.getTotalElements() // Compile error
```

### 3. Count Query Optimization Needed for Complex Queries

When using `Page` with complex JOINs, count query becomes equally complex.

```kotlin
// Optimize count query for complex queries
@Query(
    value = "SELECT u FROM User u JOIN FETCH u.orders o WHERE u.status = :status",
    countQuery = "SELECT COUNT(u) FROM User u WHERE u.status = :status" // Remove JOIN
)
fun findUsersWithOrders(status: String, pageable: Pageable): Page<User>
```

---

## Practical Application

### 1. Basic Usage

```kotlin
// Repository definition
interface UserRepository : JpaRepository<User, Long> {
    
    // Return Page - Page navigation
    fun findByStatus(status: String, pageable: Pageable): Page<User>
    
    // Return Slice - Infinite scroll
    fun findByStatusOrderByCreatedAtDesc(status: String, pageable: Pageable): Slice<User>
}

// Service usage
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

### 2. Page Navigation API (Page)

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
    val totalPages: Int,       // Total pages
    val totalElements: Long,   // Total data count
    val currentPage: Int,      // Current page
    val hasNext: Boolean,
    val hasPrevious: Boolean
)
```

### 3. Infinite Scroll API (Slice)

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
    val hasNext: Boolean,    // Only provides next page existence
    val currentPage: Int,
    val pageSize: Int
)
```

### 4. Custom Query Optimization

```kotlin
interface UserRepository : JpaRepository<User, Long> {
    
    // ✅ Optimize count query when complex JOIN exists
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
    
    // ✅ Remove count query by changing to Slice
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

### 5. Dynamic Sort Application

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

### 6. Usage with QueryDSL

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
        
        // Data retrieval
        val results = query
            .offset(pageable.offset)
            .limit(pageable.pageSize.toLong())
            .fetch()
        
        // Count query (optimized separate query)
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
            results.dropLast(1) // Remove last element
        } else {
            results
        }
        
        return SliceImpl(content, pageable, hasNext)
    }
}
```

---

## References
- [Spring Data JPA - Paging and Sorting](https://docs.spring.io/spring-data/jpa/reference/repositories/query-methods-details.html#repositories.special-parameters)
- [Spring Data Commons - Page API](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/Page.html)
- [Spring Data Commons - Slice API](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/Slice.html)
