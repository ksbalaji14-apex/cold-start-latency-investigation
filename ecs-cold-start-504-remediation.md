# ECS Cold Start Causing API Gateway 504 Timeout — Root Cause & Remediation

## Problem Statement

When hitting an API endpoint (fronted by Amazon API Gateway, backed by an ECS-hosted service) for the first time, the request fails with a **504 Gateway Timeout**. Every subsequent request to the same endpoint succeeds normally.

This is a classic **cold start** problem.

## Root Cause Analysis

1. **ECS Task Starting from Zero** — If the service has scaled down to 0 tasks, or a task was recently stopped/replaced, the very first incoming request triggers a fresh container startup.
2. **Spring Boot Initialization Overhead** — Spring Boot applications can take 15–30+ seconds to fully initialize (component scanning, bean creation, connection pool warm-up), which is well within the range that trips API Gateway's timeout.
3. **API Gateway Timeout Ceiling** — API Gateway (REST API type) has a hard timeout of 29–30 seconds. If the container isn't fully ready and serving traffic by then, the gateway gives up and returns a 504, even though the container eventually comes up fine.

## Remediation Options

### 1. Keep Containers Warm (Immediate Fix)
```json
// ECS Service configuration
{
  "desiredCount": 1,
  "minimumHealthyPercent": 100
}
```
Keeping at least one task running at all times avoids the zero-to-one cold start entirely.

### 2. Optimize Spring Boot Startup Time
```properties
# application.properties
spring.main.lazy-initialization=true
spring.jmx.enabled=false
spring.devtools.restart.enabled=false
```
```java
@SpringBootApplication(scanBasePackages = "com.yourapp.specific")
```
Narrowing component scan and disabling non-essential features cuts down bean initialization time.

### 3. Proper Health Check Wiring
```java
@Component
public class StartupHealthIndicator implements HealthIndicator {
    private volatile boolean ready = false;

    @EventListener(ApplicationReadyEvent.class)
    public void onReady() {
        ready = true;
    }

    @Override
    public Health health() {
        return ready ? Health.up().build() : Health.down().build();
    }
}
```
Ensures the load balancer / target group only routes traffic once the app is truly ready, not just once the container process has started.

### 4. Tune ECS Target Group Health Check Settings
```json
{
  "healthCheckPath": "/actuator/health",
  "healthCheckIntervalSeconds": 30,
  "healthyThresholdCount": 2,
  "unhealthyThresholdCount": 3,
  "healthCheckTimeoutSeconds": 5,
  "deregistrationDelay": 30
}
```
Reducing `deregistrationDelay` from the 300s default speeds up how quickly unhealthy/old tasks are cycled out during deploys and scaling events.

### 5. API Gateway Timeout Ceiling Awareness
REST APIs on API Gateway cap out at 29 seconds and this is not configurable. HTTP APIs allow slightly more headroom (up to 30 seconds), but neither is a substitute for eliminating the cold start itself.

### 6. Warm-Up Endpoint
```java
@RestController
public class WarmupController {

    @PostConstruct
    public void warmup() {
        // Initialize critical beans/connections eagerly
    }

    @GetMapping("/warmup")
    public ResponseEntity<String> warmup() {
        return ResponseEntity.ok("Warmed up");
    }
}
```
Forces eager initialization of critical dependencies (DB pools, caches, etc.) right after startup rather than lazily on first real request.

### 7. Scheduled Keep-Warm Pings via EventBridge
Schedule a CloudWatch/EventBridge rule to hit the service endpoint every ~5 minutes, keeping the container active and avoiding idle scale-down triggering a fresh cold start on the next real user request.

## Recommended Immediate Action

Set the ECS service `desiredCount` to at least **1**, and pair it with proper health check configuration. This alone eliminates most cold-start 504s while longer-term startup optimization work (lazy init, warm-up endpoints) is rolled out.

---
*Documented from production troubleshooting on a Spring Boot microservice running on AWS ECS, fronted by API Gateway.*
