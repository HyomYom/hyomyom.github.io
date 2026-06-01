---
layout: post
title: "mdc-structured-logging"
date: 2026-06-01
---



# MDC + Structured Logging

## 1. 개념 정리

### MDC (Mapped Diagnostic Context)

요청(Request) 전체에 **공통으로 따라다니는** 로그 데이터 저장소

- 내부적으로 `ThreadLocal` 기반으로 동작
- 요청 시작 시 값을 넣고, 요청 종료 시 반드시 `MDC.clear()` 호출
- 같은 요청에서 발생한 모든 로그에 자동으로 포함됨
**자주 쓰는 MDC 필드**

| 필드 | 설명 |
| --- | --- |
| `trace_id` | 요청 추적 ID |
| `admin_id` | 요청한 사용자 |
| `request_uri` | 호출된 API 경로 |
| `client_ip` | 클라이언트 IP |
| `http_method` | GET / POST 등 |


---

### Structured Logging

로그를 **문자열이 아닌 JSON 필드**로 기록하는 방식

**기존 방식 (문자열)**

```text
분석 완료 count=120 duration=350ms
```

→ 검색하려면 정규식 파싱 필요

**Structured Logging 방식**

```json
{
  "message": "분석 완료",
  "affected_count": 120,
  "duration_ms": 350
}
```

→ Elasticsearch에서 필드 기반 검색 즉시 가능

**자주 쓰는 이벤트 필드**

| 필드 | 설명 |
| --- | --- |
| `duration_ms` | 처리 시간 (성능 분석) |
| `affected_count` | 영향받은 데이터 수 |
| `status` | SUCCESS / FAIL |
| `target_db` | 대상 DB명 |
| `error_code` | 오류 코드 |


---

## 2. 왜 같이 써야 하는가?

둘은 역할이 다르기 때문에 **함께 쓸 때 완성**된다.

| 구성 요소 | 역할 |
| --- | --- |
| MDC | 요청 전체의 **공통 문맥** 관리 |
| Structured Logging | 특정 이벤트의 **비즈니스 데이터** 기록 |

**MDC만 있을 때** → 누가, 어떤 요청인지는 알지만 결과 데이터가 없음

```json
{
  "message": "영향도 분석 완료",
  "trace_id": "3abf1f11-...",
  "admin_id": "admin01"
}
```

**MDC + Structured Logging** → 문맥 + 비즈니스 데이터 모두 포함

```json
{
  "message": "영향도 분석 완료",
  "trace_id": "3abf1f11-...",
  "admin_id": "admin01",
  "target_db": "customer_db",
  "affected_count": 128,
  "duration_ms": 341,
  "status": "SUCCESS"
}
```

**전체 아키텍처 흐름**

```text
Client Request
    ↓
Filter (MDC 설정: trace_id, admin_id, request_uri)
    ↓
Service (Structured Logging: 비즈니스 데이터)
    ↓
Logback JSON 출력
    ↓
Logstash 수집 → Elasticsearch 저장 → Kibana 검색/시각화
```

**Kibana에서 가능해지는 것들**

```text
admin_id : "admin01"       → 특정 관리자 로그 검색
duration_ms > 300          → 오래 걸린 요청 찾기
target_db : "customer_db"  → 특정 DB 관련 로그
status : "FAIL"            → 실패 로그만 추출
```


---

## 3. 실제 구현 방법

### Gradle 의존성

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'net.logstash.logback:logstash-logback-encoder:7.4'
    implementation 'org.slf4j:slf4j-api:2.0.13'
}
```


---

### MDC Filter 구현

```java
@Slf4j
@Component
public class LogContextFilter implements Filter {

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) req;

        try {
            MDC.put("trace_id", UUID.randomUUID().toString());
            MDC.put("admin_id", request.getHeader("X-Admin-Id"));
            MDC.put("request_uri", request.getRequestURI());

            chain.doFilter(req, res);

        } finally {
            MDC.clear();
        }
    }
}
```

> MDC.clear()가 중요한 이유: Tomcat은 Thread Pool을 재사용한다. clear() 없이 두면 이전 요청의 trace_id가 다음 요청 로그에 섞인다.


---

### Service - Structured Logging 구현

**최신 방식 (권장)**

```java
log.atInfo()
   .addKeyValue("target_db", dbName)
   .addKeyValue("affected_count", affectedCount)
   .addKeyValue("duration_ms", duration)
   .addKeyValue("status", "SUCCESS")
   .log("영향도 분석 완료");
```

**Marker 방식 (레거시)**

```java
log.info(
    Markers.append("target_db", dbName)
           .and(Markers.append("affected_count", affectedCount))
           .and(Markers.append("duration_ms", duration))
           .and(Markers.append("status", "SUCCESS")),
    "영향도 분석 완료"
);
```


---

### Logback JSON 설정 (resources/logback-spring.xml)

```xml
<configuration>
    <appender name="JSON_CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp/>
                <logLevel/>
                <message/>
                <mdc/>
                <arguments/>
                <stackTrace/>
            </providers>
        </encoder>
    </appender>
    <root level="INFO">
        <appender-ref ref="JSON_CONSOLE"/>
    </root>
</configuration>
```


---

## 핵심 요약

| 구성 요소 | 역할 |
| --- | --- |
| MDC | 요청 공통 문맥 저장 |
| Structured Logging | 특정 이벤트 데이터 저장 |
| JSON Logging | ELK 검색 최적화 |
| `trace_id` | 요청 추적 |
| `duration_ms` | 성능 분석 |
| `status` | 장애 분석 |

> MDC로 요청 흐름을 추적하고, Structured Logging으로 비즈니스 데이터를 기록하며, JSON 로그로 ELK 검색 효율을 극대화한다.

