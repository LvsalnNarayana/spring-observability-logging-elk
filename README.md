
# Spring-Observability-Logging-Elk

## Overview

This project is a **multi-microservice demonstration** of **centralized observability through structured logging** using the **ELK Stack** (Elasticsearch, Logstash, Kibana) in Spring Boot 3.x.

It focuses on best practices for logging in distributed systems:
- Structured JSON logs
- Correlation ID propagation across services
- Meaningful log levels and contextual information
- Centralized collection and visualization
- Ready for alerting and advanced searching

The setup simulates enterprise-grade logging without requiring a full production-scale environment.

## Real-World Scenario (Simulated)

In enterprise platforms like **Salesforce**:
- Thousands of microservices generate logs across distributed calls.
- Tracing a user request across services is impossible with plain text logs.
- Errors must be detected quickly via alerts.
- Compliance and debugging require powerful search and dashboards.

We simulate this with a simplified booking flow (Booking → Payment → Notification) where a single user request spans multiple services, and logs are correlated for end-to-end visibility.

## Microservices Involved

| Service                  | Responsibility                                                                 | Port  |
|--------------------------|--------------------------------------------------------------------------------|-------|
| **elasticsearch**       | Storage and search engine                                                      | 9200  |
| **logstash**             | Log ingestion and parsing                                                      | 5044  |
| **kibana**               | Visualization and dashboards                                                   | 5601  |
| **eureka-server**        | Service discovery (Netflix Eureka)                                             | 8761  |
| **booking-service**      | Initiates booking, starts correlation ID, calls downstream                     | 8081  |
| **payment-service**      | Processes payment, propagates correlation ID                                   | 8082  |
| **notification-service** | Sends confirmation, logs final outcome                                         | 8083  |

## Tech Stack

- Spring Boot 3.x
- Logback with Logstash Encoder (JSON structured logging)
- Spring Cloud Sleuth (correlation ID generation – compatible alternative)
- Spring Cloud OpenFeign (inter-service calls with header propagation)
- ELK Stack (Elasticsearch, Logstash, Kibana) via Docker
- Spring Cloud Netflix Eureka
- Micrometer + Actuator (basic health)
- Lombok
- Maven (multi-module)
- Docker & Docker Compose

## Docker Containers

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    ports:
      - "9200:9200"

  logstash:
    image: docker.elastic.co/logstash/logstash:8.12.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
    ports:
      - "5044:5044"
      - "9600:9600"
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  eureka-server:
    build: ./eureka-server
    ports:
      - "8761:8761"

  booking-service:
    build: ./booking-service
    depends_on:
      - eureka-server
    ports:
      - "8081:8081"

  payment-service:
    build: ./payment-service
    depends_on:
      - eureka-server
    ports:
      - "8082:8082"

  notification-service:
    build: ./notification-service
    depends_on:
      - eureka-server
    ports:
      - "8083:8083"
```

Run with: `docker-compose up --build`

**Note**: Logstash config (`logstash/pipeline/logstash.conf`) is provided to accept Beats input on 5044 and forward JSON to Elasticsearch.

## Logging Strategy

| Feature                     | Implementation Details                                                  |
|-----------------------------|-------------------------------------------------------------------------|
| **Structured JSON Logs**    | Logstash Logback Encoder → fields: timestamp, level, thread, logger, message, traceId, spanId, service |
| **Correlation ID**          | Sleuth generates `traceId`/`spanId` → automatically added to MDC and propagated via Feign |
| **Log Levels**              | ERROR for exceptions, WARN for retries, INFO for business events, DEBUG for details |
| **Contextual Info**         | Booking ID, User ID, Payment status added via MDC                       |
| **Central Collection**      | Filebeat/Logstash TCP input → Elasticsearch → Kibana                    |

## Key Features

- End-to-end trace ID propagation across services
- Fully structured, searchable JSON logs
- Pre-configured Kibana dashboards (sample JSON provided)
- Correlation ID visible in every log entry
- Error logs with stack traces in JSON format
- Easy filtering in Kibana by traceId, service, level
- Ready for alerting rules in Kibana/Elastic

## Expected Endpoints

### Booking Service (`http://localhost:8081`)

| Method | Endpoint                  | Description                                      |
|--------|---------------------------|--------------------------------------------------|
| POST   | `/api/bookings`           | Create booking → calls payment → notification    |
| GET    | `/api/bookings/{id}`      | Get booking with traceId in response headers      |

### Payment Service (`http://localhost:8082`)

- Called internally via Feign
- Logs payment attempt with propagated traceId

### Notification Service (`http://localhost:8083`)

- Logs final success/failure with same traceId

### Kibana
- URL: `http://localhost:5601`
- Index pattern: `logstash-*`
- Search by `traceId` to see full request journey

## Architecture Overview

```
Client
   ↓
Booking Service → generates traceId
   ↓ (Feign + Sleuth header propagation)
Payment Service → same traceId in logs
   ↓
Notification Service → same traceId
   ↓
JSON Logs → Filebeat/Logstash → Elasticsearch → Kibana
```

**Log Flow**:
1. Request hits booking-service → Sleuth creates traceId
2. Feign calls propagate `X-B3-TraceId`
3. All services log with MDC → JSON includes traceId
4. Logs shipped to ELK → searchable by traceId

## How to Run

1. Clone repository
2. Start Docker: `docker-compose up --build`
3. Wait for ELK to be healthy (~2-3 minutes)
4. Create booking: POST `/api/bookings` (with JSON body)
5. Open Kibana: `http://localhost:5601`
6. Create index pattern `logstash-*`
7. Search `traceId:` → paste from response header or logs
8. See full distributed trace in logs

## Testing Observability

1. Create a booking → success path
2. Check logs in services → same traceId everywhere
3. Simulate error in payment → ERROR logs with exception
4. In Kibana: filter by traceId → see entire flow
5. Filter by level:ERROR → quick error detection

## Skills Demonstrated

- Structured JSON logging with Logstash Encoder
- Distributed tracing basics with Sleuth (traceId propagation)
- ELK stack integration in local development
- Correlation across microservices without full Zipkin
- Best practices for log context and levels
- Setting up searchable, centralized logging
- Preparing logs for alerting and dashboards

## Future Extensions

- Full distributed tracing with Zipkin/Jaeger
- Metric collection with Prometheus + Grafana
- Alerting rules in Kibana
- Log masking for PII
- Integration with OpenTelemetry

