# 🧩 Component Services

## 1. Rate Limiter Backend
- **REST API exposed to user services**
- Implements core rate-limiting logic
- Queries cache/store to enforce limits
- Logs rate-limited requestors

## 2. Rules Service
- Stores configuration for rate limits per user service/endpoint
- Only modifiable by internal admin tools
- Supports hot reloads or periodic refresh

## 3. Request Log Service
- Persists rate-limited requestor events
- May support querying (e.g., filter by time/user)
- May use tiered logging (e.g., log only violations)

---

# 🗃️ Databases & Storage

| Component        | Storage Used               | Purpose                         |
|------------------|----------------------------|---------------------------------|
| **Rate State**   | Redis                      | Token counters / sliding window |
| **Rules Store**  | Redis / DB                 | Rate limit configs              |
| **Request Logs** | PostgreSQL / ClickHouse    | Analytics, querying             |

---

# 🧰 Implementation Models: Backend vs Middleware

| Option         | Pros                              | Cons                                |
|----------------|-----------------------------------|-------------------------------------|
| **Middleware** | Integrated into user services     | Less reusable, duplicate logic per service |
| **Backend Service** | Centralized, reusable, language-agnostic | Slightly more network latency      |
| **API Gateway** | Easy if using commercial gateways | Limited algorithm flexibility, vendor lock-in |

**Our choice**: Backend Service for central control, reusability, and flexibility.

---

# 🧮 Rate Limiting Algorithms

| Algorithm              | Burst Support | Accuracy | Memory Use | Complexity | Notes                                                        |
|------------------------|---------------|----------|------------|------------|--------------------------------------------------------------|
| **Fixed Window Counter**| ❌ No         | ⚠️ Low   | ✅ Low     | ✅ Simple  | May double-allow at window edges                             |
| **Sliding Window Log**  | ✅ Yes        | ✅ High  | ❌ High    | ⚠️ Medium | Logs timestamps; ideal for small-scale precision             |
| **Sliding Window Counter**| ✅ Yes      | ✅ Medium| ⚠️ Medium | ⚠️ Medium | Balanced accuracy, uses multiple small buckets               |
| **Token Bucket**        | ✅ Yes        | ✅ High  | ✅ Low     | ⚠️ Medium | Most popular; allows burst, smooth refill                    |
| **Leaky Bucket**        | ❌ No         | ✅ High  | ✅ Low     | ❌ High   | Best for smoothing output rate; strict throttle              |

**Selected Algorithm**: **Token Bucket**
- Each request consumes a token.
- Tokens refill at a fixed rate.
- Redis used to track token state with TTL and Lua scripts for atomic updates.



# 🔁 Data Flow

```mermaid
sequenceDiagram
  participant Client
  participant UserService
  participant RateLimiter
  participant Redis
  participant RulesService
  participant Logger

  Client->>UserService: Make API request
  UserService->>RateLimiter: Check rate limit (user_id, endpoint)
  RateLimiter->>Redis: Fetch request count/token bucket
  alt Cache miss
    RateLimiter->>RulesService: Get rate limit rules
    RulesService-->>RateLimiter: Return rule
    RateLimiter->>Redis: Cache the rule
  end
  alt Request Allowed
    RateLimiter-->>UserService: Allow (200 OK)
  else Rate Limited
    RateLimiter->>Logger: Log violation
    RateLimiter-->>UserService: Deny (429)
  end
```

```mermaid
C4Context
title Rate Limiting Service - System Context

Person(ExternalUser, "External User", "Uses APIs through external clients")

System(UserService, "User Service", "Handles API traffic for different business logic")

System(RateLimiter, "Rate Limiting Service", "Controls request rate per user and endpoint")

System_Ext(RedisStore, "Redis", "Caches rate limiting counters and tokens")

System_Ext(RulesService, "Rules Service", "Stores rate limiting configuration")

System_Ext(LoggingSystem, "Logging System", "Stores audit logs of rate-limited requests")


Rel(ExternalUser, UserService, "Sends API requests")
Rel(UserService, RateLimiter, "Calls to check rate limit")
Rel(RateLimiter, RedisStore, "Reads/writes request counters")
Rel(RateLimiter, RulesService, "Fetches rate limit rules")
Rel(RateLimiter, LoggingSystem, "Logs 429/rate-limited events")



```

```mermaid 

C4Container
title Rate Limiting Service - Container Diagram

Person(ExternalUser, "External User", "Uses external APIs")

System_Boundary(rateLimiterSystem, "Rate Limiting Service") {

  Container(ApiGateway, "API Gateway / Service Mesh", "Envoy / NGINX / Kong", "Handles routing, TLS, authentication, and rate limit checks")

  Container(RateLimiterAPI, "Rate Limiter API", "Go / Node.js / FastAPI", "Exposes HTTP API for rate limiting decisions")

  Container(RulesService, "Rules Service", "Python / Go Service", "Manages and serves rate limit configurations")

  Container(RateLimiterCore, "Rate Limiting Engine", "Go / Rust", "Implements rate limiting logic using token bucket or sliding window")

  ContainerDb(RedisStore, "Redis", "In-Memory Key-Value Store", "Stores rate limiting state and counters")

  ContainerDb(LoggingStore, "Logging/Audit DB", "PostgreSQL / Elasticsearch", "Stores audit logs and rate limit violations")
}

System_Ext(UserService, "User Service", "Client-facing service that checks rate limits before processing requests")

Rel(ExternalUser, UserService, "Sends requests to APIs")

Rel(UserService, ApiGateway, "Forwards API requests through gateway")

Rel(ApiGateway, RateLimiterAPI, "Calls to check if user should be rate limited")

Rel(RateLimiterAPI, RateLimiterCore, "Delegates rate limit logic")

Rel(RateLimiterCore, RedisStore, "Reads/writes usage counters or tokens")

Rel(RateLimiterCore, RulesService, "Fetches per-endpoint rate limit configs")

Rel(RateLimiterCore, LoggingStore, "Logs 429s and rate limit events")

```