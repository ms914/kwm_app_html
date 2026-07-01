# KWM App — Pseudocode

Reference pseudocode for `kwm-mvp-11-6.html`, organized by the app's own four
layers (DATA → LOGIC → RENDER → CONTROLLER). The bonding logic — and
especially how it distinguishes single vs. double (multi-order) bonds — is
covered in depth in sections 3–5.

---

## 1. DATA layer — element & geometry constants

```
ELEMENTS = {
  symbol -> { valenceElectrons, color, coreRadius }
}   // H, He, C, N, O, F, Cl

GEOMETRIES = {
  1 -> [ one direction                              ]
  2 -> [ two directions,   opposite ends of a line   ]
  3 -> [ three directions, trigonal, 120° apart      ]
  4 -> [ four directions,  tetrahedral                ]
}
// These are the idealized KWM "maximum mutual distance" directions.

BOND_DISTANCE  = 1.7   // world units between bonded nuclei
SNAP_THRESHOLD = 0.5   // world units — clouds closer than this can bond

FUNCTION cloudStates(symbol, valenceElectrons):
    IF symbol is H:  RETURN [1]                 // one half-filled cloud
    IF symbol is He: RETURN [2]                 // one filled (lone-pair) cloud
    numClouds = min(4, valenceElectrons)
    states = array of `numClouds` ones           // start all singly-occupied
    extraElectrons = valenceElectrons - numClouds
    FOR i in 0 .. min(extraElectrons, numClouds) - 1:
        states[i] = 2                            // pair up from cloud 0 onward
    RETURN states
```

---

## 2. LOGIC layer — molecule state graph

```
CLASS AtomState:
    id, symbol, valence, numClouds
    states[]        // per-cloud occupation: 1 = single e-, 2 = lone pair
    bondMap{ cloudIndex -> { partner, partnerCloudIndex } }
    pos             // nucleus position
    cloudDirs       // null = use idealized GEOMETRIES; else explicit overrides

    FUNCTION freeClouds():
        RETURN cloud indices where states[i] == 1 AND not in bondMap
        // i.e. "bondable" clouds — singly-occupied AND not already bonded

    FUNCTION cloudWorldPos(ci):
        RETURN pos + cloudDirs[ci] * cloudDist    // world position of a cloud

CLASS BondManager:
    bonds = []   // flat list of { atomA, cloudIndexA, atomB, cloudIndexB }

    FUNCTION add(atomA, ciA, atomB, ciB):
        atomA.bondMap[ciA] = { partner: atomB, partnerCloudIndex: ciB }
        atomB.bondMap[ciB] = { partner: atomA, partnerCloudIndex: ciA }
        bonds.append({ atomA, ciA, atomB, ciB })
        // NOTE: each call records exactly ONE cloud-pair bond.
        // A double bond is represented as TWO such calls between the
        // same atom pair, on two different cloud indices each side —
        // there is no separate "bond order" field anywhere in the data
        // model. Order is always *derived* by counting.

    FUNCTION getBondsOf(atom):
        RETURN [ { partner, centralCloudIndex, partnerCloudIndex } for each
                 entry in atom.bondMap ]
```

---

## 3. Snap detection — finding bondable cloud pairs

This is the step that makes double bonds possible: it checks **every**
free-cloud pair between two atoms independently, so if two separate cloud
pairs are both within range at once, both snap.

```
FUNCTION detectSnaps(allAtoms, threshold):
    snaps = []
    FOR each unordered pair (atomA, atomB) in allAtoms:
        FOR each free cloud ciA of atomA:          // singly-occupied, unbonded
            FOR each free cloud cjB of atomB:       // singly-occupied, unbonded
                dist = distance(atomA.cloudWorldPos(ciA), atomB.cloudWorldPos(cjB))
                IF dist < threshold:
                    snaps.append({ atomA, ciA, atomB, cjB, dist })
    RETURN snaps
    // A single-bond pair (e.g. C–H) naturally produces one snap.
    // A double bond (e.g. C=C) produces TWO snaps for the same atom pair,
    // because two separate cloud pairs happen to be close simultaneously —
    // this only happens if the user has manually rotated/positioned both
    // atoms so two of their free clouds line up at once.
```

---

## 4. Bond creation — where "bond order" is *derived*

```
FUNCTION createBonds(allSnaps):
    // Group all pending snaps by which pair of atoms they connect.
    groupedByAtomPair = group(allSnaps, key = sorted(atomA.id, atomB.id))

    FOR each (atomPairKey, snapsForThisPair) in groupedByAtomPair:
        (atomA, atomB) = snapsForThisPair[0].atoms
        bondOrder = length(snapsForThisPair)     // 1 = single, 2 = double, 3 = triple

        FOR each snap in snapsForThisPair:
            BondManager.add(snap.atomA, snap.ciA, snap.atomB, snap.cjB)
        // registers each cloud-pair as its own bond; the *set* of bonds
        // between this atom pair is what later reads back as "order"

        create a BondView with strandCount = min(bondOrder, 3)   // visual only

        // ── geometry re-orientation ──
        centralAtom = whichever of (atomA, atomB) already has more
                       total bonds (tie → atomA)
        orientationResult = KWMGeometry.orientMolecule(centralAtom, BondManager)
        apply orientationResult to central atom + all bonded partners
        refresh every AtomView in the connected molecule
```

Key point: **bond order is never stored as a number on a bond.** It's
recomputed on demand by counting how many `bondMap` entries connect the same
two atoms. This is why `orientMolecule` (next section) has to *re-derive*
order too, by grouping bonds-per-partner.

---

## 5. KWM geometry — orienting a double bond correctly

This is the core of the double-bond-specific logic. Two sub-problems:

**(a)** Point the partner atom's *bonding* clouds back at the central atom.
**(b)** For a double bond specifically, twist the partner around the bond
axis so the two atoms' **lone-pair planes line up** — this is what makes the
molecule flatten into the correct planar shape (e.g. ethylene) instead of
leaving the partner's lone pairs pointing off at a random torsion angle.

```
FUNCTION orientMolecule(centralAtom, bondManager):
    centralDirs = idealized cloud directions for centralAtom.numClouds

    bondsByPartner = group(bondManager.getBondsOf(centralAtom), key = partner.id)

    FOR each (partner, bondsToThisPartner) in bondsByPartner:
        bondOrder = length(bondsToThisPartner)      // derived again, per-partner

        // bond axis = average direction of the central clouds involved
        // (for a double bond this averages 2 of the central atom's clouds)
        bondAxis = average( centralDirs[b.centralCloudIndex]
                             for b in bondsToThisPartner )
        normalize(bondAxis)

        partnerBondCloudIndices = [ b.partnerCloudIndex for b in bondsToThisPartner ]
        centralFreeDirs = centralDirs EXCLUDING the ones used for this bond
                           (i.e. central atom's lone pairs / other bonding clouds)

        mirroredDirs = mirrorDirections(
            partner.numClouds, partnerBondCloudIndices,
            bondAxis, centralFreeDirs, bondOrder
        )
        record { partner, bondAxis, mirroredDirs } for later application
    RETURN { centralDirs, partnerUpdates }


FUNCTION mirrorDirections(numClouds, bondCloudIndices, bondAxis,
                           centralFreeDirs, bondOrder):
    dirs = idealized directions for this cloud count

    // ── STEP 1 (always) — aim bonding cloud(s) at the central atom ──
    bondCloudAvg = average( dirs[i] for i in bondCloudIndices )
    normalize(bondCloudAvg)
    rotation1 = rotation that maps bondCloudAvg -> (-bondAxis)
    dirs = apply rotation1 to every direction in dirs
    // For a single bond this is the ONLY step: one cloud pointed at the
    // partner, everything else free to spin around the bond axis
    // (torsion is irrelevant with only one point of contact).

    // ── STEP 2 (only when bondOrder >= 2) — fix the torsion ──
    IF bondOrder >= 2 AND centralFreeDirs has >= 2 entries:
        partnerFreeIndices = all cloud indices NOT in bondCloudIndices

        IF partnerFreeIndices has >= 2 entries:
            // The plane containing a set of >=2 lone/free clouds has a
            // well-defined normal vector — this normal IS the molecule's
            // sigma-plane normal (the flat plane nuclei + lone pairs sit in).
            centralPlaneNormal = normalize(
                cross(centralFreeDirs[0], centralFreeDirs[1]) )
            partnerPlaneNormal = normalize(
                cross(dirs[partnerFreeIndices[0]], dirs[partnerFreeIndices[1]]) )

            // Pick whichever sign of centralPlaneNormal is the closer match,
            // since a plane normal is only defined up to sign.
            targetNormal = centralPlaneNormal is more aligned with
                            partnerPlaneNormal than its negation
                            ? centralPlaneNormal : -centralPlaneNormal

            // Signed angle, measured around bondAxis, between the two
            // plane normals — this is exactly the torsion error to correct.
            twistAngle = signedAngleAround(bondAxis, from: partnerPlaneNormal,
                                                       to: targetNormal)
            dirs = rotate every direction in dirs around bondAxis by twistAngle
        // else: fewer than 2 free clouds on the partner (e.g. it's O with
        // only 2 free clouds forming a double bond, both consumed by the
        // bond itself) — no torsion reference available, skip step 2.

    RETURN dirs
```

**Why this produces a correctly flattened double bond:** after step 1, the
partner is already pointed at the central atom, but it can still spin freely
around the bond axis like a doorknob — nothing yet forces its lone pairs
into the same plane as the central atom's lone pairs. Step 2 measures that
spin error (via the two plane-normal vectors) and rotates it out. Once both
atoms' free clouds share a plane, the two consumed "bonding" clouds on each
side (the ones now pointing at each other) automatically end up sitting
above/below that shared plane — geometrically equivalent to a σ-bond in the
shared plane plus a π-bond perpendicular to it, purely as a *side effect* of
the "maximum mutual distance" tetrahedral geometry, with no explicit
σ/π modeling anywhere in the code.

---

## 6. Rendering a double bond (visual-only, downstream of the geometry)

```
FUNCTION BondView.update():
    order = number of bonds between this atom pair          // derived, again
    strandCount = min(order, 3)                              // cap at 3 visual strands
    position strand group at bond midpoint, oriented along bond axis

    IF order == 1:
        draw single strand centered on the bond axis
        RETURN

    // ── multi-strand offset direction ──
    // Need a direction PERPENDICULAR to the bond axis, lying in the
    // pi-plane, to push the 2 (or 3) visual strands apart from each other.
    take atomA's two actual bonding-cloud directions for this partner
        (the ones with highest dot product toward the bond direction)
    piPlaneNormal = normalize(cross(cloudDirA_1, cloudDirA_2))
    offsetDir     = normalize(cross(piPlaneNormal, bondAxisDirection))

    IF offsetDir undefined (degenerate cross product):
        fall back to any axis perpendicular to the bond direction

    strandOffsets = order == 2 ? [-r, +r] : [-r, 0, +r]
    FOR each strand, i:
        position strand at offsetDir * strandOffsets[i]
    // Result: for a double bond, two ellipsoid strands sit side-by-side,
    // straddling the bond axis, in the plane perpendicular to the
    // molecule's lone-pair plane — visually reading as the two "banana
    // bonds" of the KWM double-bond convention.
```

---

## Summary: how a double bond differs from a single bond, end to end

| Stage | Single bond | Double bond |
|---|---|---|
| Snap detection | 1 cloud pair within threshold | 2 cloud pairs within threshold simultaneously |
| `BondManager` | 1 entry in `bonds[]` | 2 entries in `bonds[]` (same atom pair, different cloud indices) |
| Bond order | `bondsForThisPair.length == 1` | `bondsForThisPair.length == 2` (always *counted*, never stored) |
| `mirrorDirections` | Step 1 only (aim, torsion left free) | Step 1 **+** Step 2 (aim, then torsion-lock lone-pair planes coplanar) |
| Visual | 1 centered strand | 2 strands offset along the derived π-plane direction |
