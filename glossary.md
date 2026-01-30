# Glossary

Brief definitions of libraries and technologies. See [Tech Guide](./tech-guide.md) for detailed usage.

---

## GIS Stack

| Term | Definition |
|------|------------|
| **MapLibre** | Open-source JavaScript library for interactive vector maps (Mapbox GL JS fork) |
| **TiTiler** | High-performance tile server for Cloud-Optimized GeoTIFFs and raster formats |
| **PostGIS** | PostgreSQL extension adding geographic object support and spatial operations |
| **Martin** | Rust-based PostGIS vector tile server (10-100x faster than Node.js) |
| **DuckDB** | In-process SQL OLAP database with geospatial support for client-side analysis |
| **GeoTIFF** | TIFF format with geographic metadata for raster data |
| **Vector Tile** | Compressed protocol buffer format for map vector data at multiple zoom levels |
| **MVT** | MapBox Vector Tile format - binary protobuf encoding of vector features |
| **Web Mercator** | EPSG:3857 - standard projection for web mapping (spherical Mercator) |

---

## EdTech Stack

| Term | Definition |
|------|------------|
| **LlamaIndex** | Data framework for building LLM-powered applications with RAG support |
| **WebRTC** | Web Real-Time Communication API for peer-to-peer audio/video streaming |
| **FFmpeg** | Multimedia framework for video/audio encoding, transcoding, and streaming |
| **pgvector** | PostgreSQL extension for vector similarity search and embedding storage |
| **RAG** | Retrieval Augmented Generation - combining retrieval with LLM generation |
| **Embedding** | Dense vector representation of text/data for similarity search |
| **HLS** | HTTP Live Streaming - adaptive bitrate streaming protocol |
| **ICE** | Interactive Connectivity Establishment - NAT traversal for WebRTC |
| **STUN/TURN** | Servers for establishing peer-to-peer WebRTC connections |

---

## Backend Stack

| Term | Definition |
|------|------------|
| **Fastify** | High-performance, low-overhead Node.js web framework with schema validation |
| **PostgreSQL** | Enterprise open-source relational database with ACID compliance |
| **S3** | AWS Simple Storage Service - scalable object storage with API access |
| **Redis** | In-memory data structure store for caching, sessions, pub/sub |
| **Socket.IO** | Library for real-time bidirectional event-based communication |
| **ACID** | Atomicity, Consistency, Isolation, Durability - database transaction properties |
| **Connection Pool** | Reusable set of database connections to avoid connection overhead |
| **Prepared Statement** | Pre-compiled SQL query template for repeated execution efficiency |
| **Pub/Sub** | Publish/Subscribe pattern for messaging between services |

---

## Frontend Stack

| Term | Definition |
|------|------------|
| **Next.js** | React framework with server-side rendering, static generation, API routes |
| **React** | JavaScript library for building user interfaces with component-based architecture |
| **TailwindCSS** | Utility-first CSS framework for rapid custom UI development |
| **SSR** | Server-Side Rendering - rendering React components on the server |
| **SSG** | Static Site Generation - pre-rendering pages at build time |
| **ISR** | Incremental Static Regeneration - updating static pages after build |
| **Code Splitting** | Separating JavaScript into smaller chunks for lazy loading |
| **Virtual DOM** | Lightweight in-memory representation of actual DOM for efficient updates |
| **Hydration** | Attaching event listeners to server-rendered HTML for interactivity |

---

## Infrastructure

| Term | Definition |
|------|------------|
| **Kubernetes** | Container orchestration platform for automated deployment, scaling, management |
| **Prometheus** | Open-source monitoring and alerting toolkit with pull-based metrics |
| **Grafana** | Analytics and visualization platform for monitoring dashboards |
| **Pod** | Smallest deployable unit in Kubernetes - one or more containers |
| **Deployment** | Kubernetes resource for managing replica sets of pods |
| **Service** | Kubernetes abstraction for network access to pods |
| **Ingress** | Kubernetes resource for external HTTP/HTTPS access to services |
| **HPA** | Horizontal Pod Autoscaler - automatically scales pod replicas |
| **ConfigMap** | Kubernetes API for storing non-confidential configuration data |
| **Secret** | Kubernetes API for storing sensitive configuration data |
| **ServiceMonitor** | Prometheus CRD for discovering services to scrape metrics |
| **PromQL** | Prometheus Query Language for querying time-series metrics |
| **SLO** | Service Level Objective - target reliability metric (e.g., 99.9% uptime) |
| **SLI** | Service Level Indicator - measurable metric of service level |

---

## Common Terms

| Term | Definition |
|------|------------|
| **API** | Application Programming Interface - contract for service interaction |
| **REST** | Representational State Transfer - architectural style for APIs |
| **gRPC** | Google Remote Procedure Call - high-performance RPC framework |
| **GraphQL** | Query language for APIs enabling clients to request specific data |
| **JWT** | JSON Web Token - compact, URL-safe token for authentication |
| **OAuth** | Open Authorization - protocol for delegated access tokens |
| **2FA** | Two-Factor Authentication - requiring two verification methods |
| **CI/CD** | Continuous Integration/Continuous Deployment - automated build and release |
| **Docker** | Platform for containerizing applications with consistent environments |
| **TLS** | Transport Layer Security - cryptographic protocol for secure communication |
| **UUID** | Universally Unique Identifier - 128-bit identifier for uniqueness |

---

## Quick Index

| Category | Key Terms |
|----------|-----------|
| **Maps/GIS** | MapLibre, TiTiler, PostGIS, Martin, DuckDB, GeoTIFF, Vector Tile |
| **AI/ML** | LlamaIndex, pgvector, Embedding, RAG |
| **Media** | WebRTC, FFmpeg, HLS |
| **Backend** | Fastify, PostgreSQL, Redis, Socket.IO |
| **Frontend** | Next.js, React, TailwindCSS |
| **Infra** | Kubernetes, Prometheus, Grafana |
| **Auth** | JWT, OAuth, 2FA |
| **DevOps** | CI/CD, Docker, TLS |

**For detailed implementation guides, see [Tech Guide](./tech-guide.md).**
