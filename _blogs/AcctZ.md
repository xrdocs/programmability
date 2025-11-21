---
published: true
date: '2025-10-02 13:35 -0400'
title: AcctZ
author: Rahul Sharma
position: hidden
---

# Closing the Accounting Gap in gRPC Services with gNSI AcctZ

## 1. Introduction

Modern service providers rely on multiple interfaces to configure, monitor, and operate network devices. Traditionally, Cisco IOS XR has offered AAA (Authentication, Authorization, and Accounting) support for CLI and NETCONF operations. In these environments, executed commands or RPCs are logged in syslog, TACACS+, or other external accounting systems.

However, with the rise of gRPC-based services—such as gNMI, gNSI, gNOI, gRIBI, and P4RT—this same level of accounting visibility has not been available. This created a significant gap for operators who need unified logging, auditing, and compliance across all management interfaces.

The **gNSI AcctZ** specification, driven by the **OpenConfig** community, fills this gap by introducing a consistent accounting framework for gRPC services.

This blog explores what AcctZ is, how it works, and why it matters for operators running IOS XR in mission-critical environments.

---

## 2. What is gNSI AcctZ?

**gNSI AcctZ (Accounting Service)** defines a standardized method to stream accounting records from a network device to a remote collection system. It is built on **gRPC transport** and integrates seamlessly with existing **gNxI methods** and **YANG models** for configuration.

The service generates accounting records whenever:

- CLI or shell commands are executed  
- NETCONF RPCs are run  
- gRPC-based operations (e.g., `gNMI Set`, `gNOI OS Install`, `gNSI Cert Rotate`) are invoked  

Records are temporarily stored on the device in a **circular buffer** before being streamed to a remote collector. This ensures resilience—if the collector is temporarily unavailable, data is retained until space runs out, at which point older entries are overwritten.

---

## 3. How Does It Work?

At a high level, the workflow looks like this:

### Remote Collector Connection
- A collector connects to the router’s AcctZ service using the `RecordSubscribe` RPC.

### Timestamp-Based Retrieval
- The collector specifies the last record it received, using a timestamp in **nanoseconds since epoch**.
  - If set to `0`, the router streams **all stored records**.
  - If set to a specific time, only **new records after that time** are sent.

### Continuous Streaming
- Once subscribed, the router **continuously streams new accounting events** until the session ends or an error occurs.

### Historical Replay
- Devices can maintain a **configurable depth of history**, so both new and reconnecting collectors can retrieve past records.

---

## 4. What’s Inside an Accounting Record?

The AcctZ specification defines a set of **protobuf messages** that represent accounting data. Let’s walk through the main ones.

```
RecordSubscribe(RecordRequest) → stream RecordResponse
│
├── RecordRequest
│   └── timestamp : google.protobuf.Timestamp
│       - Nanoseconds since Unix epoch (Jan 1, 1970 UTC).
│       - A value of 0 means "send all available records".
│
└── stream RecordResponse
    ├── session_info (SessionInfo)
    │   ├── local_address : string — Local socket address
    │   ├── local_port : uint32 — Local socket port
    │   ├── remote_address : string — Remote socket address
    │   ├── remote_port : uint32 — Remote socket port
    │   ├── ip_proto : uint32 — Protocol number (e.g. 6=TCP, 17=UDP)
    │   ├── channel_id : string — Multiplexing channel (gRPC/SSH sessions)
    │   ├── tty : string — Optional tty name (e.g. console0, vty0)
    │   ├── status : enum SessionStatus
    │   │   ├── SESSION_STATUS_UNSPECIFIED = 0
    │   │   ├── SESSION_STATUS_LOGIN = 1      ("start")
    │   │   ├── SESSION_STATUS_LOGOUT = 2     ("stop")
    │   │   ├── SESSION_STATUS_ONCE = 3       (login + cmd + logout in one)
    │   │   ├── SESSION_STATUS_ENABLE = 4     (privilege change)
    │   │   ├── SESSION_STATUS_IDLE = 5       ("watchdog")
    │   │   └── SESSION_STATUS_OPERATION = 6  ("cmd")
    │   ├── user (UserDetail)
    │   │   ├── identity : string — User identity (username, SPIFFE-ID, etc.)
    │   │   ├── role : string — User’s role/group/class
    │   │   └── ssh_principal : string — SSH cert principal (if applicable)
    │   └── authn (AuthnDetail)
    │       ├── type : enum AuthnType
    │       │   ├── AUTHN_TYPE_UNSPECIFIED = 0
    │       │   ├── AUTHN_TYPE_NONE = 1
    │       │   ├── AUTHN_TYPE_PASSWORD = 2
    │       │   ├── AUTHN_TYPE_SSHKEY = 3
    │       │   ├── AUTHN_TYPE_SSHCERT = 4
    │       │   ├── AUTHN_TYPE_TLSCERT = 5
    │       │   ├── AUTHN_TYPE_PAP = 6
    │       │   └── AUTHN_TYPE_CHAP = 7
    │       ├── status : enum AuthnStatus
    │       │   ├── AUTHN_STATUS_UNSPECIFIED = 0
    │       │   ├── AUTHN_STATUS_SUCCESS = 1
    │       │   ├── AUTHN_STATUS_FAIL = 2
    │       │   └── AUTHN_STATUS_ERROR = 3
    │       └── cause : string — Reason for failure/error
    │
    ├── timestamp : google.protobuf.Timestamp  
    │   - Time event was recorded (nanoseconds since epoch).
    │
    ├── history_istruncated : bool  
    │   - True if earlier records were lost/skipped.  
    │   - Cases:
    │       * Server has no full history since requested timestamp.
    │       * Timestamp earlier than system uptime.
    │       * Server detected missed events in an ongoing stream.
    │
    ├── service_request (oneof)
    │   ├── cmd_service (CommandService)
    │   │   ├── service_type : enum CmdServiceType
    │   │   │   ├── CMD_SERVICE_TYPE_UNSPECIFIED = 0
    │   │   │   ├── CMD_SERVICE_TYPE_SHELL = 1
    │   │   │   ├── CMD_SERVICE_TYPE_CLI = 2
    │   │   │   ├── CMD_SERVICE_TYPE_WEBUI = 3
    │   │   │   ├── CMD_SERVICE_TYPE_RESTCONF = 4
    │   │   │   └── CMD_SERVICE_TYPE_NETCONF = 5
    │   │   ├── cmd : string — Executed command (expanded to full form)
    │   │   ├── cmd_istruncated : bool — True if command was truncated
    │   │   ├── cmd_args[] : repeated string — Expanded arguments
    │   │   ├── cmd_args_istruncated : bool — True if args were truncated
    │   │   └── authz (AuthzDetail)
    │   │       ├── status : enum AuthzStatus
    │   │       │   ├── AUTHZ_STATUS_UNSPECIFIED = 0
    │   │       │   ├── AUTHZ_STATUS_PERMIT = 1
    │   │       │   ├── AUTHZ_STATUS_DENY = 2
    │   │       │   └── AUTHZ_STATUS_ERROR = 3
    │   │       └── detail : string — Policy/details for PERMIT/DENY
    │   │
    │   └── grpc_service (GrpcService)
    │       ├── service_type : enum GrpcServiceType
    │       │   ├── GRPC_SERVICE_TYPE_UNSPECIFIED = 0
    │       │   ├── GRPC_SERVICE_TYPE_GNMI = 1
    │       │   ├── GRPC_SERVICE_TYPE_GNOI = 2
    │       │   ├── GRPC_SERVICE_TYPE_GNSI = 3
    │       │   ├── GRPC_SERVICE_TYPE_GRIBI = 4
    │       │   └── GRPC_SERVICE_TYPE_P4RT = 5
    │       ├── rpc_name : string — RPC path (e.g. /gnmi.gNMI/Set)
    │       ├── payloads[] : repeated Any — Legacy payload (deprecated)
    │       ├── payload_istruncated : bool — True if payload was truncated
    │       ├── authz (AuthzDetail)
    │       │   ├── status : enum AuthzStatus (see above)
    │       │   └── detail : string
    │       └── payload (oneof)
    │           ├── proto_val : Any — Structured proto payload
    │           └── string_val : string — Human-readable string payload
    │
    ├── component_name : string  
    │   - Originating component (e.g. "linecard0", "chassis0").  
    │   - Matches OpenConfig component path (/components/component[name="..."])
    │
    └── task_ids[] : repeated string  
        - IDs for internal system tasks executed as part of the request.
```
## 5. Supported Interfaces
gNSI AcctZ extends accounting to cover all major control interfaces:

XR CLI – exec and config mode commands

SHELL – bash/run operations

Sysadmin CLI and Shell – full logging

NETCONF – RPC executions

gRPC/gNxI – gNMI, gNOI, gNSI, gRIBI, P4RT

Together with existing syslog/TACACS+, this provides holistic accounting coverage across the device.

## 6. High-Level Topology
The AcctZ service runs alongside traditional AAA accounting.

```
         +-----------------+
         | Remote Collector|
         +-----------------+
                  ^
                  |  gNSI AcctZ stream
                  |
   +----------------------------------+
   |   IOS XR Router                  |
   |                                  |
   |  CLI  NETCONF  gNMI  gNOI  gRIBI |
   |   |      |       |      |     |  |
   |   +------v-------v------+-----v--+
   |             AcctZ Service        |
   +----------------------------------+
```
This model allows operators to centralize auditing across all interfaces in a consistent format.

## 7. Key Benefits
✅ Unified Visibility
Collect accounting data across CLI, shell, NETCONF, and gRPC services in one place.

✅ Compliance & Security
Session, authentication, and authorization details support regulatory compliance and audit requirements.

✅ Resilience
Circular buffer ensures no data loss during temporary collector outages.

✅ Extensibility
Based on OpenConfig standards for flexibility and interoperability.

✅ Seamless Coexistence
Works alongside existing syslog/TACACS+ logging systems.

## 8. Considerations
Record Size: RPC payloads can be large; collectors must handle this efficiently.

History Depth: Depends on device configuration and memory.

Security Controls: Collectors must be authorized.

Fallback Logging: Syslog/TACACS+ can still serve as a backup log source.

## 9. Conclusion
With gNSI AcctZ, IOS XR brings gRPC-native accounting to the forefront—closing visibility gaps in modern telemetry and automation workflows.

By extending AAA accounting into the gNxI ecosystem, AcctZ provides a consistent, secure, and future-proof framework for compliance, auditing, and trust.

As networks become increasingly API-driven, AcctZ is essential for delivering the accountability and transparency operators require.
