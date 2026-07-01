This is a **KWM (Keyword/Valence-shell Molecular) MVP** — a 3D molecular builder built in Three.js with a clean 4-layer architecture (Data → Logic → Renderer → Controller).

**What it is:** An interactive 3D molecular modeling tool where you can place atoms (H, He, C, N, O, F, Cl), move them near each other, and form chemical bonds based on electron cloud proximity detection — with real KWM geometry applied when bonds form.

**Architecture :**

- **Layer 1 — Data (`ChemData`):** Element table with valence electrons, radii, colors; tetrahedral/trigonal geometry presets; snap threshold constants.
- **Layer 2 — Logic (`KWMGeometry`, `AtomState`, `BondManager`, `SnapDetector`):** All chemistry math — cloud direction vectors, snap detection, bond formation/breaking, molecular orientation.
- **Layer 3 — Renderer (`SceneView`, `AtomView`, `BondView`):** All Three.js lives here — meshes, lights, camera orbit, cloud highlight sprites, bond cylinders. Reads from Logic, never writes back.
- **Layer 4 — Controller (`App`):** Thin glue — wires DOM events to Logic, pushes state changes to Renderer, manages mode (camera vs. move).

**Key features:**
- Electron cloud visualization (single electrons vs. lone pairs)
- Proximity-based snap detection (0.5u threshold) with gold highlighting
- KWM geometry: when bonds form, the central atom's partner is auto-positioned using tetrahedral/trigonal math with mirrored cloud directions
- Camera orbit + atom drag modes
- Algorithm log panel showing every step of snap detection and bonding
- Atom state inspector in the sidebar


