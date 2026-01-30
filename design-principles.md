# Design Principles

Core engineering imperatives for technical architecture and user experience.

---

## 1. Component Ownership

**Rule:** One team, one component.

| Responsibility | Description |
|---------------|-------------|
| Performance | Owner's SLAs define response times |
| Reliability | 99.9% uptime minimum |
| Documentation | Public interfaces documented |
| Evolution | Internal implementation at team discretion |

### Boundaries

```
┌────────────────────────────────────────────────────────────┐
│ Component                                                  │
│ ┌──────────────────────────────┐                          │
│ │ Public Interface            │ ← Versioned, stable       │
│ └──────────────────────────────┘                          │
│ ┌──────────────────────────────┐                          │
│ │ Internal Implementation     │ ← Team autonomy           │
│ └──────────────────────────────┘                          │
└────────────────────────────────────────────────────────────┘
```

### Rules of Engagement

1. **Interface Stability** — Public interfaces versioned, backwards compatible
2. **Implementation Freedom** — Refactor internal code without approval
3. **SLA Ownership** — Teams define and meet their own SLAs
4. **Cross-cutting Changes** — Require RFC + owner approval

### Registry

All components registered in `registry.json`:
- Owning team, contacts, SLA commitments, dependency graph

---

## 2. Data Contracts

**Rule:** Schema first, implementation second.

| Format | Use Case |
|--------|----------|
| JSON Schema | REST/JSON APIs |
| Protocol Buffers | gRPC services |
| GraphQL Schema | GraphQL endpoints |
| TypeScript interfaces | Frontend code |

### Contract Requirements

- Version controlled with explicit versioning
- Validation tests for all fields
- Complete field documentation

### Examples

**REST/JSON:**
```json
{
  "type": "object",
  "required": ["id", "name", "position"],
  "properties": {
    "id": { "type": "string", "format": "uuid", "description": "Unique identifier" },
 "description": "    "name": { "type": "string", "minLength": 1, "maxLength": 100 },
    "position": {
      "type": "object",
      "required": ["latitude", "longitude"],
      "properties": {
        "latitude": { "type": "number", "minimum": -90, "maximum": 90 },
        "longitude": { "type": "number", "minimum": -180, "maximum": 180 }
      }
    }
  }
}
```

**Protocol Buffer:**
```protobuf
syntax = "proto3";

message GeoPosition {
  double latitude = 1;   // -90 to 90
  double longitude = 2;  // -180 to 180
}

message Marker {
  string id = 1;         // UUID format
  string name = 2;       // 1-100 characters
  GeoPosition position = 3;
}
```

---

## 3. API Design

**Rule:** APIs are products, not implementations.

### REST

| Pattern | URL |
|---------|-----|
| Collection | `/v1/resources` |
| Resource | `/v1/resources/{id}` |
| Sub-resource | `/v1/resources/{id}/sub` |
| Query | `/v1/resources?page=1` |

**Status Codes:**
- `200` Success, `201` Created, `204` No Content
- `400` Bad Request, `401` Unauthorized, `403` Forbidden
- `404` Not Found, `409` Conflict, `429` Rate Limited
- `500` Internal Server Error

### GraphQL

```graphql
type Query {
  resource(id: ID!): Resource
  resources(filter: Filter, pagination: Pagination): ResourceConnection!
}

type ResourceConnection {
  edges: [ResourceEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

### gRPC

```protobuf
service ResourceService {
  rpc GetResource(GetResourceRequest) returns (Resource);
  rpc ListResources(ListResourcesRequest) returns (stream Resource);
  rpc CreateResources(stream CreateResourceRequest) returns (Response);
}
```

---

## 4. UX/3D Guidelines

**Rule:** Performance first, aesthetics second.

### Maps & Geospatial

| Metric | Target |
|--------|--------|
| Frame rate | 60fps on mid-range devices |
| Tile latency | <100ms p95 |
| Polygon count | 100k per scene |
| Texture size | 2048x2048 max (1024x1024 preferred) |

**MapLibre Style Structure:**
```json
{
  "version": 8,
  "name": "platform-style",
  "sources": {
    "basemap": { "type": "vector", "url": "mbtiles://{basemap}" },
    "data": { "type": "vector", "url": "mbtiles://{user_data}" }
  },
  "layers": [
    { "id": "background", "type": "background", "paint": { "background-color": "#f8f4f0" } }
  ]
}
```

### Learning Models

- WebRTC connection <1s
- First interactive frame <2s
- WCAG 2.1 AA compliance required
- GLTF 2.0 format with draco compression

---

## 5. Theming & i18n

**Rule:** Design system first, custom UI second.

### Theme Structure

```
Theme
├── Colors (primary, secondary, accent, background, text)
├── Typography (family, sizes, line heights)
├── Spacing (grid, insets, stack)
└── Components (buttons, cards, forms)
```

### Component Pattern

```tsx
interface ButtonProps {
  variant: 'primary' | 'secondary' | 'text';
  size: 'small' | 'medium' | 'large';
  disabled?: boolean;
  loading?: boolean;
  onClick?: () => void;
  children: React.ReactNode;
}
```

### i18n Structure

```json
{
  "en": {
    "common": { "buttons": { "submit": "Submit", "cancel": "Cancel" } },
    "map": { "controls": { "zoom_in": "Zoom In", "zoom_out": "Zoom Out" } }
  },
  "es": {
    "common": { "buttons": { "submit": "Enviar", "cancel": "Cancelar" } },
    "map": { "controls": { "zoom_in": "Acercar", "zoom_out": "Alejar" } }
  }
}
```

### Rules

- All text uses translation keys
- No hardcoded strings
- RTL support for all components
- Number/date formatting follows locale

---

## Quick Reference

| Principle | Key Rule |
|-----------|----------|
| Ownership | One team, one component |
| Contracts | Schema first |
| APIs | Products, not implementations |
| UX/3D | Performance first |
| Theming | Design system first |

**See Also:**
- [Dev Guide](./dev-guide.md) — Implementation guidelines
- [Glossary](./glossary.md) — Technology definitions
