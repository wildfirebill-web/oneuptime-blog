# How to Use Actors for IoT Device Management in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, IoT, Device Twin, Telemetry

Description: Model IoT devices as Dapr actors to maintain device twin state, process telemetry, send commands, and track device connectivity at scale.

---

IoT device management requires per-device state tracking, command dispatching, and telemetry ingestion at massive scale. Dapr's virtual actor model maps naturally to the device twin pattern, where each actor represents one physical device.

## Device Twin Actor Design

```
Actor Type: DeviceTwin
Actor ID:   <deviceId>
```

Each actor maintains the device's reported state, desired state, and connectivity status.

## Device Twin Actor Implementation

```go
package main

import (
  "context"
  "time"
  "github.com/dapr/go-sdk/actor"
)

type DeviceTwinState struct {
  DeviceID      string                 `json:"deviceId"`
  ReportedState map[string]interface{} `json:"reportedState"`
  DesiredState  map[string]interface{} `json:"desiredState"`
  IsOnline      bool                   `json:"isOnline"`
  LastSeen      time.Time              `json:"lastSeen"`
  FirmwareVer   string                 `json:"firmwareVersion"`
}

type DeviceTwinActor struct {
  actor.ServerImplBase
}

func (a *DeviceTwinActor) Type() string { return "DeviceTwin" }

func (a *DeviceTwinActor) ReportTelemetry(ctx context.Context, telemetry map[string]interface{}) error {
  var twin DeviceTwinState
  a.GetStateManager().Get(ctx, "twin", &twin)

  if twin.ReportedState == nil {
    twin.ReportedState = make(map[string]interface{})
  }
  for k, v := range telemetry {
    twin.ReportedState[k] = v
  }
  twin.IsOnline = true
  twin.LastSeen = time.Now().UTC()

  return a.GetStateManager().Set(ctx, "twin", twin)
}

func (a *DeviceTwinActor) SetDesiredState(ctx context.Context, desired map[string]interface{}) error {
  var twin DeviceTwinState
  a.GetStateManager().Get(ctx, "twin", &twin)

  if twin.DesiredState == nil {
    twin.DesiredState = make(map[string]interface{})
  }
  for k, v := range desired {
    twin.DesiredState[k] = v
  }

  return a.GetStateManager().Set(ctx, "twin", twin)
}

func (a *DeviceTwinActor) GetTwin(ctx context.Context) (*DeviceTwinState, error) {
  var twin DeviceTwinState
  if err := a.GetStateManager().Get(ctx, "twin", &twin); err != nil {
    return &DeviceTwinState{DeviceID: a.ID()}, nil
  }
  return &twin, nil
}
```

## Telemetry Ingestion at Scale

A telemetry ingestion service publishes device data to a Dapr pub/sub topic. A subscriber routes it to the correct device actor:

```go
func handleTelemetry(w http.ResponseWriter, r *http.Request) {
  var msg TelemetryMessage
  json.NewDecoder(r.Body).Decode(&msg)

  client.InvokeActor(r.Context(), &dapr.InvokeActorRequest{
    ActorType: "DeviceTwin",
    ActorID:   msg.DeviceID,
    Method:    "ReportTelemetry",
    Data:      mustMarshal(msg.Readings),
  })
}
```

## Sending a Command to a Device

```bash
curl -X POST http://localhost:3500/v1.0/actors/DeviceTwin/device-001/method/SetDesiredState \
  -H "Content-Type: application/json" \
  -d '{
    "targetTemperature": 22.5,
    "fanSpeed": "medium"
  }'
```

## Offline Detection with Reminders

```go
func (a *DeviceTwinActor) OnActivate() error {
  // Check for offline status every 5 minutes
  return a.AddReminder("connectivity-check", nil, 5*time.Minute, 5*time.Minute)
}

func (a *DeviceTwinActor) ReminderCall(ctx context.Context, name string, data []byte) error {
  if name == "connectivity-check" {
    var twin DeviceTwinState
    a.GetStateManager().Get(ctx, "twin", &twin)
    if time.Since(twin.LastSeen) > 10*time.Minute {
      twin.IsOnline = false
      a.GetStateManager().Set(ctx, "twin", twin)
      alertOfflineDevice(twin.DeviceID)
    }
  }
  return nil
}
```

## Summary

Dapr actors implement the device twin pattern cleanly: one actor per device maintains reported and desired state with automatic persistence and turn-based concurrency. Actor reminders handle periodic connectivity checks without external schedulers. This architecture supports millions of concurrent device actors distributed across a cluster without any custom distribution logic.
