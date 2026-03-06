# PBI Lineage Explorer

A Power BI custom visual that renders an interactive, topological lineage graph showing how Dataflows, Datasets, and Reports connect across your Power BI tenant.

---

## Features

- **Topological stage layout** — nodes arranged left-to-right by dependency depth
- **S-curve edges** with directional arrows
- **Workspace, Dataflow, and Report filters** with live search and workspace subtitle
- **Node search** — filter cards by name
- **Click a node** to highlight its full upstream/downstream lineage
- **Right panel** — upstream/downstream list with direct/indirect indicators
- **Impact bar** — shows how many datasets and reports depend on a selected node
- **Refresh status** — Success / Failed / In Progress badges with relative timestamps
- **Stage collapse** — collapse any pipeline stage column
- **Normal / Compact** view toggle
- **Failed Only** filter
- **Workspace color legend**

---

## Quick Start

### Option A — Use the pre-built visual (easiest)

1. Download the latest `.pbiviz` file from the [Releases](https://github.com/SNLSHETTY87/PBI-Lineage-Explorer/releases) page of this repository
2. In Power BI Desktop or Service: **Insert > More visuals > Import from file**
3. Select the `.pbiviz` file
4. Follow the [DAX Setup](#dax-setup) steps below

### Option B — Build from source

```bash
# Prerequisites: Node.js 18+, npm
npm install -g powerbi-visuals-tools

# Clone and install
git clone <this-repo-url>
cd LineageVisualPBI
npm install

# Dev server (hot reload in Power BI)
npm run start

# Production package
npm run package
# Output: dist/LineageVisualPBI*.pbiviz
```

---

## DAX Setup

The visual requires **two DAX measures** added to your Power BI model.

### Step 1 — Prepare your data tables

You need two tables in your model:

| Table | Required Columns |
|-------|-----------------|
| `Nodes` | `NodeId`, `NodeName`, `NodeType` (`Dataflow`/`Dataset`/`Report`), `Workspace`, `Last Successful Refresh node`, `Latest Refresh Status` (`success`/`failed`/`progress`), `PBI_URL` |
| `Edges` | `SourceId`, `TargetId`, `Source`, `Target` |

> A sample Excel file (`PBI_Lineage_SampleData.xlsx`) with the correct column structure is included in this repo. Import it into Power BI as your starting point.

### Step 2 — Create the NodesJSON measure

The visual supports **two DAX approaches** — both work identically:

#### Option A — `CONCATENATEX` (classic, compatible with all Power BI versions)

```dax
NodesJSON =
VAR _rows =
    CONCATENATEX(
        Nodes,
        "{"
            & """NodeId"":"""      & SUBSTITUTE(Nodes[NodeId],   """","'") & ""","
            & """NodeName"":"""    &
                SUBSTITUTE(
                    SUBSTITUTE(Nodes[NodeName], """", "'"),
                    "&", "&amp;"
                ) & ""","
            & """NodeType"":"""    & Nodes[NodeType]    & ""","
            & """Workspace"":"""   & SUBSTITUTE(Nodes[Workspace], """","'") & ""","
            & """RefreshTime"":""" &
                IF(
                    ISBLANK(Nodes[Last Successful Refresh node]), "",
                    FORMAT(Nodes[Last Successful Refresh node], "YYYY-MM-DDTHH:mm:SS") & "Z"
                ) & ""","
            & """RefreshStatus"":""" &
                IF(ISBLANK(Nodes[Latest Refresh Status]), "", Nodes[Latest Refresh Status])
            & ""","
            & """PbiUrl"":"""      & IF(ISBLANK(Nodes[PBI_URL]), "", Nodes[PBI_URL]) & """"
        & "}",
        ","
    )
RETURN "[" & _rows & "]"
```

#### Option B — `TOJSON` (recommended for large models, requires Power BI May 2024+)

> ⚠️ **Always specify `maxCapacity`** (the second argument). Without it, `TOJSON` silently truncates at a tiny default row limit and only a fraction of your data returns.

```dax
NodesJSON TOJSON =
TOJSON(
    SELECTCOLUMNS(
        Nodes,
        "NodeId",       Nodes[NodeId],
        "NodeName",     Nodes[NodeName],
        "NodeType",     Nodes[NodeType],
        "Workspace",    Nodes[Workspace],
        "RefreshTime",
            IF(
                ISBLANK(Nodes[Last Successful Refresh node]),
                BLANK(),
                FORMAT(Nodes[Last Successful Refresh node], "yyyy-MM-ddTHH:mm:ss") & "Z"
            ),
        "RefreshStatus",
            IF(ISBLANK(Nodes[Latest Refresh Status]), BLANK(), Nodes[Latest Refresh Status]),
        "PbiUrl",
            IF(ISBLANK(Nodes[PBI_URL]), BLANK(), Nodes[PBI_URL])
    ),
    100000   -- maxCapacity: allow up to 100,000 rows
)
```

### Step 3 — Create the EdgesJSON measure

#### Option A — `CONCATENATEX`

```dax
EdgesJSON =
VAR _rows =
    CONCATENATEX(
        Edges,
        "{"
            & """SourceId"":""" & SUBSTITUTE([SourceId], """", "'") & ""","
            & """Source"":"""   & SUBSTITUTE([Source],   """", "'") & ""","
            & """TargetId"":""" & SUBSTITUTE([TargetId], """", "'") & ""","
            & """Target"":"""   & SUBSTITUTE([Target],   """", "'") & """"
        & "}",
        ","
    )
RETURN "[" & _rows & "]"
```

#### Option B — `TOJSON`

```dax
EdgesJSON TOJSON =
TOJSON(
    SELECTCOLUMNS(
        Edges,
        "SourceId", Edges[SourceId],
        "Source",   Edges[Source],
        "TargetId", Edges[TargetId],
        "Target",   Edges[Target]
    ),
    100000   -- maxCapacity: allow up to 100,000 rows
)
```

> The visual **auto-detects** the format and parses both correctly — no code changes needed when switching.

### Step 4 — Add the visual to your report

1. Add the **PBI Lineage Explorer** visual to a report page
2. In the **Fields** pane, drag:
   - `NodesJSON` measure → **Nodes JSON** field well
   - `EdgesJSON` measure → **Edges JSON** field well
3. The lineage graph will render automatically

---

## Sample Data

`PBI_Lineage_SampleData.xlsx` contains two sheets:

| Sheet | Description |
|-------|-------------|
| `Nodes` | Sample nodes (Dataflows, Datasets, Reports) across multiple workspaces |
| `Edges` | Sample dependency edges between nodes |

Import both sheets into your Power BI model and connect them to get started immediately.

---

## NodeType Values

| Value | Colour | Represents |
|-------|--------|------------|
| `Dataflow` | Blue | Power BI Dataflows (Gen1 or Gen2) |
| `Dataset` | Green | Semantic models / Datasets |
| `Report` | Orange | Power BI Reports |

---

## RefreshStatus Values

| Value | Badge | Meaning |
|-------|-------|---------|
| `success` | Green | Last refresh succeeded |
| `failed` | Pink | Last refresh failed |
| `progress` | Yellow (animated) | Refresh currently running |
| *(blank)* | — | No refresh data |

---

## Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `Ctrl+F` | Focus the search box |
| `Esc` | Clear selection, search, and failed filter |

---

## Project Structure

```
LineageVisualPBI/
├── src/
│   ├── visual.ts              # Main visual entry point + static DOM
│   ├── interfaces.ts          # NodeData, EdgeData types
│   ├── settings.ts            # Formatting settings
│   ├── renderer/
│   │   ├── Toolbar.ts         # Filter dropdowns with search
│   │   ├── CardBuilder.ts     # Node card renderer
│   │   ├── EdgeDrawer.ts      # SVG S-curve edge drawing
│   │   ├── LayoutEngine.ts    # Topological stage layout
│   │   ├── RightPanel.ts      # Upstream/downstream panel
│   │   ├── TooltipManager.ts  # Hover tooltips
│   │   └── ImpactBar.ts       # Impact analysis bar
│   └── utils/
│       ├── helpers.ts         # Color palette, escaping
│       └── graphUtils.ts      # Ancestor/descendant traversal
├── style/
│   └── visual.less            # All styles
├── capabilities.json          # Field well definitions
├── pbiviz.json                # Visual metadata
├── PBI_Lineage_SampleData.xlsx
└── Working PBI DAX Code HTML viewer.txt   # Standalone HTML version (no custom visual needed)
```

---

## Development

```bash
npm run start    # Dev server at https://localhost:8080 (enable in Power BI settings)
npm run package  # Build .pbiviz for distribution
npm run lint     # ESLint check
```

To test with the dev server in Power BI:
1. Go to Power BI Service → Settings → Enable developer visual
2. Add the **Developer Visual** from the visualization pane to a report

---

## Author

**Sunil Shetty** — snlshetty87@gmail.com

---

## License

MIT
