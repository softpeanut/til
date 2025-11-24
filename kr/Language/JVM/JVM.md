# JVM (Java Virtual Machine)

## 개념

JVM은 **자바 바이트코드를 실행하는 가상 머신**이다.

자바 소스 코드(.java)를 컴파일하여 생성된 바이트코드(.class)를 운영체제에 독립적으로 실행할 수 있게 한다. "Write Once, Run Anywhere" 철학의 핵심 구현체이다.

### 핵심 특징

- **플랫폼 독립성**: 운영체제에 관계없이 동일한 바이트코드 실행
- **자동 메모리 관리**: GC(Garbage Collection)를 통한 자동 메모리 회수
- **런타임 최적화**: JIT(Just-In-Time) 컴파일러로 실행 중 성능 최적화

---

## 왜 필요한가?

### 해결하려는 문제

네이티브 언어(C/C++)는 각 운영체제와 하드웨어에 맞게 별도로 컴파일해야 한다. 플랫폼이 다르면 재컴파일이 필요하고, 메모리 관리도 개발자가 직접 해야 한다.

### 기존 방식의 한계

1. **플랫폼 종속성**: OS별로 다른 실행 파일 필요
2. **수동 메모리 관리**: malloc/free 등으로 직접 메모리 관리 필요
3. **이식성 부족**: 코드 변경 없이도 플랫폼마다 재컴파일 필요

### 제공하는 가치

- **이식성**: 한 번 작성한 코드가 모든 플랫폼에서 실행
- **안정성**: 자동 메모리 관리로 메모리 누수와 오류 감소
- **생산성**: 개발자가 비즈니스 로직에 집중 가능

---

## 동작 원리

### JVM 아키텍처

JVM은 **Class Loader → Runtime Data Area → Execution Engine**의 3단계로 구성된다.

```
Java Source (.java)
    ↓ javac
Bytecode (.class)
    ↓ Class Loader
Runtime Data Area
    ↓ Execution Engine
Native Machine Code
```

### 1. Class Loader

`.class` 파일을 읽어 JVM 메모리에 로드한다.

**로딩 과정**:
1. **Loading**: 클래스 파일을 읽어 메모리에 로드
2. **Linking**: 바이트코드 검증, 메모리 할당, 심볼릭 참조를 실제 참조로 변환
3. **Initialization**: static 필드 초기화

**Class Loader 계층**:
```
BootstrapClassLoader (JVM 내장, java.lang.* 등)
    ↓
PlatformClassLoader (JDK 표준 API)
    ↓
AppClassLoader (애플리케이션 클래스)
    ↓
Custom ClassLoader (사용자 정의)
```

**위임 모델 (Delegation Model)**:
- 자식 ClassLoader가 로딩 요청을 받으면 부모에게 먼저 위임
- 부모가 로딩하지 못하면 자식이 로딩 시도
- **목적**: 
  - 핵심 클래스를 애플리케이션에서 재정의하지 못하게 보호
  - 동일한 JVM 내에서 같은 이름의 클래스는 동일한 정의를 공유

### 2. Runtime Data Area

JVM이 프로그램 실행 중 사용하는 메모리 영역이다.

| 영역 | 공유 범위 | 설명 |
|------|----------|------|
| **PC Register** | 스레드별 | 현재 실행 중인 JVM 명령 주소 (CPU 레지스터와 유사) |
| **JVM Stack** | 스레드별 | 메서드 호출 시 지역 변수, 매개변수, 리턴값 등 저장 |
| **Native Method Stack** | 스레드별 | 네이티브 메서드 호출을 위한 스택 |
| **Method Area** | JVM 전체 | 클래스 메타데이터, 상수, static 변수 저장 |
| **Heap** | JVM 전체 | 객체 인스턴스가 할당되는 영역 |

#### Heap 구조 (HotSpot JVM)

```
+---------------------------+
|    Young Generation       |
|  +-----+  +----------+    |
|  |Eden |  |Survivor  |    |
|  |     |  | S0 | S1  |    |
|  +-----+  +----------+    |
+---------------------------+
|    Old Generation         |
+---------------------------+
|    Metaspace              |
+---------------------------+
```

**Young Generation**:
- **Eden**: 객체가 최초로 할당되는 영역
  - 참조되어 있으면 Survivor로, 그렇지 않으면 Garbage로 유지
  - 모두 Survivor로 이동하면 Eden 영역 정리
- **Survivor (S0, S1)**: Eden에서 살아남은 객체가 이동하는 영역

**Old Generation**:
- Young Generation에서 오래 살아남은 객체가 이동 (Promotion)
- Age 임계값을 초과한 객체가 이동
- 비교적 오래 참조되고, 앞으로도 계속 사용될 확률이 높은 객체 저장
- Young 영역 객체 참조를 관리하는 카드 테이블 존재

**Metaspace (Java 8+)**:
- Java 7 이전의 Perm 영역을 대체
- OS 네이티브 메모리에서 클래스 메타데이터 관리
- static 변수와 상수는 Heap에서 관리

### 3. Execution Engine

바이트코드를 실행하는 엔진이다.

**주요 구성 요소**:
- **Interpreter**: 바이트코드를 한 줄씩 해석하여 실행
- **JIT Compiler**: 자주 실행되는 코드를 네이티브 코드로 컴파일하여 성능 향상
- **Garbage Collector**: 더 이상 사용되지 않는 객체를 자동으로 회수 (별도 문서 참조)

---

## 주의사항

### 1. 메모리 누수 방지

Java는 자동 메모리 관리를 제공하지만 메모리 누수가 발생할 수 있다.

```java
// ❌ 잘못된 예: static 컬렉션에 계속 추가
public class Cache {
    private static Map<String, Object> cache = new HashMap<>();
    
    public void add(String key, Object value) {
        cache.put(key, value); // 계속 쌓임, 클래스가 언로드되지 않으면 계속 메모리 점유
    }
}

// ✅ 올바른 예: 크기 제한 설정
public class Cache {
    private static final int MAX_SIZE = 1000;
    private static Map<String, Object> cache = new LinkedHashMap<>(MAX_SIZE, 0.75f, true) {
        protected boolean removeEldestEntry(Map.Entry eldest) {
            return size() > MAX_SIZE; // 오래된 항목 자동 제거
        }
    };
}
```

### 2. finalize() 사용 금지

`finalize()` 메서드는 예측 불가능하고 성능 문제를 야기한다.

```java
// ❌ 잘못된 예
@Override
protected void finalize() throws Throwable {
    // 언제 호출될지 알 수 없어 리소스 해제가 지연될 수 있음
    cleanup();
}

// ✅ 올바른 예: try-with-resources 사용
try (FileInputStream fis = new FileInputStream("file.txt")) {
    // 자동으로 close() 호출
}
```

### Best Practices

- 불필요한 객체 생성 최소화 (객체 풀링, StringBuilder 사용 등)
- static 컬렉션 사용 시 주의 (메모리 누수 원인)
- 리소스는 try-with-resources로 명시적 해제
- 메모리 프로파일링 도구 활용 (VisualVM, JProfiler 등)

---

## 실전 적용

### JVM 옵션 설정

```bash
# Heap 크기 설정
java -Xms2g -Xmx4g MyApp

# Stack 크기 설정
java -Xss1m MyApp

# OOM 시 Heap Dump 생성
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/dumps MyApp

# 클래스 로딩 정보 출력
java -verbose:class MyApp
```

### 모니터링 도구

```bash
# 1. jps - 실행 중인 Java 프로세스 확인
jps -v

# 2. jinfo - JVM 설정 정보 확인
jinfo <PID>

# 3. jmap - Heap 정보 확인
jmap -heap <PID>

# 4. jstack - 스레드 덤프
jstack <PID>

# 5. jcmd - 다목적 진단 도구
jcmd <PID> VM.version
jcmd <PID> VM.flags
```

### Primitive vs Reference Type

**Primitive Type**:
- Stack 메모리에 직접 값 저장
- null 불가능, 제네릭 불가능
- 종류: `byte`, `short`, `int`, `long`, `float`, `double`, `char`, `boolean`

**Reference Type**:
- Stack에는 참조(주소), Heap에는 실제 객체
- null 가능, 제네릭 가능
- 모든 클래스, 인터페이스, 배열 (`java.lang.Object` 상속)

```java
// Primitive Type
int a = 10;  // Stack에 10 직접 저장

// Reference Type
String str = "Hello";  // Stack에 참조, Heap에 "Hello" 객체
```

---

## 참고 자료

### 공식 문서
- [Oracle JVM Specification](https://docs.oracle.com/javase/specs/jvms/se17/html/)
- [Java Garbage Collection Tuning Guide](https://docs.oracle.com/en/java/javase/17/gctuning/)

### 관련 TIL
- [Class Loader](../../Framework/Spring/Boot/Class%20Loader.md)
- [Garbage Collection](./Garbage%20Collection.md)
- [OOM](./OOM.md)
- [Analyze Heap Dump](./Analyze%20Heap%20Dump.md)
