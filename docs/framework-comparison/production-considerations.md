# Production Considerations

## Overview

Moving LLM agents from prototypes to production systems requires careful consideration of safety, reliability, monitoring, and maintenance. Deepagents and Claude SDK take fundamentally different approaches to production readiness.

## Safety & Permissions

### Deepagents: Implement Your Own

#### Permission Model
```python
# Custom permission system
class PermissionManager:
    def __init__(self, user_role: str):
        self.user_role = user_role
        self.allowed_operations = self.load_permissions(user_role)

    def check_permission(self, tool_name: str, args: dict) -> bool:
        """Validate tool call before execution."""
        # Custom authorization logic
        if tool_name.startswith("write_") and self.user_role != "admin":
            return False

        if tool_name == "delete_object":
            return self.user_role == "admin"

        # Site-based access control
        if "site" in args:
            return args["site"] in self.allowed_operations.get("sites", [])

        return True

# Wrap tools with permission checks
class SafeNetBoxAgent:
    def __init__(self, permission_manager: PermissionManager):
        self.permissions = permission_manager

    async def execute_tool(self, tool_name: str, args: dict):
        # Enforce permissions
        if not self.permissions.check_permission(tool_name, args):
            raise PermissionDenied(
                f"User not authorized for {tool_name} with {args}"
            )

        return await self.run_tool(tool_name, args)
```

**Responsibility**: Developer implements all safety logic

**Benefits**:
- Custom authorization models
- Fine-grained access control
- Domain-specific safety rules

**Costs**:
- Significant development effort
- Maintenance burden
- Security vulnerability risk

### Claude SDK: Built-In Controls

#### Declarative Permissions
```python
# Permission configuration (built-in)
options = ClaudeAgentOptions(
    permission_mode="acceptEdits",  # Options: deny, ask, acceptEdits
    allowed_tools=[
        "netbox_get_objects",  # Read-only tools
        "netbox_get_object_by_id",
        "netbox_search_objects"
    ],  # Whitelist approach
    system_prompt={
        "type": "preset",
        "preset": "claude_code",
        "append": (
            "CRITICAL: Never modify production data without explicit user confirmation. "
            "Always use read-only operations unless specifically instructed."
        )
    }
)
```

**Characteristics**:
- Declarative configuration
- SDK enforces permissions
- Audit trail automatic

**Benefits**:
- Production-ready immediately
- Battle-tested security
- Consistent enforcement

**Trade-offs**:
- Less flexibility for unusual requirements
- Constrained to SDK permission model

## Error Handling & Reliability

### Deepagents: Custom Resilience

#### Retry Logic
```python
class ResilientAgent:
    async def execute_with_retry(
        self,
        tool_func: Callable,
        *args,
        max_retries: int = 3,
        backoff: str = "exponential"
    ):
        """Custom retry strategy."""
        for attempt in range(max_retries):
            try:
                return await tool_func(*args)

            except TransientError as e:
                if attempt == max_retries - 1:
                    raise

                # Custom backoff calculation
                wait_time = self.calculate_backoff(attempt, backoff)
                logger.warning(f"Attempt {attempt + 1} failed, retrying in {wait_time}s")
                await asyncio.sleep(wait_time)

            except PermanentError:
                # Don't retry permanent failures
                raise

    def calculate_backoff(self, attempt: int, strategy: str) -> float:
        """Domain-specific backoff logic."""
        if strategy == "exponential":
            return min(2 ** attempt, 60)  # Cap at 60 seconds
        elif strategy == "linear":
            return 5 * (attempt + 1)
        else:
            return 1
```

#### Circuit Breaker Pattern
```python
class CircuitBreakerAgent:
    def __init__(self, failure_threshold: int = 5, timeout: int = 60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.circuit_open = False
        self.circuit_opened_at = None

    async def execute_with_circuit_breaker(self, tool_func: Callable):
        """Prevent cascading failures."""
        # Check circuit state
        if self.circuit_open:
            if time.time() - self.circuit_opened_at > self.timeout:
                # Try half-open state
                self.circuit_open = False
                self.failure_count = 0
            else:
                raise CircuitBreakerOpen("Circuit breaker is open")

        try:
            result = await tool_func()
            self.failure_count = 0  # Reset on success
            return result

        except Exception as e:
            self.failure_count += 1

            if self.failure_count >= self.failure_threshold:
                self.circuit_open = True
                self.circuit_opened_at = time.time()
                logger.error("Circuit breaker opened due to repeated failures")

            raise
```

**Benefits**:
- Complete control over resilience strategy
- Domain-specific error handling
- Custom recovery workflows

**Costs**:
- Must implement all patterns
- Complex testing requirements
- Ongoing maintenance

### Claude SDK: Managed Reliability

#### Automatic Error Handling
```python
# SDK handles transient failures automatically
async def query(self, message: str) -> AsyncIterator[StreamChunk]:
    """SDK manages retries and recovery."""
    try:
        await self.client.query(message)

        async for response in self.client.receive_response():
            # SDK retries transient failures internally
            yield response

    except Exception as e:
        # Only permanent failures reach application
        logger.error(f"Query failed: {e}")
        yield StreamChunk(type="error", content=str(e), completed=True)
```

#### Hook-Based Error Observation
```python
def observe_errors(tool_name: str, error: Exception) -> dict:
    """Log errors but let SDK handle recovery."""
    logger.error(f"Tool {tool_name} failed: {error}")
    metrics.increment(f"tool_error.{tool_name}")

    # SDK continues with built-in recovery
    return {"error": str(error)}

options = ClaudeAgentOptions(
    hooks={"error": observe_errors}
)
```

**Benefits**:
- Reliability out-of-box
- No retry logic to maintain
- Battle-tested patterns

**Trade-offs**:
- Can't customize retry behavior
- Limited control over recovery strategy

## Monitoring & Observability

### Deepagents: Build Custom Observability

#### Structured Logging
```python
import structlog

class ObservableAgent:
    def __init__(self):
        self.logger = structlog.get_logger()

    async def execute_tool(self, tool_name: str, args: dict):
        # Structured logging with context
        log = self.logger.bind(
            tool=tool_name,
            args=args,
            user_id=self.user_id,
            session_id=self.session_id
        )

        log.info("tool_execution_started")
        start_time = time.time()

        try:
            result = await self.run_tool(tool_name, args)

            log.info(
                "tool_execution_completed",
                duration_ms=(time.time() - start_time) * 1000,
                result_size=len(json.dumps(result))
            )

            return result

        except Exception as e:
            log.error(
                "tool_execution_failed",
                error=str(e),
                error_type=type(e).__name__,
                duration_ms=(time.time() - start_time) * 1000
            )
            raise
```

#### Metrics & Tracing
```python
from prometheus_client import Counter, Histogram, Gauge
from opentelemetry import trace

# Custom metrics
tool_calls = Counter("agent_tool_calls_total", "Total tool calls", ["tool_name", "status"])
tool_duration = Histogram("agent_tool_duration_seconds", "Tool execution time", ["tool_name"])
active_sessions = Gauge("agent_active_sessions", "Number of active sessions")

class InstrumentedAgent:
    def __init__(self):
        self.tracer = trace.get_tracer(__name__)

    async def execute_tool(self, tool_name: str, args: dict):
        """Fully instrumented tool execution."""
        # Distributed tracing
        with self.tracer.start_as_current_span("tool_execution") as span:
            span.set_attribute("tool.name", tool_name)
            span.set_attribute("tool.args", json.dumps(args))

            # Metrics
            start_time = time.time()

            try:
                result = await self.run_tool(tool_name, args)

                # Success metrics
                tool_calls.labels(tool_name=tool_name, status="success").inc()
                tool_duration.labels(tool_name=tool_name).observe(time.time() - start_time)

                return result

            except Exception as e:
                # Failure metrics
                tool_calls.labels(tool_name=tool_name, status="failure").inc()
                span.set_attribute("error", True)
                span.set_attribute("error.message", str(e))
                raise
```

**Benefits**:
- Complete observability control
- Custom metrics for domain
- Integration with any monitoring stack

**Costs**:
- Must build all instrumentation
- Maintain monitoring infrastructure
- Complex setup

### Claude SDK: Hook-Based Observability

#### Observability via Hooks
```python
# Centralized observability through hooks
import structlog

logger = structlog.get_logger()

def log_tool_use(tool_name: str, args: dict):
    """Pre-tool execution hook."""
    logger.info(
        "tool_called",
        tool=tool_name,
        args=args,
        timestamp=time.time()
    )
    return True  # Allow execution

def log_tool_result(tool_name: str, result: Any):
    """Post-tool execution hook."""
    logger.info(
        "tool_completed",
        tool=tool_name,
        result_size=len(str(result)),
        timestamp=time.time()
    )
    return result

def log_tool_error(tool_name: str, error: Exception):
    """Error hook."""
    logger.error(
        "tool_failed",
        tool=tool_name,
        error=str(error),
        error_type=type(error).__name__,
        timestamp=time.time()
    )
    return {"error": str(error)}

options = ClaudeAgentOptions(
    hooks={
        "pre_tool_use": log_tool_use,
        "post_tool_use": log_tool_result,
        "error": log_tool_error
    }
)
```

**Benefits**:
- Simple hook implementation
- SDK handles execution details
- Consistent across all agents

**Trade-offs**:
- Limited to hook interfaces
- Can't instrument SDK internals
- Less granular control

## Deployment & Scaling

### Deepagents: Deploy Like Normal Python

#### Containerization
```dockerfile
# Standard Python containerization
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy agent code
COPY . .

# Run agent service
CMD ["python", "-m", "uvicorn", "agent_api:app", "--host", "0.0.0.0", "--port", "8000"]
```

#### Horizontal Scaling
```python
# Stateless agent design enables scaling
class StatelessNetBoxAgent:
    def __init__(self, netbox_url: str, token: str):
        # No shared state between instances
        self.netbox_client = NetBoxClient(netbox_url, token)

    async def query(self, user_query: str) -> dict:
        # Each request is independent
        result = await self.process_query(user_query)
        return result

# Deploy multiple instances behind load balancer
# requests distributed across agent pool
```

**Benefits**:
- Standard Python deployment
- Easy horizontal scaling
- No special infrastructure

### Claude SDK: MCP Server Requirements

#### Deployment Architecture
```python
# Requires MCP server process management
# docker-compose.yml
version: "3.8"
services:
  agent-api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - NETBOX_URL=${NETBOX_URL}
      - NETBOX_TOKEN=${NETBOX_TOKEN}
    depends_on:
      - mcp-server

  mcp-server:
    build: ./mcp-netbox
    environment:
      - NETBOX_URL=${NETBOX_URL}
      - NETBOX_TOKEN=${NETBOX_TOKEN}
    # MCP servers communicate via stdio
```

**Considerations**:
- Manage multiple processes (agent + MCP servers)
- stdio communication requires process orchestration
- More complex deployment topology

## Maintenance & Updates

### Deepagents: Maintain Custom Code

**Monthly Maintenance Tasks**:
- Update custom orchestration logic
- Fix bugs in permission system
- Optimize custom caching strategies
- Update error handling as needed
- Keep instrumentation working
- Test resilience patterns

**Estimated Effort**: 8-12 hours/month

### Claude SDK: Leverage SDK Updates

**Monthly Maintenance Tasks**:
- Update SDK version
- Review SDK changelog
- Test with new SDK features
- Update hook implementations if needed

**Estimated Effort**: 2-4 hours/month

**SDK Benefits**:
- Security patches automatic
- Performance improvements free
- New features available
- Breaking changes documented

## Production Readiness Checklist

### Deepagents

- [ ] Permission system implemented
- [ ] Authentication & authorization
- [ ] Rate limiting
- [ ] Retry logic with backoff
- [ ] Circuit breaker pattern
- [ ] Comprehensive error handling
- [ ] Structured logging
- [ ] Metrics & monitoring
- [ ] Distributed tracing
- [ ] Health check endpoints
- [ ] Graceful shutdown
- [ ] Resource cleanup
- [ ] Unit tests (>80% coverage)
- [ ] Integration tests
- [ ] Load testing
- [ ] Security audit
- [ ] Deployment automation
- [ ] Rollback procedures
- [ ] Runbook documentation

**Timeline**: 4-6 weeks for production readiness

### Claude SDK

- [ ] Configure permission mode
- [ ] Define allowed tools whitelist
- [ ] Implement lifecycle hooks
- [ ] Add error observation
- [ ] Configure MCP servers
- [ ] Test session management
- [ ] Deploy MCP server infrastructure
- [ ] Integration tests
- [ ] Security review

**Timeline**: 1-2 weeks for production readiness

## Cost of Ownership (TCO)

### First Year Comparison

| Cost Category | Deepagents | Claude SDK |
|--------------|-----------|------------|
| **Initial Development** | 4-6 weeks | 1-2 weeks |
| **Production Hardening** | 2-4 weeks | Included |
| **Monthly Maintenance** | 8-12 hours | 2-4 hours |
| **Security Updates** | Developer responsibility | SDK managed |
| **Bug Fixes** | Developer responsibility | SDK managed |
| **Feature Updates** | Developer effort | SDK releases |
| **Team Training** | Custom architecture | Standard patterns |
| **Knowledge Transfer** | Complex | Straightforward |

### Break-Even Analysis

**Claude SDK cheaper if**:
- Standard features sufficient
- Team prefers managed solutions
- Maintenance capacity limited

**Deepagents cheaper if**:
- Significant customization needed
- Custom features justify investment
- Long-term maintenance acceptable

## Recommendations

### For Production Systems

**Start with Claude SDK** if:
- Time-to-market critical
- Team has limited LLM agent experience
- Standard capabilities sufficient
- Maintenance minimization valued

**Choose Deepagents** if:
- Novel agent behaviors required
- Custom orchestration critical
- Team has infrastructure expertise
- Flexibility justifies maintenance cost

### Migration Strategy

**Hybrid Approach**:
1. Start: Claude SDK for fast deployment
2. Identify: Limitations in SDK approach
3. Extract: Critical paths needing customization
4. Implement: Custom tools as MCP servers
5. Optional: Migrate to Deepagents if SDK too constraining

## Conclusion

Production considerations heavily favor Claude SDK for standard use cases:
- Faster time-to-production
- Lower maintenance burden
- Better security posture out-of-box
- Managed reliability

Choose Deepagents when customization needs justify the additional effort and ongoing maintenance costs.

See [Selection Guide](selection-guide.md) for final decision criteria.
