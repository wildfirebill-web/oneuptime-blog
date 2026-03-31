# How to Test Dapr Binding Triggers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Binding, Testing, Input Binding, Test Automation

Description: Learn how to test Dapr input and output binding handlers by calling the binding endpoints directly and mocking the DaprClient for output binding calls.

---

## What Are Dapr Bindings?

Dapr bindings connect your service to external systems. Input bindings trigger your service when an event arrives (a file upload, an IoT sensor reading, a timer). Output bindings let your service invoke external systems (send an email, write to a queue). Testing them means exercising the handler endpoints directly.

## Input Binding Handler

```csharp
// SensorController.cs
[ApiController]
public class SensorController : ControllerBase
{
    private readonly ISensorProcessor _processor;
    private readonly DaprClient _daprClient;

    public SensorController(ISensorProcessor processor, DaprClient daprClient)
    {
        _processor = processor;
        _daprClient = daprClient;
    }

    // Dapr calls this endpoint when a sensor reading arrives
    [HttpPost("/sensor-data")]
    public async Task<IActionResult> HandleSensorData([FromBody] SensorReading reading)
    {
        if (reading.Temperature > 90)
        {
            // Output binding: send an alert email
            await _daprClient.InvokeBindingAsync(
                bindingName: "email",
                operation: "create",
                data: new
                {
                    emailFrom    = "alerts@company.com",
                    emailTo      = "ops@company.com",
                    subject      = $"High temperature alert: {reading.SensorId}",
                    body         = $"Temperature: {reading.Temperature}C at {reading.Timestamp}"
                });
        }

        await _processor.SaveReadingAsync(reading);
        return Ok();
    }
}
```

## Unit Tests for the Input Binding Handler

```csharp
// SensorControllerTests.cs
public class SensorControllerTests
{
    private readonly Mock<ISensorProcessor> _mockProcessor;
    private readonly Mock<DaprClient> _mockDapr;
    private readonly SensorController _controller;

    public SensorControllerTests()
    {
        _mockProcessor = new Mock<ISensorProcessor>();
        _mockDapr      = new Mock<DaprClient>();
        _controller    = new SensorController(_mockProcessor.Object, _mockDapr.Object);
    }

    [Fact]
    public async Task HandleSensorData_SavesAllReadings()
    {
        var reading = new SensorReading
        {
            SensorId    = "sensor-1",
            Temperature = 25.0,
            Timestamp   = DateTime.UtcNow
        };

        _mockProcessor
            .Setup(p => p.SaveReadingAsync(reading))
            .Returns(Task.CompletedTask);

        var result = await _controller.HandleSensorData(reading);

        Assert.IsType<OkResult>(result);
        _mockProcessor.Verify(p => p.SaveReadingAsync(reading), Times.Once);
        _mockDapr.Verify(d => d.InvokeBindingAsync(
            It.IsAny<string>(), It.IsAny<string>(),
            It.IsAny<object>(), It.IsAny<IReadOnlyDictionary<string, string>>(),
            It.IsAny<CancellationToken>()), Times.Never);
    }

    [Fact]
    public async Task HandleSensorData_SendsAlertWhenTemperatureHigh()
    {
        var reading = new SensorReading
        {
            SensorId    = "sensor-2",
            Temperature = 95.0,
            Timestamp   = DateTime.UtcNow
        };

        _mockDapr
            .Setup(d => d.InvokeBindingAsync(
                "email", "create",
                It.IsAny<object>(),
                It.IsAny<IReadOnlyDictionary<string, string>>(),
                It.IsAny<CancellationToken>()))
            .Returns(Task.CompletedTask);

        _mockProcessor
            .Setup(p => p.SaveReadingAsync(reading))
            .Returns(Task.CompletedTask);

        await _controller.HandleSensorData(reading);

        _mockDapr.Verify(d => d.InvokeBindingAsync(
            "email", "create",
            It.IsAny<object>(),
            It.IsAny<IReadOnlyDictionary<string, string>>(),
            It.IsAny<CancellationToken>()), Times.Once);
    }
}
```

## Integration Test: Simulating a Binding Trigger

In integration tests, simulate an input binding by POSTing directly to the handler endpoint:

```bash
curl -X POST http://localhost:5000/sensor-data \
  -H "Content-Type: application/json" \
  -d '{
    "sensorId": "sensor-001",
    "temperature": 92.5,
    "timestamp": "2026-03-31T10:00:00Z"
  }'
```

Or from a test:

```csharp
[Fact]
[Trait("Category", "Integration")]
public async Task HighTemperatureReading_TriggersAlert()
{
    var response = await _client.PostAsJsonAsync("/sensor-data", new
    {
        sensorId    = "integration-sensor-1",
        temperature = 95.0,
        timestamp   = DateTime.UtcNow
    });

    Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    // Verify alert was queued via state store or log check
}
```

## Output Binding Component for Testing

```yaml
# components/test/email-binding.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: email
spec:
  type: bindings.localstorage
  version: v1
  metadata:
  - name: rootPath
    value: "/tmp/test-emails"
```

## Summary

Testing Dapr binding handlers is straightforward because they are plain HTTP endpoints. Unit tests call the handler method directly with mocked dependencies and verify output binding calls on `DaprClient`. Integration tests POST directly to the handler URL to simulate what Dapr would send when an input binding triggers. Swap the production email/queue binding for a local file binding in test components.
