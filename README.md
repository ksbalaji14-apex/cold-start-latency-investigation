## Investigation & Findings

### ECS Cold Start Causing API Gateway 504

A production troubleshooting case involving an intermittent 504 Gateway
Timeout on the first API request due to ECS/Spring Boot cold-start latency.

**Key areas covered:**
- ECS task cold start
- Spring Boot initialization latency
- API Gateway timeout behavior
- ECS/Target Group health checks
- Application readiness
- Warm-up strategies
- Keep-warm approaches

📄 [Read the detailed investigation](./ecs-cold-start-504-remediation.md)
