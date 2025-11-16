# Garbage Collection

## 개념

Garbage Collection(GC)은 JVM에서 더 이상 사용되지 않는 객체를 자동으로 탐지하고 메모리를 회수하는 메커니즘이다. 개발자가 명시적으로 메모리를 해제하지 않아도 JVM이 자동으로 사용하지 않는 객체를 제거하여 메모리 누수를 방지한다.

## 왜 필요한가

수동 메모리 관리는 개발자의 실수로 인한 메모리 누수와 댕글링 포인터(Dangling Pointer) 문제를 야기한다.

- **메모리 누수 방지**: 더 이상 사용하지 않는 객체가 메모리에 계속 남아 있으면 결국 OOM이 발생한다
- **개발 생산성 향상**: 개발자가 메모리 해제를 직접 관리하지 않아도 되어 비즈니스 로직에 집중할 수 있다
- **안정성 보장**: 이미 해제된 메모리를 참조하는 댕글링 포인터 문제를 원천적으로 차단한다
- **자동 최적화**: JVM이 런타임 환경에 맞춰 최적의 GC 전략을 선택하고 튜닝한다

## 동작 원리

### Heap 메모리 구조 (Hotspot JVM)

Hotspot JVM은 Heap을 Generational 방식으로 관리하여 객체의 수명에 따라 다른 영역에 배치한다.

```
Heap Memory
├── Young Generation
│   ├── Eden          : 새로운 객체가 최초로 할당되는 영역
│   └── Survivor (S0, S1) : Eden에서 살아남은 객체가 잠시 머무는 영역
└── Old Generation    : Young에서 오래 살아남은 객체가 이동하는 영역
```

**Young Generation:**
- Eden: 새로운 객체가 생성되면 최초로 할당되는 영역이다
- Survivor (S0, S1): Eden에서 Minor GC가 발생할 때 살아남은 객체가 이동하는 영역이다
- Minor GC: Eden이 가득 차면 발생하며, 살아있는 객체를 Survivor로 이동시키고 Eden을 정리한다

**Old Generation:**
- Promotion: 객체가 Minor GC를 여러 번 살아남아 Age가 일정 기준을 초과하면 Old로 이동한다
- Major GC (Full GC): Old Generation의 메모리가 부족할 때 발생하며, 전체 Heap을 정리한다
- 비교적 오래 참조되고 앞으로도 계속 사용될 확률이 높은 객체들이 저장된다
- Card Table: Young 영역의 객체를 Old 영역에서 참조하는 정보를 관리한다

### GC Root Set

GC는 Root Set에서 시작하여 도달 가능한(Reachable) 모든 객체를 탐색하고, 도달 불가능한 객체를 회수한다.

**Root Set 조건:**
1. **클래스**: 클래스의 정적 필드와 메서드는 프로그램 종료까지 참조가 유지되어 GC Root로 간주된다
   - JVM은 로드된 클래스 정보를 ClassLoaderData(CLDR)에 보관하고, GC가 CLDR 체인을 순회하며 static 필드와 ConstantPool의 참조를 추출한다

2. **스레드 스택**: 메서드의 지역 변수와 매개변수는 메서드 실행 동안 유효하여 GC Root로 간주된다
   - 스레드가 중단되면 각 스택 프레임을 탐색하고, JIT의 OopMap을 통해 어떤 변수가 객체 참조인지 확인한다

3. **활성화된 자바 스레드**: 현재 실행 중이거나 대기 중인 스레드가 종료되기 전까지 관련 객체는 GC Root로 간주된다
   - Thread Roots에서 활성 스레드를 탐색하고, 스레드 로컬과 스택 프레임의 객체 참조를 추출한다

4. **JNI 참조**: 네이티브 코드에서 참조되는 객체는 JVM 외부에서 관리되어 GC 대상이 아니다
   - JVM이 등록한 Handle Table에서 객체 참조를 추출한다

5. **동기화 모니터 객체**: `synchronized` 블록에서 사용 중인 객체는 잠금을 유지하는 동안 GC 대상에서 제외된다

```
GC Root Set
    ↓
    ├── Object A (Reachable)
    │   ├── Object B (Reachable)
    │   └── Object C (Reachable)
    │
    └── Object D (Reachable)

Object E (Unreachable) ← Garbage
Object F (Unreachable) ← Garbage
```

### Reference 종류

JVM은 참조 강도에 따라 4가지 Reference 타입을 제공한다.

| **Reference 타입** | **설명** | **GC 동작** |
|------------------|---------|-----------|
| Strong Reference | 일반적인 객체 참조 (`Object obj = new Object()`) | Root Set에서 도달 가능하면 절대 회수되지 않음 |
| Soft Reference | 캐시용 객체에 사용 | 메모리가 부족할 때만 회수 |
| Weak Reference | 약한 참조 관계 | 다음 GC 사이클에서 항상 회수 |
| Phantom Reference | 소멸 직전 객체 추적 | finalize 후 메모리 회수 전에 참조 큐에 등록 |

**Reference 판별 순서:**
1. Strong: Root Set에서 도달 가능한 객체
2. Soft: Strong이 아니면서 Soft Reference만 거치는 경로가 하나라도 있는 객체
3. Weak: Strong, Soft가 아니면서 Weak Reference만 거치는 경로가 하나라도 있는 객체
4. Phantom: Strong, Soft, Weak가 아닌 객체. `finalize()` 되었지만 메모리가 회수되지 않은 상태

```kotlin
// Weak Reference 사용 예시
val cache = WeakHashMap<String, User>()
cache["key"] = User("name")  // Weak Reference
// 다른 Strong Reference가 없으면 다음 GC에서 회수됨
```


### GC 알고리즘

GC는 객체의 생존 여부를 판별하고 메모리를 회수하기 위해 다양한 알고리즘을 사용한다.

**주요 알고리즘:**
- **Mark and Sweep**: Root Set에서 도달 가능한 객체를 마킹(Mark)하고, 마킹되지 않은 객체를 제거(Sweep)한다
- **Mark and Compact**: Mark and Sweep 후, 살아있는 객체를 Heap의 앞쪽으로 이동시켜 단편화를 줄인다
- **Copy/Scavenge**: 살아있는 객체를 다른 영역으로 복사하고, 기존 영역을 통째로 비운다 (Young Generation에서 사용)
- **Concurrent Mark/Sweep**: 애플리케이션 실행 중 동시에 마킹과 수거를 수행하여 STW(Stop-The-World) 시간을 줄인다

### GC 구현체

Java는 다양한 GC 구현체를 제공하며, 애플리케이션 특성에 따라 선택할 수 있다.

#### 1. Serial GC

단일 스레드로 GC를 수행하는 가장 단순한 방식이다.

- **Minor GC**: Young Generation에서 Mark and Sweep 수행
- **Major GC**: Old Generation에서 Mark and Sweep and Compact 수행
- **적용 대상**: CPU 코어가 1개이거나 Heap 크기가 작은 환경 (클라이언트 애플리케이션)
- **단점**: Single Thread로 동작하여 STW 시간이 길다

```bash
-XX:+UseSerialGC
```

#### 2. Parallel GC

Serial GC를 멀티 스레드로 병렬 처리하여 처리량을 높인 방식이다.

- **Minor GC**: Young Generation을 여러 스레드로 병렬 처리
- **Major GC**: Old Generation을 여러 스레드로 병렬 처리
- **적용 대상**: 멀티 코어 환경에서 처리량(Throughput)이 중요한 배치 애플리케이션
- **단점**: STW 시간이 Serial GC보다 짧지만 여전히 발생한다

```bash
-XX:+UseParallelGC
-XX:ParallelGCThreads=4  # GC 스레드 개수 지정
```

#### 3. CMS GC (Concurrent Mark Sweep)

애플리케이션 실행 중 동시에 GC를 수행하여 STW 시간을 최소화한다.

**Major GC 과정:**
1. Initial Mark (STW): Root Set에서 직접 참조하는 객체만 마킹
2. Concurrent Mark: 애플리케이션 실행 중 Old Generation 전체 마킹
3. Remark (STW): Concurrent Mark 중 변경된 객체를 재마킹
4. Concurrent Sweep: 애플리케이션 실행 중 마킹되지 않은 객체 제거

- **적용 대상**: 응답 시간(Latency)이 중요한 웹 애플리케이션
- **단점**: Compact를 수행하지 않아 메모리 단편화가 발생하고, CPU 리소스를 더 많이 사용한다

```bash
-XX:+UseConcMarkSweepGC  # Java 9 이후 Deprecated, Java 14에서 제거
```

#### 4. G1 GC (Garbage First)

Heap을 Region 단위로 나누어 관리하여 효율적으로 GC를 수행한다.

**구조:**
```
Heap (Region 단위로 분할)
├── Eden Region
├── Survivor Region
├── Old Region
├── Humongous Region  : Region 크기의 50%를 초과하는 대형 객체
└── Available/Unused  : 아직 사용되지 않은 Region
```

**동작 방식:**
- **Minor GC**: Eden Region의 객체를 Survivor 또는 Available Region으로 복사하고 Eden을 비운다
- **Mixed GC**: Minor GC로 충분하지 않을 때, Young과 일부 Old Region을 함께 수거한다
- **Region 선택 전략**: GC 효율이 가장 높은 Region(회수 가능한 메모리가 많은 Region)부터 우선 수거한다
- **Compact**: 살아있는 객체만 다른 Region으로 복사하면서 자연스럽게 단편화를 해소한다

- **적용 대상**: 대용량 Heap (4GB 이상)을 사용하는 서버 애플리케이션
- **장점**: STW 시간을 예측 가능한 수준으로 제어하고, 단편화 문제를 해결한다

```bash
-XX:+UseG1GC  # Java 9 이상 기본 GC
-XX:MaxGCPauseMillis=200  # 목표 최대 STW 시간
```

#### 5. ZGC / Shenandoah GC

초대용량 Heap (수 TB)에서도 밀리초 단위의 짧은 STW를 보장하는 저지연 GC이다.

- **ZGC**: Colored Pointer와 Load Barrier를 사용하여 대부분의 작업을 동시에 수행한다
- **Shenandoah GC**: Brooks Pointer를 사용하여 객체 이동 중에도 애플리케이션이 객체에 접근할 수 있다
- **적용 대상**: 초대용량 Heap과 극도로 짧은 응답 시간이 필요한 애플리케이션
- **STW 시간**: 10ms 이하로 제한

```bash
-XX:+UseZGC        # ZGC (Java 15 이상 정식 지원)
-XX:+UseShenandoahGC  # Shenandoah GC
```

## 주의사항

### 1. finalize() 사용 지양

`finalize()` 메서드는 GC가 객체를 회수하기 전에 호출되지만, 호출 시점을 보장하지 않고 성능 저하를 유발한다.

```kotlin
// 잘못된 방식 - finalize() 사용
class Resource {
    protected fun finalize() {
        // 정리 작업 - 언제 호출될지 모름
        cleanup()
    }
}

// 올바른 방식 - try-with-resources (Kotlin: use)
class Resource : Closeable {
    override fun close() {
        cleanup()  // 명시적으로 즉시 호출됨
    }
}

Resource().use { resource ->
    // 사용 후 자동으로 close() 호출
}
```

### 2. GC 튜닝은 신중하게

기본 GC 설정이 대부분의 경우 충분히 효율적이다. 성능 문제가 명확히 측정되지 않았다면 GC 튜닝을 시도하지 않는 것이 좋다.

**튜닝 전 확인 사항:**
- GC 로그를 분석하여 실제 STW 시간과 빈도 측정
- Heap Dump로 메모리 누수 확인
- 애플리케이션 코드 최적화 (불필요한 객체 생성 줄이기)

```bash
# GC 로그 활성화
-Xlog:gc*:file=gc.log:time,uptime,level,tags
```

### 3. Young/Old 크기 비율 고려

Young Generation이 너무 작으면 Minor GC가 빈번히 발생하고, 너무 크면 Minor GC 시간이 길어진다. 일반적으로 Heap의 1/3 정도가 적절하다.

```bash
# Young Generation 크기 지정
-XX:NewRatio=2  # Old:Young = 2:1 (Young이 전체의 1/3)
-Xmn512m        # Young Generation 고정 크기
```

## 실전 적용

### 1. GC 선택 기준

애플리케이션 특성에 따라 적절한 GC를 선택한다.

| **애플리케이션 유형** | **권장 GC** | **이유** |
|-----------------|-----------|---------|
| 웹 애플리케이션 (대용량 Heap) | G1 GC | 응답 시간 예측 가능, 단편화 해결 |
| 배치 처리 (처리량 중요) | Parallel GC | 멀티 스레드로 빠른 처리 |
| 실시간 시스템 (초저지연) | ZGC / Shenandoah | 밀리초 단위 STW |
| 소형 애플리케이션 (단일 코어) | Serial GC | 오버헤드 최소화 |

```bash
# Spring Boot 애플리케이션 예시 (4GB Heap)
-Xms4g -Xmx4g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
```

### 2. 메모리 누수 탐지 패턴

GC가 정상 동작해도 메모리 누수는 발생할 수 있다. 주요 패턴을 확인한다.

```kotlin
// 1. Static Collection에 객체 무한 추가
companion object {
    private val cache = mutableListOf<User>()  // 계속 증가
}

// 해결: 크기 제한이 있는 캐시 사용
companion object {
    private val cache = CacheBuilder.newBuilder()
        .maximumSize(1000)
        .build<String, User>()
}

// 2. Thread Local 정리 누락
private val threadLocal = ThreadLocal<Connection>()

fun process() {
    threadLocal.set(createConnection())
    // ... 작업
    // threadLocal.remove() 누락 - 스레드 풀 사용 시 누수
}

// 해결: 반드시 정리
fun process() {
    try {
        threadLocal.set(createConnection())
        // ... 작업
    } finally {
        threadLocal.remove()  // 명시적 정리
    }
}
```

### 3. GC 모니터링 지표

운영 환경에서 GC 상태를 지속적으로 모니터링한다.

```kotlin
// Spring Boot Actuator + Micrometer
@Component
class GcMonitor(private val meterRegistry: MeterRegistry) {
    
    @Scheduled(fixedRate = 60000)  // 1분마다
    fun checkGcMetrics() {
        val gcPauseTime = meterRegistry.timer("jvm.gc.pause").totalTime(TimeUnit.MILLISECONDS)
        val gcCount = meterRegistry.counter("jvm.gc.pause").count()
        
        if (gcPauseTime > 1000) {  // 1초 이상 STW
            log.warn("High GC pause time: ${gcPauseTime}ms")
        }
    }
}
```

**주요 모니터링 지표:**
- GC 빈도 (Minor/Major GC 횟수)
- STW 시간 (평균/최대)
- Heap 사용률 (Young/Old 각각)
- GC 처리량 (애플리케이션 실행 시간 / 전체 시간)

### 4. Heap Dump 분석

OOM 발생 시 자동으로 Heap Dump를 생성하여 원인을 분석한다.

```bash
# Heap Dump 자동 생성
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/logs/heapdump.hprof

# 수동으로 Heap Dump 생성
jmap -dump:live,format=b,file=heap.hprof <pid>
```

MAT(Memory Analyzer Tool)로 Heap Dump를 열어 메모리를 많이 차지하는 객체와 GC Root 경로를 확인한다.

## 참고 자료

- [Java Garbage Collection Basics](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html) - Oracle GC 튜토리얼
- [Naver D2: Java Garbage Collection](https://d2.naver.com/helloworld/1329)
- [Getting Started with the G1 Garbage Collector](https://www.oracle.com/technetwork/tutorials/tutorials-1876574.html)
- [ZGC Documentation](https://wiki.openjdk.org/display/zgc)

### 관련 TIL
- [JVM.md](./JVM.md) - JVM 메모리 구조
- [OOM.md](./OOM.md) - OOM 종류와 해결 방법
- [Analyze Heap Dump.md](./Analyze%20Heap%20Dump.md) - Heap Dump 분석 방법