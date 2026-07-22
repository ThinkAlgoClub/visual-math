## C. FRONTEND TEAM – PRACTICAL JAVASCRIPT, RENDERING & HARDWARE HELL

---

### C1. State Management & Architecture
*Decision needed: Without React/Vue, how do we prevent 5,000 lines of tightly coupled `onClick` disasters?*

- **The Pub/Sub Core (The Single Source of Truth):**
  - **Decision:** Implement a lean, standalone **EventEmitter** (Pub/Sub) pattern as the global state bus. Do NOT rely on scattered `window` variables.
  - **Exact Implementation:** Use a tiny, zero-dependency library like `mitt` (200 bytes) or build a minimal `EventBus` class with `on`, `off`, `emit`.
  - **State Structure:** Create a single `AppStore` object that holds `currentTopic`, `simulationParams`, `drawingTool`, `isFullscreen`, etc. Any UI change `emits` an event (e.g., `topic:changed`), and the respective simulation module listens to it.
  - **The Golden Rule of Unsubscribe:** Every module that calls `.on()` **MUST** call `.off()` when the teacher navigates away. Enforce this by wrapping all module initializations in a `mount()` / `unmount()` lifecycle pattern.

---

### C2. Bundle Size, Dynamic Imports & CDN Strategy (The 2-Second Rule)
*Decision needed: Load time is money. We cannot ship a 5MB bundle to a rural 2G network.*

- **Hard Bundle Budget:**
  - **Decision:** The main `app.js` chunk (Dashboard + Navigation) must be **≤ 150 KB gzipped**. 
  - Heavy libraries (Three.js @ 600KB, D3.js @ 250KB, MathJax @ 800KB) are strictly **excluded** from the main bundle.
- **Dynamic Import Strategy (Code Splitting):**
  - Use native ES6 Dynamic `import()` exclusively.
  - **Example Pattern:**
    ```javascript
    // Inside the topic loader
    async function loadSimulation(topicType) {
        if (topicType === 'geometry_3d') {
            const THREE = await import('https://cdn.skypack.dev/three@0.160.0');
            const OrbitControls = await import('https://cdn.skypack.dev/three@0.160.0/examples/jsm/controls/OrbitControls.js');
            await import('./modules/three-d-renderer.js');
        } else if (topicType === 'statistics') {
            const d3 = await import('https://cdn.skypack.dev/d3@7.8.0');
            await import('./modules/d3-chart-renderer.js');
        }
    }
    ```
  - **Failure Mode (CDN Unreachable):** If `skypack` or `cdnjs` is blocked/offline, the `import()` will throw a network error.
    - **Mitigation:** Wrap the dynamic import in a `try-catch`. On failure, display a cached fallback image of the graph (stored in the browser's Cache API) with a prominent "Retry CDN" button. *Never* show a white screen of death.
- **Import Map (The Browser Dependency Resolver):**
  - Use `<script type="importmap">` in the HTML head to alias heavy dependencies. This ensures that when multiple modules import `three`, they use the exact same singleton instance, preventing double-loading.

---

### C3. Rendering Performance (60fps Simulation + Canvas Overlay)
*Decision needed: Overlaying a transparent canvas on a WebGL animation is the #1 cause of frame drops.*

- **The Compositing Layering Strategy (GPU Acceleration):**
  - **Decision:** The HTML structure must enforce strict layer separation:
    - **Layer 1 (Bottom):** The Simulation `<div>` (contains WebGL canvas for Three.js).
    - **Layer 2 (Top):** The Drawing Canvas (HTML5 Canvas) with `pointer-events: auto`.
  - **CSS Hack for 60fps:** Apply `will-change: transform` to Layer 1 to promote it to its own GPU composite layer. Apply `pointer-events: none` to the WebGL canvas so mouse events pass straight through to the drawing canvas.
- **Throttling the `mousemove` (The 20ms Rule):**
  - Teachers draw with a stylus/mouse. Firing an API call on every `mousemove` (60 times/sec) will absolutely murder the CPU and network.
  - **Decision:** Throttle the drawing capture to **20ms (50fps)**. Use `requestAnimationFrame` for smooth local rendering, but **batch** the stroke points into a single JSON array and send it to the server **only when the teacher lifts the pen (`mouseup` / `touchend`)**.
  - *Bonus:* If a teacher draws a long continuous line, split the batch every 100 points to prevent a gigantic payload at the end.
- **OffscreenCanvas (The Unsung Hero):**
  - If the simulation is purely 2D (e.g., D3 graphs) but requires heavy data processing, move the rendering to an **OffscreenCanvas** in a Web Worker.
  - **Decision:** For complex statistical plots, the Web Worker processes the data and paints to the OffscreenCanvas, which is then transferred to the main thread. This keeps the UI responsive while the CPU crunches numbers.

---

### C4. Hardware Adaptation (The "Old Projector" & "4GB RAM" Reality)
*Decision needed: Detect the hardware and degrade gracefully instead of crashing.*

- **Resolution Scaling (Adapting to 1024x768 Projectors):**
  - **Decision:** On load, detect `window.devicePixelRatio` and `screen.width`.
  - If the screen width < 1280px, we automatically **downscale the Three.js renderer** to 75% of the viewport and use CSS `transform: scale()` to stretch it. This drastically reduces the pixel count the GPU has to push.
  - Build a "Performance Mode" toggle in the settings. If enabled, it reduces the `antialiasing: false` in the WebGL renderer and reduces shadow map resolution from 2048x2048 to 512x512.
- **FPS Monitoring (The Dynamic Downscaler):**
  - **Decision:** Implement an FPS counter that runs silently in the background using `requestAnimationFrame` timestamps.
  - If the FPS drops below **30fps** for 5 consecutive seconds, we automatically:
    1. Disable shadows and post-processing effects.
    2. Reduce polygon count on geometric shapes (e.g., render a cylinder with 16 segments instead of 64).
    3. Show a subtle toast: "Your device is running slow. Reduced visual effects for smoother teaching."
- **Memory Constraints (4GB RAM):**
  - Chrome tabs easily eat 500MB+ each. 
  - **Decision:** Limit the `History` state (undo/redo) on the drawing canvas to a maximum of **20 undo steps** instead of unlimited. Store drawing strokes as simple coordinate arrays, not high-resolution raster images, to keep memory usage under 50MB per canvas.

---

### C5. Network Resilience (Offline Mode & Request Queuing)
*Decision needed: Wi-Fi drops mid-class. The teacher must not panic.*

- **Service Worker Strategy (Cache-First for Assets):**
  - **Decision:** Use Workbox (Google's library) to generate a Service Worker.
  - *Cache Rules:*
    1. **CDN Libraries (Three.js, D3):** Cache-first with a stale-while-revalidate strategy (TTL: 30 days).
    2. **Simulation Config JSON:** Cache-first, but check network in background (TTL: 1 hour).
    3. **Lesson Thumbnails/Images:** Cache-first (TTL: 7 days).
- **Background Sync for Annotations (The Queue):**
  - When a teacher draws and saves an annotation (on `mouseup`), we fire a `fetch` request to `/api/v1/save_annotation/`.
  - **Decision:** If `navigator.onLine` is false, or the request fails with a network error, we push the unsaved payload into an **IndexedDB queue** (using the `idb` library).
  - When the network returns, a background sync event triggers and flushes the queue. The teacher sees a small "Sync Pending" icon (like Google Drive) at the top, which turns green when flushed.
- **Stale Cache Fallback (The "Last Resort"):**
  - If the teacher is completely offline and tries to open a *new* lesson they've never loaded before, the Service Worker will fail to fetch it.
  - **Decision:** Show a pre-loaded "Static Snapshot" (a high-res PNG) of that lesson's default state, cached during the last successful login. It won't be interactive, but the teacher can still view and draw over the static image, salvaging the class period.

---

### C6. Browser Support Matrix (The Legacy Trap)
*Decision needed: Stop supporting ancient browsers that break ES6 modules.*

- **The Hard Cutoff:**
  - **Decision:** Officially support **Chrome 88+, Edge 88+, Firefox 90+**. 
  - **Justification:** Chrome 88 introduced the Origin Trial for `import.meta` and robust WebGL2 support. Anything older will have catastrophic performance and memory leaks.
- **The "Block" Screen:**
  - Inject a tiny script at the very top of `<head>` (before any CSS loads) that checks `if (typeof import === 'undefined' || !window.Promise) { window.location.href = '/unsupported-browser'; }`.
  - This instantly redirects Internet Explorer 11 and ancient Safari users to a friendly page explaining they need to install the latest Chrome.
- **Polyfill Strategy (Minimal):**
  - We will **not** transpile to ES5 (Babel). This saves 40KB in bundle size.
  - **Decision:** Only use a single polyfill: `core-js/stable/structured-clone` (for deep copying simulation state), loaded conditionally if the browser doesn't support it natively. Everything else stays modern.

---

### C7. Memory Leak Prevention (The "Lesson Switch" Killer)
*Decision needed: Switching between 10 lessons in a 45-minute class should not crash the tab.*

- **The Mandatory `cleanup()` Contract:**
  - Every simulation module (Three.js, D3, Canvas) must export a `cleanup()` function.
  - **Implementation:**
    ```javascript
    // three-d-renderer.js
    let scene, camera, renderer, animationId;
    
    export function mount(container) { ... } // sets up everything
    export function cleanup() {
        cancelAnimationFrame(animationId);
        renderer.dispose(); // Crucial for WebGL
        renderer.forceContextLoss();
        renderer.domElement.remove();
        scene.traverse(obj => { if (obj.geometry) obj.geometry.dispose(); });
        // Remove all event listeners
    }
    ```
  - **Enforcement:** The main `router.js` **must** call `currentModule.cleanup()` before dynamically importing the next module. If a developer forgets to implement `cleanup`, the router throws a hard error in staging.

---

### C8. Error Handling & UX Fallbacks (No Blank Screens)
*Decision needed: When a simulation crashes, the teacher must have a way to continue the class immediately.*

- **Global Error Boundary (Replicating React's concept):**
  - Since we don't have React, wrap the main `app.render()` loop in a `try-catch`.
  - On any uncaught JS error inside a simulation, we *do not* reload the whole page (which takes 10 seconds). Instead, we hide the broken simulation pane and display a modal overlay:
    > "Oops! This simulation encountered an error. [Restart Simulation] | [Go to Dashboard] | [Download Log]"
- **The "SnapShot & Report" Feature:**
  - When the error occurs, the `window.onerror` handler automatically captures:
    1. The current `location.href`.
    2. The last 100 lines of the console logs (using a buffer array).
    3. A screenshot of the canvas (using `canvas.toDataURL()`).
  - **Decision:** This data is compressed and sent to our Sentry/Datadog backend automatically. The teacher doesn't have to file a manual ticket.
- **Degradation Path for the Restart:**
  - When "Restart Simulation" is clicked, we do a hard reset of *only* that specific module's DOM container, re-import the JS file (forcing a fresh browser cache bypass), and remount it—all without losing the teacher's active session token.

---

### Frontend Team Sign-Off Checklist (To Leave the Meeting With)

| # | Action Item | Owner | Deadline |
| :--- | :--- | :--- | :--- |
| 1 | Adopt `mitt` (or built-in EventEmitter) and define the global `AppStore` with strict lifecycle methods (`mount`/`cleanup`). | FE Lead | Sprint 1 |
| 2 | Set up the `importmap` and implement Dynamic `import()` for Three.js, D3, and MathJax. Enforce the 150KB main bundle budget. | FE Lead | Sprint 1 |
| 3 | Implement the CSS layered compositing strategy (`will-change`, `pointer-events: none`) to hit 60fps. | FE UI Dev | Sprint 2 |
| 4 | Build the FPS Monitor with automatic downscaling triggers (disable shadows, reduce poly count). | FE Lead | Sprint 2 |
| 5 | Integrate Workbox Service Worker with Cache-first strategy for assets and IndexedDB queue for offline annotations. | FE Lead | Sprint 3 |
| 6 | Write the "Block" script for outdated browsers (Chrome < 88, IE, Safari < 14). | FE UI Dev | Sprint 1 |
| 7 | Create the `cleanup()` contract and add a linter rule (ESLint) that warns if a module exports `mount` but not `cleanup`. | FE Lead | Sprint 2 |
| 8 | Build the Global Error Boundary that captures console logs + canvas screenshots and reports to Sentry on crash. | FE Lead | Sprint 3 |
