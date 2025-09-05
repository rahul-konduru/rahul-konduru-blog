+++
date = '2025-09-04T15:06:01-05:00'
draft = true
title = 'Writing a Custom Kafka Health Indicator in Spring Boot'
tags = ["Spring Boot", "Kafka", "Actuator", "Health Check", "Kubernetes"]
summary = "Learn how to create a custom Kafka health indicator in Spring Boot to improve microservice resilience. Enhance Kubernetes monitoring by verifying both Kafka producer and consumer functionality for accurate health checks."
slug = 'custom-kafka-health-indicator-spring-boot'
description = "Discover how to build a custom Kafka health indicator in Spring Boot to strengthen microservice resilience and enhance Kubernetes liveness/readiness probe accuracy. This guide covers the rationale, implementation, configuration, and best practices for production-grade health checks."
keywords = ["spring boot kafka health", "kafka custom health indicator", "kubernetes health probes", "spring actuator kafka", "advanced spring boot monitoring"]
author = "Rahul Konduru"
+++

In building resilient microservices, **health monitoring** plays a critical role in ensuring production readiness. **Spring Boot Actuator** provides powerful health endpoints that integrate seamlessly with orchestration platforms such as Kubernetes, enabling applications to be monitored and managed effectively.

However, a limitation exists: **Spring Boot Actuator does not include a built-in Kafka health indicator**, unlike the ready-made checks for databases, Redis, and other dependencies. Fortunately, **Spring Boot** allows developers to define custom health indicators for critical dependencies such as **Kafka**.

---

## Why Do We Need Health Indicators?

Kubernetes relies on *probes* to evaluate the health status of applications running in pods. These probes interact with Spring Boot Actuator health endpoints, allowing orchestrators to:

- Validate **Startup** using the Startup Probe  
- Assess **Liveness** using the Liveness Probe  
- Determine **Readiness** using the Readiness Probe 

Without accurate health indicators for critical dependencies such as **Kafka**, Kubernetes may route traffic to applications that cannot process messages. This can result in data loss, failed message processing, or service downtime. A robust Kafka health check ensures that microservices are genuinely *ready* and *operational* from an end-to-end perspective.

---

## Existing Approaches  

Many community implementations use **KafkaAdmin** to verify connectivity to a Kafka broker. This approach attempts broker-level operations to confirm basic Kafka connectivity.  

However, broker connectivity alone does not guarantee that both the **producer** and **consumer** sides of your application can function correctly. That’s why we’ll take a different approach — extending health indicators to verify **producer** and **consumer** workflows.

---

## A Custom Kafka Health Indicator  

Depending on your use case, you may implement either or both. For example:  
- A service responsible only for publishing messages may implement a **producer health check**.  
- A service that primarily processes Kafka messages would benefit more from a **consumer health check**.  
- In some cases, you may want to combine both for stronger validation.  

---

## Coding the Health Indicator

Spring Boot provides a streamlined mechanism for defining custom health indicators by either implementing the [`HealthIndicator`](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/actuate/health/HealthIndicator.html) interface or extending the [`AbstractHealthIndicator`](https://docs.spring.io/spring-boot/api/java/org/springframework/boot/actuate/health/AbstractHealthIndicator.html) base class. For enhanced simplicity and consistency, it is recommended to extend `AbstractHealthIndicator`, as this abstract class encapsulates the construction of the `Health` response and offers built-in exception handling. This approach enables developers to focus exclusively on implementing the specific health check logic relevant to their component or subsystem.

### Sample Implementation Structure

Create a class named `KafkaHealthIndicator` that extends `AbstractHealthIndicator` and annotate it with `@Component` so that it is registered as a Spring bean:

```java
@Component
public class KafkaHealthIndicator extends AbstractHealthIndicator {

    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        // Kafka health check logic
    }
}
```

#### Consumer Health Verification

Inject `KafkaListenerEndpointRegistry` and check the `connection-count` metric for any one listener to validate consumer connections:

```java
@Autowired
private KafkaListenerEndpointRegistry kafkaListenerEndpointRegistry;

private boolean isConsumerHealthy() {
    Metric metric = kafkaListenerEndpointRegistry.getListenerContainers().stream()
            .flatMap(container -> container.metrics().values().stream())
            .flatMap(metricsMap -> metricsMap.entrySet().stream())
            .filter(entry -> "connection-count".equalsIgnoreCase(entry.getKey().name()))
            .map(Map.Entry::getValue)
            .findFirst()
            .orElseThrow();
    return (Double)metric.metricValue() != 0;
}
```

**Explanation:**
The `KafkaListenerEndpointRegistry` tracks consumer metrics across all listeners. The `connection-count` metric represents the number of active broker connections. A non-zero value confirms that the application’s consumers are live.

#### Producer Health Verification

Similarly, use the `KafkaTemplate` bean to check the producer’s connection count:

```java
@Autowired
private KafkaTemplate<String, String> kafkaTemplate;

private boolean isProducerHealthy() {
    Metric metric = kafkaTemplate.metrics().entrySet().stream()
            .filter(es -> "connection-count".equalsIgnoreCase(es.getKey().name()))
            .map(Map.Entry::getValue)
            .findFirst()
            .orElseThrow();
    return (Double)metric.metricValue() != 0;
}
```

**Explanation:**
By retrieving the producer’s `connection-count` metric, the health indicator verifies that outbound Kafka connections are active and capable of sending messages.

---

### Final Health Indicator Example

The composite check integrates both producer and consumer validation:

```java
@Component
public class KafkaHealthIndicator extends AbstractHealthIndicator {

    @Autowired
    private KafkaListenerEndpointRegistry kafkaListenerEndpointRegistry;

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        if (isConsumerHealthy() && isProducerHealthy()) {
            builder.up();
        } else {
            builder.down();
        }
    }

    private boolean isConsumerHealthy() {
        // ...implementation as above
    }

    private boolean isProducerHealthy() {
        // ...implementation as above
    }
}
```

---

## Configuration: Enabling Health Groups

By default, custom indicators are **not included** in liveness/readiness probes. To configure them, update `application.yml`:

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
      group:
        liveness:
          include: kafka
        readiness:
          include: kafka
```

**Note:**
Spring automatically generates the indicator name by removing `HealthIndicator` from the class name—thus the indicator is available as `kafka`.

For enhanced diagnostics, enable detailed output:

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
      show-details: always
```

This exposes component-level statuses in the health endpoint:

```json
{"components":{"kafka":{"status":"UP"}},"status":"UP"}
```

---

## Wrapping Up  

Spring Boot’s extensibility lets us go beyond the built-in health checks and create **Kafka-specific health indicators**. These improve the reliability of your service by allowing Kubernetes (or any orchestrator) to make informed decisions about your application’s state.  

By verifying both Kafka producer and consumer functionality, your health endpoints more accurately represent the application’s readiness to handle traffic.  

When deploying a Spring Boot service that depends on Kafka, avoid relying solely on broker connectivity checks; instead, implement a robust custom health indicator.  

**Let’s get coding! 🚀**
