# Data Contract Standards

## Schema-First Design

**Schema first, implementation second.**

All data exchange uses explicit contracts:

- JSON Schema (for REST/JSON)
- Protocol Buffers (for gRPC)
- GraphQL Schema (for GraphQL)
- TypeScript interfaces (for frontend)

## Contract Requirements

Every contract must:

1. Be version controlled
2. Include explicit versioning
3. Have validation tests
4. Document all fields

## JSON Schema Example

```json
{
  "type": "object",
  "required": ["id", "name", "position"],
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid",
      "description": "Unique identifier"
    },
    "name": {
      "type": "string",
      "minLength": 1,
      "maxLength": 100,
      "description": "Display name"
    },
    "position": {
      "type": "object",
      "required": ["latitude", "longitude"],
      "properties": {
        "latitude": {
          "type": "number",
          "minimum": -90,
          "maximum": 90,
          "description": "Latitude in decimal degrees"
        },
        "longitude": {
          "type": "number",
          "minimum": -180,
          "maximum": 180,
          "description": "Longitude in decimal degrees"
        }
      }
    }
  }
}
```

## Protocol Buffer Example

```protobuf
syntax = "proto3";

message GeoPosition {
  double latitude = 1;
  double longitude = 2;
}

message Marker {
  string id = 1;
  string name = 2;
  GeoPosition position = 3;
}
```

## Validation Implementation

- Use `zod`, `joi`, or equivalent validation at API boundaries
- Implement runtime type checking in addition to TypeScript
- Write tests that explicitly verify schema adherence
- Log schema violations as critical errors

## Enforcement

- PR checks automatically verify schema compliance
- Schema regression tests run on every commit
- Schema violations in production trigger alerts
