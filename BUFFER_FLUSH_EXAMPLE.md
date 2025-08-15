# Buffer Flushing Configuration Example

This document demonstrates how to configure the New Relic Java Telemetry SDK to handle high-throughput scenarios by enabling buffer flushing to prevent the warnings like:

```
WARN com.newrelic.telemetry.LimitingScheduler: Refusing to schedule batch of size 251 (would put us over max size 500000, available = 231)
```

## The Problem

When sending large volumes of telemetry data (like Kafka metrics), the SDK's internal buffer can fill up faster than it can be processed, leading to dropped data and warning messages.

## The Solution

Configure buffer flushing to automatically reduce buffer pressure when usage exceeds a configurable threshold.

## Example Configuration

### Option 1: Configure at TelemetryClient Construction

```java
import com.newrelic.telemetry.TelemetryClient;
import com.newrelic.telemetry.http.HttpPoster;
import com.newrelic.telemetry.BaseConfig;

// Create TelemetryClient with buffer flushing enabled
TelemetryClient telemetryClient = new TelemetryClient(
    metricBatchSender,
    spanBatchSender, 
    eventBatchSender,
    logBatchSender,
    3,           // shutdown seconds
    true,        // use daemon threads
    1_000_000,   // max buffer size (1M items)
    75,          // flush threshold (75% buffer usage)
    5000         // flush interval (5 seconds)
);
```

### Option 2: Configure at Runtime

```java
// Create TelemetryClient normally
TelemetryClient telemetryClient = TelemetryClient.create(httpPosterSupplier, apiKey);

// Configure buffer flushing later
telemetryClient.configureBufferFlushing(
    80,    // flush threshold (80% buffer usage)
    1000   // flush interval (1 second)
);
```

### Option 3: Manual Buffer Management

```java
// Monitor buffer usage
int usagePercent = telemetryClient.getBufferUsagePercent();
int availableCapacity = telemetryClient.getAvailableBufferCapacity();

// Manually trigger flush when needed
if (usagePercent > 85) {
    telemetryClient.flushBuffers();
}
```

## Configuration Parameters

- **flushThresholdPercent** (0-100): Buffer usage percentage that triggers automatic flushing
  - `0` = disabled
  - `75` = flush when buffer is 75% full
  - `90` = flush when buffer is 90% full (more aggressive)

- **flushIntervalMs**: How often to check buffer usage (in milliseconds)
  - `0` = disabled
  - `1000` = check every 1 second
  - `5000` = check every 5 seconds

## Recommended Settings for High-Throughput Applications

For applications sending large volumes of Kafka metrics or other high-frequency telemetry:

```java
TelemetryClient telemetryClient = new TelemetryClient(
    metricBatchSender, spanBatchSender, eventBatchSender, logBatchSender,
    3, true, 1_000_000,
    70,    // Flush at 70% to give plenty of headroom
    2000   // Check every 2 seconds for responsive flushing
);
```

## Benefits

1. **Prevents Data Loss**: Automatic flushing reduces buffer pressure before limits are reached
2. **Configurable**: Tune threshold and interval based on your application's throughput
3. **Non-blocking**: Flush checking runs on a separate daemon thread
4. **Backward Compatible**: Existing code continues to work without changes
5. **Observable**: Monitor buffer usage to understand SDK performance

## Monitoring

Use the monitoring methods to track buffer health:

```java
// Log buffer status periodically
logger.info("Buffer usage: {}%, Available capacity: {}", 
    telemetryClient.getBufferUsagePercent(),
    telemetryClient.getAvailableBufferCapacity());
```

This configuration should eliminate the buffer overflow warnings while maintaining optimal performance for high-throughput telemetry scenarios.