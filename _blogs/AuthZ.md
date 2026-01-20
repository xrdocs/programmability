---
date: '2024-03-05 13:13 -0500'
title: gRPC Authorization using gNSI Authz
position: hidden
author: Rahul Sharma
excerpt: An introduction to AuthZ with hands-on gRPC authorization using gRPCurl.
tags:
  - iosxr
  - cisco
  - Programmability
  - gNSI
  - Authz
  - gRPC
published: true
---

This blog focuses on [AuthZ](https://github.com/openconfig/gnsi/tree/main/authz), a gNSI service that manages Authorization at gRPC service level. It is supported from XR 7.11.

# Introduction: AuthZ - Authorization at gRPC Service RPC level

<p align="justify">The gRPC suite comprises of various services including gNMI, gNOI, and gNSI, each consisting of multiple Remote Procedure Calls (RPCs) such as Get(), Set(), Subscribe(), and Rotate(). With AuthZ, network devices can be configured to restrict or allow user-level access to specific RPCs within these services.</p>

## Use Case Example

| User   | gRPC Service | RPC         | Action |
|--------|--------------|-------------|--------|
| Rahul  | gNMI         | Get()       | Allow  |
| Eric   | gNMI         | Subscribe() | Allow  |
| Cisco  | All          | All         | Allow  |
| Mike   | gNOI         | Verify()    | Deny   |

<p align="justify">To implement such controls, a <b>Service Authorization Policy</b> must be defined using a JSON-formatted Role-Based Access Control (RBAC) schema, which contains 'allow' and 'deny' rules mapping Principals to gRPC Services and RPCs.</p>

## Sample Authorization Policy

```
{
  "name": "Manage RPC level access for users Rahul, Eric, Cisco and Mike",
  "allow_rules": [
    {
      "name": "Allow user Rahul to access Get() RPC from gNMI service",
      "source": {
        "principals": ["Rahul"]
      },
      "request": {
        "paths": ["/gnmi.gNMI/Get"]
      }
    },
    {
      "name": "Allow user Eric to access Subscribe() RPC from gNMI service",
      "source": {
        "principals": ["Eric"]
      },
      "request": {
        "paths": ["/gnmi.gNMI/Subscribe"]
      }
    },
    {
      "name": "Allow user Cisco access to all RPCs",
      "source": {
        "principals": ["cisco"]
      },
      "request": {
        "paths": ["*"]
      }
    }
  ],
  "deny_rules": [
    {
      "name": "Deny user Mike to access Verify() RPC from gNOI service",
      "source": {
        "principals": ["Mike"]
      },
      "request": {
        "paths": ["/gnoi.os.OS/Verify"]
      }
    }
  ]
}
```

## Key Terminoligies

1. <b>Name</b>: Identifier for the policy or individual rule.
2. <b>Source/Principal</b>: The subject of the policy (e.g./ username or [SPIFFE ID](https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/#spiffe-id))
3. <b>Policy Behavior</b>: 
	* <b>Allow Rule</b>: Grants access to specified principals and denies all others by default.
    * <b>Deny Rule</b>: Denies access to specified principals while allowing all others.
    * <b>Precedence</b>: Deny rules override allow rules.

## Policy Deployment Methods

The policy can be onboarded to the device through the following methods:

1. During boot via a BootZ operation.
2. Using a ZTP script during initial device bootstrapping.
3. Manual copy to the designated disk location.

### Installation command

Once the policy is onboarded, it can be installed using the following CLI:

```
RP/0/RP0/CPU0:ios# gnsi load service authorization policy <path-to-policy>
```

The policy can also be installed using Rotate () RPC which is discussed later in this blog.

### Validation During Installation

* JSON schema conformance
* JSON schema version check
* Presence of 'allow_rules' section (mandatory)

If no policy is installed, the system behaves as per a 'zero-policy behavior'. By default, the zero-policy behavior is ‘allow all’. 

> Note: The policy MUST contain allow_rules section otherwise the system will behave similarly to the absence of the policy file. Even a policy file with empty allow_rules is considered a user-configured policy and does not qualify as zero-policy mode.

## AuthZ Proto RPCs

The AuthZ proto defines three main RPCs for policy management:

### 1. Rotate () RPC
This RPC changes the existing policy on the system, with each policy identified by its ‘Version’ and ‘Timestamp’. The client: 
* starts the stream,
* sends the policy to the target, 
* performs an optional test and validation, 
* and finalizes the rotation by sending a FinalizeRequest.

Before final commit, one or more probe requests could be made to validate the ‘candidate’ policy.

After receiving the ‘Finalize Request’, system will keep a backup of the current policy until ‘Finalize Request’ is done.

> NOTE: This is an exclusive RPC, meaning only one Rotate () RPC session can be active at a time. The request for second session is denied until first one continues.

The new policy changes don’t affect active gRPC sessions; it only impacts new sessions coming into the device.

> NOTE: Existing sessions created by the users who had privileges before the Rotation will continue to be active even if the new Policy removes those privileges. There will not be scope for Privilege escalation attacks as the Set RPC will try to establish a new session and will be allowed access with the new policies.

Following is a sample 'gRPCurl' command to use this RPC:

```
➜  authz git:(main) ✗ grpcurl -plaintext -d '{
"uploadRequest":
    {
        "version": "Creating policy through grpcurl",
        "created_on": 171321154967868,
        "policy": "{\"name\":\"Manage RPC level access for users Rahul, Eric, cisco and 
        Mike\",\"allow_rules\":[{\"name\":\"Allow user Rahul to access Get() RPC from gNMI 
        service\",\"request\":{\"paths\":[\"\/gnmi.gNMI\/Get\”]},\”source\":{\"principals\":
        [\"Rahul\"]}},{\"name\":\"Allow user Eric to access Subscribe() RPC from gNMI 
        service\",\"request\":{\"paths\":[\"\/gnmi.gNMI\/Subscribe\"]},\"source\":{\"principals\":
        [\"Eric\"]}},{\"name\":\"Allow user cisco access to all RPCs\",\"request\":{\"paths\":
        [\"*\"]},\"source\":{\"principals\":[\"cisco\"]}}],\"deny_rules\":[{\"name\":\"Deny user 
        Mike to access Verify() RPC from gNOI service\",\"request\":{\"paths\":
        [\"\/gnoi.os.OS\/Verify\"]},\"source\":{\"principals\":[\"Mike\"]}}]}"
    },
    "force_overwrite": true
    }{"finalize_rotation":{}}' -import-path ../authz -proto authz.proto  -H username:cisco -H
password:cisco123! 172.20.163.79:57400 gnsi.authz.v1.Authz.Rotate

Request metadata to send:
password: cisco123!
username: cisco

Response headers received:
content-type: application/grpc

Estimated response size: 0 bytes

Response contents:
{}

Estimated response size: 2 bytes

Response contents:
{
  "uploadResponse": {}
}

Response trailers received:
(empty)
Sent 2 requests and received 2 responses
```

### 2. Probe () RPC

Evaluates if a specific user action is permitted by the current/candidate policy without executing the RPC.

```
➜  authz git:(main) ✗ grpcurl -plaintext -d '{"user":"Rahul","rpc":"/gnmi.gNMI/Get"}' -
import-path ../authz -proto authz.proto  -H username:cisco -H password:cisco123! 
172.20.163.79:57400  gnsi.authz.v1.Authz.Probe

Request metadata to send:
password: cisco123!
username: cisco

Response headers received:
content-type: application/grpc

Estimated response size: 7 bytes

Response contents:
{
  "action": "ACTION_PERMIT",
  "version": "1.0"
}

Response trailers received:
(empty)
Sent 1 request and received 1 response 

➜ authz git:(main) ✗ grpcurl -plaintext -d '{"user":"Rahul","rpc":"/gnmi.gNMI/Capabilities"}' -import-path ../authz -proto authz.proto  -H 
username:cisco -H password:cisco123! 172.20.163.79:57400  gnsi.authz.v1.Authz.Probe

Request metadata to send:
password: cisco123!
username: cisco

Response headers received:
content-type: application/grpc

Estimated response size: 7 bytes

Response contents:
{
  "action": "ACTION_DENY",
  "version": "1.0"
}

Response trailers received:
(empty)
Sent 1 request and received 1 response
```
### 3. Get () RPC

Retreives the current authorization policy, including its metadata (version and timestamp).

```
➜  authz git:(main) ✗ grpcurl -plaintext -import-path ../authz -proto authz.proto  -H 
username:cisco -H password:cisco123! 172.20.163.79:57400  gnsi.authz.v1.Authz.Get

Request metadata to send:
password: cisco123!
username: cisco

Response headers received:
content-type: application/grpc

Estimated response size: 1471 bytes

Response contents:
{
  "version": "1.0",
  "createdOn": "1713207488",
  "policy": "{\n    \"name\": \"Manage RPC level access for users Rahul, Eric, cisco and Mike\",\n
  \"allow_rules\": [\n        {\n            \"name\": \"Allow user Rahul to access Get() RPC from
  gNMI service\",\n            \"source\": {\n                \"principals\": [\n     
  \"Rahul\"\n                ]\n            },\n            \"request\": {\n 
  \"paths\": [\n                    \"/gnmi.gNMI/Get\"\n                ]\n            }\n  
  },\n        {\n            \"name\": \"Allow user Eric to access Subscribe() RPC from gNMI 
  service\",\n            \"source\": {\n                \"principals\": [\n             
  \"Eric\"\n                ]\n            },\n            \"request\": {\n              
  \"paths\": [\n                    \"/gnmi.gNMI/Subscribe\"\n                ]\n            }\n
  },\n\t{\n            \"name\": \"Allow user cisco access to all RPCs\",\n            \"source\":
  {\n                \"principals\": [\n                    \"cisco\"\n                ]\n   
  },\n            \"request\": {\n                \"paths\": [\n                    \"*\"\n 
  ]\n            }\n        }\n    ],\n    \"deny_rules\": [\n        {\n            \"name\": 
  \"Deny user Mike to access Verify() RPC from gNOI service\",\n            \"source\": {\n 
  \"principals\": [\n                    \"Mike\"\n                ]\n            },\n   
  \"request\": {\n                \"paths\": [\n                    \"/gnoi.os.OS/Verify\"\n  
  ]\n            }\n        }\n    ]\n}\n\n"
}

Response trailers received:
(empty)
Sent 0 requests and received 1 response
```

## Conclusion

<p align="justify">Implementing gRPC-level authorization using the gNSI AuthZ framework enables fine-grained control over RPC access for different users across network services like gNMI and gNOI. With JSON-based policies and native CLI/gRPC tooling, administrators can define, test, and deploy authorization rules in a structured and repeatable manner.</p>



## Key Takeaways

- **gNSI AuthZ** allows user-level control of individual gRPC RPCs across services like gNMI, gNOI, and gNSI.
- Policies are defined in JSON using `allow_rules` and `deny_rules`.
- **Deny rules** always take precedence over allow rules.
- Policies can be onboarded via **BootZ**, **ZTP**, or manual disk placement.
- The `gnsi load service authorization policy` CLI command activates the policy.
- The `Rotate()`, `Probe()`, and `Get()` RPCs enable full lifecycle management of policies.
- The system defaults to **“allow all”** access unless an explicit policy is applied.
- `grpcurl` is a useful tool for interacting with the AuthZ RPCs in real time.
