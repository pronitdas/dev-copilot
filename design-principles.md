# Design Principles

This document outlines our core design principles for both technical architecture and user experience. These aren't abstract guidelines—they're engineering imperatives that drive our decisions.

## Component Ownership

**One team, one component.**

Every component in our system has clear ownership with one team responsible for its:
- Performance
- Reliability
- Documentation
- Evolution

### Implementation

```
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  Component                                                 │
│  ┌──────────────────────────────┐                          │
│  │                              │                          │
│  │  Public Interface            │ ← Team responsibility    │
│  │                              │                          │
│  └──────────────────────────────┘                          │
│                                                            │
│  ┌──────────────────────────────┐                          │
│  │                              │                          │
│  │  Internal Implementation     │ ← Team autonomy          │
│  │                              │                          │
│  └──────────────────────────────┘                          │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Rules of Engagement

1. **Interface Stability**: Public interfaces must be versioned and backwards compatible
2. **Implementation Freedom**: Teams can refactor internal implementations without approval
3. **SLA Ownership**: Teams define and meet their own SLAs
4. **Cross-cutting Changes**: Require RFC and explicit approval from all affected owners

### Component Registry

All components are registered in `registry.json` with:
- Owning team
- Primary/secondary contacts
- SLA commitments
- Dependency graph

## Data Contracts

**Schema first, implementation second.**

All data exchange uses explicit contracts defined using:
- JSON Schema (for REST/JSON)
- Protocol Buffers (for gRPC)
- GraphQL Schema (for GraphQL)
- TypeScript interfaces (for frontend)

### Implementation

Every contract must:
1. Be version controlled
2. Include explicit versioning
3. Have validation tests
4. Document all fields

### Examples

**REST/JSON Schema:**
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
      "description": "Display name for the marker"
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

**Protocol Buffer:**
```protobuf
syntax = "proto3";

message GeoPosition {
  double latitude = 1;  // Latitude in decimal degrees (-90 to 90)
  double longitude = 2; // Longitude in decimal degrees (-180 to 180)
}

message Marker {
  string id = 1;        // UUID format
  string name = 2;      // Required, 1-100 characters
  GeoPosition position = 3; // Required
}
```

## API Design

**APIs are products, not implementations.**

We follow specific patterns for each API type:

### REST APIs

- Resource-oriented URLs
- Proper HTTP method usage
- Consistent pagination
- Hypermedia links where appropriate

**URL Patterns:**
```
Collection:    /v1/resources
Resource:      /v1/resources/{id}
Sub-resource:  /v1/resources/{id}/sub-resources
Query:         /v1/resources?param=value
```

**HTTP Status Codes:**
- 200: Success
- 201: Created
- 204: No Content
- 400: Bad Request
- 401: Unauthorized
- 403: Forbidden
- 404: Not Found
- 409: Conflict
- 429: Too Many Requests
- 500: Internal Server Error

### GraphQL

- Schema-first design
- Focused queries with selection sets
- Mutations follow command pattern
- Clear error handling

**Schema Pattern:**
```graphql
type Query {
  resource(id: ID!): Resource
  resources(filter: ResourceFilter, pagination: PaginationInput): ResourceConnection!
}

type ResourceConnection {
  edges: [ResourceEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type ResourceEdge {
  node: Resource!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

### gRPC

- Service definitions with clear RPCs
- Consistent error codes
- Streaming for large data transfers
- Binary efficiency

**Service Pattern:**
```protobuf
service ResourceService {
  // Unary RPC
  rpc GetResource(GetResourceRequest) returns (Resource);
  
  // Server streaming
  rpc ListResources(ListResourcesRequest) returns (stream Resource);
  
  // Client streaming
  rpc CreateResources(stream CreateResourceRequest) returns (CreateResourcesResponse);
  
  // Bidirectional streaming
  rpc ProcessResourceStream(stream ResourceOperation) returns (stream ResourceOperationResult);
}
```

## UX/3D Guidelines

**Performance first, aesthetics second.**

### Maps and Geospatial

- Render performance target: 60fps on mid-range devices
- Tile loading latency < 100ms (p95)
- Progressive loading for complex datasets
- Pre-rendered static tiles for common views

**MapLibre Style Structure:**
```json
{
  "version": 8,
  "name": "platform-style",
  "sources": {
    "basemap": {
      "type": "vector",
      "url": "mbtiles://{basemap}"
    },
    "data": {
      "type": "vector",
      "url": "mbtiles://{user_data}"
    }
  },
  "layers": [
    {
      "id": "background",
      "type": "background",
      "paint": {
        "background-color": "#f8f4f0"
      }
    },
    {
      "id": "landcover",
      "type": "fill",
      "source": "basemap",
      "source-layer": "landcover",
      "paint": {
        "fill-color": [
          "match",
          ["get", "type"],
          "forest", "#d4e6c6",
          "farmland", "#eaeaea",
          "#f5f5f5"
        ]
      }
    }
  ]
}
```

### Learning Models

- WebRTC connection setup < 1s
- First interactive frame < 2s
- Content layout optimized for information hierarchy
- Accessibility: WCAG 2.1 AA compliance

**3D Model Performance:**
- Max polygon count: 100k per scene
- Texture size: 2048x2048 max (1024x1024 preferred)
- LOD implementation for complex models
- GLTF 2.0 format with draco compression

## Theming and i18n

**Design system first, custom UI second.**

### Theme Structure

```
┌─────────────────────────┐
│ Theme                   │
├─────────────────────────┤
│ - Colors                │
│   - Primary             │
│   - Secondary           │
│   - Accent              │
│   - Background          │
│   - Text                │
├─────────────────────────┤
│ - Typography            │
│   - Font family         │
│   - Font sizes          │
│   - Line heights        │
├─────────────────────────┤
│ - Spacing               │
│   - Grid                │
│   - Insets              │
│   - Stack               │
├─────────────────────────┤
│ - Components            │
│   - Buttons             │
│   - Cards               │
│   - Forms               │
└─────────────────────────┘
```

### Component Structure

All UI components follow this pattern:
```tsx
interface ButtonProps {
  variant: 'primary' | 'secondary' | 'text';
  size: 'small' | 'medium' | 'large';
  disabled?: boolean;
  loading?: boolean;
  onClick?: () => void;
  children: React.ReactNode;
}

const Button: React.FC<ButtonProps> = ({
  variant = 'primary',
  size = 'medium',
  disabled = false,
  loading = false,
  onClick,
  children
}) => {
  // Implementation
};
```

### MapLibre Theming

Map styles follow a layered approach:
1. Base layer (terrain, water)
2. Reference layer (roads, boundaries)
3. Data layer (user data)
4. Interaction layer (highlights, selection)

Each has its own theme properties:
```json
{
  "map": {
    "base": {
      "water": "#c9e6ff",
      "land": "#f4f4f0",
      "forest": "#d4e6c6"
    },
    "reference": {
      "road": {
        "major": "#ffffff",
        "minor": "#f8f8f8",
        "stroke": "#e0e0e0"
      },
      "boundary": {
        "admin": "#8d8d8d",
        "disputed": "#d9d9d9"
      }
    },
    "data": {
      "sequential": [
        "#f7fbff", "#deebf7", "#c6dbef", 
        "#9ecae1", "#6baed6", "#4292c6", 
        "#2171b5", "#08519c", "#08306b"
      ],
      "categorical": [
        "#4e79a7", "#f28e2c", "#e15759", 
        "#76b7b2", "#59a14f", "#edc949", 
        "#af7aa1", "#ff9da7", "#9c755f", "#bab0ab"
      ]
    }
  }
}
```

### i18n Strategy

- All text uses translation keys
- No hardcoded strings
- RTL support for all components
- Number/date formatting follows locale

**Translation Structure:**
```json
{
  "en": {
    "common": {
      "buttons": {
        "submit": "Submit",
        "cancel": "Cancel",
        "save": "Save"
      }
    },
    "map": {
      "controls": {
        "zoom_in": "Zoom In",
        "zoom_out": "Zoom Out",
        "fullscreen": "Fullscreen"
      }
    }
  },
  "es": {
    "common": {
      "buttons": {
        "submit": "Enviar",
        "cancel": "Cancelar",
        "save": "Guardar"
      }
    },
    "map": {
      "controls": {
        "zoom_in": "Acercar",
        "zoom_out": "Alejar",
        "fullscreen": "Pantalla completa"
      }
    }
  }
}
```

## Next Steps

For implementation details, see:
- [Architecture Overview](./architecture-overview.md) for system design
- [Dev Guide](./dev-guide.md) for implementation guidelines
- [Component Library](http://localhost:6006) for UI component documentation
