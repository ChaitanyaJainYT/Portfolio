# Master Prompt & Specification: Gravitational Solar System Portfolio

> **Instructions for AI Developer:** Build a fully responsive, single-file (`index.html`) interactive portfolio web application from scratch based on the architectural specification below. You have total creative freedom over the visual aesthetics, UI layout, CSS micro-interactions, dark mode styling, ambient effects, and HUD drawer components, provided the core N-body physics engine, orbital math, and data requirements are met.

---

## 1. Core Vision & Concept
The goal is an immersive, science-fiction interactive developer portfolio where projects are represented as celestial bodies (planets and orbiting moons) revolving around a central portfolio star core. 

Rather than a static grid or traditional menu, visitors navigate an interconnected planetary system governed by real-time N-body gravitational physics ($F = G \frac{m_1 m_2}{r^2}$).

---

## 2. Technical Architecture & Constraints
* **Single-File Delivery:** All code (HTML structural layout, embedded CSS/Tailwind, and JavaScript logic) must reside entirely inside a single `index.html` file.
* **Pure 2D Canvas Engine:** Use native HTML5 Canvas (`CanvasRenderingContext2D`) for 60FPS physics rendering, starfields, and constellation lines to eliminate heavy WebGL/3D library setup issues.
* **Zero Script Dependability:** Do not rely on external 3D engines (e.g., Three.js). You may use Tailwind CSS via CDN and FontAwesome icon libraries for UI styling.
* **Mobile-First Touch Compatibility:** Implement canvas drag-to-pan, pinch/scroll zoom, and tap selection with movement-threshold tolerance (preventing unintentional clicks while panning).

---

## 3. Core Engine Mechanics (Mandatory Physics Rules)

### A. N-Body Gravitational Physics
Implement gravitational attraction between all celestial bodies using Newton's Law of Universal Gravitation:
$$F = G \frac{m_1 m_2}{r^2}$$
* **Sun Core Mass:** The central star possesses the highest mass in the system (`M_sun ≈ 85,000`), acting as the primary orbital anchor.
* **Planet Mass & Radius Scaling:** Larger/more complex projects have a higher mass (`m`) and physical radius (`r`). 
* **Inter-Planetary Gravitational Sway:** Massive planets must exert subtle gravitational forces on neighboring planets, introducing organic orbital perturbations rather than rigid circular tracks.
* **Initial Velocity Vectors:** Assign initial velocity vectors ($v = \sqrt{\frac{G \cdot M_{sun}}{r}}$) perpendicular to the Sun vector to establish stable elliptical orbits upon initialization.

### B. Satellites & Constellation Networks
* **Orbiting Tech Moons:** Individual technologies, tools, or framework tags within a project orbit their respective parent planet as smaller satellite sub-nodes.
* **Constellation Lines:** Dynamically draw faint glowing laser lines connecting planets that share tech stack categories or explicit project relationships.

---

## 4. Creative Freedom: Theme, Aesthetics & UI/UX

*The implementing AI agent is encouraged to design and polish the visuals and UI using its own creative judgment, adhering to the following broad guidelines:*

* **Theme & Vibe:** High-tech tactical sci-fi command deck / deep space telemetry observer. It should feel like an active space radar or planetary scanner terminal.
* **Visual Palette:** Deep space dark mode (dark blues, deep slate, pitch black) highlighted by glowing neon stellar accents (cyan, magenta, amber sun core, emerald).
* **Interface Layout:** Design intuitive floating glassmorphism overlays for search, category filtering (e.g., Game Dev, AI, Web Apps, Systems), orbital speed control toggles (`Paused`, `1.0x`, `2.5x`), and project detail panels.
* **Project Detail Drawer/Modal:** When a user taps or locks camera target onto a planet, display its telemetry (title, description, project metrics, gravity mass, tech stack tags, key features, and live webpage link button).

---

## 5. Sample Dataset Structure

Initialize the engine with at least 6–8 distinct project planets with varying masses, sizes, orbital distances, and technology moons.

```json
[
  {
    "id": "project-id",
    "title": "Project Name",
    "sector": "Domain Category",
    "catKey": "GAME / AI / WEB / SYS",
    "desc": "Short overview description of the project.",
    "scale": "Scope level (e.g., Enterprise / Large / Medium)",
    "mass": 2500,
    "size": 24,
    "dist": 350,
    "color": "#00f0ff",
    "url": "https://example.com",
    "tags": ["Tech 1", "Tech 2", "Tech 3"],
    "features": ["Highlight 1", "Highlight 2"],
    "moons": ["MoonTech 1", "MoonTech 2"],
    "constellations": ["other-project-id-1", "other-project-id-2"]
  }
]
```

---

## 6. Execution Checklist for AI Agent
1. [ ] Build high-performance Canvas rendering and physics update loops (`requestAnimationFrame`).
2. [ ] Render procedural starfields, glowing central sun core, planet bodies, particle trails, and interactive constellations.
3. [ ] Implement camera auto-tracking and smooth pan/zoom math.
4. [ ] Build floating glassmorphic UI controls, category filter buttons, and project drawer.
5. [ ] Ensure full responsiveness and smooth touch support across iOS/Android mobile screens.