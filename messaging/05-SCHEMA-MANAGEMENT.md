# Schema Management

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Why Schema Management](#why-schema-management)
   - [Contract Enforcement](#contract-enforcement)
   - [Schema Evolution](#schema-evolution-overview)
   - [Backwards Compatibility](#backwards-compatibility)
3. [Serialization Formats](#serialization-formats)
   - [JSON](#json)
   - [Avro](#avro-format)
   - [Protocol Buffers](#protocol-buffers-format)
   - [Format Comparison](#format-comparison)
4. [Schema Registry](#schema-registry)
   - [Confluent Schema Registry](#confluent-schema-registry)
   - [AWS Glue Schema Registry](#aws-glue-schema-registry)
   - [Azure Schema Registry](#azure-schema-registry)
   - [Registry Comparison](#registry-comparison)
5. [Schema Evolution](#schema-evolution)
   - [Forward Compatibility](#forward-compatibility)
   - [Backward Compatibility](#backward-compatibility-1)
   - [Full Compatibility](#full-compatibility)
   - [Breaking vs Non-Breaking Changes](#breaking-vs-non-breaking-changes)
6. [Avro](#avro)
   - [Schema Definition](#schema-definition)
   - [Unions and Defaults](#unions-and-defaults)
   - [Complex Types](#complex-types)
   - [Avro Evolution Rules](#avro-evolution-rules)
7. [Protocol Buffers](#protocol-buffers)
   - [Message Definitions](#message-definitions)
   - [Field Numbering](#field-numbering)
   - [Wire Format](#wire-format)
   - [Protobuf Evolution Rules](#protobuf-evolution-rules)
8. [JSON Schema](#json-schema)
   - [Validation](#validation)
   - [Draft Versions](#draft-versions)
   - [Tooling](#tooling)
9. [Schema Governance](#schema-governance)
   - [Ownership](#ownership)
   - [Review Process](#review-process)
   - [Versioning Strategy](#versioning-strategy)
   - [CI/CD Integration](#cicd-integration)
10. [Migration Strategies](#migration-strategies)
    - [Dual-Write](#dual-write)
    - [Gradual Migration](#gradual-migration)
    - [Schema Translation](#schema-translation)
11. [Next Steps](#next-steps)
12. [Version History](#version-history)

---

## Overview

Schema management is the discipline of defining, versioning, and enforcing the structure of messages exchanged between producers and consumers in a messaging system. Without explicit schema management, distributed systems accumulate implicit contracts that break silently at runtime, leading to data corruption, processing failures, and costly outages.

### Target Audience

- Backend and platform engineers building event-driven systems
- Architects designing data contracts across service boundaries
- Data engineers responsible for streaming pipelines and data quality
- Teams adopting or migrating to schema-first development workflows

### Scope

This document covers serialization formats, schema registries, evolution strategies, and governance practices for messaging systems. It is **broker-agnostic** — the concepts apply whether you use Apache Kafka, RabbitMQ, AWS SNS/SQS, Azure Service Bus, or Google Cloud Pub/Sub. For broker-specific details, see the companion files listed in [Next Steps](#next-steps).

---

## Why Schema Management

In a distributed system, producers and consumers are developed, deployed, and scaled independently. The message format is the contract between them. Schema management makes that contract explicit, versioned, and enforceable.

### Contract Enforcement

Without schema enforcement, any producer can emit any payload shape. Consumers discover mismatches only at runtime — often in production.

```
Without Schema Management
──────────────────────────

  Producer A          Broker            Consumer X
  ┌──────────┐      ┌────────┐      ┌────────────┐
  │ { "name": │─────▶│        │─────▶│ Expects    │
  │   "Alice",│      │        │      │ "user_name"│  ✗ Field mismatch
  │   "age":  │      │        │      │ field      │    at runtime
  │   30 }    │      │        │      │            │
  └──────────┘      └────────┘      └────────────┘

With Schema Management
───────────────────────

  Producer A          Registry         Broker            Consumer X
  ┌──────────┐      ┌──────────┐    ┌────────┐      ┌────────────┐
  │ Serialize │─────▶│ Validate │    │        │─────▶│ Deserialize│
  │ against   │      │ schema   │───▶│        │      │ with known │  ✓ Contract
  │ schema v2 │      │ v2       │    │        │      │ schema v2  │    enforced
  └──────────┘      └──────────┘    └────────┘      └────────────┘
```

**Benefits of explicit contracts:**

- Producers cannot publish messages that violate the schema
- Consumers know exactly what fields and types to expect
- Schema changes are reviewed and versioned before deployment
- Incompatible changes are caught at build time, not runtime

### Schema Evolution Overview

Systems are never static. Business requirements change, new fields are added, old fields become obsolete. Schema evolution defines the rules for changing a schema while preserving compatibility between different versions of producers and consumers.

```
Schema Evolution Timeline
─────────────────────────

  v1: { name, email }
       │
       ▼  (add optional field)
  v2: { name, email, phone? }         ◀── Non-breaking change
       │
       ▼  (add required field with default)
  v3: { name, email, phone?, role="user" }  ◀── Non-breaking change
       │
       ▼  (remove field without default)
  v4: { name, role }                   ◀── BREAKING change
```

### Backwards Compatibility

**Backwards compatibility** means that consumers using an older schema version can still read messages produced with a newer schema version. This is the most important compatibility guarantee in practice because consumers are typically upgraded after producers.

| Compatibility Direction | Who Must Understand Whom | Typical Use Case |
|------------------------|-------------------------|-----------------|
| **Backward** | New consumer reads old data | Consumer upgrades before producer |
| **Forward** | Old consumer reads new data | Producer upgrades before consumer |
| **Full** | Both directions | Independent deployment of any version |
| **None** | No guarantees | Prototype/development only |

---

## Serialization Formats

The serialization format determines how structured data is encoded into bytes for transmission and decoded back into structured data on the consumer side. The choice of format affects performance, schema enforcement, and evolution capabilities.

### JSON

**JSON** (JavaScript Object Notation) is a human-readable, text-based format. It is the most common format for REST APIs and is widely used in messaging systems due to its simplicity.

**Strengths:**

- Human-readable and easy to debug
- Ubiquitous tooling and library support
- No compilation step required
- Self-describing — field names are included in every message

**Weaknesses:**

- No built-in schema enforcement (requires JSON Schema as an add-on)
- Verbose — field names repeated in every message increase payload size
- No native support for schema evolution rules
- Slower serialization/deserialization compared to binary formats

### Avro Format

**Apache Avro** is a binary serialization format with a rich schema definition language. Schemas are defined in JSON and the encoded payload is compact because field names are not included — the schema provides the structure.

**Strengths:**

- Compact binary encoding (no field names in payload)
- First-class schema evolution with compatibility rules
- Schema is stored separately (in a registry), reducing message size
- Strong integration with Kafka and the Confluent ecosystem

**Weaknesses:**

- Not human-readable (binary format)
- Reader and writer must agree on a schema (or use a registry)
- Schema resolution can be complex with deeply nested types

### Protocol Buffers Format

**Protocol Buffers** (protobuf) is a binary serialization format developed by Google. Schemas are defined in `.proto` files and compiled into language-specific code.

**Strengths:**

- Compact binary encoding with field tags (not names)
- Code generation for many languages
- Strong backward and forward compatibility via field numbering
- Widely adopted in gRPC and internal Google systems

**Weaknesses:**

- Requires a compilation step (protoc)
- Less human-readable than JSON
- No native union types (use `oneof` instead)
- Removed required fields in proto3 (everything is optional by default)

### Format Comparison

| Feature | JSON | Avro | Protocol Buffers |
|---------|------|------|-----------------|
| **Encoding** | Text | Binary | Binary |
| **Human-readable** | Yes | No | No |
| **Schema required** | No (optional via JSON Schema) | Yes | Yes (.proto file) |
| **Schema in payload** | Field names included | No (schema ID reference) | No (field tags only) |
| **Code generation** | Optional | Optional | Required |
| **Payload size** | Large | Small | Small |
| **Serialization speed** | Slow | Fast | Fast |
| **Evolution support** | Manual (JSON Schema) | Native (compatibility modes) | Native (field numbering) |
| **Ecosystem** | Universal | Kafka/Hadoop | gRPC/Google |
| **Best for** | APIs, debugging | Kafka event streaming | gRPC, high-performance systems |

---

## Schema Registry

A schema registry is a centralized service that stores, versions, and validates schemas. Producers register schemas before publishing messages, and consumers fetch schemas to deserialize messages. The registry enforces compatibility rules, preventing breaking changes from reaching production.

```
Schema Registry Architecture
─────────────────────────────

  ┌──────────┐    1. Register      ┌─────────────────┐
  │ Producer │───────schema v3────▶│                 │
  └────┬─────┘                     │  Schema         │
       │                           │  Registry       │
       │  2. Serialize with        │                 │
       │     schema ID = 3         │  ┌───────────┐  │
       ▼                           │  │ v1        │  │
  ┌──────────┐                     │  │ v2        │  │
  │  Broker  │                     │  │ v3 (new)  │  │
  └────┬─────┘                     │  └───────────┘  │
       │                           └────────┬────────┘
       │  3. Consume message                │
       ▼                                    │
  ┌──────────┐    4. Fetch schema   ────────┘
  │ Consumer │───────for ID = 3────▶
  └──────────┘
```

**Core responsibilities of a schema registry:**

- **Storage**: Persist schema versions for each subject (topic)
- **Versioning**: Track schema history and assign unique IDs
- **Compatibility checks**: Validate new schemas against the configured compatibility mode
- **Lookup**: Allow producers and consumers to fetch schemas by ID or subject

### Confluent Schema Registry

The **Confluent Schema Registry** is the most widely adopted registry, designed primarily for Apache Kafka. It stores schemas in a Kafka topic (`_schemas`) for durability and supports Avro, Protobuf, and JSON Schema.

**Key features:**

- Compatibility modes per subject (backward, forward, full, none)
- REST API for schema registration and lookup
- Schema ID embedded in the message payload (5-byte header)
- Integrates with Kafka Connect, ksqlDB, and Kafka Streams
- Available as part of Confluent Platform or Confluent Cloud

```
Confluent Schema Registry — Message Format
───────────────────────────────────────────

  ┌──────────────────────────────────────────┐
  │  Kafka Message Value                     │
  │                                          │
  │  ┌──────┬───────────┬──────────────────┐ │
  │  │ 0x00 │ Schema ID │ Avro/Protobuf    │ │
  │  │      │ (4 bytes) │ encoded payload  │ │
  │  │ magic│           │                  │ │
  │  │ byte │           │                  │ │
  │  └──────┴───────────┴──────────────────┘ │
  │                                          │
  │  Total overhead: 5 bytes per message     │
  └──────────────────────────────────────────┘
```

### AWS Glue Schema Registry

**AWS Glue Schema Registry** is a serverless registry integrated with the AWS ecosystem. It supports Avro, JSON Schema, and Protobuf and integrates with Kinesis Data Streams, MSK (Managed Kafka), and AWS Glue ETL.

**Key features:**

- Serverless — no infrastructure to manage
- IAM-based access control
- Compatibility enforcement (backward, forward, full, none)
- Schema versioning with auto-assigned version numbers
- Native integration with AWS Glue, Kinesis, and MSK

### Azure Schema Registry

**Azure Schema Registry** is part of Azure Event Hubs and provides schema management for event-driven systems on Azure. It supports Avro schemas and integrates with Event Hubs and the Kafka protocol support in Event Hubs.

**Key features:**

- Managed service within Azure Event Hubs namespace
- Avro schema support (JSON Schema and Protobuf in preview)
- Role-based access control via Azure AD
- Schema groups for organizing schemas by application or team
- Client libraries for .NET, Java, Python, and JavaScript

### Registry Comparison

| Feature | Confluent | AWS Glue | Azure Schema Registry |
|---------|-----------|----------|----------------------|
| **Managed** | Cloud or self-hosted | Serverless | Managed (Event Hubs) |
| **Formats** | Avro, Protobuf, JSON Schema | Avro, JSON Schema, Protobuf | Avro (others in preview) |
| **Broker integration** | Kafka | Kinesis, MSK | Event Hubs |
| **Compatibility modes** | All (per-subject) | All | Backward, forward, full, none |
| **Schema ID transport** | 5-byte header in message | Header or payload | Content-type header |
| **Access control** | ACLs, RBAC | IAM policies | Azure AD RBAC |
| **Pricing** | Confluent license / Cloud tier | Pay-per-use | Included with Event Hubs |

---

## Schema Evolution

Schema evolution is the process of changing a schema over time while maintaining compatibility between producers and consumers running different versions. A well-defined evolution strategy prevents runtime failures during rolling deployments.

```
Compatibility Modes
────────────────────

  Backward Compatible (new consumer, old data)
  ──────────────────────────────────────────────

    Producer (v1) ──▶ Broker ──▶ Consumer (v2)     ✓

    Consumer v2 can read messages written by Producer v1.
    Rule: New schema can read data written with the old schema.


  Forward Compatible (old consumer, new data)
  ─────────────────────────────────────────────

    Producer (v2) ──▶ Broker ──▶ Consumer (v1)     ✓

    Consumer v1 can read messages written by Producer v2.
    Rule: Old schema can read data written with the new schema.


  Full Compatible (both directions)
  ──────────────────────────────────

    Producer (v1) ──▶ Broker ──▶ Consumer (v2)     ✓
    Producer (v2) ──▶ Broker ──▶ Consumer (v1)     ✓

    Any version can read data written by any adjacent version.
```

### Forward Compatibility

**Forward compatibility** guarantees that consumers using an older schema can still process messages produced with a newer schema. This is critical when producers are upgraded before consumers.

**Rules for forward compatibility:**

- New fields must have default values (old consumer ignores unknown fields)
- No fields may be removed unless they had defaults
- Field types must not change in incompatible ways

### Backward Compatibility

**Backward compatibility** guarantees that consumers using a newer schema can still process messages produced with an older schema. This is critical when consumers are upgraded before producers.

**Rules for backward compatibility:**

- New fields must have default values (so missing fields in old data get defaults)
- Removed fields must have had default values
- Field types must not change in incompatible ways

### Full Compatibility

**Full compatibility** is the combination of forward and backward compatibility. Any consumer version can read data produced by any adjacent producer version. This is the safest mode and is recommended for production systems.

### Breaking vs Non-Breaking Changes

| Change Type | Example | Backward | Forward | Full |
|------------|---------|----------|---------|------|
| Add optional field with default | `phone: string = ""` | ✓ Safe | ✓ Safe | ✓ Safe |
| Add required field without default | `phone: string` (required) | ✗ Breaks | ✓ Safe | ✗ Breaks |
| Remove field with default | Remove `phone` (had default) | ✓ Safe | ✓ Safe | ✓ Safe |
| Remove field without default | Remove `name` (no default) | ✓ Safe | ✗ Breaks | ✗ Breaks |
| Rename field | `name` → `full_name` | ✗ Breaks | ✗ Breaks | ✗ Breaks |
| Change field type | `age: int` → `age: string` | ✗ Breaks | ✗ Breaks | ✗ Breaks |
| Add enum value | Add `PREMIUM` to `UserType` | ✓ Safe | ✗ Breaks | ✗ Breaks |
| Remove enum value | Remove `TRIAL` from `UserType` | ✗ Breaks | ✓ Safe | ✗ Breaks |

> **Best practice:** Default to **full compatibility** mode. Only relax to backward or forward when you have strong deployment ordering guarantees and understand the implications.

---

## Avro

Apache Avro is the most popular serialization format in the Kafka ecosystem. Its tight integration with schema registries and native support for schema evolution make it the default choice for many event streaming platforms.

### Schema Definition

Avro schemas are defined in JSON. A record schema specifies the record name, namespace, and an ordered list of fields.

```json
{
  "type": "record",
  "name": "UserCreated",
  "namespace": "com.example.events",
  "fields": [
    { "name": "user_id",    "type": "string" },
    { "name": "email",      "type": "string" },
    { "name": "created_at", "type": "long",   "logicalType": "timestamp-millis" },
    { "name": "role",       "type": "string", "default": "user" }
  ]
}
```

**Avro primitive types:**

| Type | Description | Example |
|------|-------------|---------|
| `null` | No value | `null` |
| `boolean` | True/false | `true` |
| `int` | 32-bit signed integer | `42` |
| `long` | 64-bit signed integer | `1622505600000` |
| `float` | 32-bit IEEE 754 | `3.14` |
| `double` | 64-bit IEEE 754 | `3.141592653589793` |
| `bytes` | Sequence of bytes | binary data |
| `string` | Unicode character sequence | `"hello"` |

### Unions and Defaults

Avro **unions** allow a field to hold one of several types. The most common use is making a field nullable by combining `null` with another type.

```json
{
  "name": "phone",
  "type": ["null", "string"],
  "default": null
}
```

**Union rules:**

- The default value must match the **first type** in the union
- For nullable fields, place `"null"` first and set `"default": null`
- Unions cannot directly contain other unions
- Unions cannot contain multiple schemas of the same type

```
Avro Union Resolution
──────────────────────

  Writer schema:  ["null", "string"]
  Writer value:   "555-1234"

  Resolution: Match against union branches in order.
              "555-1234" is a string ──▶ branch index 1.

  Encoded: ┌───────────┬──────────────────┐
           │ index = 1 │ "555-1234"       │
           │ (varint)  │ (Avro string)    │
           └───────────┴──────────────────┘
```

### Complex Types

Avro supports complex types beyond primitives and unions.

**Arrays:**

```json
{ "type": "array", "items": "string" }
```

**Maps:**

```json
{ "type": "map", "values": "long" }
```

**Enums:**

```json
{
  "type": "enum",
  "name": "UserRole",
  "symbols": ["ADMIN", "USER", "GUEST"]
}
```

**Fixed (fixed-length byte arrays):**

```json
{
  "type": "fixed",
  "name": "MD5",
  "size": 16
}
```

**Nested records:**

```json
{
  "type": "record",
  "name": "Address",
  "fields": [
    { "name": "street", "type": "string" },
    { "name": "city",   "type": "string" },
    { "name": "zip",    "type": "string" }
  ]
}
```

### Avro Evolution Rules

Avro uses **writer schema** (the schema used to encode the data) and **reader schema** (the schema used to decode the data). The Avro library performs schema resolution to map fields between the two schemas.

```
Avro Schema Resolution
───────────────────────

  Writer Schema (v1)              Reader Schema (v2)
  ┌──────────────────┐           ┌──────────────────────┐
  │ user_id: string  │──────────▶│ user_id: string      │  ✓ Match
  │ email:   string  │──────────▶│ email:   string      │  ✓ Match
  │ name:    string  │     ╳     │                      │  (ignored by reader)
  │                  │           │ role: string = "user" │  ✓ Default used
  └──────────────────┘           └──────────────────────┘

  - Fields in writer but not reader: ignored
  - Fields in reader but not writer: must have a default value
  - Field order does not matter — matching is by field name
```

**Safe Avro evolution changes:**

| Change | Backward | Forward | Full |
|--------|----------|---------|------|
| Add field with default | ✓ | ✓ | ✓ |
| Remove field with default | ✓ | ✓ | ✓ |
| Add field without default | ✗ | ✓ | ✗ |
| Remove field without default | ✓ | ✗ | ✗ |
| Widen type (`int` → `long`) | ✓ | ✗ | ✗ |
| Add enum symbol at end | ✓ | ✗ | ✗ |
| Add union branch | Depends | Depends | Depends |

---

## Protocol Buffers

Protocol Buffers (protobuf) is Google's language-neutral, platform-neutral serialization mechanism. It uses a compiled schema (`.proto` files) and field numbering for wire-format stability.

### Message Definitions

Protobuf messages are defined in `.proto` files using the proto3 syntax.

```protobuf
syntax = "proto3";

package example.events;

message UserCreated {
  string user_id    = 1;
  string email      = 2;
  int64  created_at = 3;
  string role       = 4;
  Address address   = 5;
}

message Address {
  string street = 1;
  string city   = 2;
  string zip    = 3;
}
```

**Key concepts:**

- Each field has a **unique number** (tag) that identifies it in the binary format
- In proto3, all fields are optional by default and have zero-value defaults
- Messages can be nested and reference other message types
- Enums define named constants with integer values

### Field Numbering

Field numbers are the foundation of protobuf's evolution model. The wire format uses field numbers — not names — to identify fields. This means field names can change freely without breaking compatibility.

```
Protobuf Wire Format
─────────────────────

  message UserCreated {
    string user_id = 1;     ──▶  Tag 1, type = string
    string email   = 2;     ──▶  Tag 2, type = string
    int64  age     = 3;     ──▶  Tag 3, type = varint
  }

  Encoded bytes:
  ┌─────────┬────────────┬─────────┬────────────┬─────────┬──────┐
  │ Tag 1   │ "user-123" │ Tag 2   │ "a@b.com"  │ Tag 3   │  30  │
  │ (field  │ (length-   │ (field  │ (length-   │ (field  │(var- │
  │  num +  │  delimited)│  num +  │  delimited)│  num +  │ int) │
  │  wire   │            │  wire   │            │  wire   │      │
  │  type)  │            │  type)  │            │  type)  │      │
  └─────────┴────────────┴─────────┴────────────┴─────────┴──────┘

  Field numbers (tags) are stable across versions.
  Unknown tags are preserved (not discarded) by default.
```

**Field number rules:**

- Field numbers must be unique within a message
- Numbers 1–15 use 1 byte in the wire format (use for frequently set fields)
- Numbers 16–2047 use 2 bytes
- **Never reuse** a field number after deleting a field — use `reserved`
- Reserved ranges prevent accidental reuse: `reserved 3, 6 to 8;`

### Wire Format

Protobuf encodes data as a sequence of key-value pairs where the key is the field number combined with the wire type.

| Wire Type | Meaning | Used For |
|-----------|---------|----------|
| 0 | Varint | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
| 1 | 64-bit | fixed64, sfixed64, double |
| 2 | Length-delimited | string, bytes, embedded messages, repeated fields |
| 5 | 32-bit | fixed32, sfixed32, float |

### Protobuf Evolution Rules

Protobuf provides strong forward and backward compatibility as long as field numbering rules are followed.

**Safe changes:**

- Add new fields (with new field numbers)
- Remove fields (mark old numbers as `reserved`)
- Rename fields (wire format uses numbers, not names)
- Change `int32` to `int64` (or `uint32` to `uint64`) — values are compatible

**Breaking changes:**

- Reuse a deleted field number for a different type
- Change a field's wire type (e.g., `int32` to `string`)
- Change a `repeated` field to a scalar (or vice versa)
- Change the meaning of a field number

```
Protobuf Reserved Fields
─────────────────────────

  // v1: original message
  message UserCreated {
    string user_id = 1;
    string name    = 2;
    int32  age     = 3;    // Will be removed
  }

  // v2: age removed, number 3 reserved
  message UserCreated {
    reserved 3;
    reserved "age";

    string user_id = 1;
    string name    = 2;
    string email   = 4;    // New field with NEW number
  }

  Field 3 is permanently retired.
  If anyone tries to reuse 3, the protoc compiler rejects it.
```

---

## JSON Schema

JSON Schema is a vocabulary for annotating and validating JSON documents. While JSON itself has no built-in schema enforcement, JSON Schema adds structural validation, making it suitable for messaging systems that use JSON payloads.

### Validation

JSON Schema validates the structure, types, and constraints of a JSON document.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.com/user-created.schema.json",
  "title": "UserCreated",
  "type": "object",
  "required": ["user_id", "email"],
  "properties": {
    "user_id": {
      "type": "string",
      "description": "Unique user identifier"
    },
    "email": {
      "type": "string",
      "format": "email"
    },
    "role": {
      "type": "string",
      "enum": ["admin", "user", "guest"],
      "default": "user"
    },
    "created_at": {
      "type": "string",
      "format": "date-time"
    }
  },
  "additionalProperties": false
}
```

**Validation capabilities:**

| Keyword | Purpose | Example |
|---------|---------|---------|
| `type` | Restrict value type | `"type": "string"` |
| `required` | Mandatory fields | `"required": ["user_id"]` |
| `enum` | Allowed values | `"enum": ["A", "B"]` |
| `minimum` / `maximum` | Numeric range | `"minimum": 0` |
| `minLength` / `maxLength` | String length | `"maxLength": 255` |
| `pattern` | Regex validation | `"pattern": "^[A-Z]{3}$"` |
| `format` | Semantic format | `"format": "email"` |
| `additionalProperties` | Allow/disallow extra fields | `"additionalProperties": false` |

### Draft Versions

JSON Schema has evolved through multiple draft versions. Choose the right draft based on tooling support and feature requirements.

| Draft | Year | Key Features |
|-------|------|-------------|
| Draft-04 | 2013 | Widely supported, basic validation |
| Draft-06 | 2017 | `const`, `contains`, `propertyNames` |
| Draft-07 | 2018 | `if`/`then`/`else`, `readOnly`, `writeOnly` |
| 2019-09 | 2019 | Renamed from "draft-08", vocabulary system, `$recursiveRef` |
| 2020-12 | 2020 | `$dynamicRef`, stable vocabulary, recommended for new projects |

> **Recommendation:** Use **2020-12** for new projects. Use Draft-07 if your tooling does not yet support the latest version.

### Tooling

| Tool | Language | Purpose |
|------|----------|---------|
| ajv | JavaScript/TypeScript | Fast JSON Schema validator |
| jsonschema | Python | Validation library |
| json-schema-validator | Java | Validation library (supports all drafts) |
| Newtonsoft.Json.Schema | .NET | Validation and generation |
| json-schema-to-typescript | JavaScript | Generate TypeScript types from schema |
| quicktype | Multi-language | Generate types from JSON Schema |

---

## Schema Governance

Schema governance establishes the organizational processes and policies around schema creation, review, and evolution. Without governance, schemas drift, contracts break, and teams spend significant time debugging incompatibilities.

### Ownership

Every schema should have a clear **owner** — typically the team that produces the messages.

```
Schema Ownership Model
───────────────────────

  ┌───────────────────────────────────────────────────┐
  │  Topic: user-events                               │
  │                                                   │
  │  Owner: Identity Team                             │
  │  Schemas:                                         │
  │    - UserCreated (v1, v2, v3)                     │
  │    - UserUpdated (v1, v2)                         │
  │    - UserDeleted (v1)                             │
  │                                                   │
  │  Consumers:                                       │
  │    - Billing Service (reads UserCreated)           │
  │    - Notification Service (reads UserCreated,      │
  │      UserUpdated)                                 │
  │    - Analytics Pipeline (reads all)               │
  │                                                   │
  │  Approval required from: Identity Team lead       │
  │  Compatibility mode: FULL                         │
  └───────────────────────────────────────────────────┘
```

**Ownership responsibilities:**

- Define and evolve the schema
- Review and approve schema change requests
- Communicate planned breaking changes to consumers
- Maintain schema documentation and changelog

### Review Process

Schema changes should follow a review process similar to code reviews.

```
Schema Change Review Process
─────────────────────────────

  Developer ──▶ Create schema   ──▶ Automated      ──▶ Peer review   ──▶ Merge
                change PR            compatibility       by schema
                                     check (CI)          owner

                     │                    │                    │
                     ▼                    ▼                    ▼
                 .avsc / .proto      Pass/Fail            Approve /
                 file changes        against registry     Request changes
```

**Review checklist:**

- Does the change maintain the required compatibility mode?
- Are new fields documented with clear descriptions?
- Do new fields have sensible defaults?
- Has the consumer impact been assessed?
- Are deprecated fields marked and scheduled for removal?
- Has the schema been tested with sample data?

### Versioning Strategy

| Strategy | Description | When to Use |
|----------|-------------|-------------|
| **Schema Registry versioning** | Registry auto-assigns version numbers per subject | Standard approach for Avro/Protobuf with a registry |
| **Semantic versioning** | Major.minor.patch applied to schema versions | When schemas are managed as independent artifacts |
| **Topic-per-version** | Publish different versions to different topics | Last resort when compatibility cannot be maintained |
| **Envelope versioning** | Include a `schema_version` field in the message payload | JSON-based systems without a registry |

### CI/CD Integration

Integrate schema validation into your CI/CD pipeline to catch breaking changes before deployment.

```
CI/CD Schema Validation Pipeline
──────────────────────────────────

  ┌────────────┐     ┌────────────────┐     ┌──────────────────┐
  │ Git Push   │────▶│ CI Pipeline    │────▶│ Deploy           │
  │ (schema    │     │                │     │                  │
  │  change)   │     │ 1. Lint schema │     │ 1. Register      │
  └────────────┘     │ 2. Compile     │     │    schema in     │
                     │    (protoc)    │     │    registry      │
                     │ 3. Compat.    │     │ 2. Deploy app    │
                     │    check vs   │     │    with new      │
                     │    registry   │     │    schema        │
                     │ 4. Generate   │     └──────────────────┘
                     │    code       │
                     │ 5. Run tests  │
                     └────────────────┘
```

**CI/CD integration tools:**

| Tool | Purpose |
|------|---------|
| `confluent schema-registry test-compatibility` | Test Avro/Protobuf compatibility against Confluent Registry |
| `buf lint` / `buf breaking` | Lint and check breaking changes for Protobuf |
| `avro-tools` | Validate Avro schemas |
| `ajv-cli` | Validate JSON documents against JSON Schema |
| Custom scripts | Fetch latest schema from registry, compare with proposed change |

---

## Migration Strategies

When changing serialization formats, schema registries, or making large-scale schema changes, you need a migration strategy that avoids downtime and data loss.

### Dual-Write

In a **dual-write** migration, the producer writes messages in both the old and new formats simultaneously during a transition period. Consumers switch to the new format at their own pace.

```
Dual-Write Migration
─────────────────────

  Phase 1: Dual write
  ┌──────────┐     ┌────────────────┐     ┌───────────────┐
  │ Producer │────▶│ Old topic      │────▶│ Old consumers │
  │ (writes  │     │ (JSON)         │     │ (JSON)        │
  │  both)   │     └────────────────┘     └───────────────┘
  │          │
  │          │     ┌────────────────┐     ┌───────────────┐
  │          │────▶│ New topic      │────▶│ New consumers │
  │          │     │ (Avro)         │     │ (Avro)        │
  └──────────┘     └────────────────┘     └───────────────┘

  Phase 2: All consumers migrated
  ┌──────────┐     ┌────────────────┐     ┌───────────────┐
  │ Producer │────▶│ New topic      │────▶│ All consumers │
  │ (Avro    │     │ (Avro)         │     │ (Avro)        │
  │  only)   │     └────────────────┘     └───────────────┘
  └──────────┘

  Phase 3: Decommission old topic
```

**Dual-write trade-offs:**

| Advantage | Disadvantage |
|-----------|-------------|
| Zero downtime | Double write amplification during transition |
| Consumers migrate independently | Must keep old and new topics in sync |
| Easy rollback — revert to old topic | Increased producer complexity |

### Gradual Migration

A **gradual migration** uses a translation layer (adapter service) to convert between formats, allowing consumers to migrate one at a time without any producer changes.

```
Gradual Migration with Adapter
────────────────────────────────

  ┌──────────┐     ┌────────────┐     ┌──────────────┐
  │ Producer │────▶│ Topic      │────▶│ Adapter      │
  │ (Avro v2)│     │ (Avro v2)  │     │ Service      │
  └──────────┘     └────────────┘     └──────┬───────┘
                                             │
                                   ┌─────────┼─────────┐
                                   ▼         ▼         ▼
                            ┌──────────┐ ┌────────┐ ┌────────┐
                            │ Consumer │ │ Legacy │ │ Legacy │
                            │ (Avro v2)│ │ (JSON) │ │ (v1)   │
                            └──────────┘ └────────┘ └────────┘
                                           ▲            ▲
                                     Adapter translates to
                                     legacy format on the fly
```

**When to use gradual migration:**

- Many consumers with different teams and release schedules
- Cannot coordinate a synchronized migration window
- Need to support legacy consumers for an extended period

### Schema Translation

**Schema translation** converts messages between formats at the infrastructure level, typically using a stream processor or a connector.

| Tool | Translation Capability |
|------|----------------------|
| Kafka Connect + converters | Convert between Avro, JSON, Protobuf at the connector level |
| ksqlDB | Transform and reserialize streams in real time |
| AWS Glue ETL | Convert between formats in batch and streaming pipelines |
| Apache Flink | Custom serialization/deserialization in stream processing |
| Custom service | Application-level translation for complex mappings |

```
Schema Translation Pipeline
─────────────────────────────

  ┌──────────┐     ┌──────────┐     ┌──────────────┐     ┌──────────┐
  │ Source   │────▶│ Topic A  │────▶│ Stream       │────▶│ Topic B  │
  │ (Avro)  │     │ (Avro)   │     │ Processor    │     │ (JSON)   │
  └──────────┘     └──────────┘     │              │     └──────────┘
                                    │ - Deserialize│
                                    │   Avro       │
                                    │ - Transform  │
                                    │ - Serialize  │
                                    │   JSON       │
                                    └──────────────┘
```

**Translation guidelines:**

- **Test field mapping thoroughly** — subtle differences (e.g., null handling, default values) cause silent data loss
- **Monitor translation lag** — the translation service is a bottleneck if it falls behind
- **Plan for schema changes on both sides** — the translator must handle evolution in both source and target schemas
- **Consider data fidelity** — some types do not have exact equivalents across formats (e.g., Avro logical types vs JSON)

---

## Next Steps

Continue your messaging learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | Messaging Fundamentals | Core messaging concepts, delivery guarantees, event-driven architecture |
| [01-APACHE-KAFKA.md](01-APACHE-KAFKA.md) | Apache Kafka | Distributed event streaming, topics, partitions, consumer groups |
| [02-RABBITMQ.md](02-RABBITMQ.md) | RabbitMQ | Message queues, exchanges, bindings, AMQP protocol |
| [03-CLOUD-SERVICES.md](03-CLOUD-SERVICES.md) | Cloud Messaging Services | AWS SQS/SNS, Azure Service Bus, Google Cloud Pub/Sub |
| [04-PATTERNS.md](04-PATTERNS.md) | Messaging Patterns | Pub/sub, request-reply, saga, event sourcing, outbox pattern |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial schema management documentation |
