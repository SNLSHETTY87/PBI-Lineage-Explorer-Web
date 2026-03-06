# How to Create the Nodes & Edges Tables for LineageVisual

This visual requires **two flat tables** — `Nodes` and `Edges` — loaded into your
Power BI semantic model. Both can be generated from the Power BI REST API using
Power Query M, or defined manually for testing.

---

## 1. Required Table Schemas

### Nodes Table

| Column | Type | Required | Description |
|---|---|---|---|
| `NodeId` | Text | Yes | Globally unique ID (e.g. `df_wsId_objectId`) |
| `NodeName` | Text | Yes | Display name of the item |
| `NodeType` | Text | Yes | Exactly: `Dataflow`, `Dataset`, or `Report` |
| `Workspace` | Text | Yes | Workspace display name |
| `RefreshTime` | DateTime | No | Last successful refresh timestamp |
| `RefreshStatus` | Text | No | One of: `success`, `failed`, `progress` |
| `PbiUrl` | Text | No | Direct URL to the item in Power BI Service |
| `SourceId` | Text | No | Leave blank / null for node rows |
| `TargetId` | Text | No | Leave blank / null for node rows |

### Edges Table

| Column | Type | Required | Description |
|---|---|---|---|
| `SourceId` | Text | Yes | `NodeId` of the upstream item |
| `TargetId` | Text | Yes | `NodeId` of the downstream item |
| `NodeId` | Text | No | Leave blank / null for edge rows |

> **Tip:** You MUST keep these as two separate tables. Do not append or union them, as they serve different purposes. They will later be combined into JSON strings via DAX measures to pass into the visual.

---

## 2. Power Query — REST API Approach (User Permissions)

This method works with any Power BI user account that has access to the
workspaces. It calls the standard Power BI REST API.

### Step 1 — Enable API Access

In Power BI Admin Portal → Tenant Settings, ensure:
- **Allow service principals to use read-only Power BI admin APIs** — On
- **Allow service principals to use Power BI APIs** — On (for service principal auth)

Or use your own Power BI account credentials (Power Query will prompt for OAuth).

---

### Step 2 — Create the `Nodes` Table Query

Paste this M code into a new blank query (`Home → Advanced Editor`):

```m
let
    BaseUrl = "https://api.powerbi.com/v1.0/myorg/",

    // ── Helper: call API and return value list ──────────────────────────────
    ApiGet = (relPath as text) =>
        let
            Raw  = Web.Contents(BaseUrl & relPath, [Headers = [Accept = "application/json"]]),
            Json = Json.Document(Raw)
        in
            if Record.HasFields(Json, "value") then Json[value] else {},

    // ── Get all accessible workspaces ───────────────────────────────────────
    Workspaces = ApiGet("groups?$top=500&$filter=type eq 'Workspace'"),

    // ── Build node rows for each workspace ─────────────────────────────────
    WorkspaceNodes = List.Transform(Workspaces, each
        let
            WsId   = _[id],
            WsName = _[name],

            // Dataflows
            Dataflows = ApiGet("groups/" & WsId & "/dataflows"),
            DfNodes = List.Transform(Dataflows, each [
                NodeId        = "df_" & WsId & "_" & _[objectId],
                NodeName      = _[name],
                NodeType      = "Dataflow",
                Workspace     = WsName,
                RefreshTime   = try _[configuredBy] otherwise null,   // placeholder
                RefreshStatus = null,
                PbiUrl        = "https://app.powerbi.com/groups/" & WsId & "/dataflows/" & _[objectId],
                _DfId         = _[objectId],
                _WsId         = WsId
            ]),

            // Datasets (semantic models)
            Datasets = ApiGet("groups/" & WsId & "/datasets"),
            DsNodes = List.Transform(
                List.Select(Datasets, each not Text.StartsWith(_[name], "Report Usage Metrics")),
                each [
                    NodeId        = "ds_" & WsId & "_" & _[id],
                    NodeName      = _[name],
                    NodeType      = "Dataset",
                    Workspace     = WsName,
                    RefreshTime   = try _[lastRefreshed] otherwise null,
                    RefreshStatus = try (if _[isRefreshable] then "success" else null) otherwise null,
                    PbiUrl        = "https://app.powerbi.com/groups/" & WsId & "/datasets/" & _[id],
                    _DsId         = _[id],
                    _WsId         = WsId
                ]
            ),

            // Reports
            Reports = ApiGet("groups/" & WsId & "/reports"),
            RpNodes = List.Transform(Reports, each [
                NodeId        = "rp_" & WsId & "_" & _[id],
                NodeName      = _[name],
                NodeType      = "Report",
                Workspace     = WsName,
                RefreshTime   = null,
                RefreshStatus = null,
                PbiUrl        = _[webUrl],
                _RpDsId       = try _[datasetId] otherwise null,
                _WsId         = WsId
            ])
        in
            DfNodes & DsNodes & RpNodes
    ),

    AllNodes     = List.Combine(WorkspaceNodes),
    NodesTable   = Table.FromRecords(AllNodes),
    KeepCols     = {"NodeId","NodeName","NodeType","Workspace","RefreshTime","RefreshStatus","PbiUrl"},
    NodesFinal   = Table.SelectColumns(NodesTable, KeepCols, MissingField.UseNull)
in
    NodesFinal
```

---

### Step 3 — Create the `Edges` Table Query

```m
let
    BaseUrl = "https://api.powerbi.com/v1.0/myorg/",

    ApiGet = (relPath as text) =>
        let
            Raw  = Web.Contents(BaseUrl & relPath, [Headers = [Accept = "application/json"]]),
            Json = Json.Document(Raw)
        in
            if Record.HasFields(Json, "value") then Json[value] else {},

    Workspaces = ApiGet("groups?$top=500&$filter=type eq 'Workspace'"),

    WorkspaceEdges = List.Transform(Workspaces, each
        let
            WsId    = _[id],
            Datasets = ApiGet("groups/" & WsId & "/datasets"),
            Reports  = ApiGet("groups/" & WsId & "/reports"),

            // Edge: Dataflow → Dataset
            // Datasets expose their upstream dataflow via datasources
            DfDsEdges = List.Combine(List.Transform(Datasets, each
                let
                    DsId      = _[id],
                    DsNodeId  = "ds_" & WsId & "_" & DsId,
                    Sources   = try ApiGet("groups/" & WsId & "/datasets/" & DsId & "/datasources") otherwise {},
                    // Datasources with type "Extension" or containing "dataflows" are dataflow references
                    DfSources = List.Select(Sources, each
                        try Text.Contains(Text.Lower(_[datasourceType]), "dataflow") otherwise false
                    ),
                    Edges = List.Transform(DfSources, each
                        let
                            // connectionDetails.path looks like:
                            // "workspaces/{wsGuid}/dataflows/{dfGuid}"
                            Path   = try _[connectionDetails][path] otherwise "",
                            Parts  = Text.Split(Path, "/"),
                            DfGuid = try Parts{List.Count(Parts)-1} otherwise null,
                            SrcWs  = try Parts{1} otherwise WsId,
                            SrcId  = "df_" & SrcWs & "_" & DfGuid
                        in
                            if DfGuid <> null
                            then [SourceId = SrcId, TargetId = DsNodeId]
                            else null
                    ),
                    ValidEdges = List.Select(Edges, each _ <> null)
                in
                    ValidEdges
            )),

            // Edge: Dataset → Report
            DsRpEdges = List.Transform(
                List.Select(Reports, each Record.HasFields(_, "datasetId") and _[datasetId] <> null),
                each [
                    SourceId = "ds_" & WsId & "_" & _[datasetId],
                    TargetId = "rp_" & WsId & "_" & _[id]
                ]
            )
        in
            DfDsEdges & DsRpEdges
    ),

    AllEdges   = List.Combine(WorkspaceEdges),
    EdgeTable  = Table.FromRecords(AllEdges),
    EdgeFinal  = Table.SelectColumns(EdgeTable, {"SourceId", "TargetId"})
in
    EdgeFinal
```

---

## 3. Admin Scanner API Approach (Best for Complete Cross-Workspace Lineage)

The **Workspace Scanner API** returns full lineage including cross-workspace
dataflow references in a single call. Requires a Power BI admin account or
service principal with admin read permission.

```m
let
    // ── Phase 1: trigger a workspace scan ──────────────────────────────────
    AdminBase = "https://api.powerbi.com/v1.0/myorg/admin/workspaces/",

    // Get all workspace IDs
    AllWs = Json.Document(
        Web.Contents(AdminBase & "modified?excludePersonalWorkspaces=True",
                     [Headers = [Accept = "application/json"]])
    )[workspaceIds],

    // Post scan request (up to 100 workspaces per request)
    Batches    = List.Split(AllWs, 100),
    ScanIds    = List.Transform(Batches, each
        let
            Body    = Json.FromValue([workspaces = _]),
            PostRaw = Web.Contents(AdminBase & "getInfo?datasetExpressions=False&lineage=True",
                          [Content = Body,
                           Headers = [#"Content-Type" = "application/json",
                                      Accept          = "application/json"]])
        in Json.Document(PostRaw)[id]
    ),

    // ── Phase 2: poll until scan is complete, then fetch results ───────────
    // (In practice: wait ~30s, then call /scanResult/{scanId})
    FetchScan = (scanId as text) =>
        Json.Document(
            Web.Contents(AdminBase & "scanResult/" & scanId,
                         [Headers = [Accept = "application/json"]])
        )[workspaces],

    WorkspacesData = List.Combine(List.Transform(ScanIds, FetchScan)),

    // ── Phase 3: build Nodes ───────────────────────────────────────────────
    AllNodeRecords = List.Combine(List.Transform(WorkspacesData, each
        let
            WsId   = _[id],
            WsName = _[name],

            DfNodes = List.Transform(try _[dataflows] otherwise {}, each [
                NodeId        = "df_" & WsId & "_" & _[objectId],
                NodeName      = _[name],
                NodeType      = "Dataflow",
                Workspace     = WsName,
                RefreshTime   = null,
                RefreshStatus = null,
                PbiUrl        = "https://app.powerbi.com/groups/" & WsId & "/dataflows/" & _[objectId]
            ]),

            DsNodes = List.Transform(try _[datasets] otherwise {}, each [
                NodeId        = "ds_" & WsId & "_" & _[id],
                NodeName      = _[name],
                NodeType      = "Dataset",
                Workspace     = WsName,
                RefreshTime   = try _[lastRefreshTime] otherwise null,
                RefreshStatus = try (if _[lastRefreshTime] <> null then "success" else null) otherwise null,
                PbiUrl        = "https://app.powerbi.com/groups/" & WsId & "/datasets/" & _[id]
            ]),

            RpNodes = List.Transform(try _[reports] otherwise {}, each [
                NodeId        = "rp_" & WsId & "_" & _[id],
                NodeName      = _[name],
                NodeType      = "Report",
                Workspace     = WsName,
                RefreshTime   = null,
                RefreshStatus = null,
                PbiUrl        = try _[webUrl] otherwise null
            ])
        in
            DfNodes & DsNodes & RpNodes
    )),

    // ── Phase 4: build Edges ───────────────────────────────────────────────
    AllEdgeRecords = List.Combine(List.Transform(WorkspacesData, each
        let
            WsId = _[id],

            // Dataflow → Dataset (via dataset.upstreamDataflows)
            DfDsEdges = List.Combine(List.Transform(try _[datasets] otherwise {}, each
                let
                    DsId  = _[id],
                    UpDfs = try _[upstreamDataflows] otherwise {}
                in
                    List.Transform(UpDfs, each [
                        SourceId = "df_" & _[groupId] & "_" & _[targetDataflowId],
                        TargetId = "ds_" & WsId & "_" & DsId
                    ])
            )),

            // Dataflow → Dataflow (chained dataflows via upstreamDataflows on dataflow)
            DfDfEdges = List.Combine(List.Transform(try _[dataflows] otherwise {}, each
                let
                    DfId  = _[objectId],
                    UpDfs = try _[upstreamDataflows] otherwise {}
                in
                    List.Transform(UpDfs, each [
                        SourceId = "df_" & _[groupId] & "_" & _[targetDataflowId],
                        TargetId = "df_" & WsId & "_" & DfId
                    ])
            )),

            // Dataset → Report
            DsRpEdges = List.Transform(
                List.Select(try _[reports] otherwise {}, each Record.HasFields(_, "datasetId")),
                each [
                    SourceId = "ds_" & WsId & "_" & _[datasetId],
                    TargetId = "rp_" & WsId & "_" & _[id]
                ]
            )
        in
            DfDsEdges & DfDfEdges & DsRpEdges
    ))
in
    // Return both — use "AllNodeRecords" or "AllEdgeRecords" as the query result
    // by renaming / duplicating this query
    AllEdgeRecords   // swap to AllNodeRecords for the Nodes table
```

> Create **two separate queries** from this code — one returning `AllNodeRecords`
> and one returning `AllEdgeRecords` — by renaming the final `in` expression.

---

## 4. TMDL Table Definitions

Add these table definitions to your semantic model's `.tmdl` files.
Place each file in the `tables/` folder of your TMDL project.

### `tables/Nodes.tmdl`

```tmdl
table Nodes

    column NodeId
        dataType: string
        sourceColumn: NodeId
        summarizeBy: none

    column NodeName
        dataType: string
        sourceColumn: NodeName
        summarizeBy: none

    column NodeType
        dataType: string
        sourceColumn: NodeType
        summarizeBy: none
        /// Valid values: Dataflow | Dataset | Report

    column Workspace
        dataType: string
        sourceColumn: Workspace
        summarizeBy: none

    column RefreshTime
        dataType: dateTime
        sourceColumn: RefreshTime
        formatString: General Date
        summarizeBy: none

    column RefreshStatus
        dataType: string
        sourceColumn: RefreshStatus
        summarizeBy: none
        /// Valid values: success | failed | progress | (blank)

    column PbiUrl
        dataType: string
        sourceColumn: PbiUrl
        summarizeBy: none

    annotation PBI_ResultType = Table

```

### `tables/Edges.tmdl`

```tmdl
table Edges

    column SourceId
        dataType: string
        sourceColumn: SourceId
        summarizeBy: none
        /// Must match a NodeId in the Nodes table

    column TargetId
        dataType: string
        sourceColumn: TargetId
        summarizeBy: none
        /// Must match a NodeId in the Nodes table

    annotation PBI_ResultType = Table

```

### `model.tmdl` — add partition sources

For each table, add a partition pointing to your Power Query query:

```tmdl
    partition Nodes-partition
        mode: import
        source
            type: m
            expression =
                let
                    Source = Nodes   // references your Power Query query named "Nodes"
                in
                    Source

    partition Edges-partition
        mode: import
        source
            type: m
            expression =
                let
                    Source = Edges   // references your Power Query query named "Edges"
                in
                    Source
```

---

## 5. Manual Sample Data (for testing)

Use `Enter Data` in Power BI Desktop to paste these rows directly.

### Nodes (paste into Enter Data)

| NodeId | NodeName | NodeType | Workspace | RefreshTime | RefreshStatus | PbiUrl |
|---|---|---|---|---|---|---|
| df_ws1_001 | Sales Source Dataflow | Dataflow | Analytics WS | 2026-02-24 06:00:00 | success | |
| df_ws1_002 | HR Source Dataflow | Dataflow | Analytics WS | 2026-02-24 07:30:00 | failed | |
| ds_ws1_101 | Sales Dataset | Dataset | Analytics WS | 2026-02-24 06:10:00 | success | |
| ds_ws1_102 | HR Dataset | Dataset | Analytics WS | | | |
| rp_ws1_201 | Sales Dashboard | Report | Analytics WS | | | |
| rp_ws1_202 | HR Overview | Report | Analytics WS | | | |
| rp_ws1_203 | Executive Summary | Report | Analytics WS | | | |

### Edges (paste into Enter Data)

| SourceId | TargetId |
|---|---|
| df_ws1_001 | ds_ws1_101 |
| df_ws1_002 | ds_ws1_102 |
| ds_ws1_101 | rp_ws1_201 |
| ds_ws1_101 | rp_ws1_203 |
| ds_ws1_102 | rp_ws1_202 |
| ds_ws1_102 | rp_ws1_203 |

---

## 6. Create JSON Measures & Field Well Mapping

After successfully loading the `Nodes` and `Edges` tables into your Power BI semantic model, you need to create **two DAX measures** that convert your table columns into JSON strings so the visual can consume them efficiently.

The visual supports **two DAX approaches** — both are parsed automatically:

### Step 1 — Create the `NodesJSON` Measure

#### Option A — `CONCATENATEX` (classic, works with all Power BI versions)

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
                    ISBLANK(Nodes[RefreshTime]), "",
                    FORMAT(Nodes[RefreshTime], "YYYY-MM-DDTHH:mm:SS") & "Z"
                ) & ""","
            & """RefreshStatus"":""" &
                IF(ISBLANK(Nodes[RefreshStatus]), "", Nodes[RefreshStatus])
            & ""","
            & """PbiUrl"":"""      & IF(ISBLANK(Nodes[PbiUrl]), "", Nodes[PbiUrl]) & """"
        & "}",
        ","
    )
RETURN "[" & _rows & "]"
```

#### Option B — `TOJSON` (recommended for large models, Power BI May 2024+)

> ⚠️ **Always specify `maxCapacity`** (second argument). Without it, `TOJSON` silently truncates at a tiny default row limit — only a few rows are returned regardless of your table size.

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
                ISBLANK(Nodes[RefreshTime]),
                BLANK(),
                FORMAT(Nodes[RefreshTime], "yyyy-MM-ddTHH:mm:ss") & "Z"
            ),
        "RefreshStatus",
            IF(ISBLANK(Nodes[RefreshStatus]), BLANK(), Nodes[RefreshStatus]),
        "PbiUrl",
            IF(ISBLANK(Nodes[PbiUrl]), BLANK(), Nodes[PbiUrl])
    ),
    100000   -- maxCapacity: allow up to 100,000 rows
)
```

### Step 2 — Create the `EdgesJSON` Measure

#### Option A — `CONCATENATEX`

```dax
EdgesJSON =
VAR _rows =
    CONCATENATEX(
        Edges,
        "{"
            & """SourceId"":""" & SUBSTITUTE([SourceId], """", "'") & ""","
            & """TargetId"":""" & SUBSTITUTE([TargetId], """", "'") & """"
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

> The visual **auto-detects** the format and parses both correctly. You can freely switch without changing any visual settings.

### Step 3 — Field Well Mapping in the Visual

Finally, map the two new DAX measures to the visual:

1. Insert the **LineageVisual** onto your report canvas.
2. In the **Fields** pane, drag:
   - Your `NodesJSON` measure into the **Nodes JSON** field well.
   - Your `EdgesJSON` measure into the **Edges JSON** field well.

The lineage graph will render automatically!

---

## 7. Tips & Troubleshooting

| Issue | Fix |
|---|---|
| API returns 401 | Sign in with an account that has access to the workspaces; in Power Query go to **Data Source Settings → Edit Credentials** and choose **Organizational account** |
| No edges visible | Check that `SourceId` / `TargetId` values exactly match the `NodeId` values — they are case-sensitive |
| Scanner API returns empty | The scan may still be running; wait 30–60 seconds and refresh, or add a `Function.InvokeAfter` delay |
| Dataset → Dataflow edge missing | Not all datasets expose their dataflow datasource via the standard API; use the Admin Scanner API for complete cross-workspace lineage |
| RefreshStatus not showing bars | Values must be lowercase: `success`, `failed`, or `progress` (the visual lower-cases on import automatically) |
| Too many items / slow load | Filter workspaces in the M code with `$filter=name eq 'My Workspace'` or limit to specific workspace IDs |

---

## 8. Refresh Status via Refresh History API

To populate `RefreshStatus` accurately, call the refresh history endpoint per dataset:

```m
// For a single dataset:
GetRefreshStatus = (wsId as text, dsId as text) =>
    let
        Raw     = Json.Document(
                      Web.Contents("https://api.powerbi.com/v1.0/myorg/groups/" &
                                   wsId & "/datasets/" & dsId & "/refreshes?$top=1",
                                   [Headers = [Accept = "application/json"]])
                  ),
        History = try Raw[value] otherwise {},
        Latest  = if List.Count(History) > 0 then History{0} else null,
        Status  = if Latest = null then null
                  else if Latest[status] = "Completed" then "success"
                  else if Latest[status] = "Failed"    then "failed"
                  else if Latest[status] = "Unknown"   then "progress"
                  else null
    in
        Status
```

Call this function for each dataset row in your Nodes query and use the
result as the `RefreshStatus` value.
