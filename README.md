# SmartRoute — Intelligent Dynamic Route Optimization System

An interactive route-planning visualizer that demonstrates **Dijkstra's algorithm** and
**A\*** on a simulated Chennai road network. It combines a modern glassmorphism UI,
live traffic controls, and step-by-step search animation. Every routing algorithm is
implemented by hand—no routing library is used.

---

## 1. Installation

Requirements: **Node.js 18+** and npm.

```bash
cd smartroute
npm install
npm run dev
```

Open the URL Vite prints (default **http://localhost:5173**). That's it — no backend,
no API keys, no database required for this first version.

```bash
npm run build      # production build -> dist/
npm run preview    # serve the production build locally
```

---

## 2. What you can do in the app

1. Pick a **Start Location** and **Destination** from 25 simulated Chennai-area
   locations.
2. Choose **Dijkstra** or **A\*** as the algorithm.
3. Choose a **Route Preference** (Shortest Distance / Fastest Route / Avoid Traffic /
   Balanced) — this changes how edge weights are computed.
4. Click **Calculate Route** for an instant answer, or **Run Animation** to watch the
   algorithm explore the graph node-by-node, with pause / resume / reset / speed
   controls.
5. Open **Traffic & Road Controls** to change a road's traffic level (LOW / MEDIUM /
   HIGH / SEVERE) or mark it **CLOSED**, then hit **Recalculate Route** to see the
   route change live.
6. Click **Compare Algorithms** to run Dijkstra and A\* back-to-back on the same query
   and see nodes explored / execution time side by side.
7. Explore a glassmorphism interface with animated search states, glowing route effects,
   and a color-coded map legend.

### Map colors

| Color | Meaning |
| --- | --- |
| Slate | Unvisited location / normal road |
| Cyan | Visited location |
| Amber | Currently explored location |
| Cyan → green → violet | Final shortest route, shown with moving dashes |
| Orange | Heavy traffic |
| Rose | Closed road |

---

## 3. Project structure

```
smartroute/
  src/
    components/
      MapGraph.jsx            SVG map: nodes, edges, live color states
      ControlPanel.jsx        Start/destination/algorithm/mode + action buttons
      RouteInfo.jsx           Right-panel: distance, time, path, stats
      AlgorithmVisualizer.jsx Bottom console: log, distance table, priority queue
      TrafficPanel.jsx        Per-road traffic level + close/open toggle
      ComparisonPanel.jsx     Dijkstra vs A* table + bar chart
      ExplanationPanel.jsx    "How Dijkstra Works" teaching panel
    algorithms/
      dijkstra.js             Manual Dijkstra, emits a step-by-step trace
      astar.js                Manual A*, f(n) = g(n) + h(n)
      priorityQueue.js        Hand-written binary min-heap
    data/
      locations.js            25 graph vertices (id, label, x, y)
      roads.js                ~38 graph edges (distance, base time, traffic, closed)
    utils/
      graphUtils.js           Adjacency list builder, traffic-aware edge cost, heuristic
      pathReconstruction.js   Walks the `previous` map back into a path
    App.jsx                   Wires state, animation loop, and layout together
    index.css                 Inter typography and glassmorphism visual theme
  index.html
  package.json
  vite.config.js
```

The routing logic (`algorithms/`, `utils/`) has **zero dependency on React** — it's
plain JavaScript operating on a plain adjacency list, so it could be lifted into a
FastAPI backend later (e.g. behind a `POST /route` endpoint) without changing the core
algorithm code at all. That's intentional, per the brief's request to keep the engine
frontend-only for v1 but structured for an eventual backend move.

---

## 4. The graph data structure

The road network is represented as a **weighted, undirected graph** using an
**adjacency list**:

```js
{
  Chennai: [{ to: 'Guindy', road: {...} }, { to: 'Tambaram', road: {...} }],
  Guindy:  [{ to: 'Chennai', road: {...} }, { to: 'Velachery', road: {...} }, ...],
  ...
}
```

`utils/graphUtils.js` builds this from a flat list of `roads` (`data/roads.js`),
**skipping any road marked `closed: true`** — so a closed road is not just visually
greyed out, it is structurally absent from the graph the algorithm sees. All 25
locations and ~38 roads are Chennai-inspired, but every distance, travel time, and
traffic figure is simulated for this demo.

---

## 5. Dijkstra's algorithm (manual implementation)

`algorithms/dijkstra.js`, built entirely from `algorithms/priorityQueue.js` (a
hand-written binary min-heap) and a plain distance/previous map:

1. Set every node's distance to `Infinity`, source to `0`.
2. Push the source into the min-heap.
3. Pop the node with the smallest known distance.
4. For each neighbor, compute `currentDistance + edgeWeight` ("edge relaxation").
5. If that's smaller than the neighbor's known distance, update it, record the
   predecessor in a `previous` map, and push the neighbor back into the heap.
6. Repeat until the destination is popped (or the heap empties, meaning no route
   exists).
7. Reconstruct the path by walking `previous` backwards from destination to source
   (`utils/pathReconstruction.js`).

Every push/pop/relax operation also appends a structured step (`{type, node,
distances, pq, log}`) to a `steps` array, which is what powers the animation, the live
distance table, and the priority-queue panel in the UI — the algorithm doesn't know or
care that it's being visualized; the trace is just a side-effect log.

**Complexity:** `O((V + E) log V)` time (heap push/pop is `O(log V)`, done up to `E`
times), `O(V + E)` space for the adjacency list, distance map, and heap.

---

## 6. A\* implementation (manual)

`algorithms/astar.js` is structurally the same search as Dijkstra, with one change:
the heap is keyed by `f(n) = g(n) + h(n)` instead of by `g(n)` alone.

- `g(n)` — actual accumulated cost from the source to `n` (same as Dijkstra's
  distance).
- `h(n)` — **straight-line (Euclidean) distance** between node `n` and the
  destination's `(x, y)` map coordinates, scaled down so it never overestimates the
  true remaining road distance (this keeps the heuristic *admissible*, which is what
  guarantees A\* still finds the optimal path, not just *a* path).

Because nodes closer to the destination get a lower `f(n)`, A\* is pulled toward the
goal instead of expanding roughly uniformly in every direction like Dijkstra. Try
**Compare Algorithms** on a couple of routes — A\* consistently visits fewer nodes for
the same optimal-cost path.

---

## 7. Traffic model

Each road has a `traffic` level and a `baseTimeMin`. The effective travel cost is:

```
effectiveCost = baseCost × trafficFactor
LOW = 1.0   MEDIUM = 1.3   HIGH = 2.0   SEVERE = 3.0
```

What "baseCost" means depends on the selected **Route Preference**:

| Mode              | Edge cost formula                                    |
|-------------------|-------------------------------------------------------|
| Shortest Distance | `distanceKm` (traffic ignored — physical distance only) |
| Fastest Route     | `baseTimeMin × trafficFactor`                          |
| Avoid Traffic     | `baseTimeMin × trafficFactor²` (penalizes congestion hard) |
| Balanced          | `0.5 × distanceKm + 0.5 × (baseTimeMin × trafficFactor)` |

Changing a road's traffic level in the Traffic Panel mutates that road's weight
immediately; clicking **Recalculate Route** reruns the selected algorithm over the
updated graph.

---

## 8. Road closure system

Marking a road **CLOSED** sets `closed: true` on that edge. `buildAdjacencyList`
filters closed edges out entirely before Dijkstra/A\* ever see the graph — so a closed
road isn't "weighted very high," it's structurally unreachable, exactly like a real
road closure. The map renders closed roads in red with a dashed line and an `X` in
place of the distance label. Recalculating after a closure will route around it if any
path exists, or report "No route found" if the closure disconnects the destination.

---

## 9. How the animation works

Both algorithms produce a full `steps` array up front (this is fast — even the full
25-node graph runs in well under a millisecond). The UI then **replays** that array
one step at a time:

- `Run Animation` resets `stepIndex` to `0` and starts a timer (`setTimeout`, delay
  controlled by the speed slider) that increments `stepIndex` until the trace ends.
- At any `stepIndex`, `App.jsx` derives the current visualization state (which nodes
  are visited/exploring, the live distance table, the priority-queue contents, and the
  console log) by folding over `steps[0..stepIndex]`.
- **Pause** stops the timer without discarding progress; **Resume** restarts it from
  the same `stepIndex`; **Reset** rewinds to `0` without recomputing the algorithm.
- Once a `done` step is replayed, the reconstructed path is revealed on the map in
  a glowing animated gradient and the Route Information panel populates with distance,
  time, and stats.

Because the trace is precomputed, scrubbing the animation is just changing an array
index — the algorithm itself is never re-run mid-animation, which keeps
pause/resume/speed changes instant and glitch-free.

---

## 10. Notes on the data

Distances, travel times, and traffic levels are **simulated** for demonstration and do
not reflect real Chennai road conditions. Location names and relative geography are
loosely Chennai-inspired to give the demo a concrete, recognizable feel for a capstone
presentation.

---

## Tech stack

- React 18
- Vite 5
- Plain CSS with Inter from Google Fonts
- SVG for the interactive road network

## Available scripts

| Command | Description |
| --- | --- |
| `npm run dev` | Start the local development server. |
| `npm run build` | Create an optimized production build in `dist/`. |
| `npm run preview` | Serve the production build locally. |
