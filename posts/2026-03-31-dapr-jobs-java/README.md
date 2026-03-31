# How to Use Dapr Jobs with Java SDK

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Job, Java, Scheduler, Microservice

Description: Learn how to schedule and manage one-time and recurring jobs in Java using the Dapr Jobs API for reliable task execution in distributed systems.

---

## Introduction

Dapr Jobs provides a durable job scheduling API that lets you schedule work to run at a specific time or on a recurring schedule. The Java SDK makes it straightforward to create and manage jobs from your application code.

## Adding the Dependency

Add the Dapr Java SDK to your `pom.xml`:

```xml
<dependency>
  <groupId>io.dapr</groupId>
  <artifactId>dapr-sdk</artifactId>
  <version>1.13.0</version>
</dependency>
```

## Creating a DaprClient

```java
import io.dapr.client.DaprClient;
import io.dapr.client.DaprClientBuilder;

DaprClient client = new DaprClientBuilder().build();
```

## Scheduling a One-Time Job

Schedule a job to run once at a specific time:

```java
import io.dapr.client.domain.ScheduleJobRequest;
import java.time.OffsetDateTime;

ScheduleJobRequest request = ScheduleJobRequest.newBuilder()
    .setName("send-report")
    .setData("monthly-report".getBytes())
    .setSchedule("@every 1m")
    .build();

client.scheduleJob(request).block();
System.out.println("Job scheduled successfully");
```

## Scheduling a Recurring Job

Use cron expressions for recurring jobs:

```java
ScheduleJobRequest recurringRequest = ScheduleJobRequest.newBuilder()
    .setName("cleanup-job")
    .setData("cleanup-old-records".getBytes())
    .setSchedule("0 2 * * *")  // Daily at 2 AM
    .setRepeats(0)              // 0 means unlimited repeats
    .build();

client.scheduleJob(recurringRequest).block();
```

## Implementing the Job Handler

Register an HTTP endpoint to handle job triggers:

```java
import org.springframework.web.bind.annotation.*;

@RestController
public class JobController {

    @PostMapping("/job/send-report")
    public ResponseEntity<Void> handleSendReport(@RequestBody byte[] data) {
        String jobData = new String(data);
        System.out.println("Executing job with data: " + jobData);
        // perform job work
        return ResponseEntity.ok().build();
    }

    @PostMapping("/job/cleanup-job")
    public ResponseEntity<Void> handleCleanup(@RequestBody byte[] data) {
        // cleanup logic
        return ResponseEntity.ok().build();
    }
}
```

## Getting Job Details

Retrieve information about a scheduled job:

```java
import io.dapr.client.domain.GetJobResponse;

GetJobResponse job = client.getJob("send-report").block();
System.out.println("Job name: " + job.getName());
System.out.println("Job schedule: " + job.getSchedule());
```

## Deleting a Job

Cancel a job when it is no longer needed:

```java
client.deleteJob("send-report").block();
System.out.println("Job deleted");
```

## Dapr Component Configuration

Ensure your Dapr sidecar has the scheduler service enabled in your component YAML:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: myconfig
spec:
  features:
    - name: SchedulerReminders
      enabled: true
```

## Summary

Dapr Jobs in the Java SDK provides a simple API to schedule one-time and recurring tasks with durability and fault tolerance built in. The scheduler service handles persistence so jobs survive restarts, and the handler pattern keeps your job logic cleanly separated from scheduling concerns.
