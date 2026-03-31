# How to Use Dapr Service Invocation Between .NET and Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, .NET, Python, Service Invocation, Microservice

Description: Learn how to invoke a Python Flask service from a .NET application using Dapr service invocation with the .NET SDK and Python HTTP endpoints.

---

Dapr service invocation lets a .NET service call a Python service without knowing its network address. The .NET application calls the Dapr sidecar, which resolves and forwards the request to the Python service's sidecar.

## Python Flask Service (Target)

Create a simple Python Flask service that handles incoming Dapr invocations:

```python
# app.py
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/calculate', methods=['POST'])
def calculate():
    data = request.json
    result = data.get('a', 0) + data.get('b', 0)
    return jsonify({'result': result})

@app.route('/health', methods=['GET'])
def health():
    return jsonify({'status': 'healthy'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

Kubernetes deployment annotation for the Python service:

```yaml
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "calculator-python"
  dapr.io/app-port: "5000"
```

## .NET Service (Caller)

Use the Dapr .NET SDK to invoke the Python service:

```bash
dotnet add package Dapr.Client
```

```csharp
using Dapr.Client;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDaprClient();

var app = builder.Build();

app.MapPost("/compute", async (DaprClient daprClient, ComputeRequest request) =>
{
    var response = await daprClient.InvokeMethodAsync<CalculateRequest, CalculateResponse>(
        HttpMethod.Post,
        "calculator-python",  // Dapr app-id of Python service
        "calculate",          // Method name maps to URL path
        new CalculateRequest { A = request.ValueA, B = request.ValueB }
    );

    return Results.Ok(new { Sum = response.Result });
});

app.Run();

record ComputeRequest(double ValueA, double ValueB);
record CalculateRequest { public double A { get; set; } public double B { get; set; } }
record CalculateResponse { public double Result { get; set; } }
```

## Kubernetes Deployment for .NET Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: compute-dotnet
spec:
  replicas: 1
  selector:
    matchLabels:
      app: compute-dotnet
  template:
    metadata:
      labels:
        app: compute-dotnet
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "compute-dotnet"
        dapr.io/app-port: "8080"
    spec:
      containers:
      - name: compute-dotnet
        image: myrepo/compute-dotnet:latest
        ports:
        - containerPort: 8080
```

## Testing the Cross-Language Call

Deploy both services and test from within the cluster:

```bash
# Port-forward the .NET service
kubectl port-forward deployment/compute-dotnet 8080:8080

# Call the .NET service which invokes Python via Dapr
curl -X POST http://localhost:8080/compute \
  -H "Content-Type: application/json" \
  -d '{"valueA": 10, "valueB": 25}'

# Expected response: {"sum": 35}
```

## Error Handling Across Languages

Handle Dapr invocation errors in .NET:

```csharp
try
{
    var response = await daprClient.InvokeMethodAsync<CalculateRequest, CalculateResponse>(
        HttpMethod.Post, "calculator-python", "calculate", request
    );
    return Results.Ok(response);
}
catch (InvocationException ex) when (ex.Response.StatusCode == HttpStatusCode.NotFound)
{
    return Results.NotFound("Python calculator service not found");
}
catch (InvocationException ex)
{
    return Results.Problem($"Python service error: {ex.Message}");
}
```

## Summary

Dapr service invocation between .NET and Python requires no shared libraries or protocol negotiation. The .NET SDK's `InvokeMethodAsync` serializes the request and sends it to the Dapr sidecar, which routes it to the Python Flask service's sidecar. Error handling uses standard HTTP status codes that both languages understand.
