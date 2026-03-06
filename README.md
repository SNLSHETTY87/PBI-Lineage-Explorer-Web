# PBI Lineage Explorer (Standalone Web Version)

A self-contained HTML webpage that renders an interactive, topological lineage graph showing how Dataflows, Datasets, and Reports connect across your Power BI tenant. 

This repository contains the **standalone web version** of the PBI Lineage Explorer, which requires zero dependencies or backend infrastructure.

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

## How to Use

Because this tool is completely self-contained within `index.html`, you have two easy options to use it:

### Option A — Run Locally

1. Clone or download this repository.
2. Double-click `index.html` to open it in your web browser.
3. Your data stays entirely local in your browser.

### Option B — GitHub Pages (Hosted)

If you have enabled GitHub Pages for your fork of this repository:
1. Navigate to your repository's **Settings > Pages**.
2. Under "Build and deployment", select **Deploy from a branch**.
3. Choose the **`main`** branch and the **`/(root)`** folder.
4. Open your generated GitHub Pages URL!

---

## Inputting Data

Once the page is open, you will see the **Data Input Landing Page**, which supports three ways to load your JSON data:

1. **Combined JSON**: Paste an object containing both `{"nodes": [...], "edges": [...]}`.
2. **Separate Inputs**: Paste your Nodes JSON array and Edges JSON array into two distinct fields.
3. **Upload File**: Drag and drop a combined `.json` file for immediate loading.

You can also use the **Load Sample Data** button to quickly populate the inputs and explore the visualization's features.

---

## Power BI Integration (Generating the JSON)

To generate the required JSON structure directly from your Power BI data model, you need two tables (`Nodes` and `Edges`). The standalone web page includes an expandable **DAX Measures** section on the landing page that provides the necessary `CONCATENATEX` and `TOJSON` measures you can use.

### Required Tables

| Table | Required Columns |
|-------|-----------------|
| `Nodes` | `NodeId`, `NodeName`, `NodeType` (`Dataflow`/`Dataset`/`Report`), `Workspace`, `Last Successful Refresh node`, `Latest Refresh Status` (`success`/`failed`/`progress`), `PBI_URL` |
| `Edges` | `SourceId`, `TargetId`, `Source`, `Target` |

> A sample Excel file (`PBI_Lineage_SampleData.xlsx`) with the correct column structure is included in this repo. Import it into Power BI as your starting point.

---

## Author

**Sunil Shetty** — snlshetty87@gmail.com

<a href="https://buymeacoffee.com/snlshetty87" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" height="40" width="142"></a>

---

## License

MIT
