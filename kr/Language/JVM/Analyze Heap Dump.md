# Heap Dump 분석

## 개념

Heap Dump는 **특정 시점의 JVM 힙 메모리 상태를 파일로 저장한 스냅샷**이다.

애플리케이션의 메모리 사용 현황, 객체 간 참조 관계, 메모리 누수 원인 등을 분석하기 위해 사용한다. 힙에 존재하는 모든 객체의 정보와 그들 간의 참조 구조가 포함되어 있다.

### 핵심 특징

- **시점 스냅샷**: 특정 순간의 힙 메모리 상태를 완전히 캡처한다
- **메모리 진단**: OOM(Out Of Memory) 발생 원인을 분석할 수 있다
- **객체 추적**: 어떤 객체가 메모리를 많이 차지하는지 확인 가능하다

---

## 왜 필요한가?

### 해결하려는 문제

운영 중인 애플리케이션에서 메모리 누수나 OOM이 발생했을 때, 로그만으로는 원인을 파악하기 어렵다. 어떤 객체가 메모리를 점유하고 있는지, 왜 GC되지 않는지 알아야 한다.

### 기존 방식의 한계

1. **로그 분석**: 메모리 사용량은 알 수 있지만 구체적인 객체 정보는 파악 불가
2. **모니터링 도구**: 실시간 메모리 사용량은 보이지만 과거 시점 분석 불가
3. **추측 기반 디버깅**: 코드만 보고 메모리 누수 원인을 찾기 어렵다

### 제공하는 가치

- **정확한 진단**: 실제 메모리에 존재하는 객체와 참조 관계를 직접 확인
- **재현 불가능한 문제 해결**: 운영 환경에서만 발생하는 문제도 분석 가능
- **근거 기반 최적화**: 추측이 아닌 데이터 기반으로 메모리 최적화

---

## 동작 원리

### Heap Dump 생성 방식

Heap Dump는 JVM이 힙 메모리의 모든 객체 정보를 직렬화하여 파일로 저장한다. 생성 시점에 STW(Stop The World)가 발생하여 애플리케이션이 일시 중지된다.

### 생성 방법

```bash
# 1. jmap 명령어로 수동 생성
jmap -dump:live,format=b,file=heap.hprof <PID>

# 2. OOM 발생 시 자동 생성 (JVM 옵션)
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/path/to/dumps \
     -jar application.jar

# 3. jcmd 명령어 사용
jcmd <PID> GC.heap_dump /path/to/heap.hprof
```

### 파일 포맷

Heap Dump 파일은 `.hprof` 확장자를 가지며, 바이너리 형식으로 저장된다. 파일 크기는 힙 메모리 사용량과 비례하며, 수 GB에 달할 수 있다.

---

## 주의사항

### 1. 프로덕션 환경에서 생성 시 주의

Heap Dump 생성 중 STW가 발생하여 애플리케이션이 멈춘다.

```bash
# ❌ 잘못된 예: 피크 타임에 무작정 생성
jmap -dump:format=b,file=heap.hprof <PID>  # 서비스 중단 가능

# ✅ 올바른 예: 자동 생성 옵션 설정 또는 한가한 시간 선택
-XX:+HeapDumpOnOutOfMemoryError  # OOM 발생 시에만 자동 생성
```

### 2. 디스크 공간 확인

Heap Dump 파일은 매우 크기 때문에 디스크 공간이 부족하면 생성 실패한다.

```bash
# ✅ 생성 전 디스크 공간 확인
df -h /path/to/dumps

# ✅ 필요시 압축 또는 이전 파일 삭제
gzip old-heap.hprof
```

### 3. 민감 정보 포함 가능성

Heap Dump에는 메모리에 있던 모든 데이터가 포함되므로 비밀번호, API 키 등 민감 정보가 노출될 수 있다.

```bash
# ✅ 분석 후 즉시 삭제
rm -f heap.hprof

# ✅ 외부 공유 금지
# Heap Dump는 내부에서만 분석하고 외부로 전송하지 않는다
```

### Best Practices

- OOM 자동 덤프 옵션(`-XX:+HeapDumpOnOutOfMemoryError`)을 운영 환경에 항상 설정
- Heap Dump 저장 경로를 충분한 디스크 공간이 있는 곳으로 지정
- 분석 후 파일은 보안을 위해 즉시 삭제
- 주기적인 테스트 환경에서 덤프 생성/분석 연습

---

## 실전 적용

### 기본 분석 과정

MAT(Memory Analyzer Tool)를 사용한 분석

```bash
# 1. MAT 다운로드 및 설치
# https://eclipse.dev/mat/downloads.php

# 2. Heap Dump 파일 열기
# File > Open Heap Dump > heap.hprof 선택
```

### 주요 분석 패턴

#### 패턴 1: Leak Suspects Report로 빠른 진단

MAT의 자동 분석 기능을 활용

```
1. Heap Dump 파일 열기
2. "Leak Suspects Report" 자동 실행
3. Suspect 섹션에서 메모리를 많이 차지하는 객체 확인
4. "See stacktrace" 클릭하여 객체 생성 위치 추적
```

**확인 포인트**:
- Problem Suspect 1, 2, 3... 순서대로 확인
- Accumulated Objects by Class 에서 의심되는 클래스 찾기
- Shortest Paths to GC Roots 에서 왜 GC되지 않는지 확인

#### 패턴 2: Dominator Tree로 메모리 점유율 분석

메모리를 가장 많이 사용하는 객체를 찾는다.

```
1. Dominator Tree 뷰 열기
2. Retained Heap 기준으로 정렬
3. 상위 객체들의 참조 관계 추적
4. "List objects" > "with outgoing references" 로 참조 확인
```

**예시**:
```
com.example.Cache @ 0x7f8a1234
  └─ Retained Heap: 500 MB
     └─ java.util.HashMap @ 0x7f8a5678
        └─ Entry[] (size: 1,000,000)  // 여기가 문제!
```

#### 패턴 3: Histogram으로 클래스별 인스턴스 확인

특정 클래스의 인스턴스 개수와 메모리 사용량을 분석

```
1. Histogram 뷰 열기
2. "Shallow Heap" 또는 "Retained Heap" 기준 정렬
3. 의심되는 클래스 우클릭
4. "List objects" > "with incoming references" 선택
5. 어디서 참조하고 있는지 역추적
```

#### 패턴 4: OQL로 특정 객체 검색

SQL과 유사한 쿼리로 객체를 검색

```sql
-- 특정 클래스의 모든 인스턴스 찾기
SELECT * FROM com.example.User

-- 특정 조건의 객체 찾기
SELECT * FROM java.lang.String s WHERE s.count > 1000

-- 크기가 큰 컬렉션 찾기
SELECT * FROM java.util.ArrayList a WHERE a.size > 10000
```

### 실제 분석 예시

```
문제: OOM 발생 with java.lang.OutOfMemoryError: Java heap space

분석 과정:
1. MAT로 heap.hprof 파일 열기
2. Leak Suspects Report 확인
   → Problem Suspect 1: com.example.CacheManager (Retained: 3.2GB)
3. Dominator Tree에서 CacheManager 확인
   → HashMap에 100만 개의 Entry 존재
4. Histogram에서 com.example.CacheEntry 검색
   → 100만 개의 인스턴스 존재, 각 3KB
5. Path to GC Roots 확인
   → static 필드에서 참조되어 GC 불가
   
원인: CacheManager의 static Map이 무한정 증가
해결: LRU 캐시로 변경 + 최대 크기 제한 추가
```

### Before/After 비교

```kotlin
// ❌ 문제 코드: 캐시가 무한정 증가
object CacheManager {
    private val cache = mutableMapOf<String, Data>()
    
    fun put(key: String, data: Data) {
        cache[key] = data  // 계속 쌓임
    }
}

// ✅ 개선 코드: LRU 캐시로 크기 제한
class CacheManager(private val maxSize: Int = 10000) {
    private val cache = object : LinkedHashMap<String, Data>(
        maxSize, 0.75f, true
    ) {
        override fun removeEldestEntry(
            eldest: MutableMap.MutableEntry<String, Data>?
        ): Boolean {
            return size > maxSize  // 오래된 항목 자동 제거
        }
    }
    
    fun put(key: String, data: Data) {
        cache[key] = data
    }
}
```

---

## 참고 자료

### 공식 문서
- [Eclipse Memory Analyzer (MAT)](https://eclipse.dev/mat/)
- [Oracle JVM Troubleshooting Guide](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/)

### 추천 아티클
- [Heap Dump 분석 가이드](https://incheol-jung.gitbook.io/docs/q-and-a/java/heap-dump-feat.-oom) - OOM 발생 시 Heap Dump 분석 방법
- [MAT 실전 활용법](https://shonm.tistory.com/646) - MAT 도구 사용법 상세 가이드
- [Heap Dump로 OOM 해결하기](https://blog.yevgnenll.me/posts/heap-dump-out-of-memory) - 실제 사례 기반 분석 과정

### 관련 TIL
- [JVM](./JVM.md)
- [Garbage Collection](./Garbage%20Collection.md)
- [OOM](./OOM.md)