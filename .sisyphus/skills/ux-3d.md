# UX and 3D Guidelines

## Core Principle

**Performance first, aesthetics second.**

## Maps and Geospatial

### Performance Targets

- Render performance: 60fps on mid-range devices
- Tile loading latency < 100ms (p95)
- Progressive loading for complex datasets
- Pre-rendered static tiles for common views

### MapLibre Style Structure

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
    }
  ]
}
```

## Learning Models

- WebRTC connection setup < 1s
- First interactive frame < 2s
- Content layout optimized for information hierarchy
- Accessibility: WCAG 2.1 AA compliance

## 3D Model Performance

- Max polygon count: 100k per scene
- Texture size: 2048x2048 max (1024x1024 preferred)
- LOD implementation for complex models
- GLTF 2.0 format with draco compression

## Accessibility

- WCAG 2.1 AA compliance required
- Keyboard navigation for all interactive elements
- Screen reader support
- Color contrast requirements met
- Focus indicators visible
- Skip links for navigation

## Responsive Design

- Mobile-first approach
- Touch targets minimum 44x44 pixels
- Breakpoints for common screen sizes
- Fluid typography and spacing
