# Class Loader

## 개념

Class Loader는 **JVM이 클래스 파일(.class)을 메모리에 로드하는 메커니즘**이다. 자바 애플리케이션이 실행될 때 모든 클래스를 한 번에 로드하지 않고, 필요한 시점에 동적으로 로드한다.

### 핵심 특징

- **동적 로딩**: 클래스가 실제로 필요한 시점에 로드된다
- **계층 구조**: Class Loader는 부모-자식 관계의 계층 구조를 가진다
- **위임 모델**: 클래스 로딩 요청을 부모에게 위임하는 Parent Delegation Model을 따른다
- **네임스페이스 분리**: 각 Class Loader는 독립적인 네임스페이스를 가져 클래스 충돌을 방지한다

---

## 왜 필요한가

### 해결하려는 문제

모든 클래스를 애플리케이션 시작 시 한 번에 로드하면 메모리 낭비와 긴 시작 시간이 발생한다. 또한 동일한 이름의 클래스가 여러 곳에 존재할 때 충돌이 발생할 수 있다.

### 기존 방식의 한계

1. **메모리 낭비**: 사용하지 않는 클래스까지 모두 로드
2. **긴 시작 시간**: 모든 클래스를 로드할 때까지 대기
3. **클래스 충돌**: 동일한 이름의 클래스 구분 불가
4. **보안 이슈**: 신뢰할 수 없는 코드와 시스템 코드의 분리 어려움

### 제공하는 가치

- **효율적인 메모리 사용**: 필요한 클래스만 로드하여 메모리 절약
- **빠른 시작 시간**: 애플리케이션 시작 시 최소한의 클래스만 로드
- **모듈화**: 각 Class Loader가 독립적인 네임스페이스로 클래스 충돌 방지
- **보안**: 시스템 클래스와 애플리케이션 클래스를 분리하여 보안 강화
- **핫 스왑**: 개발 중 클래스를 재로드하여 빠른 피드백

---

## 동작 원리

### Class Loader 계층 구조

JVM의 Class Loader는 3단계 계층 구조로 구성된다.

```
Bootstrap ClassLoader (null)
    ↓
Platform ClassLoader (PlatformClassLoader)
    ↓
Application ClassLoader (AppClassLoader)
    ↓
Custom ClassLoader (예: RestartClassLoader)
```

**각 Class Loader의 역할:**

1. **Bootstrap ClassLoader**
   - JVM의 네이티브 코드로 구현되어 Java 객체가 아님 (null로 표시됨)
   - JDK 핵심 라이브러리 로드 (`java.lang.*`, `java.util.*` 등)
   - `$JAVA_HOME/lib` 디렉토리의 rt.jar, core 라이브러리

2. **Platform ClassLoader** (Java 9 이전: Extension ClassLoader)
   - Java 플랫폼 확장 라이브러리 로드
   - `$JAVA_HOME/lib/ext` 또는 `java.ext.dirs` 경로의 클래스
   - JDBC 드라이버 같은 표준 확장 라이브러리 (`javax.*`)

3. **Application ClassLoader** (AppClassLoader)
   - 애플리케이션 클래스패스의 클래스 로드
   - 개발자가 작성한 애플리케이션 코드
   - 외부 라이브러리 JAR 파일

4. **Custom ClassLoader**
   - 특수한 목적을 위한 사용자 정의 Class Loader
   - Spring Boot Devtools의 RestartClassLoader (Hot Reload)
   - 애플리케이션 서버의 WAR 격리

### Parent Delegation Model (부모 위임 모델)

클래스 로딩 요청이 발생하면 다음 순서로 처리된다.

```
1. 클래스 로딩 요청 발생
   ↓
2. 현재 Class Loader에서 이미 로드되었는지 확인
   ↓
3. 부모 Class Loader에게 위임 (재귀적으로 Bootstrap까지)
   ↓
4. 부모가 로드 실패 시 현재 Class Loader에서 로드 시도
   ↓
5. 로드 성공 또는 ClassNotFoundException 발생
```

**동작 흐름:**

```kotlin
// 예: String 클래스 로딩
AppClassLoader.loadClass("java.lang.String")
    → PlatformClassLoader.loadClass("java.lang.String")
        → BootstrapClassLoader.loadClass("java.lang.String")
            → ✅ Bootstrap이 로드 성공 (rt.jar)
```

**이점:**
- **보안**: 핵심 시스템 클래스를 먼저 로드하여 악의적인 코드로부터 보호
- **일관성**: 동일한 클래스가 여러 위치에 있어도 항상 부모가 로드한 버전 사용
- **중복 방지**: 이미 로드된 클래스를 재로드하지 않음

### Spring Boot에서의 Class Loader

Spring Boot 애플리케이션에서는 AppClassLoader가 주로 사용되며, devtools를 활성화하면 RestartClassLoader가 추가된다.

**AppClassLoader 로딩 순서:**

1. **부모 위임**: PlatformClassLoader → BootstrapClassLoader
2. **애플리케이션 클래스**: `build/classes/kotlin/main` 디렉토리
3. **라이브러리 JAR**: `libs/*.jar` 또는 의존성 JAR 파일

**RestartClassLoader (Spring Boot Devtools):**
- 애플리케이션 코드만 RestartClassLoader로 로드
- 라이브러리는 AppClassLoader로 로드 (재시작 시 유지)
- 코드 변경 시 RestartClassLoader만 재생성하여 빠른 재시작

---

## 주의사항

### 1. 클래스 중복 정의 시 우선순위 이해

부모 위임 모델로 인해 동일한 이름의 클래스가 여러 위치에 있어도 부모가 먼저 로드한다.

```kotlin
// kotlin.String 재정의 시도
package kotlin

class String {
    // 재정의된 내용
}

// 실제 로드되는 클래스:
// → BootstrapClassLoader가 로드한 java.lang.String
// → 재정의한 클래스는 절대 사용되지 않음
```

**ClassLoader 확인 방법:**

```kotlin
@RestController
class ClassLoaderController {
    
    @GetMapping("/classloader")
    fun checkClassLoader(): Map<String, Any?> {
        return mapOf(
            "String" to String::class.java.classLoader, // null (Bootstrap)
            "DataSource" to javax.sql.DataSource::class.java.classLoader, // Platform
            "RestController" to org.springframework.web.bind.annotation.RestController::class.java.classLoader, // App
            "MyClass" to this::class.java.classLoader // App or Restart
        )
    }
}
```

### 2. 클래스 로딩 충돌 주의

같은 클래스가 서로 다른 Class Loader로 로드되면 다른 클래스로 인식된다.

```kotlin
// ClassLoader A로 로드된 User
val user1 = classLoaderA.loadClass("com.example.User").newInstance()

// ClassLoader B로 로드된 User
val user2 = classLoaderB.loadClass("com.example.User").newInstance()

// ClassCastException 발생!
// user1과 user2는 서로 다른 클래스로 인식됨
```

### 3. 메모리 누수 가능성

Class Loader가 제거되지 않으면 로드된 모든 클래스가 메모리에 남는다.

```kotlin
// Metaspace OOM 원인
// - 동적으로 생성된 Class Loader가 해제되지 않음
// - Reflection으로 생성된 프록시 클래스가 계속 증가
// - 클래스 언로딩이 발생하지 않음

// 해결 방법:
// 1. Class Loader 참조를 명시적으로 해제
// 2. MaxMetaspaceSize 설정
// 3. 불필요한 동적 클래스 생성 줄이기
```

---

## 실전 적용

### 1. 클래스 로딩 순서 확인

실제로 어떤 Class Loader가 클래스를 로드했는지 확인한다.

```kotlin
@RestController
class ClassLoaderTestController {
    
    @GetMapping("/test-classloader")
    fun testClassLoader(): Map<String, String> {
        val results = mutableMapOf<String, String>()
        
        // 1. kotlin.String - Bootstrap이 로드 (null)
        results["kotlin.String"] = String::class.java.classLoader?.toString() ?: "BootstrapClassLoader"
        
        // 2. javax.sql.DataSource - Platform이 로드
        results["javax.sql.DataSource"] = javax.sql.DataSource::class.java.classLoader.toString()
        
        // 3. Spring 라이브러리 - App이 로드
        results["RestController"] = org.springframework.web.bind.annotation.RestController::class.java.classLoader.toString()
        
        // 4. 사용자 정의 클래스 - App 또는 Restart가 로드
        results["MyController"] = this::class.java.classLoader.toString()
        
        return results
    }
}
```

**예상 출력:**
```json
{
  "kotlin.String": "BootstrapClassLoader",
  "javax.sql.DataSource": "jdk.internal.loader.ClassLoaders$PlatformClassLoader@xxx",
  "RestController": "org.springframework.boot.loader.LaunchedURLClassLoader@xxx",
  "MyController": "org.springframework.boot.devtools.restart.classloader.RestartClassLoader@xxx"
}
```

### 2. 클래스 재정의 실험

동일한 패키지 경로에 클래스를 재정의하면 어떻게 되는지 확인한다.

```kotlin
// 1. javax.sql.DataSource 재정의 시도
package javax.sql

interface DataSource {
    fun customMethod(): String
}

// 2. 실제 로드 확인
@Service
class DataSourceTest(
    private val dataSource: javax.sql.DataSource // 어떤 DataSource가 주입될까?
) {
    fun test() {
        println(dataSource::class.java.classLoader)
        // PlatformClassLoader - 재정의된 클래스가 아닌 표준 라이브러리가 로드됨
    }
}
```

**결과:**
- `javax.sql.DataSource`: PlatformClassLoader가 로드 (표준 라이브러리 우선)
- 재정의한 클래스는 절대 사용되지 않음

### 3. Spring Boot Devtools 활용

RestartClassLoader로 빠른 재시작을 구현한다.

```kotlin
// build.gradle.kts
dependencies {
    developmentOnly("org.springframework.boot:spring-boot-devtools")
}

// application.yml
spring:
  devtools:
    restart:
      enabled: true
      additional-paths: src/main/kotlin
      exclude: static/**,public/**
```

**동작 방식:**
```
변경 감지
    ↓
RestartClassLoader만 재생성
    ↓
애플리케이션 코드만 재로드
    ↓
라이브러리는 기존 AppClassLoader 유지 (빠른 재시작)
```

### 4. 클래스 로딩 디버깅

클래스 로딩 과정을 로그로 확인한다.

```bash
# JVM 옵션
-verbose:class
# 또는
-XX:+TraceClassLoading

# 출력 예시:
[Loaded java.lang.String from /Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home/lib/modules]
[Loaded javax.sql.DataSource from /Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home/lib/modules]
[Loaded com.example.MyController from file:/app/build/classes/kotlin/main/]
```

---

## 참고 자료

### 공식 문서
- [Oracle ClassLoader Documentation](https://docs.oracle.com/javase/8/docs/api/java/lang/ClassLoader.html)
- [JVM Specification - Loading](https://docs.oracle.com/javase/specs/jvms/se17/html/jvms-5.html)

### 추천 아티클
- [클래스 로더 이해하기](https://kkang-joo.tistory.com/10)
- [Spring Boot Devtools의 RestartClassLoader](https://docs.spring.io/spring-boot/docs/current/reference/html/using.html#using.devtools.restart)

### 관련 TIL
- [JVM](../../../Language/JVM/JVM.md) - JVM 아키텍처와 Class Loader의 위치