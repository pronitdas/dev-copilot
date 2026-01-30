# Technology Guide

Detailed guides for all technologies in our stack.

---

## GIS Stack

### MapLibre

**What:** Open-source JavaScript library for interactive vector maps (Mapbox GL JS fork).

**Why:**
- Complete styling control, no licensing costs
- High-performance vector tile rendering
- 3D terrain capabilities, custom WebGL layers

**Basic Implementation:**
```javascript
import { Map } from 'maplibre-gl';

const map = new Map({
  container: 'map-container',
  style: '/assets/styles/default.json',
  center: [-122.4194, 37.7749],
  zoom: 12
});

map.on('load', () => {
  map.addSource('data-source', {
    type: 'vector',
    tiles: ['/api/tiles/{z}/{x}/{y}.mvt'],
    minzoom: 0,
    maxzoom: 14
  });
  
  map.addLayer({
    id: 'data-layer',
    type: 'fill',
    source: 'data-source',
    'source-layer': 'layer-name',
    paint: {
      'fill-color': ['interpolate', ['linear'], ['get', 'value'], 0, '#f7fbff', 100, '#08306b'],
      'fill-opacity': 0.7
    }
  });
});
```

**Performance:**
```javascript
// Pre-load critical tiles
const criticalBounds = [[-123, 37], [-122, 38]];
map.fitBounds(criticalBounds, { padding: 0, animate: false, duration: 0 });

// Switch to raster at low zoom
map.on('zoom', () => {
  if (map.getZoom() < 10) {
    map.setLayoutProperty('vector-layer', 'visibility', 'none');
  }
});
```

---

### TiTiler

**What:** High-performance tile server for Cloud-Optimized GeoTIFFs.

**Why:**
- On-the-fly tile generation
- Built-in optimization for large datasets
- Scalable cloud deployment

**Deployment:**
```yaml
version: '3'
services:
  titiler:
    image: ghcr.io/developmentseed/titiler:latest
    environment:
      - CPL_TMPDIR=/tmp
      - GDAL_CACHEM      - VSI_CACHE=TRUE
   AX=200
 ports:
      - "8000:8000"
```

**Custom Endpoints:**
```python
from fastapi import FastAPI
from titiler.core.factory import TilerFactory

app = FastAPI()
cog = TilerFactory()
app.include_router(cog.router, prefix="/cog")
```

---

### PostGIS

**What:** PostgreSQL spatial database extension.

**Why:**
- Industry standard for spatial databases
- Rich spatial functions, transaction support
- High-performance spatial indexing

**Spatial Queries:**
```sql
-- Find points within 5km
SELECT id, name, ST_AsGeoJSON(geom)
FROM points
WHERE ST_DWithin(
  geom,
  ST_SetSRID(ST_MakePoint(-73.9857, 40.7484), 4326)::geography,
  5000
);

-- Generate vector tiles
SELECT ST_AsMVT(q, 'layer_name', 4096, 'geom')
FROM (
  SELECT id, name, ST_AsMVTGeom(
    ST_Transform(geom, 3857),
    ST_TileEnvelope(z, x, y),
    4096, 256, true
  ) as geom
  FROM spatial_table
  WHERE ST_Intersects(geom, ST_Transform(ST_TileEnvelope(z, x, y), 4326))
) as q;
```

**Performance:**
```sql
CREATE INDEX spatial_table_geom_idx ON spatial_table USING GIST (geom);
CREATE INDEX spatial_table_webmerc_idx ON spatial_table USING GIST (ST_Transform(geom, 3857));
CLUSTER spatial_table USING spatial_table_geom_idx;
ANALYZE spatial_table;
```

---

### Martin

**What:** Rust-based PostGIS vector tile server.

**Why:**
- 10-100x faster than Node.js alternatives
- Low memory footprint, simple config

**Config:**
```toml
[database]
host = "localhost"
port = 5432
user = "postgres"
password = "password"
dbname = "gis_db"
pool_size = 20

[[table]]
name = "public.spatial_table"
id_column = "id"
geometry_column = "geom"
srid = 4326
```

**MapLibre Integration:**
```javascript
map.addSource('vector-source', {
  type: 'vector',
  tiles: ['http://localhost:3000/{z}/{x}/{y}/public.spatial_table.mvt'],
  minzoom: 0,
  maxzoom: 14
});
```

---

### DuckDB

**What:** In-process SQL OLAP database with geospatial support.

**Why:**
- Lightning-fast in-memory queries
- Columnar storage, direct GeoParquet processing
- Minimal resource footprint

**Browser Setup:**
```javascript
import * as duckdb from '@duckdb/duckdb-wasm';

async function initDuckDB() {
  const bundle = await duckdb.selectBundle(duckdb.getJsDelivrBundles());
  const worker = new Worker(bundle.mainWorker);
  const db = new duckdb.AsyncDuckDB(new duckdb.ConsoleLogger(), worker);
  await db.instantiate(bundle.mainModule);
  return db;
}

const db = await initDuckDB();
const conn = await db.connect();
await conn.query('INSTALL spatial; LOAD spatial;');
```

**Spatial Query:**
```sql
SELECT category, COUNT(*) as count, SUM(ST_Area(geometry)) / 1000000 as area_km2
FROM geodata
GROUP BY category
ORDER BY area_km2 DESC;
```

---

## EdTech Stack

### LlamaIndex

**What:** Data framework for LLM-powered applications.

**Why:**
- Structured RAG approach
- Diverse data source support
- Query planning and routing

**Basic Setup:**
```python
from llama_index import VectorStoreIndex, SimpleDirectoryReader
from llama_index.vector_stores import PGVectorStore

documents = SimpleDirectoryReader("./data").load_data()

vector_store = PGVectorStore.from_params(
    database="postgres",
    host="localhost",
    password="postgres",
    port=5432,
    user="postgres",
    table_name="document_embeddings",
    embed_dim=1536
)

index = VectorStoreIndex.from_documents(documents, vector_store=vector_store)
query_engine = index.as_query_engine()
response = query_engine.query("What is the capital of France?")
```

---

### WebRTC

**What:** Real-time communication API for browsers.

**Why:**
- Low-latency video/audio streaming
- Peer-to-peer architecture
- Native browser support, encrypted by default

**Peer Connection:**
```javascript
const configuration = {
  iceServers: [
    { urls: 'stun:stun.example.org:3478' },
    { urls: 'turn:turn.example.org:3478', username: 'user', credential: 'pass' }
  ]
};

const peerConnection = new RTCPeerConnection(configuration);

localStream.getTracks().forEach(track => {
  peerConnection.addTrack(track, localStream);
});

peerConnection.ontrack = event => {
  remoteVideo.srcObject = event.streams[0];
};

peerConnection.onicecandidate = event => {
  if (event.candidate) {
    signalServer.send(JSON.stringify({ type: 'candidate', candidate: event.candidate }));
  }
};
```

**Optimize for Low Latency:**
```javascript
const videoConstraints = { width: { ideal: 640 }, height: { ideal: 480 }, frameRate: { min: 15, ideal: 30 } };
navigator.mediaDevices.getUserMedia({ video: videoConstraints, audio: audioConstraints });
```

---

### FFmpeg

**What:** Multimedia framework for encoding, transcoding, streaming.

**Why:**
- Industry standard, versatile
- Hardware acceleration support
- All media formats

**Transcoding:**
```javascript
import { spawn } from 'child_process';

function transcodeVideo(inputPath, outputPath, options = {}) {
  return new Promise((resolve, reject) => {
    const args = ['-i', inputPath, '-c:v', options.videoCodec || 'libx264',
      '-c:a', options.audioCodec || 'aac', '-b:v', options.videoBitrate || '1M',
      '-b:a', options.audioBitrate || '128k', outputPath];
    
    const ffmpeg = spawn('ffmpeg', args);
    ffmpeg.on('close', (code) => code === 0 ? resolve() : reject());
  });
}
```

**HLS Streaming:**
```bash
ffmpeg -i input.mp4 \
  -profile:v main -sc_threshold 0 -g 48 -keyint_min 48 \
  -c:v libx264 -b:v:0 5M -s:v:0 1920x1080 \
  -c:v libx264 -b:v:1 3M -s:v:1 1280x720 \
  -c:a aac -b:a 192k \
  -f hls -hls_time 6 -hls_list_size 0 \
  -var_stream_map "v:0,a:0 v:1,a:0" \
  output.m3u8
```

---

### pgvector

**What:** PostgreSQL extension for vector similarity search.

**Why:**
- Integrates with existing PostgreSQL
- High-performance similarity searches
- IVF and HNSW indexing support

**Setup:**
```sql
CREATE EXTENSION vector;

CREATE TABLE embeddings (
  id SERIAL PRIMARY KEY,
  content TEXT NOT NULL,
  embedding VECTOR(1536) NOT NULL,
  metadata JSONB
);

CREATE INDEX ON embeddings USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);
```

**Usage:**
```javascript
async function storeEmbedding(content, embedding) {
  await pool.query(
    'INSERT INTO embeddings (content, embedding) VALUES ($1, $2) RETURNING id',
    [content, embedding]
  );
}

async function findSimilar(embedding, limit = 5) {
  const result = await pool.query(`
    SELECT id, content, 1 - (embedding <=> $1) as similarity
    FROM embeddings
    ORDER BY embedding <=> $1
    LIMIT $2
  `, [embedding, limit]);
  return result.rows;
}
```

---

## Backend Stack

### Fastify

**What:** High-performance Node.js web framework.

**Why:**
- Extremely fast, low overhead
- Schema-based validation
- Robust plugin architecture

**Basic Server:**
```typescript
import fastify from 'fastify';

const server = fastify({ logger: true });

server.post('/users', {
  schema: {
    body: { name: { type: 'string' }, email: { type: 'string', format: 'email' } },
    response: { 200: { id: { type: 'string' }, success: { type: 'boolean' } } }
  }
}, async (request, reply) => {
  const { name, email } = request.body;
  const id = await createUser({ name, email });
  return { id, success: true };
});

server.listen({ port: 3000 }, (err) => {
  if (err) { console.error(err); process.exit(1); }
  console.log('Server listening on port 3000');
});
```

**Plugins:**
```typescript
server.register(fastifyJwt, { secret: process.env.JWT_SECRET });
server.register(fastifyCors, { origin: ['https://example.com'] });

server.addHook('onRequest', async (request) => {
  request.log.info(`Request: ${request.method} ${request.url}`);
});
```

---

### PostgreSQL

**What:** Enterprise-class open-source relational database.

**Why:**
- ACID-compliant, advanced indexing
- Rich data types, robust replication
- Extensible (PostGIS, pgvector)

**Connection Pool:**
```typescript
import { Pool } from 'pg';

const pool = new Pool({
  host: process.env.POSTGRES_HOST,
  port: 5432,
  user: process.env.POSTGRES_USER,
  password: process.env.POSTGRES_PASSWORD,
  database: process.env.POSTGRES_DB,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000
});

async function query(text: string, params: any[] = []) {
  const start = Date.now();
  const result = await pool.query(text, params);
  if (Date.now() - start > 200) console.warn(`Slow query: ${text}`);
  return result;
}
```

**Transaction:**
```typescript
async function executeTransaction<T>(callback: (client: PoolClient) => Promise<T>): Promise<T> {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const result = await callback(client);
    await client.query('COMMIT');
    return result;
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

---

### S3

**What:** Object storage service (AWS S3 or MinIO).

**Why:**
- Highly scalable, cost-effective
- Built-in versioning, lifecycle policies
- High durability

**Client Setup:**
```typescript
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3Client = new S3Client({
  region: process.env.AWS_REGION,
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY
  },
  endpoint: process.env.S3_ENDPOINT,
  forcePathStyle: true
});

async function uploadFile(bucket: string, key: string, body: Buffer, contentType: string) {
  await s3Client.send(new PutObjectCommand({ Bucket: bucket, Key: key, Body: body, ContentType: contentType }));
}

async function getDownloadUrl(bucket: string, key: string, expiresIn = 3600) {
  return getSignedUrl(s3Client, new GetObjectCommand({ Bucket: bucket, Key: key }), { expiresIn });
}
```

**Multipart Upload:**
```typescript
import { Upload } from '@aws-sdk/lib-storage';

const upload = new Upload({
  client: s3Client,
  params: { Bucket: bucket, Key: key, Body: fileStream },
  partSize: 10 * 1024 * 1024,
  leavePartsOnError: false
});

upload.on('httpUploadProgress', (progress) => {
  console.log(`Uploaded ${progress.loaded} of ${progress.total}`);
});

await upload.done();
```

---

### Redis

**What:** In-memory data structure store.

**Why:**
- Extremely fast in-memory operations
- Pub/sub messaging, distributed locking
- Perfect for caching, sessions

**Setup:**
```typescript
import { createClient } from 'redis';

const redisClient = createClient({
  url: process.env.REDIS_URL,
  socket: { reconnectStrategy: (retries) => Math.min(retries * 50, 3000) }
});

redisClient.on('error', (err) => console.error('Redis error:', err));
await redisClient.connect();
```

**Caching:**
```typescript
async function setCache(key: string, value: any, ttlSeconds = 3600) {
  await redisClient.set(key, JSON.stringify(value), { EX: ttlSeconds });
}

async function getCache<T>(key: string): Promise<T | null> {
  const value = await redisClient.get(key);
  return value ? JSON.parse(value) : null;
}

async function withCache<T>(key: string, fn: () => Promise<T>, ttl = 3600): Promise<T> {
  const cached = await getCache<T>(key);
  if (cached) return cached;
  const result = await fn();
  await setCache(key, result, ttl);
  return result;
}
```

**Pub/Sub:**
```typescript
const publisher = redisClient.duplicate();
const subscriber = redisClient.duplicate();
await Promise.all([publisher.connect(), subscriber.connect()]);

await subscriber.subscribe('notifications', (message) => {
  console.log('Received:', JSON.parse(message));
});

await publisher.publish('notifications', JSON.stringify({ type: 'alert', data: '...' }));
```

---

### Socket.IO

**What:** Real-time bidirectional event-based communication.

**Why:**
- Automatic fallback, reconnection logic
- Room-based broadcasting
- Cross-browser compatibility

**Server:**
```typescript
import { Server } from 'socket.io';

const io = new Server(httpServer, {
  cors: { origin: ['https://example.com'], credentials: true },
  transports: ['websocket', 'polling']
});

io.on('connection', (socket) => {
  console.log(`User connected: ${socket.data.user.id}`);
  
  socket.on('join-room', (roomId) => {
    socket.join(roomId);
    socket.to(roomId).emit('user-joined', { userId: socket.data.user.id });
  });
  
  socket.on('send-message', ({ roomId, message }) => {
    io.to(roomId).emit('new-message', { userId: socket.data.user.id, message });
  });
  
  socket.on('disconnect', () => {
    console.log(`User disconnected`);
  });
});

httpServer.listen(3000);
```

---

## Frontend Stack

### Next.js

**What:** React framework with SSR, SSG, API routes.

**Why:**
- Server-side rendering for SEO
- Built-in API routes, ISR
- Automatic code splitting, image optimization

**API Route:**
```typescript
import { NextResponse } from 'next/server';
import { z } from 'zod';

const userSchema = z.object({ name: z.string().min(2), email: z.string().email() });

export async function POST(request: Request) {
  try {
    const body = await request.json();
    const user = userSchema.parse(body);
    return NextResponse.json({ user }, { status: 201 });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json({ error: 'Invalid data', details: error.errors }, { status: 400 });
    }
    return NextResponse.json({ error: 'Internal Server Error' }, { status: 500 });
  }
}
```

**Server Component:**
```typescript
import { getUsers } from '@/lib/data';

export default async function UsersPage() {
  const users = await getUsers();
  return (
    <div>
      <h1>Users</h1>
      <ul>{users.map(user => <li key={user.id}>{user.name}</li>)}</ul>
    </div>
  );
}
```

**Lazy Loading:**
```typescript
import { lazy, Suspense } from 'react';

const HeavyComponent = lazy(() => import('./HeavyComponent'));

export default function LazyComponent() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <HeavyComponent />
    </Suspense>
  );
}
```

---

### React

**What:** JavaScript library for building user interfaces.

**Why:**
- Component-based architecture
- Virtual DOM for efficient updates
- Large ecosystem, TypeScript support

**Component Pattern:**
```tsx
type ButtonVariant = 'primary' | 'secondary' | 'text';
type ButtonSize = 'small' | 'medium' | 'large';

interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: ButtonVariant;
  size?: ButtonSize;
  isLoading?: boolean;
  leftIcon?: React.ReactNode;
}

export const Button: React.FC<ButtonProps> = ({
  variant = 'primary', size = 'medium', isLoading, leftIcon, children, className, disabled, ...rest
}) => {
  return (
    <button className={`btn btn-${variant} btn-${size} ${className || ''}`} disabled={disabled || isLoading} {...rest}>
      {isLoading && <span className="loader" />}
      {leftIcon && <span className="left-icon">{leftIcon}</span>}
      {children}
    </button>
  );
};
```

**Custom Hook:**
```typescript
export function useLocalStorage<T>(key: string, initialValue: T): [T, (value: T) => void] {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch { return initialValue; }
  });

  const setValue = (value: T) => {
    try {
      setStoredValue(value instanceof Function ? value(storedValue) : value);
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch (error) { console.warn(error); }
  };

  return [storedValue, setValue];
}
```

**Performance:**
```typescript
import React, { memo, useCallback, useMemo } from 'react';

const Item = memo(({ id, name, onSelect }: ItemProps) => {
  const handleClick = useCallback(() => onSelect(id), [id, onSelect]);
  return <div onClick={handleClick}>{name}</div>;
});

export const List: React.FC<ListProps> = ({ items, onSelectItem }) => {
  const handleSelect = useCallback((id: string) => onSelectItem(id), [onSelectItem]);
  const sortedItems = useMemo(() => [...items].sort((a, b) => a.name.localeCompare(b.name)), [items]);
  return <div>{sortedItems.map(item => <Item key={item.id} {...item} onSelect={handleSelect} />)}</div>;
};
```

---

### TailwindCSS

**What:** Utility-first CSS framework.

**Why:**
- Rapid UI development
- Consistent design system
- Minimal CSS overhead

**Config:**
```javascript
module.exports = {
  content: ['./src/**/*.{js,ts,jsx,tsx}'],
  theme: {
    extend: {
      colors: {
        primary: { 50: '#f0f9ff', 500: '#0ea5e9', 900: '#0c4a6e' },
        secondary: { 100: '#e0f2fe', 800: '#075985' }
      },
      fontFamily: { sans: ['Inter', 'sans-serif'] }
    }
  },
  plugins: [require('@tailwindcss/forms'), require('@tailwindcss/typography')]
};
```

**Component:**
```tsx
interface CardProps { title?: string; description?: string; image?: string; children?: React.ReactNode; }

export const Card: React.FC<CardProps> = ({ title, description, image, children, className = '' }) => {
  return (
    <div className={`bg-white rounded-lg shadow-md overflow-hidden ${className}`}>
      {image && <div className="relative h-48 w-full"><img src={image} alt={title} className="absolute h-full w-full object-cover" /></div>}
      <div className="p-4">
        {title && <h3 className="text-lg font-medium text-gray-900">{title}</h3>}
        {description && <p className="mt-1 text-sm text-gray-500">{description}</p>}
        {children && <div className="mt-4">{children}</div>}
      </div>
    </div>
  );
};
```

**Dark Mode:**
```tsx
export const ThemeToggle: React.FC = () => {
  const { theme, setTheme } = useTheme();
  return (
    <button onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}
      className="rounded-full p-2 hover:bg-gray-100 dark:hover:bg-gray-800">
      {theme === 'dark' ? '☀️' : '🌙'}
    </button>
  );
};
```

---

## Infrastructure

### Kubernetes

**What:** Container orchestration platform.

**Why:**
- Declarative configuration
- Automated scaling, recovery
- Service discovery, load balancing

**Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  namespace: platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-service
  template:
    metadata:
      labels:
        app: api-service
    spec:
      containers:
      - name: api
        image: registry.example.com/api-service:v1.2.3
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
```

**Service & Ingress:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api-service
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-cert
```

**HPA:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

### Prometheus

**What:** Open-source monitoring and alerting toolkit.

**Why:**
- Pull-based metrics collection
- Powerful PromQL
- Service discovery integration

**Custom Metrics:**
```typescript
import prometheus from 'prom-client';

const register = new prometheus.Registry();
prometheus.collectDefaultMetrics({ register });

const httpRequestDuration = new prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5]
});

const httpRequestTotal = new prometheus.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status_code']
});

register.registerMetric(httpRequestDuration);
register.registerMetric(httpRequestTotal);

export const httpMetricsMiddleware = (req, res, next) => {
  const end = httpRequestDuration.startTimer();
  res.on('finish', () => {
    const route = req.route?.path || req.path;
    end({ method: req.method, route, status_code: res.statusCode });
    httpRequestTotal.inc({ method: req.method, route, status_code: res.statusCode });
  });
  next();
};

export const metricsHandler = async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
};
```

**PromQL Examples:**
```promql
# Error rate over 5%
sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100 > 5

# 95th percentile latency
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# Memory pressure
container_memory_usage_bytes / container_spec_memory_limit_bytes * 100 > 85
```

---

### Grafana

**What:** Analytics and visualization platform.

**Why:**
- Powerful visualizations
- Multi-source support
- Alerting, dashboards

**Data Source Config:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus:9090
      isDefault: true
    - name: Loki
      type: loki
      url: http://loki:3100
```

**Alert Notification:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-notifications
data:
  notifications.yaml: |
    apiVersion: 1
    notifiers:
    - name: Slack
      type: slack
      uid: slack1
      org_id: 1
      is_default: true
      settings:
        url: https://hooks.slack.com/services/TXXXXX/BXXXXX/XXXXXXX
        recipient: "#alerts"
```

---

## Quick Reference

| Technology | Purpose | Key Benefit |
|------------|---------|-------------|
| MapLibre | Vector maps | Complete styling control |
| TiTiler | Tile server | On-the-fly generation |
| PostGIS | Spatial DB | Rich spatial functions |
| Martin | Vector tiles | 10-100x faster than Node |
| DuckDB | In-process OLAP | Lightning-fast queries |
| LlamaIndex | LLM framework | Structured RAG |
| WebRTC | Real-time comm | Low-latency streaming |
| FFmpeg | Media processing | Hardware acceleration |
| pgvector | Vector search | PostgreSQL integration |
| Fastify | Web framework | Extreme performance |
| PostgreSQL | Relational DB | ACID compliant |
| S3 | Object storage | Highly scalable |
| Redis | In-memory store | Fast caching |
| Socket.IO | Real-time | Bidirectional events |
| Next.js | React framework | SSR/SSG support |
| React | UI library | Virtual DOM |
| TailwindCSS | Styling | Utility-first |
| Kubernetes | Orchestration | Auto-scaling |
| Prometheus | Monitoring | Pull-based metrics |
| Grafana | Visualization | Dashboards |

**See Also:**
- [Design Principles](./design-principles.md) — Architecture decisions
- [Fundamentals](./fundamentals.md) — Engineering standards
- [Dev Guide](./dev-guide.md) — Workflow
- [Glossary](./glossary.md) — Term definitions
