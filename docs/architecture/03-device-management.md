# Device Management

## Device Management

This section documents how the platform manages IoT devices — command delivery via Device Shadow, certificate-based authentication via Fleet Provisioning, per-device authorization via IoT policy variables, and fleet organization via Thing Types and Thing Groups.

---

### Device Shadow — Command Delivery to Disconnected Devices

Devices in this architecture connect hourly for data push, meaning they are offline most of the time. AWS IoT Core Device Shadow provides a persistent cloud-side state document for each device, containing `desired` (what the cloud wants), `reported` (what the device confirmed), and a computed `delta` (the difference). When an operator sends a command, the cloud writes to `desired`; when the device reconnects, it receives the `delta` containing only changed fields, applies them, and updates `reported` to confirm. The shadow reconciles automatically when `desired` equals `reported`.

#### Key Concepts

| Concept | Description |
|---|---|
| `desired` state | Set by cloud (Lambda) — represents the target configuration or command |
| `reported` state | Set by device — represents the actual device configuration after applying commands |
| `delta` state | Computed by IoT Core — contains fields where `desired` differs from `reported` |
| `version` | Monotonically increasing integer — used for optimistic locking and idempotency |
| Named shadow | Each device can have multiple named shadows for different concerns (e.g., `config`, `firmware`) |

#### Command Delivery Sequence Diagram

The following diagram shows the complete lifecycle of a command from operator action to device confirmation. This flow handles the device being offline at the time of command issuance — the shadow persists the desired state until the device reconnects.

```mermaid
sequenceDiagram
    participant WebUI as Web UI (Operator)
    participant APIGW as API Gateway
    participant LambdaCmd as Lambda (Command Handler)
    participant Shadow as IoT Core Shadow
    participant Dev as IoT Device (hourly connect)
    participant Audit as DynamoDB (Audit)

    WebUI->>APIGW: POST /devices/{thingName}/commands<br/>{command: "setSampleRate", value: 300}
    APIGW->>LambdaCmd: invoke with command payload
    activate LambdaCmd
    LambdaCmd->>Shadow: UpdateThingShadow<br/>{desired: {sampleRate: 300}}
    LambdaCmd->>Audit: PutItem {commandId, thingName, status: PENDING}
    Shadow-->>LambdaCmd: accepted {version: 15, state: {desired: {sampleRate: 300}}}
    deactivate LambdaCmd
    APIGW-->>WebUI: 202 Accepted {commandId}

    rect rgb(240, 240, 240)
        Note over Shadow,Dev: Device is OFFLINE — desired state persisted in Shadow (version 15)
    end

    Dev->>Shadow: CONNECT (MQTT TLS 8883, per-device X.509 cert)
    Dev->>Shadow: SUBSCRIBE $aws/things/{thingName}/shadow/update/delta
    Shadow-->>Dev: delta {state: {sampleRate: 300}, version: 15}

    Note over Dev: version 15 > last applied (12) — apply command (idempotency guard)

    activate Dev
    Dev->>Dev: Apply sampleRate = 300
    deactivate Dev

    Dev->>Shadow: PUBLISH $aws/things/{thingName}/shadow/update<br/>{reported: {sampleRate: 300}, version: 15}
    Shadow-->>Dev: shadow/update/accepted (delta cleared)
    Shadow->>LambdaCmd: IoT Rule triggers on shadow update
    LambdaCmd->>Audit: UpdateItem {commandId, status: DELIVERED}
```

#### Version-Based Idempotency

Every shadow update increments the version number, enabling the device to detect and discard duplicate commands:

- Every shadow update increments the `version` counter atomically in IoT Core.
- The device stores the last successfully applied version in local persistent storage.
- When a delta arrives, the device compares the received `version` to its last applied version.
- If received `version` <= last applied, the delta is discarded — this prevents double-execution from retransmissions or MQTT QoS 1 duplicates.
- If the device sends an update with a stale version, IoT Core returns **HTTP 409 Conflict** — the device must fetch the latest shadow and retry.

This mechanism ensures exactly-once execution semantics for commands even when the device receives the same delta multiple times due to network retries.

#### Device Shadow vs Alternatives

| Approach | Offline Support | Complexity | AWS Integration | Recommendation |
|---|---|---|---|---|
| Device Shadow (desired/reported/delta) | Built-in — persists while device is offline | Low — managed by IoT Core | Native — Rules Engine, Lambda triggers | **Recommended** |
| SQS queue per device | Yes — messages retained 14 days | High — custom polling logic, per-device queues | Manual — must poll from device or proxy | Not recommended for this use case |
| DynamoDB polling | Yes — records persist indefinitely | High — custom poll + mark-as-read logic | Manual — device or proxy must query | Not recommended — polling waste |
