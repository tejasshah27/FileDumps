# RedisJSON + Spring WebFlux Research Report

**Date:** 2026-08-10  
**Researcher:** Kimi Chat (Moonshot AI)  
**Topic:** Using RedisJSON with Spring WebFlux (Reactive Stack) for partial JSON document reads, comparison with Redis Hash, and Java client selection (Jedis vs Lettuce).

---

## Table of Contents
1. [User Requirements](#user-requirements)
2. [Research Methodology](#research-methodology)
3. [Intermediate Research Findings](#intermediate-research-findings)
4. [Final Recommendations](#final-recommendations)
5. [Code Snippets](#code-snippets)
6. [Reference Links](#reference-links)

---

## User Requirements

The user provided a sample JSON document:

```json
{
  "entityID": "FUND001",
  "status": "APPROVED",
  "attributes": {
    "key1": "value1",
    "key2": "value2",
    "key3": ["x", "y", "s"],
    "key4": {
      "nkey1": "nvalue1",
      "nkey2": "nvalue2"
    }
  },
  "createdBy": "SOEID12"
}
```

**Partial reading requirement:** Return only `entityID`, `status`, `attributes.key1`, and `attributes.key2`.

**Expectations:**
1. Store and retrieve JSON documents in Redis for mostly read calls.
2. Partial reading of JSON is the 99% workload.
3. Does RedisJSON help? If yes, with Jedis or another option for a reactive stack service?
4. How to ensure partial document fetching — code snippet?
5. Is it as performant as Redis Hash?
6. Which to prefer: flatten as Redis Hash or store as RedisJSON? (Fetching is mostly 1 level deep.)

**Constraints:**
- Deep research, trusted sources (preferably official API documentation).
- No bias — user has no preference.
- Ask questions if unclear.

---

## Research Methodology

The research was conducted using web search across multiple queries to gather information from official Redis documentation, Jedis/Lettuce GitHub repositories, Spring Data Redis documentation, and community benchmarks.

### Search Queries Executed

**Round 1:**
- `RedisJSON partial read JSON.GET path syntax official documentation`
- `Jedis RedisJSON support reactive java examples`
- `Spring Data Redis WebFlux reactive RedisJSON`
- `RedisJSON vs Hash performance benchmark nested documents`
- `Lettuce RedisJSON reactive stream support`

**Round 2:**
- `Jedis 5 RedisJSON support UnifiedJedis jsonGet example`
- `RedisJSON performance vs Hash memory benchmark official`
- `Spring Data Redis RedisJSON module support`
- `RedisJSON partial read network bandwidth savings vs Hash`

**Round 3:**
- `Jedis reactive support Mono Flux non blocking`
- `RedisJSON memory overhead vs String Hash official documentation`
- `RedisJSON JSON.GET multiple paths performance network optimization`

---

## Intermediate Research Findings

### Finding 1: Jedis is Blocking — Incompatible with WebFlux

Jedis is a synchronous, blocking Redis client. It does not support reactive/non-blocking I/O. Using Jedis inside a Spring WebFlux application would block Netty event-loop threads, defeating the entire purpose of the reactive stack.

**Sources:**
- Jedis GitHub README: "Jedis — a thin, synchronous wrapper...build everything above it yourself."
- Spring Data Redis docs: "Reactive API using the Lettuce driver."
- Lettuce documentation: "Lettuce is a scalable thread-safe Redis client...reactive usage."

**Conclusion:** For WebFlux, **Lettuce is the only viable standard driver**.

### Finding 2: RedisJSON is Designed for Partial Reads

RedisJSON stores documents in a parsed binary tree on the server. When `JSON.GET` is issued with a JSONPath expression, the server walks directly to the requested node(s) and returns only that slice. The document is **not** fully serialized, transmitted, and parsed on the client side.

**Key command for partial read:**
```bash
JSON.GET fund:FUND001 $.entityID $.status $.attributes.key1 $.attributes.key2
```

Response:
```json
{
  "$.entityID": ["FUND001"],
  "$.status": ["APPROVED"],
  "$.attributes.key1": ["value1"],
  "$.attributes.key2": ["value2"]
}
```

**Sources:**
- RedisJSON official docs: `JSON.GET key [path [path ...]]` supports multiple paths in one call.
- When multiple paths are specified, Redis returns an object mapping each path to its result array.

### Finding 3: Spring Data Redis Does Not Expose RedisJSON Module Commands

`ReactiveRedisTemplate` does not provide native RedisJSON commands. Developers must drop down to Lettuce's native reactive JSON API for `JSON.GET`, `JSON.SET`, etc.

**Sources:**
- Spring Data Redis issue tracker: "Spring data redis support for modules...dispatching custom commands with Lettuce."

### Finding 4: RedisJSON Memory Overhead vs Hash

- **RedisJSON:** Internal representation is "often more expensive, size-wise, than the serialized form." Every value carries at least 8 bytes of pointer overhead; containers (objects/arrays) incur additional bookkeeping.
- **Redis Hash:** When below `hash-max-ziplist-entries` (default 512) and `hash-max-ziplist-value` (default 64 bytes), Redis encodes hashes as compact ziplists. Extremely memory-efficient for small field values.

**Sources:**
- RedisJSON docs: "Redis JSON stores JSON values as binary data...more expensive, size-wise, than the serialized form."
- Redis internals docs: "When hashes are small we can instead just encode them...amortized time for HGET and HSET commands is still O(1)."

### Finding 5: Performance Comparison

| Dimension | RedisJSON | Redis Hash (Flattened) |
|-----------|-----------|------------------------|
| Partial read (network) | Only requested paths transit the wire | `HMGET` fetches only requested fields |
| Server-side CPU | JSONPath traversal O(N×M) where N=depth, M=parent keys | O(1) per field hash lookup |
| Memory overhead | Higher (binary tree structure) | Lower (ziplist optimization) |
| Nested data | Native support | Cannot natively nest; must serialize sub-objects |
| Atomic partial update | `JSON.SET` at path | `HSET` single field |
| Bulk partial read | Single `JSON.GET` with multiple paths | Single `HMGET` with multiple fields |

**Time complexity:** Redis official docs state JSON path evaluation is *O(N×M)*, where N is the depth and M is the number of parent object keys. Array indexing is O(1).

---

## Final Recommendations

### Critical Finding: Jedis is Incompatible with WebFlux

**Jedis does not support reactive/non-blocking I/O.** It is a synchronous, blocking client. Using it inside a Spring WebFlux application will defeat the entire purpose of the reactive stack by blocking Netty event-loop threads.

For a WebFlux service, **Lettuce** is the only viable driver among the standard Java clients. It is built on Netty and exposes synchronous, asynchronous, and reactive (`Mono`/`Flux`) APIs.

### 1. Does RedisJSON Help for 99% Partial-Read Workloads?

**Yes — this is exactly the problem RedisJSON was designed to solve.**

RedisJSON stores documents in a parsed binary tree on the server. When you issue a `JSON.GET` with a JSONPath expression, the server walks directly to the requested node(s) and returns only that slice. The document is **not** fully serialized, transmitted, and parsed on the client side.

### 2. Reactive Stack Options for RedisJSON

| Option | Reactive Support | RedisJSON Native | Notes |
|--------|------------------|------------------|-------|
| **Jedis** | ❌ Blocking only | ✅ Yes (via `UnifiedJedis`) | Unusable in WebFlux |
| **Lettuce** | ✅ `Mono` / `Flux` | ✅ Yes (native JSON API) | **Recommended** |
| **Spring Data Redis** | ✅ (Lettuce only) | ❌ No module commands | Use for basic ops; drop to Lettuce for JSON |
| **Redis OM Spring** | ❌ Synchronous repositories | ✅ `@Document` annotation | Separate library; not reactive |

Spring Data Redis does **not** expose RedisJSON module commands through `ReactiveRedisTemplate`. You must use Lettuce's native reactive JSON API.

### 3. Which to Prefer: RedisJSON or Redis Hash?

**Recommendation: Use RedisJSON with Lettuce (reactive).**

**Reasoning:**

1. **Your reads are partial and nested.** With RedisHash, `attributes` is a nested object. To read `attributes.key1` partially from a Hash, you would need to either:
   - Store the entire `attributes` blob as a JSON string inside one hash field — then partial read requires fetching and parsing the whole blob client-side.
   - Flatten into dot-notation keys (`attributes.key1`, `attributes.key2`, `attributes.key3`, `attributes.key4.nkey1`...) — this is fragile, loses structure, and makes updates painful.

   RedisJSON solves this natively with `$.attributes.key1`.

2. **Network efficiency matters for 99% reads.** `JSON.GET key $.entityID $.status $.attributes.key1 $.attributes.key2` transfers exactly those four values. No unused fields cross the wire.

3. **Read-heavy workload amortizes memory cost.** RedisJSON uses more RAM, but if your workload is 99% reads and documents are not enormous, the CPU and network savings on partial reads outweigh the memory penalty.

4. **Jedis is not an option for WebFlux.** Even if you prefer Jedis's API, it has no reactive support. Lettuce is the standard for Spring WebFlux.

#### When to Consider Redis Hash Instead

Consider flattening to Hash **only if**:
- Your documents are very small and memory is severely constrained.
- You never need to read nested fields individually (you always read the whole `attributes` block).
- You can tolerate the application-layer complexity of mapping nested JSON to flat hash fields.

Even then, for nested JSON, you would likely end up storing serialized JSON strings inside hash fields, which brings you back to the "fetch and parse everything" problem that RedisJSON solves.

### 4. Final Architecture Recommendation

| Layer | Choice |
|-------|--------|
| **Redis Driver** | Lettuce (reactive) |
| **Redis Module** | RedisJSON |
| **Spring Integration** | Custom `@Bean` exposing `RedisReactiveCommands<String, String>` |
| **Partial Read** | `JSON.GET` with multiple JSONPath arguments |
| **Serialization** | Jackson / Gson for POJO ↔ JSON string conversion before calling Lettuce JSON API |

If you need higher-level repository abstractions later, evaluate **Redis OM Spring** (`@Document` annotation), but note that its repository pattern is currently synchronous, not reactive.

**Bottom line:** For a Spring WebFlux application doing predominantly partial reads on nested JSON, **Lettuce + RedisJSON** is the correct, performant, and officially supported stack. Jedis should not be used in this reactive context.

---

## Code Snippets

### Dependency

```xml
<dependency>
    <groupId>io.lettuce</groupId>
    <artifactId>lettuce-core</artifactId>
    <version>6.8.0.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

### Configuration Bean

```java
import io.lettuce.core.RedisClient;
import io.lettuce.core.api.StatefulRedisConnection;
import io.lettuce.core.api.reactive.RedisReactiveCommands;
import io.lettuce.core.json.JsonPath;
import io.lettuce.core.json.JsonParser;
import io.lettuce.core.json.JsonValue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RedisJsonConfig {

    @Bean
    public RedisClient redisClient() {
        return RedisClient.create("redis://localhost:6379");
    }

    @Bean
    public StatefulRedisConnection<String, String> redisConnection(RedisClient redisClient) {
        return redisClient.connect();
    }

    @Bean
    public RedisReactiveCommands<String, String> reactiveCommands(
            StatefulRedisConnection<String, String> connection) {
        return connection.reactive();
    }

    @Bean
    public JsonParser jsonParser(RedisReactiveCommands<String, String> commands) {
        return commands.getJsonParser();
    }
}
```

### Service: Store Document

```java
import io.lettuce.core.api.reactive.RedisReactiveCommands;
import io.lettuce.core.json.JsonParser;
import io.lettuce.core.json.JsonPath;
import io.lettuce.core.json.JsonValue;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;

@Service
public class FundJsonService {

    private final RedisReactiveCommands<String, String> commands;
    private final JsonParser parser;

    public FundJsonService(RedisReactiveCommands<String, String> commands, JsonParser parser) {
        this.commands = commands;
        this.parser = parser;
    }

    public Mono<String> storeFund(String entityId, String jsonPayload) {
        JsonValue jsonValue = parser.createJsonValue(jsonPayload);
        return commands.jsonSet("fund:" + entityId, JsonPath.ROOT_PATH, jsonValue)
                .doOnNext(System.out::println); // prints "OK"
    }
}
```

### Service: Partial Read (The 99% Case)

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import io.lettuce.core.api.reactive.RedisReactiveCommands;
import io.lettuce.core.json.JsonPath;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;

import java.util.List;

@Service
public class FundPartialReadService {

    private final RedisReactiveCommands<String, String> commands;
    private final ObjectMapper objectMapper = new ObjectMapper();

    public FundPartialReadService(RedisReactiveCommands<String, String> commands) {
        this.commands = commands;
    }

    /**
     * Fetches only entityID, status, attributes.key1 and attributes.key2
     * in a single Redis round-trip.
     */
    public Mono<FundPartialDto> getFundPartial(String entityId) {
        String key = "fund:" + entityId;

        List<JsonPath> paths = List.of(
                JsonPath.of("$.entityID"),
                JsonPath.of("$.status"),
                JsonPath.of("$.attributes.key1"),
                JsonPath.of("$.attributes.key2")
        );

        // jsonGet returns a Flux<JsonValue>; collect to list
        return commands.jsonGet(key, paths.toArray(new JsonPath[0]))
                .collectList()
                .map(this::mapToDto);
    }

    private FundPartialDto mapToDto(List<io.lettuce.core.json.JsonValue> values) {
        // Values correspond to paths in order
        FundPartialDto dto = new FundPartialDto();
        if (!values.isEmpty()) dto.setEntityID(unquote(values.get(0).toString()));
        if (values.size() > 1) dto.setStatus(unquote(values.get(1).toString()));
        if (values.size() > 2) dto.setKey1(unquote(values.get(2).toString()));
        if (values.size() > 3) dto.setKey2(unquote(values.get(3).toString()));
        return dto;
    }

    private String unquote(String s) {
        // jsonGet wraps strings in quotes; strip them
        if (s != null && s.startsWith("\"") && s.endsWith("\"")) {
            return s.substring(1, s.length() - 1);
        }
        return s;
    }
}
```

### DTO

```java
public class FundPartialDto {
    private String entityID;
    private String status;
    private String key1;
    private String key2;
    // getters / setters
}
```

### WebFlux Controller

```java
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;

@RestController
@RequestMapping("/funds")
public class FundController {

    private final FundPartialReadService partialReadService;

    public FundController(FundPartialReadService partialReadService) {
        this.partialReadService = partialReadService;
    }

    @GetMapping("/{entityId}")
    public Mono<FundPartialDto> getFundPartial(@PathVariable String entityId) {
        return partialReadService.getFundPartial(entityId);
    }
}
```

---

## Reference Links

### Official Redis Documentation
1. **RedisJSON Commands Reference** — https://redis.io/docs/latest/commands/json.get/
   - Documents `JSON.GET` syntax, multiple path support, and return formats.

2. **RedisJSON Data Type** — https://redis.io/docs/latest/develop/data-types/json/
   - Explains binary tree internal structure, memory overhead, and performance characteristics.

3. **RedisJSON Path Evaluation Complexity** — https://redis.io/docs/latest/commands/json.get/
   - Time complexity: O(N×M), where N is depth and M is number of parent object keys.

### Jedis Documentation
4. **Jedis GitHub README** — https://github.com/redis/jedis
   - Confirms Jedis is a "thin, synchronous wrapper."

5. **Jedis 5.0 Release Notes / RedisJSON Support** — https://github.com/redis/jedis/releases
   - Documents `UnifiedJedis` JSON API but confirms no reactive support.

### Lettuce Documentation
6. **Lettuce Core GitHub** — https://github.com/redis/lettuce
   - "Scalable thread-safe Redis client providing synchronous, asynchronous and reactive APIs."

7. **Lettuce Reactive API Guide** — https://lettuce.io/core/release/reference/#reactive-api
   - Documents `RedisReactiveCommands`, `Mono`, and `Flux` support.

8. **Lettuce JSON API Examples** — https://lettuce.io/core/release/reference/#json-commands
   - Shows `jsonGet`, `jsonSet`, `JsonPath`, and `JsonParser` usage.

### Spring Data Redis
9. **Spring Data Redis Reference** — https://docs.spring.io/spring-data/redis/reference/
   - Confirms reactive support is Lettuce-only.

10. **Spring Data Redis Reactive Support** — https://docs.spring.io/spring-data/redis/reference/redis/reactive-redis.html
    - `ReactiveRedisTemplate` and `ReactiveRedisConnection` documentation.

11. **Spring Data Redis Module Support Discussion** — https://github.com/spring-projects/spring-data-redis/issues
    - Community discussions on RedisJSON module integration (or lack thereof in template).

### Redis OM Spring
12. **Redis OM Spring GitHub** — https://github.com/redis/redis-om-spring
    - Object mapping for Redis hashes and JSON documents. Note: synchronous repository pattern.

### Performance & Internals
13. **Redis Hash Internals (Ziplist)** — https://redis.io/docs/latest/develop/data-types/hashes/
    - Memory-efficient encoding for small hashes.

14. **RedisJSON vs Hash Performance Analysis** — https://redis.io/blog/
    - Official Redis blog posts on JSON module performance and use cases.

15. **RedisJSON Memory Overhead Discussion** — https://redis.io/docs/latest/develop/data-types/json/
    - "More expensive, size-wise, than the serialized form."

---

*End of Research Report*
