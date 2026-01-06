# Geofence Corridor Generation

![Nano Banana Pro Visualization](./geofence_logic_vibe.png)

## What is a Geofence Corridor?

A **geofence corridor** is a polygon buffer zone around a route path. It defines the acceptable area where a vehicle/object should stay while traveling along its planned route.

---

## Architecture

The geofence system is split into two layers:

```
┌─────────────────────────────────────────────────────────────┐
│                   GeofenceCore (Pure JS)                    │
│  - No Leaflet, No DOM dependencies                          │
│  - Input: Array of routes, buffer distance                  │
│  - Output: GeoJSON FeatureCollection                        │
├─────────────────────────────────────────────────────────────┤
│  Core Functions:                                            │
│  • calculateBearing()        - Compass bearing              │
│  • destinationPoint()        - Point at bearing/distance    │
│  • haversineDistance()       - Distance between points      │
│  • consolidateRoutes()       - Merge connected segments     │
│  • createCorridorPolygon()   - Build corridor polygon       │
│  • generateCorridorGeoJSON() - Main API (returns GeoJSON)   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│               Visualization Layer (Leaflet)                 │
│  - Uses GeofenceCore output                                 │
│  - Renders polygons on map                                  │
│  - Handles UI interactions                                  │
├─────────────────────────────────────────────────────────────┤
│  Functions:                                                 │
│  • generateGeofences()   - Orchestrator (Core + Render)     │
│  • exportGeofences()     - Download GeoJSON file            │
└─────────────────────────────────────────────────────────────┘
```

---

## Code Location

All geofence logic is in **`index.html`**:

| Function | Lines | Layer | Purpose |
|----------|-------|-------|---------|
| `calculateBearing()` | 589-598 | Core | Compass bearing between two points |
| `destinationPoint()` | 601-617 | Core | Point at bearing/distance |
| `haversineDistance()` | 619-632 | Core | Distance in meters (Leaflet-free!) |
| `consolidateRoutes()` | 635-657 | Core | Merge connected segments |
| `createCorridorPolygon()` | 660-733 | Core | Build flat-ended corridor polygon |
| `generateCorridorGeoJSON()` | 735-798 | Core | **Main API** - returns GeoJSON |
| `generateGeofences()` | 800-848 | Viz | Orchestrator (calls Core + renders) |
| `exportGeofences()` | 850-865 | Viz | Download GeoJSON file |

---

## Main API: `generateCorridorGeoJSON()`

This is the **Leaflet-independent** function for generating geofence corridors.

### Signature
```javascript
generateCorridorGeoJSON(allRoutes, distanceKm, options = {})
```

### Parameters
| Parameter | Type | Description |
|-----------|------|-------------|
| `allRoutes` | `Array` | Array of routes. Each route is an array of `[lat, lng]` points. |
| `distanceKm` | `number` | Buffer distance in kilometers |
| `options.simplifyTolerance` | `number` | Turf.js simplify tolerance (default: 0.00005 ≈ 5m) |
| `options.connectionThreshold` | `number` | Max distance (meters) to merge segments (default: 1) |

### Returns
```javascript
{
  type: 'FeatureCollection',
  features: [
    {
      type: 'Feature',
      properties: {
        routeIndex: 1,
        bufferDistanceKm: 0.1,
        type: 'geofence_corridor'
      },
      geometry: {
        type: 'Polygon',
        coordinates: [[[lng, lat], ...]]
      }
    }
  ]
}
```

### Example Usage
```javascript
const routes = [
  [[0, 0], [0, 1], [1, 1]],  // Route 1: 3 points
  [[1, 1], [2, 1], [2, 2]]   // Route 2: 3 points (connected to Route 1)
];

const geoJSON = generateCorridorGeoJSON(routes, 0.1); // 100m buffer
console.log(JSON.stringify(geoJSON, null, 2));
```

---

## The 3-Step Algorithm

### Step 1: Consolidate Connected Segments
**Function: `consolidateRoutes()`**

Multiple route segments that share endpoints are merged into one continuous line.

**Why?** Generating corridors for separate segments creates gaps at connection points.

```javascript
const dist = haversineDistance(prevEnd[0], prevEnd[1], nextStart[0], nextStart[1]);

if (dist < connectionThresholdMeters) {
    // Connect: append next segment (skipping duplicate first point)
    currentRoute.push(...allRoutes[i].slice(1));
}
```

---

### Step 2: Simplify the Line
**Using Turf.js**

The consolidated line is simplified using the **Ramer-Douglas-Peucker algorithm**.

**Why?** Raw curves have 50+ points per segment. Too many points = slow performance and bumpy edges.

```javascript
const line = turf.lineString(geoJSONPoints);
const simplifiedLine = turf.simplify(line, { 
    tolerance: 0.00005, // ≈ 5 meters
    highQuality: true 
});
```

---

### Step 3: Generate Flat-Ended Corridor Polygon
**Function: `createCorridorPolygon()`**

For each point on the simplified line:
1. Calculate the perpendicular direction (90° from travel direction)
2. Create a left offset point and a right offset point
3. At corners, use **miter join** to fill gaps

```javascript
const leftBearing = (offsetBearing - 90 + 360) % 360;
const rightBearing = (offsetBearing + 90) % 360;

const leftPoint = destinationPoint(lat, lng, leftBearing, offsetDistance);
const rightPoint = destinationPoint(lat, lng, rightBearing, offsetDistance);
```

**Polygon Construction**:
```javascript
const polygonCoords = [
    ...leftSide,
    ...rightSide.reverse(),
    leftSide[0] // Close the polygon
];
```

---

## Deep Dive: Corridor Geometry Creation

![Corridor Geometry Deep Dive](./corridor_geometry_deep_dive.png)

The core challenge of generating a clean geofence is transforming a zero-width line into a precise, flat-ended polygon.

### 1. Perpendicular Vector Engineering
For every point on the path, we calculate the travel direction (bearing) to the next point. We rotate this vector by ±90° to find the "Left" and "Right" offset directions.

### 2. The Miter Join Resolution
When the route turns, a simple perpendicular offset creates overlaps on the inside and gaps on the outside. We resolve this using **Miter Joins**.

The distance from the center point to the corner is increased based on the sharpness of the turn:

$$Distance_{miter} = \frac{Distance_{base}}{\cos(\theta/2)}$$

*where θ is the angle of the turn.*

```javascript
const halfAngle = Math.abs(angleDiff / 2);
const halfAngleRad = halfAngle * Math.PI / 180;
const miterLimit = 3; // Max extension factor
const miterFactor = Math.min(1 / Math.cos(halfAngleRad), miterLimit);
offsetDistance = widthKm * miterFactor;
```

### 3. Flat End Cap Precision
Unlike `turf.buffer()` which creates circular "pill" shapes, our algorithm treats the first and last points as hard boundaries:
- **Start Cap**: Perpendicular to the first segment
- **End Cap**: Perpendicular to the last segment

---

## Why Not Use `turf.buffer()`?

The standard Turf.js buffer function creates **rounded end caps** and **rounded corners**.

Our custom `createCorridorPolygon()` creates:
- ✅ Flat ends (no circles)
- ✅ Miter corners (filled, not rounded)

---

## Summary

| Step | Function | Purpose |
|------|----------|---------|
| 1 | `consolidateRoutes()` | Eliminate gaps between connected segments |
| 2 | `turf.simplify()` | Reduce points for performance |
| 3 | `createCorridorPolygon()` | Create flat-ended buffer with miter corners |

**Result**: Clean, continuous corridor polygons without gaps or circles.

---

## Dependencies

| Library | Purpose |
|---------|---------|
| **Turf.js** | Line simplification (`turf.simplify`, `turf.lineString`) |
| **Leaflet.js** | Map visualization only (not required for core) |
