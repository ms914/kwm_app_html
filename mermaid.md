flowchart TD

subgraph L1["Layer 1 — DATA (no imports, no side-effects)"]
  direction TB
  CD["ChemData\n──────────────\nELEMENTS table\nGEOMETRIES table\nBOND_DISTANCE · SNAP_THRESHOLD\ncloudStates()"]
end

subgraph L2["Layer 2 — LOGIC (plain JS, zero Three.js)"]
  direction TB
  V3["Vec3\n──────────────\nnormalize · dot · cross\nadd · sub · scale"]
  KWM["KWMGeometry\n──────────────\nbaseDirections()\nmirrorDirections()\norientMolecule()"]
  AS["AtomState\n──────────────\nid · symbol · pos · rot\nstates[] · bondMap{}\ncloudDirs[]\ncloudWorldPos(ci)\nfreeClouds()"]
  BM["BondManager\n──────────────\nadd() · breakBetween()\ngetBondsOf() · areBonded()\nremoveAtom()"]
  SD["SnapDetector\n──────────────\ndetect(atoms, bondMgr)\n→ [{aA,ciA,aB,ciB,dist}]"]
end

subgraph L3["Layer 3 — RENDERER ▣ THREE.js boundary"]
  direction TB
  SV["SceneView\n──────────────\nWebGLRenderer\nPerspectiveCamera\nAmbientLight · DirectionalLight\nGridHelper\norbitBy() · zoomBy()"]
  AV["AtomView\n──────────────\nGroup + SphereGeometry (core)\nSphereGeometry × n (clouds)\nCanvasTexture (label)\nhighlightCloud()\nsetSelected()\nrefresh() ← reads AtomState"]
  BV["BondView\n──────────────\nGroup + SphereGeometry × order\nsetFromUnitVectors (orient)\nupdate() ← reads AtomState.pos"]
end

subgraph L4["Layer 4 — CONTROLLER"]
  direction TB
  APP["App\n──────────────\nmouse events → logic writes\nRaycaster (THREE.js hitTest)\ncheckSnaps() → SnapDetector\ncreateBonds() → BondManager\n → KWMGeometry\n → AtomView.refresh()"]
end

CD -->|"cloudStates()\nGEOMETRIES"| AS
CD -->|"SNAP_THRESHOLD"| SD
CD -->|"GEOMETRIES"| KWM
V3 --> KWM
KWM -->|"orientMolecule()\nreturns Vec3 directions"| APP
AS -->|"pos · cloudDirs\nbondMap"| BM
AS -->|"cloudWorldPos()"| SD
SD -->|"snap list"| APP
BM -->|"getBondsOf()"| KWM
SV --> AV
SV --> BV
AS -.->|"read by"| AV
AS -.->|"pos read by"| BV
APP -->|"creates / writes"| AS
APP -->|"calls"| BM
APP -->|"calls"| SD
APP -->|"creates / calls refresh()"| AV
APP -->|"creates / calls update()"| BV
APP -->|"orbitBy · zoomBy"| SV
