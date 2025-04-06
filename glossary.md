jwt-secret: <base64-encoded-value>
```

StatefulSets for databases:
```yaml
# postgres-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: platform
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14-alpine
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: password
        - name: POSTGRES_DB
          value: platform
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 4Gi
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 5
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: standard
      resources:
        requests:
          storage: 100Gi
```

Custom resource definitions (CRDs):
```yaml
# certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-certificate
  namespace: platform
spec:
  secretName: api-tls-cert
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - api.example.com
  - api-staging.example.com
```

Jobs and CronJobs:
```yaml
# cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
  namespace: platform
spec:
  schedule: "0 2 * * *"  # Every day at 2am
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: registry.example.com/db-backup:v1.0.0
            env:
            - name: DB_HOST
              value: postgres
            - name: DB_NAME
              value: platform
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: password
            - name: S3_BUCKET
              value: platform-backups
            - name: S3_PREFIX
              value: postgres
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: backup-secrets
                  key: aws-access-key
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: backup-secrets
                  key: aws-secret-key
          restartPolicy: OnFailure
```

Network policies:
```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: platform
spec:
  podSelector:
    matchLabels:
      app: api-service
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

Custom Prometheus metrics:
```yaml
# prometheus-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api-service-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: api-service
  endpoints:
  - port: metrics
    interval: 15s
    path: /metrics

---
# service-with-metrics.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: platform
  labels:
    app: api-service
spec:
  selector:
    app: api-service
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: metrics
    port: 9090
    targetPort: 9090
  type: ClusterIP
```

**Internal Resources:**
- [Kubernetes Architecture Guide](/docs/infrastructure/kubernetes-architecture.md)
- [Deployment Pipeline](/docs/infrastructure/ci-cd-pipeline.md)
- [Cluster Management](/docs/infrastructure/cluster-management.md)

### Prometheus

**What:** Prometheus is an open-source systems monitoring and alerting toolkit designed for reliability and scalability.

**Why:**
- Pull-based metrics collection
- Powerful query language (PromQL)
- Time-series data model
- Service discovery integration
- Rich visualization with Grafana

**How to Use:**

Basic setup:
```yaml
# prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
      
    scrape_configs:
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
        - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https
          
      - job_name: 'kubernetes-nodes'
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        kubernetes_sd_configs:
        - role: node
        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics
          
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name
```

Alert rules:
```yaml
# prometheus-alerts.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-alerts
  namespace: monitoring
data:
  alerts.yml: |
    groups:
    - name: kubernetes
      rules:
      - alert: KubernetesPodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[5m]) > 0
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"
          description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is restarting {{ $value }} times every 5 minutes"
          
      - alert: KubernetesPodNotReady
        expr: sum by (namespace, pod) (kube_pod_status_phase{phase=~"Pending|Unknown"}) > 0
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} not ready"
          description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has been in non-ready state for more than 15 minutes"
          
      - alert: KubernetesContainerOOMKilled
        expr: kube_pod_container_status_terminated_reason{reason="OOMKilled"} == 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Container {{ $labels.container }} in pod {{ $labels.namespace }}/{{ $labels.pod }} OOM killed"
          description: "Container {{ $labels.container }} in pod {{ $labels.namespace }}/{{ $labels.pod }} was OOM killed"
          
    - name: node
      rules:
      - alert: HostHighCPULoad
        expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Host {{ $labels.instance }} high CPU load"
          description: "CPU load is > 80% for more than 10 minutes"
          
      - alert: HostLowDiskSpace
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Host {{ $labels.instance }} low disk space"
          description: "Disk space is less than 10% on {{ $labels.device }} mount at {{ $labels.instance }}"
          
      - alert: HostMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Host {{ $labels.instance }} high memory usage"
          description: "Memory usage is > 90% for more than 5 minutes"
```

Custom application metrics:
```typescript
// metrics.ts
import prometheus from 'prom-client';

// Create a Registry
const register = new prometheus.Registry();

// Add default metrics
prometheus.collectDefaultMetrics({ register });

// Define custom metrics
const httpRequestDuration = new prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10]
});

const httpRequestTotal = new prometheus.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code']
});

const databaseQueryDuration = new prometheus.Histogram({
  name: 'database_query_duration_seconds',
  help: 'Duration of database queries in seconds',
  labelNames: ['query_name', 'success'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5]
});

const activeSessions = new prometheus.Gauge({
  name: 'active_sessions',
  help: 'Number of currently active sessions'
});

// Register metrics
register.registerMetric(httpRequestDuration);
register.registerMetric(httpRequestTotal);
register.registerMetric(databaseQueryDuration);
register.registerMetric(activeSessions);

// Middleware for HTTP metrics
export const httpMetricsMiddleware = (req, res, next) => {
  const end = httpRequestDuration.startTimer();
  
  res.on('finish', () => {
    const route = req.route ? req.route.path : req.path;
    const method = req.method;
    const statusCode = res.statusCode;
    
    end({ method, route, status_code: statusCode });
    httpRequestTotal.inc({ method, route, status_code: statusCode });
  });
  
  next();
};

// Database query timer
export const measureDatabaseQuery = async (queryName, queryFn) => {
  const end = databaseQueryDuration.startTimer();
  
  try {
    const result = await queryFn();
    end({ query_name: queryName, success: 'true' });
    return result;
  } catch (error) {
    end({ query_name: queryName, success: 'false' });
    throw error;
  }
};

// Session tracking
export const trackSession = {
  create: () => {
    activeSessions.inc();
  },
  end: () => {
    activeSessions.dec();
  }
};

// Metrics endpoint
export const metricsHandler = async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
};
```

PromQL examples:
```
# High error rate (over 5%)
sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100 > 5

# Slow queries (95th percentile over 500ms)
histogram_quantile(0.95, sum(rate(database_query_duration_seconds_bucket[5m])) by (le, query_name)) > 0.5

# Memory pressure
container_memory_usage_bytes{namespace="platform"} / container_spec_memory_limit_bytes{namespace="platform"} * 100 > 85

# Latency SLO (99% of requests under 300ms)
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{route="/api/v1/important"}[5m])) by (le)) < 0.3

# Service apdex score (satisfaction score)
(
  sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m]))
  +
  sum(rate(http_request_duration_seconds_bucket{le="1.2"}[5m])) / 2
) / sum(rate(http_request_duration_seconds_count[5m]))

# Node disk pressure prediction (will run out in 4 hours)
predict_linear(node_filesystem_avail_bytes[6h], 4 * 3600) < 0
```

**Internal Resources:**
- [Monitoring Architecture](/docs/infrastructure/monitoring-architecture.md)
- [Custom Metrics Guide](/docs/backend/application-metrics.md)
- [Alert Management](/docs/infrastructure/alerting-strategy.md)

### Grafana

**What:** Grafana is an open-source analytics and visualization platform that integrates with various data sources to create dashboards and alerts.

**Why:**
- Powerful visualization
- Support for multiple data sources
- Alert functionality
- User-friendly dashboard creation
- Customizable and extensible

**How to Use:**

Basic deployment:
```yaml
# grafana.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - containerPort: 3000
          name: http
        env:
        - name: GF_SECURITY_ADMIN_USER
          valueFrom:
            secretKeyRef:
              name: grafana-secrets
              key: admin-user
        - name: GF_SECURITY_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: grafana-secrets
              key: admin-password
        - name: GF_AUTH_ANONYMOUS_ENABLED
          value: "false"
        volumeMounts:
        - name: grafana-storage
          mountPath: /var/lib/grafana
        - name: grafana-datasources
          mountPath: /etc/grafana/provisioning/datasources
        - name: grafana-dashboards-config
          mountPath: /etc/grafana/provisioning/dashboards
        - name: grafana-dashboards
          mountPath: /var/lib/grafana/dashboards
      volumes:
      - name: grafana-storage
        persistentVolumeClaim:
          claimName: grafana-pvc
      - name: grafana-datasources
        configMap:
          name: grafana-datasources
      - name: grafana-dashboards-config
        configMap:
          name: grafana-dashboards-config
      - name: grafana-dashboards
        configMap:
          name: grafana-dashboards
```

Data source configuration:
```yaml
# grafana-datasources.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  datasources.yaml: |
    apiVersion: 1
    
    datasources:
    - name: Prometheus
      type: prometheus
      access: proxy
      url: http://prometheus:9090
      isDefault: true
      editable: false
      
    - name: Loki
      type: loki
      access: proxy
      url: http://loki:3100
      editable: false
      
    - name: PostgreSQL
      type: postgres
      access: proxy
      url: postgres:5432
      database: metrics
      user: grafana
      secureJsonData:
        password: ${POSTGRES_PASSWORD}
      jsonData:
        sslmode: "disable"
      editable: false
```

Dashboard provisioning:
```yaml
# grafana-dashboards-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards-config
  namespace: monitoring
data:
  dashboards.yaml: |
    apiVersion: 1
    
    providers:
    - name: 'default'
      orgId: 1
      folder: ''
      type: file
      disableDeletion: false
      editable: true
      options:
        path: /var/lib/grafana/dashboards
```

Example dashboard JSON:
```yaml
# grafana-dashboards.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: monitoring
data:
  api-dashboard.json: |
    {
      "annotations": {
        "list": [
          {
            "builtIn": 1,
            "datasource": "-- Grafana --",
            "enable": true,
            "hide": true,
            "iconColor": "rgba(0, 211, 255, 1)",
            "name": "Annotations & Alerts",
            "type": "dashboard"
          }
        ]
      },
      "editable": true,
      "gnetId": null,
      "graphTooltip": 0,
      "id": 1,
      "links": [],
      "panels": [
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "Prometheus",
          "fieldConfig": {
            "defaults": {
              "unit": "short"
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 0
          },
          "hiddenSeries": false,
          "id": 2,
          "legend": {
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "show": true,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.5.7",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "expr": "sum(rate(http_requests_total[5m])) by (route)",
              "interval": "",
              "legendFormat": "{{route}}",
              "refId": "A"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "HTTP Request Rate",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "short",
              "label": "Requests / Second",
              "logBase": 1,
              "max": null,
              "min": "0",
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        }
      ],
      "refresh": "10s",
      "schemaVersion": 27,
      "style": "dark",
      "tags": [],
      "templating": {
        "list": []
      },
      "time": {
        "from": "now-6h",
        "to": "now"
      },
      "timepicker": {},
      "timezone": "",
      "title": "API Service Dashboard",
      "uid": "api-service",
      "version": 1
    }
```

Alert configuration:
```yaml
# grafana-alerts.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-notifications
  namespace: monitoring
data:
  notifications.yaml: |
    apiVersion: 1
    
    notifiers:
    - name: Slack
      type: slack
      uid: slack1
      org_id: 1
      is_default: true
      send_reminder: true
      frequency: 1h
      disable_resolve_message: false
      settings:
        url: https://hooks.slack.com/services/TXXXXX/BXXXXX/XXXXXXX
        recipient: "#alerts"
        mentionChannel: "here"
        token: XYZABC
        iconEmoji: ":ghost:"
        username: Grafana
        
    - name: PagerDuty
      type: pagerduty
      uid: pd1
      org_id: 1
      is_default: false
      send_reminder: false
      disable_resolve_message: false
      settings:
        integrationKey: XXXXXXXXX
```

**Internal Resources:**
- [Grafana Dashboard Guide](/docs/infrastructure/grafana-dashboards.md)
- [Alert Configuration](/docs/infrastructure/grafana-alerts.md)
- [Dashboard Templates](/infrastructure/monitoring/dashboards)

---

## Next Steps

Now that you have an overview of our technology stack, consider these next steps:

1. Set up your [Local Development Environment](/docs/local-setup.md)
2. Explore the [Architecture Overview](/docs/architecture-overview.md)
3. Review the [Design Principles](/docs/design-principles.md)
4. Follow the [Development Guide](/docs/dev-guide.md)
5. Check out our [API Documentation](http://localhost:8080/api-docs)
6. Join the #dev-help channel on Slack for assistance

For any questions or clarifications, contact the platform team at platform@example.com.
// Start server
httpServer.listen(3000, () => {
  console.log('Server listening on port 3000');
});
```

Scaling with Redis adapter:
```typescript
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

// Create Redis clients for adapter
const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();

await Promise.all([pubClient.connect(), subClient.connect()]);

// Create adapter
io.adapter(createAdapter(pubClient, subClient));

// Now broadcasts will work across multiple Socket.IO servers
io.on('connection', (socket) => {
  // This will reach all clients across all Socket.IO servers
  io.emit('global-event', { data: 'message' });
});
```

Performance optimization:
```typescript
// Configure for better performance
const io = new Server(httpServer, {
  // Prefer websockets to reduce latency
  transports: ['websocket'],
  
  // Increase ping timeout for unreliable networks
  pingTimeout: 60000,
  
  // Disable per-event acks to reduce overhead
  perMessageDeflate: false,
  
  // Increase maxHttpBufferSize for larger payloads
  maxHttpBufferSize: 1e6, // 1MB
  
  // Use binary for more efficient transfer
  serveClient: false // Don't serve client library
});

// Compress and batch messages on server
import msgpack from 'socket.io-msgpack-parser';
io.parser = msgpack;

// Rate limiting
const rateLimitMap = new Map();

io.use((socket, next) => {
  const userId = socket.data.user.id;
  const now = Date.now();
  
  if (!rateLimitMap.has(userId)) {
    rateLimitMap.set(userId, {
      count: 1,
      firstRequest: now,
      lastRequest: now
    });
    return next();
  }
  
  const limit = rateLimitMap.get(userId);
  
  // Reset counter after 1 minute
  if (now - limit.firstRequest > 60000) {
    rateLimitMap.set(userId, {
      count: 1,
      firstRequest: now,
      lastRequest: now
    });
    return next();
  }
  
  // Allow maximum 100 events per minute
  if (limit.count >= 100) {
    return next(new Error('Rate limit exceeded'));
  }
  
  // Update rate limit info
  limit.count += 1;
  limit.lastRequest = now;
  
  next();
});
```

Implementing namespaces:
```typescript
// Create namespaced socket servers for different features
const chatNamespace = io.of('/chat');
const editorNamespace = io.of('/editor');
const presenceNamespace = io.of('/presence');

// Configure different authentication and handlers for each namespace
chatNamespace.use(chatAuthMiddleware);
chatNamespace.on('connection', handleChatConnection);

editorNamespace.use(editorAuthMiddleware);
editorNamespace.on('connection', handleEditorConnection);

// Implement real-time presence tracking
presenceNamespace.on('connection', (socket) => {
  const { user } = socket.data;
  
  // Track user online status
  const presenceKey = `presence:${user.id}`;
  pubClient.set(presenceKey, 'online', { EX: 60 });
  
  // Set up presence heartbeat
  const heartbeatInterval = setInterval(() => {
    pubClient.set(presenceKey, 'online', { EX: 60 });
  }, 30000);
  
  // Handle disconnect
  socket.on('disconnect', () => {
    clearInterval(heartbeatInterval);
    pubClient.set(presenceKey, 'offline');
  });
});
```

**Internal Resources:**
- [Socket.IO Architecture Guide](/docs/backend/socketio-architecture.md)
- [Real-time Event System](/services/core/src/realtime)
- [Scaling WebSocket Servers](/docs/infrastructure/websocket-scaling.md)

## Frontend Stack

### Next.js

**What:** Next.js is a React framework that enables server-side rendering, static site generation, API routes, and more, providing a comprehensive solution for React applications.

**Why:**
- Server-side rendering for better SEO and performance
- Built-in API routes
- Incremental Static Regeneration for dynamic content
- Automatic code splitting
- Image optimization

**How to Use:**

Basic project structure:
```
next-app/
├── public/              # Static assets
├── src/
│   ├── app/             # App router components
│   │   ├── layout.tsx   # Root layout
│   │   ├── page.tsx     # Home page
│   │   └── [...]/       # Other routes
│   ├── components/      # Reusable components
│   │   ├── ui/          # UI components
│   │   └── feature/     # Feature components
│   ├── lib/             # Utility functions
│   ├── styles/          # Global styles
│   └── types/           # TypeScript types
├── next.config.js       # Next.js configuration
└── package.json         # Dependencies
```

API routes:
```typescript
// src/app/api/users/route.ts
import { NextResponse } from 'next/server';
import { z } from 'zod';

// Schema validation
const userSchema = z.object({
  name: z.string().min(2),
  email: z.string().email()
});

export async function POST(request: Request) {
  try {
    // Parse and validate request body
    const body = await request.json();
    const validatedData = userSchema.parse(body);
    
    // Create user in database
    const user = await createUser(validatedData);
    
    return NextResponse.json({ user }, { status: 201 });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { error: 'Invalid data', details: error.errors },
        { status: 400 }
      );
    }
    
    return NextResponse.json(
      { error: 'Internal Server Error' },
      { status: 500 }
    );
  }
}

export async function GET(request: Request) {
  // Get query parameters
  const url = new URL(request.url);
  const page = parseInt(url.searchParams.get('page') || '1');
  const limit = parseInt(url.searchParams.get('limit') || '10');
  
  // Fetch users from database
  const { users, total } = await getUsers(page, limit);
  
  return NextResponse.json({
    users,
    pagination: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit)
    }
  });
}
```

Data fetching:
```typescript
// src/app/users/page.tsx
import { getUsers } from '@/lib/data';

// Server component with fetch
export default async function UsersPage() {
  // This runs on the server
  const users = await getUsers();
  
  return (
    <div>
      <h1>Users</h1>
      <ul>
        {users.map(user => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>
    </div>
  );
}

// src/app/users/[id]/page.tsx
import { getUser } from '@/lib/data';
import { notFound } from 'next/navigation';

// Dynamic route with params
export default async function UserPage({ params }: { params: { id: string } }) {
  try {
    const user = await getUser(params.id);
    
    return (
      <div>
        <h1>{user.name}</h1>
        <p>Email: {user.email}</p>
      </div>
    );
  } catch (error) {
    // Handle user not found
    notFound();
  }
}
```

Performance optimization:
```typescript
// src/components/LazyComponent.tsx
import { Suspense, lazy } from 'react';

// Lazy load heavy components
const HeavyComponent = lazy(() => import('./HeavyComponent'));

export default function LazyComponent() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <HeavyComponent />
    </Suspense>
  );
}

// src/app/blog/[slug]/page.tsx
import { getBlogPost } from '@/lib/data';

// Generate static pages at build time
export async function generateStaticParams() {
  const posts = await getBlogPosts();
  
  return posts.map(post => ({
    slug: post.slug
  }));
}

// Enable Incremental Static Regeneration
export const revalidate = 3600; // Revalidate every hour

export default async function BlogPost({ params }: { params: { slug: string } }) {
  const post = await getBlogPost(params.slug);
  
  return (
    <article>
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  );
}
```

**Internal Resources:**
- [Next.js Architecture Guide](/docs/frontend/nextjs-architecture.md)
- [SSR vs. SSG Decision Tree](/docs/frontend/rendering-strategies.md)
- [Performance Optimization Checklist](/docs/frontend/nextjs-performance.md)

### React

**What:** React is a JavaScript library for building user interfaces, particularly single-page applications where dynamic and responsive UI is critical.

**Why:**
- Component-based architecture
- Virtual DOM for efficient updates
- Large ecosystem and community
- Declarative programming model
- Seamless integration with TypeScript

**How to Use:**

Component patterns:
```tsx
// src/components/ui/Button/Button.tsx
import React from 'react';
import styles from './Button.module.css';

type ButtonVariant = 'primary' | 'secondary' | 'text';
type ButtonSize = 'small' | 'medium' | 'large';

interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: ButtonVariant;
  size?: ButtonSize;
  isLoading?: boolean;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
}

export const Button: React.FC<ButtonProps> = ({
  variant = 'primary',
  size = 'medium',
  isLoading = false,
  leftIcon,
  rightIcon,
  children,
  className,
  disabled,
  ...rest
}) => {
  return (
    <button
      className={`${styles.button} ${styles[variant]} ${styles[size]} ${className || ''}`}
      disabled={disabled || isLoading}
      {...rest}
    >
      {isLoading && <span className={styles.loader} />}
      {leftIcon && <span className={styles.leftIcon}>{leftIcon}</span>}
      <span className={styles.content}>{children}</span>
      {rightIcon && <span className={styles.rightIcon}>{rightIcon}</span>}
    </button>
  );
};
```

Custom hooks:
```tsx
// src/hooks/useLocalStorage.ts
import { useState, useEffect } from 'react';

export function useLocalStorage<T>(key: string, initialValue: T): [T, (value: T) => void] {
  // Get from local storage then
  // parse stored json or return initialValue
  const readValue = (): T => {
    // Prevent build error "window is undefined" but keep working
    if (typeof window === 'undefined') {
      return initialValue;
    }

    try {
      const item = window.localStorage.getItem(key);
      return item ? (JSON.parse(item) as T) : initialValue;
    } catch (error) {
      console.warn(`Error reading localStorage key "${key}":`, error);
      return initialValue;
    }
  };

  // State to store our value
  // Pass initial state function to useState so logic is only executed once
  const [storedValue, setStoredValue] = useState<T>(readValue);

  // Return a wrapped version of useState's setter function that
  // persists the new value to localStorage.
  const setValue = (value: T) => {
    try {
      // Allow value to be a function so we have same API as useState
      const valueToStore =
        value instanceof Function ? value(storedValue) : value;
      
      // Save state
      setStoredValue(valueToStore);
      
      // Save to local storage
      if (typeof window !== 'undefined') {
        window.localStorage.setItem(key, JSON.stringify(valueToStore));
      }
    } catch (error) {
      console.warn(`Error setting localStorage key "${key}":`, error);
    }
  };

  useEffect(() => {
    setStoredValue(readValue());
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  return [storedValue, setValue];
}
```

Context with reducers:
```tsx
// src/context/AuthContext.tsx
import React, { createContext, useContext, useReducer, useEffect } from 'react';
import { User } from '@/types';

// Define state type
interface AuthState {
  user: User | null;
  isLoading: boolean;
  error: string | null;
}

// Define actions
type AuthAction =
  | { type: 'LOGIN_REQUEST' }
  | { type: 'LOGIN_SUCCESS'; payload: User }
  | { type: 'LOGIN_FAILURE'; payload: string }
  | { type: 'LOGOUT' };

// Define context type
interface AuthContextType {
  state: AuthState;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
}

// Create context
const AuthContext = createContext<AuthContextType | undefined>(undefined);

// Reducer function
function authReducer(state: AuthState, action: AuthAction): AuthState {
  switch (action.type) {
    case 'LOGIN_REQUEST':
      return {
        ...state,
        isLoading: true,
        error: null
      };
    case 'LOGIN_SUCCESS':
      return {
        ...state,
        isLoading: false,
        user: action.payload,
        error: null
      };
    case 'LOGIN_FAILURE':
      return {
        ...state,
        isLoading: false,
        user: null,
        error: action.payload
      };
    case 'LOGOUT':
      return {
        ...state,
        user: null,
        error: null
      };
    default:
      return state;
  }
}

// Create provider
export const AuthProvider: React.FC<React.PropsWithChildren<{}>> = ({ children }) => {
  const [state, dispatch] = useReducer(authReducer, {
    user: null,
    isLoading: false,
    error: null
  });

  // Check for stored token on mount
  useEffect(() => {
    const checkAuth = async () => {
      const token = localStorage.getItem('auth_token');
      if (token) {
        try {
          dispatch({ type: 'LOGIN_REQUEST' });
          const user = await validateToken(token);
          dispatch({ type: 'LOGIN_SUCCESS', payload: user });
        } catch (error) {
          localStorage.removeItem('auth_token');
          dispatch({ type: 'LOGIN_FAILURE', payload: 'Invalid token' });
        }
      }
    };
    
    checkAuth();
  }, []);

  // Auth methods
  const login = async (email: string, password: string) => {
    try {
      dispatch({ type: 'LOGIN_REQUEST' });
      
      // Call API
      const response = await fetch('/api/auth/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email, password })
      });
      
      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.message || 'Login failed');
      }
      
      const { user, token } = await response.json();
      
      // Store token
      localStorage.setItem('auth_token', token);
      
      dispatch({ type: 'LOGIN_SUCCESS', payload: user });
    } catch (error) {
      dispatch({ 
        type: 'LOGIN_FAILURE', 
        payload: error instanceof Error ? error.message : 'Login failed' 
      });
    }
  };

  const logout = () => {
    localStorage.removeItem('auth_token');
    dispatch({ type: 'LOGOUT' });
  };

  return (
    <AuthContext.Provider value={{ state, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};

// Create hook for easy context use
export const useAuth = (): AuthContextType => {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
};

// Helper function to validate token
async function validateToken(token: string): Promise<User> {
  const response = await fetch('/api/auth/validate', {
    headers: { Authorization: `Bearer ${token}` }
  });
  
  if (!response.ok) {
    throw new Error('Invalid token');
  }
  
  return response.json();
}
```

Performance optimization:
```tsx
import React, { memo, useCallback, useMemo } from 'react';

interface ItemProps {
  id: string;
  name: string;
  onSelect: (id: string) => void;
}

// Memoize component to prevent unnecessary re-renders
const Item = memo(({ id, name, onSelect }: ItemProps) => {
  // Memoize callback to prevent recreation on every render
  const handleClick = useCallback(() => {
    onSelect(id);
  }, [id, onSelect]);
  
  console.log(`Rendering item ${id}`);
  
  return (
    <div onClick={handleClick}>
      {name}
    </div>
  );
});

interface ListProps {
  items: Array<{ id: string; name: string }>;
  onSelectItem: (id: string) => void;
}

export const List: React.FC<ListProps> = ({ items, onSelectItem }) => {
  // Memoize callback at parent level
  const handleSelect = useCallback((id: string) => {
    onSelectItem(id);
  }, [onSelectItem]);
  
  // Memoize expensive computation
  const sortedItems = useMemo(() => {
    console.log('Sorting items');
    return [...items].sort((a, b) => a.name.localeCompare(b.name));
  }, [items]);
  
  return (
    <div>
      {sortedItems.map(item => (
        <Item 
          key={item.id}
          id={item.id}
          name={item.name}
          onSelect={handleSelect}
        />
      ))}
    </div>
  );
};
```

**Internal Resources:**
- [React Component Architecture](/docs/frontend/react-architecture.md)
- [State Management Patterns](/docs/frontend/state-management.md)
- [Performance Optimization Guide](/docs/frontend/react-performance.md)

### TailwindCSS

**What:** Tailwind CSS is a utility-first CSS framework that allows you to build custom designs without leaving your HTML.

**Why:**
- Rapid UI development
- Consistent design system
- Minimal CSS overhead with purging
- Responsive design out of the box
- Highly customizable

**How to Use:**

Basic setup:
```javascript
// tailwind.config.js
module.exports = {
  content: [
    './src/**/*.{js,ts,jsx,tsx}',
  ],
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#f0f9ff',
          100: '#e0f2fe',
          200: '#bae6fd',
          300: '#7dd3fc',
          400: '#38bdf8',
          500: '#0ea5e9',
          600: '#0284c7',
          700: '#0369a1',
          800: '#075985',
          900: '#0c4a6e',
        },
        secondary: {
          // Custom colors
        },
      },
      spacing: {
        '128': '32rem',
      },
      fontFamily: {
        sans: ['Inter', 'sans-serif'],
        display: ['Lexend', 'sans-serif'],
      },
    },
  },
  plugins: [
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography'),
  ],
}
```

Component patterns:
```tsx
// src/components/ui/Card.tsx
import React from 'react';

interface CardProps {
  title?: string;
  description?: string;
  image?: string;
  children?: React.ReactNode;
  className?: string;
}

export const Card: React.FC<CardProps> = ({
  title,
  description,
  image,
  children,
  className = '',
}) => {
  return (
    <div className={`bg-white rounded-lg shadow-md overflow-hidden ${className}`}>
      {image && (
        <div className="relative h-48 w-full">
          <img
            src={image}
            alt={title || 'Card image'}
            className="absolute h-full w-full object-cover"
          />
        </div>
      )}
      <div className="p-4">
        {title && <h3 className="text-lg font-medium text-gray-900">{title}</h3>}
        {description && <p className="mt-1 text-sm text-gray-500">{description}</p>}
        {children && <div className="mt-4">{children}</div>}
      </div>
    </div>
  );
};
```

Custom utilities:
```css
/* src/styles/globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer components {
  /* Custom component styles */
  .btn {
    @apply inline-flex items-center justify-center rounded-md border border-transparent px-4 py-2 text-sm font-medium shadow-sm focus:outline-none focus:ring-2 focus:ring-offset-2;
  }
  
  .btn-primary {
    @apply bg-primary-600 text-white hover:bg-primary-700 focus:ring-primary-500;
  }
  
  .btn-secondary {
    @apply bg-secondary-100 text-secondary-800 hover:bg-secondary-200 focus:ring-secondary-500;
  }
  
  .input {
    @apply block w-full rounded-md border-gray-300 shadow-sm focus:border-primary-500 focus:ring-primary-500 sm:text-sm;
  }
}

@layer utilities {
  /* Custom utility classes */
  .text-shadow {
    text-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  }
  
  .text-shadow-lg {
    text-shadow: 0 4px 8px rgba(0, 0, 0, 0.12);
  }
}
```

Responsive design:
```tsx
// src/components/ui/ResponsiveGrid.tsx
import React from 'react';

interface ResponsiveGridProps {
  children: React.ReactNode;
  cols?: {
    xs?: number;
    sm?: number;
    md?: number;
    lg?: number;
    xl?: number;
  };
  gap?: string;
  className?: string;
}

export const ResponsiveGrid: React.FC<ResponsiveGridProps> = ({
  children,
  cols = { xs: 1, sm: 2, md: 3, lg: 4, xl: 5 },
  gap = '4',
  className = '',
}) => {
  const getGridCols = () => {
    return [
      cols.xs && `grid-cols-${cols.xs}`,
      cols.sm && `sm:grid-cols-${cols.sm}`,
      cols.md && `md:grid-cols-${cols.md}`,
      cols.lg && `lg:grid-cols-${cols.lg}`,
      cols.xl && `xl:grid-cols-${cols.xl}`,
    ].filter(Boolean).join(' ');
  };

  return (
    <div className={`grid ${getGridCols()} gap-${gap} ${className}`}>
      {children}
    </div>
  );
};
```

Dark mode support:
```tsx
// src/components/ui/ThemeToggle.tsx
import React from 'react';
import { useTheme } from '@/hooks/useTheme';

export const ThemeToggle: React.FC = () => {
  const { theme, setTheme } = useTheme();
  
  return (
    <button
      type="button"
      className="rounded-full p-2 text-gray-500 hover:bg-gray-100 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-primary-500 dark:text-gray-400 dark:hover:bg-gray-800"
      onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}
    >
      {theme === 'dark' ? (
        <svg className="h-5 w-5" fill="currentColor" viewBox="0 0 20 20">
          {/* Sun icon */}
          <path
            fillRule="evenodd"
            d="M10 2a1 1 0 011 1v1a1 1 0 11-2 0V3a1 1 0 011-1zm4 8a4 4 0 11-8 0 4 4 0 018 0zm-.464 4.95l.707.707a1 1 0 001.414-1.414l-.707-.707a1 1 0 00-1.414 1.414zm2.12-10.607a1 1 0 010 1.414l-.706.707a1 1 0 11-1.414-1.414l.707-.707a1 1 0 011.414 0zM17 11a1 1 0 100-2h-1a1 1 0 100 2h1zm-7 4a1 1 0 011 1v1a1 1 0 11-2 0v-1a1 1 0 011-1zM5.05 6.464A1 1 0 106.465 5.05l-.708-.707a1 1 0 00-1.414 1.414l.707.707zm1.414 8.486l-.707.707a1 1 0 01-1.414-1.414l.707-.707a1 1 0 011.414 1.414zM4 11a1 1 0 100-2H3a1 1 0 000 2h1z"
            clipRule="evenodd"
          />
        </svg>
      ) : (
        <svg className="h-5 w-5" fill="currentColor" viewBox="0 0 20 20">
          {/* Moon icon */}
          <path d="M17.293 13.293A8 8 0 016.707 2.707a8.001 8.001 0 1010.586 10.586z" />
        </svg>
      )}
    </button>
  );
};
```

**Internal Resources:**
- [Tailwind Best Practices](/docs/frontend/tailwind-best-practices.md)
- [Design System Implementation](/docs/frontend/design-system.md)
- [Component Library](/components)

## Infrastructure

### Kubernetes

**What:** Kubernetes is an open-source platform for automating deployment, scaling, and operations of application containers.

**Why:**
- Declarative configuration
- Automated scaling and recovery
- Efficient resource utilization
- Service discovery and load balancing
- Robust secret and configuration management

**How to Use:**

Basic deployment:
```yaml
# deployment.yaml
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
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        env:
        - name: NODE_ENV
          value: production
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: database-url
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
```

Service and ingress:
```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: platform
spec:
  selector:
    app: api-service
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP

---
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: platform
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
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

Horizontal pod autoscaling:
```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-service-hpa
  namespace: platform
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
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

ConfigMaps and Secrets:
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: platform
data:
  cache-ttl: "3600"
  feature-flags: |
    {
      "enableNewFeature": true,
      "enableBetaFeature": false
    }
  log-level: "info"

---
# secret.yaml (Never commit actual secrets to Git)
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
  namespace: platform
type: Opaque
data:
  database-url: <base64-encoded-value>
  api-key: <base64-encoded-value>
  jwt# Library Glossary

This document provides a detailed breakdown of all libraries and technologies used in our stack. Each entry includes what the technology is, why we use it, how to use it effectively, and links to internal resources.

## Table of Contents

- [GIS Stack](#gis-stack)
  - [MapLibre](#maplibre)
  - [TiTiler](#titiler)
  - [PostGIS](#postgis)
  - [Martin](#martin)
  - [DuckDB](#duckdb)
- [EdTech Stack](#edtech-stack)
  - [LlamaIndex](#llamaindex)
  - [WebRTC](#webrtc)
  - [FFmpeg](#ffmpeg)
  - [pgvector](#pgvector)
- [Backend Stack](#backend-stack)
  - [Fastify](#fastify)
  - [PostgreSQL](#postgresql)
  - [S3](#s3)
  - [Redis](#redis)
  - [Socket.IO](#socketio)
- [Frontend Stack](#frontend-stack)
  - [Next.js](#nextjs)
  - [React](#react)
  - [TailwindCSS](#tailwindcss)
- [Infrastructure](#infrastructure)
  - [Kubernetes](#kubernetes)
  - [Prometheus](#prometheus)
  - [Grafana](#grafana)

---

## GIS Stack

### MapLibre

**What:** MapLibre GL JS is an open-source JavaScript library for interactive, customizable vector maps on the web. It's a fork of Mapbox GL JS that's fully independent and community-driven.

**Why:** 
- Complete control over styling and rendering
- No commercial licensing costs
- High-performance vector tile rendering
- 3D terrain capabilities
- Custom layer support for WebGL integration

**How to Use:**

Basic implementation:
```javascript
import { Map } from 'maplibre-gl';

const map = new Map({
  container: 'map-container',
  style: '/assets/styles/default.json',
  center: [-122.4194, 37.7749],
  zoom: 12
});

// Add custom layers
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
      'fill-color': ['interpolate', ['linear'], ['get', 'value'], 
        0, '#f7fbff',
        100, '#08306b'
      ],
      'fill-opacity': 0.7
    }
  });
});
```

Performance optimization:
```javascript
// Pre-load critical tiles
const criticalBounds = [[-123, 37], [-122, 38]];
map.fitBounds(criticalBounds, { padding: 0, animate: false, duration: 0 });

// Wait for tiles to load, then restore view
map.once('idle', () => {
  map.fitBounds(actualBounds, { padding: 20, animate: true });
});

// Use raster tiles at low zoom for performance
map.on('zoom', () => {
  const zoom = map.getZoom();
  if (zoom < 10) {
    map.setLayoutProperty('vector-layer', 'visibility', 'none');
    map.setLayoutProperty('raster-layer', 'visibility', 'visible');
  } else {
    map.setLayoutProperty('vector-layer', 'visibility', 'visible');
    map.setLayoutProperty('raster-layer', 'visibility', 'none');
  }
});
```

**Internal Resources:**
- [Vector Tile Style Guide](/docs/gis/vector-tile-style-guide.md)
- [Custom Layer Examples](/libs/gis/src/layers)
- [Performance Optimization Guide](/docs/gis/maplibre-performance.md)

### TiTiler

**What:** TiTiler is a high-performance tile server that dynamically generates map tiles from Cloud-Optimized GeoTIFFs and other geospatial formats.

**Why:**
- On-the-fly tile generation
- Built-in optimization for large datasets
- Advanced raster operations (colorizing, hillshading)
- Scalable architecture for cloud deployment
- Rapid serving of user-uploaded data

**How to Use:**

Basic deployment:
```yaml
# docker-compose.yml
version: '3'
services:
  titiler:
    image: ghcr.io/developmentseed/titiler:latest
    environment:
      - CPL_TMPDIR=/tmp
      - GDAL_CACHEMAX=200
      - VSI_CACHE=TRUE
      - VSI_CACHE_SIZE=5000000
      - PYTHONWARNINGS=ignore
    volumes:
      - ./data:/data
    ports:
      - "8000:8000"
```

Custom endpoints:
```python
# custom_titiler.py
from fastapi import FastAPI, Depends
from titiler.core.factory import TilerFactory
from titiler.core.dependencies import PathParams
from titiler.mosaic.factory import MosaicTilerFactory

app = FastAPI()

cog = TilerFactory()
app.include_router(cog.router, prefix="/cog")

mosaic = MosaicTilerFactory()
app.include_router(mosaic.router, prefix="/mosaic")

@app.get("/custom/{z}/{x}/{y}.png")
def custom_tile(
    tile_params: PathParams = Depends(),
):
    # Custom tile generation logic
    return custom_tile_function(tile_params.z, tile_params.x, tile_params.y)
```

**Internal Resources:**
- [TiTiler Deployment Guide](/docs/gis/titiler-deployment.md)
- [Custom TiTiler Extensions](/services/gis/tile-server)
- [Raster Processing Pipeline](/libs/gis/src/raster)

### PostGIS

**What:** PostGIS is a spatial database extension for PostgreSQL that adds support for geographic objects and spatial operations.

**Why:**
- Industry standard for spatial databases
- Rich set of spatial functions
- Transaction support for data integrity
- Integrated with our existing PostgreSQL infrastructure
- High-performance spatial indexing

**How to Use:**

Basic spatial queries:
```sql
-- Find all points within 5km of a location
SELECT id, name, ST_AsGeoJSON(geom) as geometry
FROM points
WHERE ST_DWithin(
  geom,
  ST_SetSRID(ST_MakePoint(-73.9857, 40.7484), 4326)::geography,
  5000
);

-- Calculate area of polygons
SELECT id, name, ST_Area(geom::geography) / 10000 as area_hectares
FROM polygons;
```

Vector tile generation:
```sql
-- Generate vector tiles on the fly
SELECT ST_AsMVT(q, 'layer_name', 4096, 'geom')
FROM (
  SELECT 
    id, 
    name,
    property_value,
    ST_AsMVTGeom(
      ST_Transform(geom, 3857),
      ST_TileEnvelope(z, x, y),
      4096,
      256,
      true
    ) as geom
  FROM spatial_table
  WHERE ST_Intersects(
    geom,
    ST_Transform(ST_TileEnvelope(z, x, y), 4326)
  )
) as q;
```

Performance optimization:
```sql
-- Create spatial index
CREATE INDEX spatial_table_geom_idx ON spatial_table USING GIST (geom);

-- Create functional index for transformed geometries
CREATE INDEX spatial_table_webmerc_idx ON spatial_table 
USING GIST (ST_Transform(geom, 3857));

-- Cluster table by spatial index
CLUSTER spatial_table USING spatial_table_geom_idx;

-- Analyze for query planner
ANALYZE spatial_table;
```

**Internal Resources:**
- [PostGIS Query Optimization](/docs/databases/postgis-optimization.md)
- [Spatial Index Management](/scripts/db/manage-indexes.sh)
- [Vector Tile Generation](/services/gis/src/tiles)

### Martin

**What:** Martin is a PostGIS vector tile server written in Rust, designed for high-performance serving of vector tiles directly from a PostgreSQL/PostGIS database.

**Why:**
- Extremely high performance (10-100x faster than Node.js alternatives)
- Low memory footprint
- Direct PostgreSQL connection
- Simple configuration
- Perfect for self-hosted tile servers

**How to Use:**

Configuration:
```toml
# config.toml
[database]
host = "localhost"
port = 5432
user = "postgres"
password = "password"
dbname = "gis_db"
pool_size = 20

[webserver]
port = 3000
bind = "0.0.0.0"
cors = true
worker_processes = 8

[[table]]
name = "public.spatial_table"
id_column = "id"
geometry_column = "geom"
srid = 4326
extent = 4096
buffer = 64
clip_geom = true
geometry_type = "GEOMETRY"
```

Docker deployment:
```yaml
# docker-compose.yml
version: '3'
services:
  martin:
    image: urbica/martin:latest
    environment:
      - DATABASE_URL=postgres://postgres:postgres@postgres:5432/gis_db
      - WORKER_PROCESSES=8
      - POOL_SIZE=20
    ports:
      - "3000:3000"
    depends_on:
      - postgres
```

Usage with MapLibre:
```javascript
// Add vector tile source from Martin
map.addSource('vector-source', {
  type: 'vector',
  tiles: ['http://localhost:3000/{z}/{x}/{y}/public.spatial_table.mvt'],
  minzoom: 0,
  maxzoom: 14
});
```

**Internal Resources:**
- [Martin Configuration Guide](/docs/gis/martin-config.md)
- [Performance Tuning](/docs/gis/martin-performance.md)
- [Kubernetes Deployment](/infrastructure/k8s/martin)

### DuckDB

**What:** DuckDB is an in-process SQL OLAP database management system with advanced analytical capabilities, including geospatial support.

**Why:**
- In-memory processing for lightning-fast queries
- Columnar storage for efficient analytics
- Direct processing of GeoParquet files
- Perfect for client-side GIS analysis
- Minimal resource footprint

**How to Use:**

Basic setup:
```javascript
import * as duckdb from 'duckdb';

// Create an in-memory database
const db = new duckdb.Database(':memory:');
const conn = db.connect();

// Load spatial extension
conn.exec('INSTALL spatial; LOAD spatial;');

// Create table from GeoJSON
conn.exec(`
  CREATE TABLE geodata AS
  SELECT * FROM ST_Read('data.geojson');
`);

// Perform spatial analysis
const result = conn.exec(`
  SELECT 
    category,
    COUNT(*) as count,
    SUM(ST_Area(geometry)) / 1000000 as area_km2
  FROM geodata
  GROUP BY category
  ORDER BY area_km2 DESC;
`);

console.table(result);
```

Working with GeoParquet:
```javascript
// Load GeoParquet file
conn.exec(`
  CREATE TABLE large_dataset AS
  SELECT * FROM READ_PARQUET('large_dataset.parquet');
`);

// Create spatial index
conn.exec(`
  CREATE TABLE spatial_index AS
  SELECT 
    id, 
    ST_TileEnvelope(ST_TileX(geometry, 10), ST_TileY(geometry, 10), 10) as tile_geom
  FROM large_dataset;
`);

// Query using tile index
const results = conn.exec(`
  SELECT l.*
  FROM large_dataset l
  JOIN spatial_index i ON l.id = i.id
  WHERE ST_Intersects(i.tile_geom, ST_GeomFromText('POLYGON((...))'));
`);
```

Browser integration:
```javascript
import * as duckdb from '@duckdb/duckdb-wasm';

async function initDuckDB() {
  const JSDELIVR_BUNDLES = duckdb.getJsDelivrBundles();
  
  // Select a bundle based on browser feature detection
  const bundle = await duckdb.selectBundle(JSDELIVR_BUNDLES);
  
  const worker = new Worker(bundle.mainWorker);
  const logger = new duckdb.ConsoleLogger();
  
  // Create a database
  const db = new duckdb.AsyncDuckDB(logger, worker);
  await db.instantiate(bundle.mainModule);
  
  return db;
}

const db = await initDuckDB();
const conn = await db.connect();

// Now use like regular DuckDB
await conn.query(`
  INSTALL spatial;
  LOAD spatial;
`);
```

**Internal Resources:**
- [DuckDB Integration Guide](/docs/gis/duckdb-integration.md)
- [Client-side Analysis Patterns](/libs/gis/src/analysis)
- [Browser Performance Optimizations](/docs/gis/duckdb-browser.md)

## EdTech Stack

### LlamaIndex

**What:** LlamaIndex is a data framework for building LLM-powered applications, focusing on knowledge retrieval, routing, and augmentation.

**Why:**
- Structured approach to RAG (Retrieval Augmented Generation)
- Support for diverse data sources
- Query planning and routing
- Optimized vector storage integration
- Advanced context handling

**How to Use:**

Basic document indexing:
```python
from llama_index import VectorStoreIndex, SimpleDirectoryReader
from llama_index.storage.storage_context import StorageContext
from llama_index.vector_stores import PGVectorStore

# Load documents
documents = SimpleDirectoryReader("./data").load_data()

# Create PG vector store
vector_store = PGVectorStore.from_params(
    database="postgres",
    host="localhost",
    password="postgres",
    port=5432,
    user="postgres",
    table_name="document_embeddings",
    embed_dim=1536  # For OpenAI embeddings
)

# Create storage context
storage_context = StorageContext.from_defaults(vector_store=vector_store)

# Build index
index = VectorStoreIndex.from_documents(
    documents, 
    storage_context=storage_context
)

# Query the index
query_engine = index.as_query_engine()
response = query_engine.query("What is the capital of France?")
print(response)
```

Advanced retrieval with metadata filtering:
```python
from llama_index.schema import MetadataFilter

# Create query engine with metadata filters
query_engine = index.as_query_engine(
    filters=MetadataFilter(
        filters=[
            {"key": "category", "value": "geography"},
            {"key": "date", "value": "2023", "operator": ">="}
        ]
    )
)

# Execute query
response = query_engine.query("What are the largest cities?")
```

Custom retrieval pipeline:
```python
from llama_index.retrievers import VectorIndexRetriever
from llama_index.query_engine import RetrieverQueryEngine
from llama_index.response_synthesizers import CompactAndRefine

# Custom retriever
retriever = VectorIndexRetriever(
    index=index,
    similarity_top_k=5,
    filters=MetadataFilter(filters=[...])
)

# Custom response synthesizer
response_synthesizer = CompactAndRefine(
    llm=llm,
    verbose=True
)

# Build query engine
query_engine = RetrieverQueryEngine(
    retriever=retriever,
    response_synthesizer=response_synthesizer
)

# Execute query
response = query_engine.query("Complex question here?")
```

**Internal Resources:**
- [LlamaIndex Architecture](/docs/edtech/llamaindex-architecture.md)
- [Custom Retrievers](/services/edtech/src/knowledge)
- [Query Optimization](/docs/edtech/llm-query-optimization.md)

### WebRTC

**What:** WebRTC (Web Real-Time Communication) is an open-source project that provides web browsers and mobile applications with real-time communication via simple APIs.

**Why:**
- Low-latency video/audio streaming
- Peer-to-peer architecture reduces server load
- Industry standard for real-time communication
- Native browser support
- Secure by default (encrypted)

**How to Use:**

Basic peer connection:
```javascript
// Signaling is handled separately through your server
const configuration = {
  iceServers: [
    { urls: 'stun:stun.example.org:3478' },
    { 
      urls: 'turn:turn.example.org:3478',
      username: 'username',
      credential: 'password'
    }
  ]
};

// Create peer connection
const peerConnection = new RTCPeerConnection(configuration);

// Add local stream
localStream.getTracks().forEach(track => {
  peerConnection.addTrack(track, localStream);
});

// Handle incoming remote stream
peerConnection.ontrack = event => {
  remoteVideo.srcObject = event.streams[0];
};

// ICE candidate handling
peerConnection.onicecandidate = event => {
  if (event.candidate) {
    // Send candidate to peer via signaling server
    signalServer.send(JSON.stringify({
      type: 'candidate',
      candidate: event.candidate
    }));
  }
};

// Create and send offer
async function createOffer() {
  const offer = await peerConnection.createOffer();
  await peerConnection.setLocalDescription(offer);
  
  // Send offer to peer via signaling server
  signalServer.send(JSON.stringify({
    type: 'offer',
    sdp: peerConnection.localDescription
  }));
}

// Handle incoming offers/answers/candidates from signaling
function handleSignalingMessage(message) {
  const data = JSON.parse(message);
  
  switch(data.type) {
    case 'offer':
      peerConnection.setRemoteDescription(new RTCSessionDescription(data.sdp));
      // Create and send answer
      createAnswer();
      break;
    case 'answer':
      peerConnection.setRemoteDescription(new RTCSessionDescription(data.sdp));
      break;
    case 'candidate':
      peerConnection.addIceCandidate(new RTCIceCandidate(data.candidate));
      break;
  }
}
```

Optimizing for low-latency:
```javascript
// Configure for lower latency
const videoConstraints = {
  width: { ideal: 640 },
  height: { ideal: 480 },
  frameRate: { min: 15, ideal: 30 }
};

const audioConstraints = {
  echoCancellation: true,
  noiseSuppression: true,
  autoGainControl: true
};

// Get optimized media stream
navigator.mediaDevices.getUserMedia({
  video: videoConstraints,
  audio: audioConstraints
}).then(stream => {
  // Use stream
});

// Set codec preferences for lower latency
const transceiver = peerConnection.addTransceiver('video');
const capabilities = RTCRtpSender.getCapabilities('video');
const preferredCodecs = capabilities.codecs.filter(
  codec => codec.mimeType === 'video/VP8' || codec.mimeType === 'video/H264'
);
transceiver.setCodecPreferences(preferredCodecs);
```

Error handling and connection monitoring:
```javascript
// Monitor connection state
peerConnection.onconnectionstatechange = () => {
  switch(peerConnection.connectionState) {
    case 'connected':
      // Connection established
      console.log('Connection established');
      break;
    case 'disconnected':
    case 'failed':
      // Connection lost, attempt reconnection
      console.log('Connection lost, reconnecting...');
      reconnect();
      break;
    case 'closed':
      // Connection closed
      console.log('Connection closed');
      break;
  }
};

// Monitor ICE connection state
peerConnection.oniceconnectionstatechange = () => {
  console.log('ICE connection state:', peerConnection.iceConnectionState);
  
  if (peerConnection.iceConnectionState === 'failed') {
    // ICE failure, restart ICE
    peerConnection.restartIce();
  }
};

// Stats monitoring for quality
setInterval(() => {
  peerConnection.getStats().then(stats => {
    stats.forEach(report => {
      if (report.type === 'inbound-rtp' && report.kind === 'video') {
        console.log('Packets lost:', report.packetsLost);
        console.log('Jitter:', report.jitter);
        console.log('Frames decoded:', report.framesDecoded);
        
        // Adapt quality based on stats
        if (report.packetsLost > threshold) {
          // Reduce video quality
          reduceVideoQuality();
        }
      }
    });
  });
}

// Example usage
const transcodeOptions = {
  videoCodec: 'libx264',
  audioCodec: 'aac',
  videoBitrate: '2M',
  audioBitrate: '192k',
  format: 'mp4'
};

transcodeVideo('input.mov', 'output.mp4', transcodeOptions)
  .then(result => console.log('Transcoding complete'))
  .catch(error => console.error('Transcoding failed:', error));
```

Hardware-accelerated transcoding:
```javascript
// Using NVIDIA GPU acceleration with NVENC
const nvencArgs = [
  '-hwaccel', 'cuda',
  '-hwaccel_output_format', 'cuda',
  '-i', inputPath,
  '-c:v', 'h264_nvenc',
  '-preset', 'p4',
  '-tune', 'll',
  '-b:v', '5M',
  '-c:a', 'aac',
  '-b:a', '192k',
  outputPath
];

// Using Intel QuickSync
const quicksyncArgs = [
  '-hwaccel', 'qsv',
  '-i', inputPath,
  '-c:v', 'h264_qsv',
  '-b:v', '5M',
  '-c:a', 'aac',
  '-b:a', '192k',
  outputPath
];
```

Creating HLS adaptive streaming:
```javascript
const hlsArgs = [
  '-i', inputPath,
  '-profile:v', 'main',
  '-sc_threshold', '0',
  '-g', '48',
  '-keyint_min', '48',
  
  // 1080p
  '-map', '0',
  '-c:v:0', 'libx264',
  '-b:v:0', '5M',
  '-s:v:0', '1920x1080',
  
  // 720p
  '-map', '0',
  '-c:v:1', 'libx264',
  '-b:v:1', '3M',
  '-s:v:1', '1280x720',
  
  // 480p
  '-map', '0',
  '-c:v:2', 'libx264',
  '-b:v:2', '1M',
  '-s:v:2', '854x480',
  
  // Audio
  '-map', '0:a',
  '-c:a', 'aac',
  '-b:a', '192k',
  
  // Output
  '-var_stream_map', 'v:0,a:0 v:1,a:0 v:2,a:0',
  '-master_pl_name', 'master.m3u8',
  '-f', 'hls',
  '-hls_time', '6',
  '-hls_list_size', '0',
  '-hls_segment_filename', 'stream_%v/segment_%d.ts',
  'stream_%v/playlist.m3u8'
];
```

**Internal Resources:**
- [FFmpeg Processing Pipeline](/services/edtech/src/media)
- [Hardware Acceleration Guide](/docs/edtech/ffmpeg-hardware-acceleration.md)
- [Adaptive Streaming Implementation](/libs/edtech/src/streaming)

### pgvector

**What:** pgvector is a PostgreSQL extension that adds vector similarity search capabilities, enabling efficient storage and querying of embeddings.

**Why:**
- Integrates with our existing PostgreSQL infrastructure
- High-performance vector similarity searches
- Support for multiple indexing methods (IVF, HNSW)
- Transaction support for data integrity
- Perfect for semantic search and AI-powered features

**How to Use:**

Database setup:
```sql
-- Install the extension
CREATE EXTENSION vector;

-- Create a table with vector column
CREATE TABLE embeddings (
  id SERIAL PRIMARY KEY,
  content TEXT NOT NULL,
  embedding VECTOR(1536) NOT NULL, -- 1536 for OpenAI embeddings
  metadata JSONB
);

-- Create an index for faster similarity search
CREATE INDEX ON embeddings USING ivfflat (embedding vector_l2_ops) WITH (lists = 100);
-- Or use HNSW index for better accuracy
CREATE INDEX ON embeddings USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);
```

Node.js integration:
```javascript
import { Pool } from 'pg';

// Setup connection pool
const pool = new Pool({
  host: 'localhost',
  port: 5432,
  user: 'postgres',
  password: 'postgres',
  database: 'platform_dev',
});

// Store embeddings
async function storeEmbedding(content, embedding, metadata = {}) {
  const client = await pool.connect();
  try {
    const query = `
      INSERT INTO embeddings (content, embedding, metadata)
      VALUES ($1, $2, $3)
      RETURNING id;
    `;
    const result = await client.query(query, [content, embedding, metadata]);
    return result.rows[0].id;
  } finally {
    client.release();
  }
}

// Search similar content
async function findSimilar(embedding, limit = 5) {
  const client = await pool.connect();
  try {
    const query = `
      SELECT 
        id, 
        content, 
        metadata,
        1 - (embedding <=> $1) as similarity
      FROM embeddings
      ORDER BY embedding <=> $1
      LIMIT $2;
    `;
    const result = await client.query(query, [embedding, limit]);
    return result.rows;
  } finally {
    client.release();
  }
}
```

Performance optimization:
```sql
-- Tune for large datasets
ALTER TABLE embeddings SET (autovacuum_vacuum_scale_factor = 0.05);
ALTER TABLE embeddings SET (autovacuum_analyze_scale_factor = 0.02);

-- Cluster table for better locality
CLUSTER embeddings USING embeddings_embedding_idx;

-- Partitioning for huge datasets
CREATE TABLE embeddings (
  id SERIAL,
  content TEXT NOT NULL,
  embedding VECTOR(1536) NOT NULL,
  metadata JSONB,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE embeddings_202401 PARTITION OF embeddings
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE embeddings_202402 PARTITION OF embeddings
FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Create indexes on each partition
CREATE INDEX ON embeddings_202401 USING hnsw (embedding vector_cosine_ops);
CREATE INDEX ON embeddings_202402 USING hnsw (embedding vector_cosine_ops);
```

**Internal Resources:**
- [pgvector Performance Guide](/docs/databases/pgvector-optimization.md)
- [Semantic Search Implementation](/services/edtech/src/search)
- [Embedding Management](/libs/core/src/embeddings)

## Backend Stack

### Fastify

**What:** Fastify is a high-performance, low overhead web framework for Node.js, focusing on developer experience, performance, and plugins.

**Why:**
- Extremely fast request handling
- Schema-based validation
- Robust plugin architecture
- Built-in TypeScript support
- Lower memory footprint than Express

**How to Use:**

Basic setup:
```typescript
import fastify, { FastifyInstance } from 'fastify';
import { TypeBoxTypeProvider } from '@fastify/type-provider-typebox';
import { Type } from '@sinclair/typebox';

// Create fastify instance with TypeBox for validation
const server: FastifyInstance = fastify({
  logger: true,
}).withTypeProvider<TypeBoxTypeProvider>();

// Define route schema
const schema = {
  body: Type.Object({
    name: Type.String(),
    email: Type.String({ format: 'email' }),
    age: Type.Optional(Type.Number({ minimum: 18 }))
  }),
  response: {
    200: Type.Object({
      id: Type.String({ format: 'uuid' }),
      success: Type.Boolean()
    })
  }
};

// Register route with schema
server.post('/users', { schema }, async (request, reply) => {
  const { name, email, age } = request.body;
  
  // Process request
  const id = await createUser({ name, email, age });
  
  return { id, success: true };
});

// Start server
server.listen({ port: 3000 }, (err) => {
  if (err) {
    server.log.error(err);
    process.exit(1);
  }
});
```

Plugins and hooks:
```typescript
import fastify from 'fastify';
import fastifyJwt from '@fastify/jwt';
import fastifyCors from '@fastify/cors';

const server = fastify({ logger: true });

// Register plugins
server.register(fastifyJwt, {
  secret: process.env.JWT_SECRET || 'supersecret'
});

server.register(fastifyCors, {
  origin: ['https://example.com', 'https://subdomain.example.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  credentials: true
});

// Add hooks
server.addHook('onRequest', async (request, reply) => {
  // Log every request
  request.log.info(`Request received: ${request.method} ${request.url}`);
});

server.addHook('preValidation', async (request, reply) => {
  // Authenticate requests to protected routes
  if (request.url.startsWith('/api/protected')) {
    try {
      await request.jwtVerify();
    } catch (err) {
      reply.code(401).send({ error: 'Unauthorized' });
    }
  }
});

server.addHook('onResponse', async (request, reply) => {
  // Log response time
  request.log.info(`Response time: ${reply.getResponseTime()}ms`);
});
```

Performance optimization:
```typescript
import fastify from 'fastify';
import fastifyCompress from '@fastify/compress';
import fastifyCaching from '@fastify/caching';

const server = fastify({
  // Disable JSON parsing for routes that don't need it
  disableRequestLogging: true,
  jsonShorthand: false,
  
  logger: {
    level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
    // Use pino-pretty only in development
    transport: process.env.NODE_ENV !== 'production'
      ? { target: 'pino-pretty' }
      : undefined
  }
});

// Enable compression
server.register(fastifyCompress, {
  encodings: ['gzip', 'deflate']
});

// Enable caching headers
server.register(fastifyCaching, {
  privacy: fastifyCaching.privacy.PUBLIC,
  expiresIn: 600
});

// Use custom serializer for faster JSON serialization
server.setSerializerCompiler(({ schema, method, url, httpStatus }) => {
  return data => JSON.stringify(data);
});
```

**Internal Resources:**
- [Fastify Architecture Guide](/docs/backend/fastify-architecture.md)
- [API Service Template](/services/core/template)
- [Performance Benchmarks](/docs/backend/api-benchmarks.md)

### PostgreSQL

**What:** PostgreSQL is an advanced, enterprise-class, open-source relational database known for reliability, feature robustness, and performance.

**Why:**
- ACID-compliant transactions
- Advanced indexing capabilities
- Rich data types (including JSON, array, geometric)
- Robust replication and high availability
- Extensibility (PostGIS, pgvector, etc.)

**How to Use:**

Connection management:
```typescript
import { Pool } from 'pg';

// Create connection pool
const pool = new Pool({
  host: process.env.POSTGRES_HOST || 'localhost',
  port: parseInt(process.env.POSTGRES_PORT || '5432'),
  user: process.env.POSTGRES_USER || 'postgres',
  password: process.env.POSTGRES_PASSWORD || 'postgres',
  database: process.env.POSTGRES_DB || 'platform_dev',
  
  // Connection pool settings
  max: 20, // Maximum clients in pool
  idleTimeoutMillis: 30000, // Close idle clients after 30s
  connectionTimeoutMillis: 2000, // Return error after 2s if no connection
  
  // SSL configuration for production
  ssl: process.env.NODE_ENV === 'production' ? {
    rejectUnauthorized: true,
    ca: fs.readFileSync('/path/to/ca-cert.pem').toString()
  } : undefined
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  console.log('Closing database connections...');
  await pool.end();
  console.log('Database connections closed');
  process.exit(0);
});

// Helper function for queries
async function query(text: string, params: any[] = []) {
  const start = Date.now();
  try {
    const result = await pool.query(text, params);
    const duration = Date.now() - start;
    
    if (duration > 200) {
      console.warn(`Slow query (${duration}ms): ${text}`);
    }
    
    return result;
  } catch (error) {
    console.error('Database query error:', error);
    throw error;
  }
}
```

Transaction management:
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

// Example usage
async function transferFunds(fromAccount: string, toAccount: string, amount: number) {
  return executeTransaction(async (client) => {
    // Debit from account
    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
      [amount, fromAccount]
    );
    
    // Credit to account
    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
      [amount, toAccount]
    );
    
    // Record transaction
    await client.query(
      'INSERT INTO transactions (from_account, to_account, amount) VALUES ($1, $2, $3)',
      [fromAccount, toAccount, amount]
    );
    
    return { success: true };
  });
}
```

Performance optimization:
```typescript
// Use prepared statements for repeated queries
const preparedStatement = {
  name: 'fetch-user',
  text: 'SELECT * FROM users WHERE id = $1',
  values: [userId]
};

const result = await pool.query(preparedStatement);

// Batch inserts for better performance
async function batchInsert(items: any[]) {
  const values = items.map((item, index) => 
    `(${index * 3 + 1}, ${index * 3 + 2}, ${index * 3 + 3})`
  ).join(', ');
  
  const flatParams = items.flatMap(item => [item.name, item.email, item.role]);
  
  const query = `
    INSERT INTO users (name, email, role)
    VALUES ${values}
    RETURNING id
  `;
  
  return pool.query(query, flatParams);
}

// Use COPY for very large inserts
async function bulkInsert(filePath: string) {
  const client = await pool.connect();
  
  try {
    const stream = fs.createReadStream(filePath);
    const query = `COPY users (name, email, role) FROM STDIN WITH (FORMAT CSV, HEADER true)`;
    
    await new Promise((resolve, reject) => {
      const copyStream = client.query(copyFrom(query));
      stream.pipe(copyStream);
      
      copyStream.on('error', reject);
      copyStream.on('end', resolve);
    });
    
    return { success: true };
  } finally {
    client.release();
  }
}
```

**Internal Resources:**
- [Database Architecture Guide](/docs/databases/postgres-architecture.md)
- [Query Optimization Patterns](/docs/databases/query-optimization.md)
- [Replication Setup](/docs/databases/postgres-replication.md)

### S3

**What:** S3 (Simple Storage Service) is an object storage service designed to store and retrieve any amount of data from anywhere. We use AWS S3 or compatible alternatives like MinIO.

**Why:**
- Highly scalable object storage
- Cost-effective for large datasets
- Built-in versioning and lifecycle policies
- High durability and availability
- Works well with CloudFront for content delivery

**How to Use:**

Basic setup:
```typescript
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

// Initialize S3 client
const s3Client = new S3Client({
  region: process.env.AWS_REGION || 'us-east-1',
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID || '',
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY || ''
  },
  // For MinIO or other S3-compatible services
  endpoint: process.env.S3_ENDPOINT,
  forcePathStyle: true // Required for MinIO
});

// Upload file
async function uploadFile(bucket: string, key: string, body: Buffer, contentType: string) {
  const command = new PutObjectCommand({
    Bucket: bucket,
    Key: key,
    Body: body,
    ContentType: contentType
  });
  
  const response = await s3Client.send(command);
  return { key, etag: response.ETag };
}

// Generate presigned URL for download
async function getDownloadUrl(bucket: string, key: string, expiresIn = 3600) {
  const command = new GetObjectCommand({
    Bucket: bucket,
    Key: key
  });
  
  return getSignedUrl(s3Client, command, { expiresIn });
}

// Generate presigned URL for upload
async function getUploadUrl(bucket: string, key: string, contentType: string, expiresIn = 3600) {
  const command = new PutObjectCommand({
    Bucket: bucket,
    Key: key,
    ContentType: contentType
  });
  
  return getSignedUrl(s3Client, command, { expiresIn });
}
```

File management:
```typescript
import { ListObjectsV2Command, DeleteObjectCommand, CopyObjectCommand } from '@aws-sdk/client-s3';

// List files in a directory
async function listFiles(bucket: string, prefix: string) {
  const command = new ListObjectsV2Command({
    Bucket: bucket,
    Prefix: prefix,
    MaxKeys: 1000
  });
  
  const response = await s3Client.send(command);
  
  return response.Contents?.map(item => ({
    key: item.Key,
    size: item.Size,
    lastModified: item.LastModified
  })) || [];
}

// Delete file
async function deleteFile(bucket: string, key: string) {
  const command = new DeleteObjectCommand({
    Bucket: bucket,
    Key: key
  });
  
  await s3Client.send(command);
  return { success: true };
}

// Move/rename file
async function moveFile(bucket: string, sourceKey: string, destinationKey: string) {
  // Copy to new location
  const copyCommand = new CopyObjectCommand({
    Bucket: bucket,
    CopySource: `${bucket}/${sourceKey}`,
    Key: destinationKey
  });
  
  await s3Client.send(copyCommand);
  
  // Delete from old location
  const deleteCommand = new DeleteObjectCommand({
    Bucket: bucket,
    Key: sourceKey
  });
  
  await s3Client.send(deleteCommand);
  
  return { success: true };
}
```

Optimizing for large uploads:
```typescript
import { Upload } from '@aws-sdk/lib-storage';
import { createReadStream } from 'fs';

// Multipart upload for large files
async function uploadLargeFile(bucket: string, key: string, filePath: string, contentType: string) {
  const fileStream = createReadStream(filePath);
  
  const upload = new Upload({
    client: s3Client,
    params: {
      Bucket: bucket,
      Key: key,
      Body: fileStream,
      ContentType: contentType
    },
    // Optional settings
    partSize: 10 * 1024 * 1024, // 10MB parts
    leavePartsOnError: false    // Clean up on failure
  });
  
  // Add progress monitoring
  upload.on('httpUploadProgress', (progress) => {
    console.log(`Uploaded ${progress.loaded} of ${progress.total} bytes`);
  });
  
  const result = await upload.done();
  return { key, etag: result.ETag };
}
```

**Internal Resources:**
- [S3 Architecture Guide](/docs/infrastructure/s3-architecture.md)
- [Asset Management Service](/services/core/src/assets)
- [File Upload Workflow](/docs/backend/file-upload-patterns.md)

### Redis

**What:** Redis is an in-memory data structure store used as a database, cache, message broker, and streaming engine.

**Why:**
- Extremely fast in-memory operations
- Support for various data structures
- Built-in pub/sub messaging
- Perfect for caching, session management
- Distributed locking capabilities

**How to Use:**

Basic setup:
```typescript
import { createClient } from 'redis';

// Create Redis client
const redisClient = createClient({
  url: process.env.REDIS_URL || 'redis://localhost:6379',
  socket: {
    reconnectStrategy: (retries) => Math.min(retries * 50, 3000)
  }
});

// Event handlers
redisClient.on('error', (err) => console.error('Redis error:', err));
redisClient.on('connect', () => console.log('Connected to Redis'));
redisClient.on('reconnecting', () => console.log('Reconnecting to Redis...'));

// Connect to Redis
await redisClient.connect();

// Graceful shutdown
process.on('SIGTERM', async () => {
  console.log('Closing Redis connection...');
  await redisClient.quit();
  console.log('Redis connection closed');
  process.exit(0);
});
```

Caching with TTL:
```typescript
// Set cache with expiration
async function setCache(key: string, value: any, ttlSeconds = 3600) {
  await redisClient.set(key, JSON.stringify(value), {
    EX: ttlSeconds
  });
}

// Get from cache
async function getCache<T>(key: string): Promise<T | null> {
  const value = await redisClient.get(key);
  if (!value) return null;
  
  try {
    return JSON.parse(value) as T;
  } catch (error) {
    console.error('Error parsing cached value:', error);
    return null;
  }
}

// Cache wrapper for expensive functions
async function withCache<T>(key: string, fn: () => Promise<T>, ttlSeconds = 3600): Promise<T> {
  // Try to get from cache first
  const cached = await getCache<T>(key);
  if (cached) return cached;
  
  // Execute the function
  const result = await fn();
  
  // Cache the result
  await setCache(key, result, ttlSeconds);
  
  return result;
}

// Example usage
const userData = await withCache(
  `user:${userId}`,
  () => fetchUserFromDatabase(userId),
  1800 // 30 minutes
);
```

Pub/Sub messaging:
```typescript
// Create dedicated clients for pub/sub
const publisher = redisClient.duplicate();
const subscriber = redisClient.duplicate();

await publisher.connect();
await subscriber.connect();

// Subscribe to a channel
await subscriber.subscribe('notifications', (message) => {
  try {
    const notification = JSON.parse(message);
    processNotification(notification);
  } catch (error) {
    console.error('Error processing notification:', error);
  }
});

// Publish a message
async function publishNotification(userId: string, type: string, data: any) {
  const notification = {
    userId,
    type,
    data,
    timestamp: Date.now()
  };
  
  await publisher.publish('notifications', JSON.stringify(notification));
  return { success: true };
}
```

Distributed locking:
```typescript
async function acquireLock(resource: string, ttlMs = 30000): Promise<string | null> {
  const lockId = crypto.randomUUID();
  
  // Try to set key only if it doesn't exist
  const result = await redisClient.set(`lock:${resource}`, lockId, {
    NX: true,
    PX: ttlMs
  });
  
  return result ? lockId : null;
}

async function releaseLock(resource: string, lockId: string): Promise<boolean> {
  // Use Lua script to ensure atomic operation
  const script = `
    if redis.call("GET", KEYS[1]) == ARGV[1] then
      return redis.call("DEL", KEYS[1])
    else
      return 0
    end
  `;
  
  const result = await redisClient.eval(script, {
    keys: [`lock:${resource}`],
    arguments: [lockId]
  });
  
  return result === 1;
}

// Example usage
async function processResource(resource: string) {
  const lockId = await acquireLock(resource);
  if (!lockId) {
    throw new Error('Failed to acquire lock');
  }
  
  try {
    // Process the resource
    await doSomething(resource);
  } finally {
    // Always release the lock
    await releaseLock(resource, lockId);
  }
}
```

**Internal Resources:**
- [Redis Architecture Guide](/docs/infrastructure/redis-architecture.md)
- [Caching Patterns](/docs/backend/caching-strategies.md)
- [Distributed Locking Implementation](/libs/core/src/locking)

### Socket.IO

**What:** Socket.IO is a library that enables real-time, bidirectional and event-based communication between browsers and servers.

**Why:**
- Real-time bidirectional communication
- Automatic fallback to long polling if WebSockets unavailable
- Built-in reconnection logic
- Room-based broadcasting capabilities
- Cross-browser compatibility

**How to Use:**

Server setup:
```typescript
import { createServer } from 'http';
import { Server } from 'socket.io';
import express from 'express';

const app = express();
const httpServer = createServer(app);

// Create Socket.IO server
const io = new Server(httpServer, {
  cors: {
    origin: ['https://example.com'],
    methods: ['GET', 'POST'],
    credentials: true
  },
  pingTimeout: 30000,
  pingInterval: 25000,
  upgradeTimeout: 10000,
  transports: ['websocket', 'polling']
});

// Authentication middleware
io.use((socket, next) => {
  const token = socket.handshake.auth.token;
  
  if (!token) {
    return next(new Error('Authentication error'));
  }
  
  try {
    const decoded = verifyJwt(token);
    socket.data.user = decoded;
    next();
  } catch (error) {
    next(new Error('Invalid token'));
  }
});

// Connection handler
io.on('connection', (socket) => {
  const { user } = socket.data;
  console.log(`User connected: ${user.id}`);
  
  // Join user-specific room
  socket.join(`user:${user.id}`);
  
  // Handle events
  socket.on('join-room', (roomId) => {
    // Join a room
    socket.join(roomId);
    
    // Notify others in room
    socket.to(roomId).emit('user-joined', {
      userId: user.id,
      username: user.name
    });
  });
  
  socket.on('leave-room', (roomId) => {
    socket.leave(roomId);
    socket.to(roomId).emit('user-left', {
      userId: user.id
    });
  });
  
  socket.on('send-message', (data) => {
    const { roomId, message } = data;
    
    // Broadcast to everyone in the room except sender
    socket.to(roomId).emit('new-message', {
      userId: user.id,
      username: user.name,
      message,
      timestamp: Date.now()
    });
  });
  
  socket.on('disconnect', () => {
    console.log(`User disconnected: ${user.id}`);
    // Handle cleanup
  });
});

// Start server
httpServer.listen(3000,, 3000);
```

**Internal Resources:**
- [WebRTC Architecture](/docs/edtech/webrtc-architecture.md)
- [Signaling Server Implementation](/services/edtech/src/signaling)
- [Connection Quality Monitoring](/libs/edtech/src/webrtc)

### FFmpeg

**What:** FFmpeg is a powerful multimedia framework that can decode, encode, transcode, mux, demux, stream, filter, and play virtually any type of audio and video.

**Why:**
- Industry standard for media processing
- Extremely versatile and feature-rich
- High performance with hardware acceleration
- Perfect for video trimming, encoding, and conversion
- Supports virtually all media formats

**How to Use:**

Basic video transcoding:
```javascript
import { spawn } from 'child_process';

function transcodeVideo(inputPath, outputPath, options = {}) {
  return new Promise((resolve, reject) => {
    const args = [
      '-i', inputPath,
      '-c:v', options.videoCodec || 'libx264',
      '-c:a', options.audioCodec || 'aac',
      '-b:v', options.videoBitrate || '1M',
      '-b:a', options.audioBitrate || '128k',
      '-f', options.format || 'mp4',
      outputPath
    ];
    
    const ffmpeg = spawn('ffmpeg', args);
    
    let stdoutData = '';
    let stderrData = '';
    
    ffmpeg.stdout.on('data', (data) => {
      stdoutData += data.toString();
    });
    
    ffmpeg.stderr.on('data', (data) => {
      stderrData += data.toString();
    });
    
    ffmpeg.on('close', (code) => {
      if (code === 0) {
        resolve({
          success: true,
          output: stdoutData
        });
      } else {
        reject({
          success: false,
          error: stderrData
        });
      }
    });
  });
}
