# Jaeger Distributed Tracing: Complete Enterprise Guide

This guide provides the required Verification Scenarios, Error Tracking insights, and Performance Analysis strategies for your Jaeger deployment at **[http://localhost:16686](http://localhost:16686)**.

---

## 1. Trace Propagation & Service Names

Spring Boot 3 natively propagates the W3C `traceparent` headers. Because we configured `spring.application.name` in each `application.properties`, Jaeger automatically groups the traces into the following unique services in the dropdown:

- `api-gateway`
- `user-service`
- `order-service`
- `notification-service`

---

## 2. Verification Scenarios

### Scenario 1: Create User

1. **Action**: Open Swagger UI and POST a new user to `/api/users/register`.
2. **Expected Trace**: Select `api-gateway` in Jaeger and click "Find Traces". You will see a trace with 2 spans:
   - `Gateway Route Resolution`
   - Downstream to `user-service` where the `user.register` custom business span is executed.

### Scenario 2: Create Order

1. **Action**: Obtain a JWT token and POST a new order to `/api/orders`.
2. **Expected Trace**: You will see a massive End-to-End trace containing:
   - Gateway -> Order Service (`order.create` span)
   - Order Service -> User Service (Feign Client span to validate user)
   - Order Service -> Notification Service (`notification.send` span via Feign)

### Scenario 3: Downstream Failure (Resilience4j)

1. **Action**: Stop the user service: `docker stop user-service`. Send a `POST /api/orders` request.
2. **Expected Trace**:
   - The trace will show the Gateway calling the Order Service.
   - The Order Service Feign call to User Service will show a **Red Error Span** indicating a timeout or connection refused.
   - You will see the Resilience4j Fallback method execute successfully, returning the 400 Bad Request to the Gateway.

---

## 3. Error Tracking in Jaeger

Micrometer Tracing automatically attaches exception stack traces and HTTP status codes directly to the Jaeger spans.

- **Search by Error**: In the Jaeger search panel, add the Tag: `error=true`. This will instantly filter for any HTTP 4xx, HTTP 5xx, or unhandled Java exceptions across the entire cluster.
- **Circuit Breaker Activations**: When an external call fails, the Feign span will be marked with `error=true`. The subsequent span will show the fallback execution path, allowing you to visually see the circuit breaker activating.

---

## 4. Performance Analysis

Jaeger is an incredible tool for finding bottlenecks.

- **Database Latency**: Look for spans labeled `query` or `jdbc`. You can see exactly how many milliseconds the MySQL database took to execute the insert.
- **External Call Latency**: Compare the length of the `order-service` span to the nested `user-service` Feign span. If the `user-service` span takes up 90% of the bar length, you know the User Service is your bottleneck.
- **Compare Traces**: Use the "Compare" feature in Jaeger to select two traces of the same endpoint (one fast, one slow) to visually see where the latency degraded.

---

## 5. Troubleshooting Guide

**No Traces appearing in Jaeger?**

1. **Check OTLP Endpoint**: Ensure `OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4318/v1/traces` is set correctly in your environment variables.
2. **Check Sampling Rate**: Verify `management.tracing.sampling.probability=1.0` is in `application.properties`. If it's set to 0.1, you will only see 1 out of 10 requests!
3. **Verify Header Propagation**: If a trace suddenly breaks and starts a new trace ID in the downstream service, ensure OpenFeign is configured correctly to pass the `Authorization` and `traceparent` headers (which we fixed via the `FeignClientInterceptor` earlier).

---

## 6. Docker & OTLP Architecture Recap

```yaml
# How Jaeger was integrated into docker-compose.yml
jaeger:
  image: jaegertracing/all-in-one:latest
  ports:
    - "16686:16686" # UI
    - "4318:4318"   # OTLP HTTP Receiver
  environment:
    - COLLECTOR_OTLP_ENABLED=true
```
