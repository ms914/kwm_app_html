´´´mermaid
<style>
#erd { padding: 1rem 0; }
#erd svg { width: 100%; height: auto; }
</style>
<div id="erd"></div>
<script type="module">
import mermaid from 'https://esm.sh/mermaid@11/dist/mermaid.esm.min.mjs';
const dark = matchMedia('(prefers-color-scheme: dark)').matches;
await document.fonts.ready;
mermaid.initialize({
  startOnLoad: false,
  theme: 'base',
  fontFamily: '"Anthropic Sans", sans-serif',
  themeVariables: {
    darkMode: dark,
    fontSize: '13px',
    fontFamily: '"Anthropic Sans", sans-serif',
    primaryColor:       dark ? '#1a2240' : '#e6f1fb',
    primaryTextColor:   dark ? '#b5d4f4' : '#0c447c',
    primaryBorderColor: dark ? '#378add' : '#185fa5',
    secondaryColor:     dark ? '#1a3020' : '#eaf3de',
    secondaryTextColor: dark ? '#c0dd97' : '#3b6d11',
    secondaryBorderColor: dark ? '#639922' : '#3b6d11',
    tertiaryColor:      dark ? '#2a1a10' : '#faeeda',
    tertiaryTextColor:  dark ? '#fac775' : '#633806',
    tertiaryBorderColor:dark ? '#ba7517' : '#854f0b',
    edgeLabelBackground: dark ? '#0b0d12' : '#ffffff',
    lineColor:          dark ? '#9c9a92' : '#73726c',
    textColor:          dark ? '#c2c0b6' : '#3d3d3a',
    clusterBkg:         dark ? '#111520' : '#f8f8f5',
    clusterBorder:      dark ? '#2a3050' : '#d3d1c7',
    titleColor:         dark ? '#d0d8f0' : '#1a1a18',
    nodeBorder:         dark ? '#378add' : '#185fa5',
    mainBkg:            dark ? '#1a2240' : '#e6f1fb',
  },
  flowchart: { curve: 'basis', padding: 20 },
});

const diagram = `flowchart TD

subgraph L1["Layer 1 — DATA  (no imports, no side-effects)"]
  direction TB
  CD["ChemData\n──────────────\nELEMENTS table\nGEOMETRIES table\nBOND_DISTANCE · SNAP_THRESHOLD\ncloudStates()"]
end

subgraph L2["Layer 2 — LOGIC  (plain JS, zero Three.js)"]
  direction TB
  V3["Vec3\n──────────────\nnormalize · dot · cross\nadd · sub · scale"]
  KWM["KWMGeometry\n──────────────\nbaseDirections()\nmirrorDirections()\norientMolecule()"]
  AS["AtomState\n──────────────\nid · symbol · pos · rot\nstates[] · bondMap{}\ncloudDirs[]\ncloudWorldPos(ci)\nfreeClouds()"]
  BM["BondManager\n──────────────\nadd() · breakBetween()\ngetBondsOf() · areBonded()\nremoveAtom()"]
  SD["SnapDetector\n──────────────\ndetect(atoms, bondMgr)\n→ [{aA,ciA,aB,ciB,dist}]"]
end

subgraph L3["Layer 3 — RENDERER  ▣ THREE.js boundary"]
  direction TB
  SV["SceneView\n──────────────\nWebGLRenderer\nPerspectiveCamera\nAmbientLight · DirectionalLight\nGridHelper\norbitBy() · zoomBy()"]
  AV["AtomView\n──────────────\nGroup + SphereGeometry (core)\nSphereGeometry × n (clouds)\nCanvasTexture (label)\nhighlightCloud()\nsetSelected()\nrefresh()  ← reads AtomState"]
  BV["BondView\n──────────────\nGroup + SphereGeometry × order\nsetFromUnitVectors (orient)\nupdate()  ← reads AtomState.pos"]
end

subgraph L4["Layer 4 — CONTROLLER"]
  direction TB
  APP["App\n──────────────\nmouse events → logic writes\nRaycaster (THREE.js hitTest)\ncheckSnaps() → SnapDetector\ncreateBonds() → BondManager\n             → KWMGeometry\n             → AtomView.refresh()"]
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
`;

const { svg } = await mermaid.render('kwm-svg', diagram);
document.getElementById('erd').innerHTML = svg;
</script>
´´´
