# Out Of Memory (OOM)

## 개념

Out Of Memory는 JVM이나 컨테이너 환경에서 사용 가능한 메모리가 부족할 때 발생하는 오류이다. JVM은 Heap, Metaspace, Direct Memory, Native Memory 등 다양한 메모리 영역을 관리하며, 각 영역에서 메모리 부족이 발생하면 서로 다른 형태의 OOM 오류가 발생한다.

## 왜 필요한가

OOM은 시스템의 안정성과 성능에 직접적인 영향을 미치는 중요한 문제이다.

- **장애 예방**: OOM이 발생하면 애플리케이션이 비정상 종료되거나 성능이 심각하게 저하된다
- **리소스 관리**: 메모리 사용 패턴을 이해하고 적절히 제한하여 안정적인 운영을 보장한다
- **성능 최적화**: OOM의 원인을 분석하면 불필요한 메모리 사용을 줄이고 애플리케이션 성능을 개선할 수 있다
- **비용 절감**: 메모리를 효율적으로 사용하면 서버 리소스를 절약하고 운영 비용을 줄일 수 있다

## 동작 원리

### OOM 발생 메커니즘

JVM과 컨테이너는 각각 독립적으로 메모리를 관리하며, 두 레벨 모두에서 OOM이 발생할 수 있다.

```
Container Memory Limit (예: 1GB)
├── JVM Heap (-Xmx) (예: 512MB)
├── Metaspace (클래스 메타데이터)
├── Direct Memory (NIO 버퍼)
├── Native Memory (Thread Stack, JNI 등)
└── OS 오버헤드
```

컨테이너 메모리 제한을 초과하면 `Container OOMKilled`가 발생하고, JVM 내부의 각 메모리 영역이 한계에 도달하면 해당 영역의 OOM이 발생한다.

### OOM 종류와 원인

| **OOM 종류**                         | **원인**                                                                                | **해결 방안**                                                                                                        | **비고**                                             |
| ---------------------------------- | ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- | -------------------------------------------------- |
| Container OOMKilled                | - JVM heap 설정이 컨테이너 메모리 제한보다 큼                                                        | - Heap 메모리를 컨테이너의 60~80% 정도로 제한<br>    <br>- MaxRAMPercentage 설정 (기본 25%)                                        | - 100%로 제한해두지 않는 이유: Native 메모리에 대한 고려             |
| Java heap space                    | - 많은 객체 생성 및 유지<br><br>- 메모리 누수 (컬렉션 등)<br>    <br>- 대용량 파일/데이터 적재<br><br>- 비효율적인 직렬화 | - 데이터 스트리밍 처리 적용 (파일, json 등)<br>    <br>- 불필요하게 참조가 유지되는 부분이 없는지 확인 (컬렉션, 캐시 등)<br>    <br>- HeapDump 확인        |                                                    |
| Metaspace                          | - 많은 클래스 동적 로딩 (리플렉션, 프록시)<br>    <br>- ClassLoader 누수                                | - 프록시, 리플렉션 줄이기<br>    <br>- MaxMetaspaceSize 설정 (기본 제한 없음)<br>    <br>- Elastic Metaspace 활용 (Java 16 이후 기본 설정) | - Elastic Metaspace는 사용하지 않는 메모리를 즉시 반환            |
| GC overhead limit exceeded         | - 오랫동안 참조되는 객체가 많음<br>    <br>- 객체가 GC로 수거되지 않음                                       | - G1GC 사용<br>    <br>- 캐시 줄이기<br>    <br>- UseStringDeduplication 활용 (G1GC 필요)                                   | - UseStringDeduplication로 중복되는 String 인스턴스를 하나로 관리 |
| unable to create new native thread | - 많은 비동기/병렬 처리 시 OS 스레드 수 제한 초과<br>    <br>- 사용되는 stack 크기가 커서 native 메모리 부족          | - 생성 가능한 스레드 최대 개수 제한<br>    <br>- Virtual Thread 활용<br>    - 혹은 스레드별 stack 크기 지정 (스택오버플로우 주의)                   | - Virtual Thread는 native stack 부담을 주지 않음           |
| Direct buffer memory               | - Netty/NIO의 direct buffer 사용<br>    <br>- 명시적인 해제 누락                                 | - MaxDirectMemorySize 설정 (기본 Xmx와 동일)<br>    <br>- 버퍼 재사용                                                        | - Direct memory는 GC 대상 아니기 때문에 수동 해제 필요            |

## 주의사항

### 1. 컨테이너 메모리 설정 시 여유 공간 확보

JVM Heap을 컨테이너 메모리의 100%로 설정하면 안 된다. Native Memory (Thread Stack, Direct Memory, JNI 등)와 OS 오버헤드를 고려하여 60~80% 정도로 제한해야 한다.

```bash
# 잘못된 설정 - Container OOMKilled 발생 가능
-Xmx1024m  # 컨테이너 메모리도 1GB

# 올바른 설정
-XX:MaxRAMPercentage=75.0  # 컨테이너 메모리의 75%를 Heap으로 사용
```

### 2. GC overhead limit exceeded의 본질

이 오류는 단순히 메모리가 부족한 것이 아니라, GC가 CPU 시간의 98% 이상을 소비하면서도 Heap의 2% 미만만 회수하는 상황에서 발생한다. 이는 Old 영역에 살아있는 객체가 너무 많거나 메모리 단편화가 심각한 경우이다.

**G1GC가 해결하는 방식:**
- Heap을 region 단위로 나누고, 살아있는 객체만 다른 region으로 압축(compacting)하면서 단편화를 줄인다
- 회수 효율이 높은 region부터 우선 수집하여 효율적으로 메모리를 확보한다
- 대형 객체(Humongous)를 별도의 연속 region에 배치하여 단편화를 최소화한다

```bash
# G1GC 활성화 (Java 9 이상에서는 기본값)
-XX:+UseG1GC
```

Java 9 이상부터 기본 GC는 G1GC이다 (CPU 코어가 1개인 경우 Serial GC 선택)

### 3. Virtual Thread로 스레드 제한 극복

일반 스레드(Platform Thread)는 OS 네이티브 스레드와 1:1 매핑되어 OS의 스레드 수 제한을 받는다. Virtual Thread는 JVM이 스케줄링하는 경량 스레드로, 여러 Virtual Thread가 소수의 OS 스레드 위에서 번갈아 실행되므로 I/O 바운드 작업이 많은 환경에서 효율적이다.

```kotlin
// Platform Thread - OS 스레드 수 제한에 걸림
repeat(100_000) {
    Thread.ofPlatform().start { /* 작업 */ }
}

// Virtual Thread - 제한 없이 많은 스레드 생성 가능
repeat(100_000) {
    Thread.ofVirtual().start { /* 작업 */ }
}
```

## 실전 적용

### 1. 컨테이너 환경에서 JVM 메모리 설정

Kubernetes Pod의 메모리 제한이 1GB일 때 안정적인 JVM 설정:

```yaml
# Kubernetes Deployment
resources:
  limits:
    memory: "1Gi"
  requests:
    memory: "1Gi"
```

```bash
# JVM 옵션
-XX:MaxRAMPercentage=75.0         # Heap은 750MB
-XX:+UseG1GC                      # G1GC 사용 (Java 9 이상 기본)
-XX:MaxGCPauseMillis=200          # GC 최대 중단 시간 목표
```

### 2. Heap Dump 분석으로 메모리 누수 확인

`Java heap space` OOM이 발생하면 Heap Dump를 분석하여 원인을 파악한다.

```bash
# Heap Dump 자동 생성 설정
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/logs/heapdump.hprof
```

MAT(Memory Analyzer Tool)로 Heap Dump를 분석하여 메모리를 많이 차지하는 객체를 확인한다.

```bash
# 예시: Collection에 불필요한 객체가 계속 쌓이는 경우
Map<String, User> userCache = new ConcurrentHashMap<>();  // 캐시가 계속 증가
// 해결: Caffeine과 같은 제한된 크기의 캐시 사용
```

### 3. Metaspace 크기 제한

Metaspace는 기본적으로 크기 제한이 없어 무한정 증가할 수 있다. 리플렉션이나 동적 프록시를 많이 사용하는 애플리케이션은 Metaspace 크기를 명시적으로 제한해야 한다.

```bash
# Metaspace 크기 제한
-XX:MaxMetaspaceSize=256m
-XX:MetaspaceSize=128m  # 초기 크기
```

Java 16 이상에서는 Elastic Metaspace가 기본으로 활성화되어 사용하지 않는 메모리를 즉시 반환한다.

### 4. 모니터링 지표 설정

OOM을 사전에 예방하기 위해 메모리 사용량을 지속적으로 모니터링한다.

```kotlin
// Spring Boot Actuator + Micrometer
// JVM 메모리 메트릭 확인
val heapUsed = meterRegistry.gauge("jvm.memory.used", Tags.of("area", "heap"))
val metaspaceUsed = meterRegistry.gauge("jvm.memory.used", Tags.of("id", "Metaspace"))

// 임계값 초과 시 알림 설정
if (heapUsed!! > maxHeap * 0.9) {
    log.warn("Heap memory usage is over 90%")
}
```

## 참고 자료

### 관련 TIL
- [JVM.md](./JVM.md) - JVM 메모리 구조와 각 영역의 역할
- [Garbage Collection.md](./Garbage%20Collection.md) - GC 알고리즘과 튜닝 방법
- [Analyze Heap Dump.md](./Analyze%20Heap%20Dump.md) - MAT를 사용한 Heap Dump 분석 방법

