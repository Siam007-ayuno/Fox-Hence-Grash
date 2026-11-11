# Fox--Goose-Beans


[3d_river_crossing_code_architecture_explanation.md](https://github.com/user-attachments/files/34679687/3d_river_crossing_code_architecture_explanation.md)
# Comprehensive Technical Architecture & Code Walkthrough
## 3D River Crossing & BFS State-Space Laboratory (`index.html`)
https://sites.google.com/view/realistic-3d-river-crossing-al/full-page-embed
---

### Executive Overview & Technology Stack

The project is structured as a self-contained, client-side single-page application (SPA) built without build-step overhead or external asset dependencies. Everything—from the 3D geometries and shaders to procedural audio soundscapes and graph visualization—runs directly in modern browsers via standard Web APIs and CDN packages.

| Layer | Library / API | Version / Source | Engineering Purpose |
| :--- | :--- | :--- | :--- |
| **3D Graphics Engine** | **Three.js** | `r128` | WebGL scene graph, custom procedural geometry meshes, PBR standard materials, shadow mapping, and camera projection. |
| **Camera Controls** | **OrbitControls.js** | `0.128.0` | Spherical orbit navigation, drag-to-rotate, pinch/scroll zoom, and polar angle constraints. |
| **Kinematics & Easing** | **GSAP (GreenSock)** | `3.12.2` | Parabolic boarding arc trajectories, boat recoil buoyancy, smooth camera transitions, and oar rowing strokes. |
| **Design System** | **Tailwind CSS** | `3.x (Play CDN)` | Utility styling configured in dark "Tactile Studio" palette (`#090A0F`, `#12131A`, `#181922`). |
| **Sound Engine** | **Web Audio API** | Native Browser API | Pure procedural audio synthesis (zero `.mp3` downloads): bandpass noise splash generator, dual oscillator alerts, and pentatonic fanfare. |
| **Celebration Effects** | **canvas-confetti** | `1.6.0` | Particle confetti burst triggered upon goal state verification. |
| **Typography & Icons** | **Google Fonts & FA** | `Plus Jakarta Sans`, `JetBrains Mono`, `Instrument Serif`, `FontAwesome 6` | Editorial typography pairing and tactical UI iconography. |

---

## 1. Algorithmic Foundation & State-Space Mathematics

### 1.1 State Vector Definition
Every configuration of the puzzle is formalized as an ordered 4-tuple:
$$S = (F, X, G, B) \in \{L, R\}^4$$
where:
- $F \in \{L, R\}$: Farmer's position (West Bank $L$, East Bank $R$)
- $X \in \{L, R\}$: Fox position (*Vulpes*)
- $G \in \{L, R\}$: Goose position (*Anser*)
- $B \in \{L, R\}$: Beans position (*Phaseolus*)

Total theoretical state configurations:
$$|S| = 2^4 = 16 \text{ states}$$

### 1.2 Constraint Safety Predicates
A state $S = (F, X, G, B)$ is valid if and only if neither unsupervised predation nor foraging occurs:

$$\text{Safe}(S) \iff \neg \Big(\big(X = G \land F \ne G\big) \lor \big(G = B \land F \ne G\big)\Big)$$

- **Predator Hazard**: $(X = G \land F \ne G)$ — The Fox devours the Goose if left together without the Farmer.
- **Forage Hazard**: $(G = B \land F \ne G)$ — The Goose devours the Beans if left together without the Farmer.

Evaluating all 16 states with this predicate prunes **6 invalid conflict states**, leaving exactly **10 valid states**:

```
Valid States (10):
(L,L,L,L), (R,L,R,L), (L,L,R,L), (R,R,R,L), (R,L,R,R),
(L,R,L,L), (L,L,L,R), (R,R,L,R), (L,R,L,R), (R,R,R,R)

Pruned Conflict States (6):
(R,L,L,L) -> Fox & Goose left together; Goose & Beans left together on L
(R,R,L,L) -> Goose & Beans left together on L
(R,L,L,R) -> Fox & Goose left together on L
(L,R,R,L) -> Fox & Goose left together on R
(L,L,R,R) -> Goose & Beans left together on R
(L,R,R,R) -> Fox & Goose left together; Goose & Beans left together on R
```

### 1.3 Breadth-First Search (BFS) & The "Step 4 Paradox"
Because the state-space is an unweighted directed graph $G = (V, E)$, Breadth-First Search (BFS) is mathematically guaranteed to find the minimal path length.

1. **Step 1: Goose First ($L,L,L,L \to R,L,R,L$)**: The Goose is the only common variable in both hazards. Leaving the Fox and Beans on Bank $L$ is safe because foxes do not eat beans.
2. **Steps 2 & 3: The Diamond Bifurcation**: After returning alone ($L,L,R,L$), the Farmer can take either the Fox ($R,R,R,L$, Branch A) or the Beans ($R,L,R,R$, Branch B). Both branches are mathematically symmetric.
3. **Step 4: The Backward Pivot**: In state $(R, R, R, L)$, if the Farmer returns alone, the Fox devours the Goose. If the Farmer leaves the Fox and takes nothing, no progress is made. Greedy hill-climbing algorithms fail here because moving an item back toward the start bank increases heuristic distance to the goal. BFS discovers that **returning the Goose to the West Bank ($L, R, L, L$)** is the unique safe transition bridge to depth 4.
4. **Steps 5–7: Convergence & Goal**: The Farmer rows the Beans East, returns West alone, and rows the Goose East for the final time to reach $(R, R, R, R)$ in exactly **7 optimal crossings**.

---

## 2. In-Depth Code Architecture by Module

### Module 1: Game State & Solution Matrices (Lines 490–630)
```javascript
const gameState = {
  farmerBank: 'L',    // Current bank of the boat and Farmer ('L' | 'R')
  boatCargo: null,    // Single item currently aboard ('fox' | 'goose' | 'beans' | null)
  positions: {
    fox: 'L',         // Entity location: 'L', 'R', or 'boat'
    goose: 'L',
    beans: 'L'
  },
  movesCount: 0,
  history: [],        // Full undo stack storing deep clones of previous states
  isRowing: false,    // Input lock flag preventing race conditions during animation
  gameOver: false,
  autoSolveTimer: null,
  activeBranch: 'A'   // Toggles Branch A (Fox first) or Branch B (Beans first)
};
```
- `SOLUTION_PATHS`: Contains full metadata for both symmetric 7-step solutions, including textual explanations, target vectors, and step numbers.
- `DAG_NODES` & `DAG_EDGES`: Pre-calculated 2D coordinates for the 16-node state diagram rendered into the SVG element.

---

### Module 2: Procedural Web Audio Synthesizer (Lines 632–718)
To eliminate external `.mp3` loading failures or latency, audio is synthesized directly on the audio bus:
- **`initAudio()`**: Instantiates `window.AudioContext` or `window.webkitAudioContext`, resuming suspended contexts following user gesture requirements.
- **`playTone(freq, type, duration, volume)`**: Creates an `OscillatorNode` connected to a `GainNode`, applying an exponential gain ramp (`exponentialRampToValueAtTime`) to eliminate audible audio clicks.
- **`playSplashSound()`**: Generates procedural water splashing by filling an `AudioBuffer` with white noise:
  $$x(t) = (2 \cdot \text{rand}() - 1) \cdot e^{-t / \tau}$$
  It routes the noise buffer through a `BiquadFilterNode` (`bandpass`, sweeping downward from $520\,\text{Hz}$ to $160\,\text{Hz}$) to mimic the resonant acoustic cavity of displaced river water.
- **`soundConflict()`**: Plays dual low-frequency sawtooth waves ($175\,\text{Hz} \to 130\,\text{Hz}$) to signal constraint failure.
- **`soundVictory()`**: Plays an ascending pentatonic arpeggio ($440\,\text{Hz}, 554\,\text{Hz}, 659\,\text{Hz}, 880\,\text{Hz}, 1108\,\text{Hz}$) using triangle oscillators.

---

### Module 3: Three.js World, Lighting & Water Shader (Lines 720–895)
- **Renderer Settings**: Configured with `ACESFilmicToneMapping` (exposure $1.15$), high-performance preference, and `PCFSoftShadowMap` for diffused contact shadows.
- **Dynamic Water Displacement**:
  A `THREE.PlaneGeometry(16, 44, 68, 68)` has its base vertex coordinates stored in an immutable array `baseWaterVerts`. In the `animate()` render loop, the Z-height of all $69 \times 69 = 4,761$ vertices is dynamically modulated using superimposed sinusoidal waves:
  $$z(u, v, t) = z_0 + 0.09 \sin(1.7u + 2.3t) + 0.06 \cos(1.3v + 1.8t)$$
  Calling `waterGeo.computeVertexNormals()` every frame re-calculates the surface normal vectors, creating authentic specular glints from the directional sunlight.
- **Landscape Generation**:
  `populateLandscape()` procedurally generates multi-tier pine trees, autumn golden foliage, mossy river rocks (using `THREE.DodecahedronGeometry`), and cattail reeds along the water's edge.

---

### Module 4: Procedural 3D Models & Clinker Boat Construction (Lines 897–1145)
Rather than loading bulky external GLTF/GLB files, all entities are built procedurally from geometric primitives:

1. **Clinker/Lapstrake Wooden Rowboat**:
   - **Hull Strakes**: Three stepped concentric rectangular boxes with varying widths ($1.65\,\text{m}$, $1.92\,\text{m}$, $2.15\,\text{m}$) simulate overlapping cedar planks.
   - **Framing**: Internal mahogany rib timbers, curved bow stem post, flat stern transom, slatted deck floorboards, and dual thwart benches.
   - **Rowlocks & Oars**: Brass torus rowlocks holding wooden shafts tipped with carved blades and copper edge bands.
2. **Farmer**:
   - Cylindrical denim overalls, articulated flannel sleeves, skin-toned head mesh, boots, and a wide-brim straw hat with a red ribbon band.
3. **Fox**:
   - Russet body cylinder, white fur chest bib, tapered hexagonal snout with dark nose sphere, erect ears, dark sock paws, and an articulated tail that wags via sinusoidal math in the render loop (`tail.rotation.z = Math.sin(t * 3.2) * 0.12`).
4. **Goose**:
   - Ellipsoidal plumage body, folded wing feather geometries, curved cylindrical neck, alert head with orange conical bill, dark eyes, and webbed feet.
5. **Beans**:
   - Burlap-textured sack cylinder with a cinched top cuff, tied rope ring, and 6 procedural green bean spheres clustered in the top opening.

---

### Module 5: Kinematics, GSAP Physics & Raycasting (Lines 1147–1280)
- **Boarding & Disembarking**:
  `animateBoardingJump()` and `animateDisembarkJump()` calculate coordinates between bank slots and the boat. GSAP tweens the horizontal axes ($X, Z$) linearly while applying a parabolic arc to the vertical axis ($Y$):
  $$Y(t) = Y_{\text{target}} + 1.65 \cdot \sin(\pi t)$$
  Simultaneously, the rowboat reacts to passenger weight with a physical recoil dip ($Y = -0.14$) and rotational list ($Z = \pm 0.05$).
- **Rowing Crossing**:
  `animateRowingCrossing()` translates the boat across the river ($X = -4.5 \leftrightarrow +4.5$) with `power2.inOut` easing over $1.65\,\text{s}$. The oars oscillate rhythmically (`repeat: 5`), and water wake particles (`THREE.RingGeometry`) are continuously spawned behind the boat stern.
- **Interactive Raycasting**:
  A `THREE.Raycaster` tracks the cursor coordinates. The `resolveActorRoot()` helper function walks up the Three.js scene hierarchy until finding `fox`, `goose`, `beans`, or `boat`, providing instant hover tooltips and single-click interaction.

---

### Module 6: Game Loop, Rule Verification & State Telemetry (Lines 1282–1530)
- **`getEffectivePositions()`**:
  Resolves passenger locations. If an item is in the boat, its effective bank is computed as `gameState.farmerBank`.
- **`evaluateBoardOutcome()`**:
  Evaluates the safety predicate after every move. If a conflict occurs, it halts the simulation, plays the error tone, and displays the **Conflict Modal** with the exact violation reason and an immediate **Undo** button.
- **Auto-Docking Victory Detection**:
  ```javascript
  if (eff.F === 'R' && eff.X === 'R' && eff.G === 'R' && eff.B === 'R') {
    triggerVictory();
  }
  ```
  Even if the final passenger (the Goose) is still sitting in the boat docked at the East Bank, victory triggers immediately, automatically animating the Goose out of the boat onto the grass, firing confetti, and displaying efficiency telemetry.

---

### Module 7: Live SVG 16-State DAG & Step Walkthrough (Lines 1532–1750)
- **Tab 1 (`1. All 7 Steps`)**:
  Renders an interactive step timeline. Clicking any step invokes `jumpToStateVector()`, instantly animating all 3D actors to match the selected step.
- **Tab 2 (`2. 16-State DAG`)**:
  Builds an SVG representation of all 16 states. Emerald boxes represent valid states, crimson dotted lines represent pruned hazard edges, and an amber pulsating circle tracks the player's active state in real time.
- **Tab 3 (`3. BFS Queue`)**:
  Exposes the real-time Boolean constraint engine, showing whether `(X == G) ∧ (F ≠ G)` or `(G == B) ∧ (F ≠ G)` evaluates to `TRUE` or `FALSE`, while displaying active BFS depth exploration layers.

---

## 3. Data Flow & Interaction Lifecycle Diagram

```
[ User Interaction ]
   │
   ├─► Click 3D Actor (Raycaster) OR Click Dock Button
   │      │
   │      ▼
   ├─► toggleItemBoarding(itemKey)
   │      ├─ Record Snapshot in gameState.history[] (Undo Stack)
   │      ├─ GSAP Parabolic Jump (animateBoardingJump / animateDisembarkJump)
   │      └─ Web Audio: soundHop()
   │
   ├─► rowBoatAcross()
   │      ├─ Update gameState.farmerBank (L <-> R)
   │      ├─ GSAP Boat Translation (X: -4.5 <-> 4.5) & Oar Stroking
   │      ├─ Spawn Dynamic Water Wake Rings
   │      └─ Web Audio: Procedural Water Splash Noise
   │
   └─► evaluateBoardOutcome()
          │
          ├──► Conflict Detected? ((X==G && F!=G) || (G==B && F!=G))
          │       ├─ soundConflict()
          │       ├─ Open Conflict Modal (Pruned State Alert)
          │       └─ Offer 1-Click Undo
          │
          ├──► Goal Reached? (F==R && X==R && G==R && B==R)
          │       ├─ Auto-Disembark Remaining Cargo to East Shore
          │       ├─ soundVictory() + Canvas Confetti Burst
          │       └─ Open Victory Modal with Efficiency Stats
          │
          └──► State Valid & Ongoing
                  ├─ Synchronize Live 4-Bit HUD: (F,X,G,B) & Binary Word
                  ├─ Pulse Active Node in SVG 16-State DAG
                  └─ Update BFS Candidate Transition Queue
```

---

## 4. Key Engineering Highlights & Best Practices Used

1. **Zero External Assets**: No `.gltf`, `.obj`, or `.mp3` files are fetched over the network. All models are procedurally composed of Three.js primitives and materials, and all audio is synthesized via the Web Audio API, making the application launch instantly with zero 404 or CORS risks.
2. **State Decoupling & Pure Kinematics**: Game logic is governed entirely by lightweight pure JavaScript primitives (`'L'`, `'R'`, `'boat'`). The 3D layer acts purely as a synchronized view layer driven by GSAP tweens.
3. **Comprehensive Edge-Case Handling**:
   - Single-item boat capacity is enforced automatically; attempting to board a second item smoothly unloads the first.
   - Input locking (`isRowing`) prevents concurrent clicks from corrupting animation state.
   - Automatic symmetric branch switching allows users to start with either the Fox or the Beans at Step 3 without desynchronizing the tutor.
