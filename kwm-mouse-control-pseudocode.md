# KWM App — Mouse Control Pseudocode

Companion to `kwm-app-pseudocode.md`, covering the CONTROLLER layer's input
handling: mode switching, hit testing, the drag-mode decision tree, and the
per-mode drag behaviors (camera orbit, atom translate, cloud rotate, and
their "whole bonded group" equivalents).

---

## 1. Two top-level modes

```
mode = 'cam' | 'move'          // toggled by the Camera/Move buttons

// 'cam'  -> every drag on the canvas orbits the camera, full stop.
// 'move' -> a drag either manipulates a hit atom, or — if nothing was
//           hit — falls back to orbiting the camera too.
```

---

## 2. Mouse-down — deciding what a drag will do

This is the important branch point: it inspects *what* was clicked (nothing
/ atom core / atom cloud) and *whether that atom is part of a bonded group*,
then stores one of several drag-mode strings for `onMove` to interpret.

```
FUNCTION onMouseDown(event):
    didDrag = false                      // becomes true on first real movement

    IF mode == 'cam':
        drag = { kind: 'cam', prevX: event.x, prevY: event.y }
        RETURN

    // mode == 'move': raycast into the scene from the mouse position
    hit = raycastAtoms(event.x, event.y)

    IF hit is nothing:
        drag = { kind: 'cam', prevX: event.x, prevY: event.y }   // fallback
        RETURN

    (atom, part) = hit                    // part = 'core' | 'cloud'
                                           // (H/He always report 'core')
    group = connectedComponent(atom)      // BFS over bondMap; size 1 if unbonded

    IF group.size > 1:
        // ── bonded group: whole-molecule manipulation ──
        IF part == 'core':
            dragKind = (event.button == RIGHT) ? 'group-translateY'
                                                : 'group-translateXZ'
        ELSE:  // part == 'cloud'
            dragKind = 'group-rotate'
        drag = { kind: dragKind, group,
                 startPositions: snapshot of group's current positions,
                 prevX: event.x, prevY: event.y }

    ELSE:
        // ── single, unbonded atom ──
        IF part == 'core':
            dragKind = (event.button == RIGHT) ? 'atom-translateY'
                                                : 'atom-translateXZ'
        ELSE:  // part == 'cloud'
            dragKind = 'atom-rotateClouds'
        drag = { kind: dragKind, atom, prevX: event.x, prevY: event.y }
```

**Decision table** (what a left/right drag does, by what you grabbed):

| Hit part | Bonded? | Left-drag | Right-drag |
|---|---|---|---|
| nothing | — | orbit camera | orbit camera |
| core | no | translate atom in XZ | translate atom in Y |
| cloud | no | rotate that atom's clouds | rotate that atom's clouds |
| core | yes (group) | translate whole group in XZ | translate whole group in Y |
| cloud | yes (group) | rotate whole group | rotate whole group |

Note that for a bonded atom, clicking *either* its core or a cloud that maps
to a `'cloud'` hit routes to `group-rotate` — the distinction between
"translate" and "rotate" is still governed by core-vs-cloud, it's just that
translate/rotate now apply to the *entire connected molecule* rather than
one atom.

---

## 3. Mouse-move — executing the stored drag mode

```
FUNCTION onMouseMove(event):
    IF drag is null: RETURN

    dx = event.x - drag.prevX
    dy = event.y - drag.prevY
    IF dx != 0 OR dy != 0: didDrag = true

    SWITCH drag.kind:

      CASE 'cam':
          orbitCamera(theta -= dx * SENSITIVITY, phi -= dy * SENSITIVITY)
          // phi is clamped to a safe range so the camera can't flip over the poles

      CASE 'group-translateXZ':
          (right, forward) = cameraRightAndForward()   // forward flattened to XZ plane
          worldDx = right * dx * pixelScale + forward * (-dy) * pixelScale
          FOR each atom, i in drag.group:
              atom.pos.xz = drag.startPositions[i].xz + worldDx
              refreshView(atom)
          drag.startPositions = re-snapshot current positions   // so next delta is relative
          refreshAllBondViews()

      CASE 'group-translateY':
          deltaY = (event.y - drag.prevY) * pixelScale
          FOR each atom in drag.group:
              atom.pos.y -= deltaY
              refreshView(atom)
          drag.startPositions = re-snapshot current positions
          refreshAllBondViews()

      CASE 'atom-translateXZ':
          (right, forward) = cameraRightAndForward()
          atom.pos.x += right.x * dx * pixelScale + forward.x * (-dy) * pixelScale
          atom.pos.z += right.z * dx * pixelScale + forward.z * (-dy) * pixelScale
          refreshView(atom); refreshAllBondViews()

      CASE 'atom-translateY':
          atom.pos.y -= dy * pixelScale
          refreshView(atom); refreshAllBondViews()

      CASE 'atom-rotateClouds':
          rotateAtomClouds(atom, dx, dy)          // see section 4

      CASE 'group-rotate':
          rotateGroupAroundCentroid(drag.group, dx, dy)   // see section 4

    drag.prevX = event.x
    drag.prevY = event.y
    recheckSnapCandidates()          // every move can create/break a snap
```

---

## 4. Rotation math (single atom vs. whole group)

Both rotation modes build the **same incremental quaternion** from mouse
delta, then apply it either to one atom's cloud directions or to an entire
group's positions *and* cloud directions.

```
FUNCTION buildDragRotation(dx, dy):
    speed = ROTATE_SENSITIVITY
    camRight = cameraRightVector()
    qYaw   = quaternionFromAxisAngle(worldUp,  angle: -dx * speed)
    qPitch = quaternionFromAxisAngle(camRight, angle: -dy * speed)
    RETURN qYaw * qPitch              // combined incremental rotation


FUNCTION rotateAtomClouds(atom, dx, dy):
    IF atom.isCentered: RETURN        // H / He have no orientable clouds

    q = buildDragRotation(dx, dy)
    currentDirs = atom.cloudDirs OR idealizedDirections(atom.numClouds)
    atom.cloudDirs = [ normalize(rotate(d, q)) for d in currentDirs ]
    atom.rot = IDENTITY               // cloudDirs now fully drive rendering
    refreshView(atom)
    refreshAllBondViews()             // bond visuals depend on cloud directions


FUNCTION rotateGroupAroundCentroid(group, dx, dy):
    centroid = average(atom.pos for atom in group)
    q = buildDragRotation(dx, dy)

    FOR each atom in group:
        // 1. orbit the atom's *position* around the shared centroid
        relative = atom.pos - centroid
        atom.pos = centroid + rotate(relative, q)

        // 2. also spin the atom's own cloud directions by the same q,
        //    so bond angles/orientations are preserved as the group turns
        IF NOT atom.isCentered:
            currentDirs = atom.cloudDirs OR idealizedDirections(atom.numClouds)
            atom.cloudDirs = [ normalize(rotate(d, q)) for d in currentDirs ]
            atom.rot = IDENTITY

        refreshView(atom)
    refreshAllBondViews()
```

Both cases rotate around a **camera-relative** axis pair (world-up for
horizontal drag, camera-right for vertical drag) rather than the object's
own local axes — this is what makes the rotation always feel like "drag
right, thing turns right" regardless of current camera orbit angle.

---

## 5. Mouse-up — click vs. drag, and selection

```
FUNCTION onMouseUp(event):
    IF NOT didDrag:
        // pure click (no movement happened) → treat as a selection toggle
        hit = raycastAtoms(event.x, event.y)
        IF hit exists:
            toggleSelection(hit.atom)
        ELSE:
            clearSelection()
    // if didDrag was true, the drag itself already did the work —
    // mouseup does nothing further to the model

    drag = null
    recheckSnapCandidates()


FUNCTION toggleSelection(atom):
    IF atom already in selected[]:
        remove it, un-highlight its view
    ELSE:
        IF selected[] already has 2 atoms:
            drop the oldest one (FIFO), un-highlight it
        add atom, highlight its view
    updateToolbarButtons()     // e.g. show "Break Bond" only when 2 bonded atoms selected
    updateInspectorPanel()
```

Selection is capped at 2 atoms at a time (enough to select a bond pair for
bonding/breaking); adding a 3rd bumps the oldest selection out, FIFO-style.

---

## 6. Hit testing (raycaster)

```
FUNCTION raycastAtoms(screenX, screenY):
    ndc = convert (screenX, screenY) to normalized device coordinates [-1, 1]
    ray = camera-space ray through ndc               // THREE.Raycaster
    candidateMeshes = flatten( every visible atom's [core, ...visibleClouds] )
                       // bonded clouds are hidden (invisible), so they can't be hit
    hits = ray.intersect(candidateMeshes), sorted nearest-first
    IF hits is empty: RETURN nothing

    (atomId, part) = hits[0].userData
    atom = lookup atom by atomId
    RETURN { atom, part: atom.isCentered ? 'core' : part }
    // H/He are always reported as 'core' even if the ray hit their single
    // cloud mesh, since centered atoms don't support independent cloud rotation
```

---

## 7. Screen-to-world scaling

Both translate modes need to convert a pixel delta into a world-space
distance that "feels" consistent regardless of zoom level.

```
FUNCTION pixelScale():
    halfFovRadians = camera.fieldOfView_radians / 2
    visibleWorldHeightAtCamDist = camDist * tan(halfFovRadians) * 2
    RETURN visibleWorldHeightAtCamDist / canvasHeightInPixels
    // world-units-per-pixel: bigger camDist (zoomed out) => bigger scale,
    // so a fixed pixel drag always moves the atom the same *visible* amount


FUNCTION cameraRightAndForward():
    right   = normalize(column 0 of camera's world matrix)
    forward = normalize(-1 * column 2 of camera's world matrix)
    forward.y = 0; normalize(forward)     // flatten to the ground plane for XZ dragging
    RETURN (right, forward)
```

---

## Summary: single atom vs. bonded group, side by side

| Action | Single unbonded atom | Bonded group |
|---|---|---|
| Left-drag core | translate that atom in XZ | translate **every** atom in the group in XZ together |
| Right-drag core | translate that atom in Y | translate the whole group in Y together |
| Left/right-drag cloud | rotate that atom's own cloud directions | rotate **positions + cloud directions** of every atom in the group around the group's centroid |
| What moves | one `AtomState.pos` / `.cloudDirs` | every member's `.pos` and (for non-centered atoms) `.cloudDirs` |
| Reference frame | camera-relative axes | same camera-relative axes, applied group-wide |

The group case is really the single-atom case applied in a loop with one
shared centroid/rotation — there's no separate "rigid body" data structure;
`connectedComponent()` just re-derives group membership from the bond graph
every time a drag starts.
