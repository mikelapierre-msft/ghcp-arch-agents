---
name: diagram-generation
description: "Generate architecture diagrams using Draw.io with Azure shapes. Opens the Draw.io editor, creates a new empty diagram, loads Azure shape libraries, builds the diagram, exports it as an image, and references it in the target markdown file."
---

# Diagram Generation

## When to Use

- Creating architecture diagrams for Contoso architecture documents
- Visualizing infrastructure topology, data flows, or networking layouts
- Generating Azure-based architecture diagrams with official Azure icons
- Exporting diagrams as images (PNG/SVG) for embedding in markdown documentation

## Prerequisites

- The Draw.io MCP server must be running at `http://localhost:3000/`
- The `drawio-mcp-se` MCP tools must be available

## Procedure

### Step 1: Open the Draw.io Editor

Open the Draw.io editor in the VS Code integrated browser so the diagram is visible during creation:

```
Use #tool:open_browser_page with url "http://localhost:3000/"
```

This returns a `pageId` — keep it for later browser interactions if needed.

### Step 2: Create a New Empty Diagram

Discard any existing diagram content and start fresh by importing an empty diagram:

```
Use #tool:mcp_drawio-mcp-se_import-diagram with:
  - data: "<mxGraphModel><root><mxCell id='0'/><mxCell id='1' parent='0'/></root></mxGraphModel>"
  - format: "xml"
  - mode: "replace"
```

This replaces the current diagram with a blank canvas containing only the root cells.

### Step 3: Azure Shape Style Reference

Azure shapes are **not discoverable** via the MCP shape API (`get-shape-categories` / `get-shape-by-name`). They are rendered by the Draw.io editor UI but not indexed by the MCP server.

Azure shapes use `mxgraph.azure.*` style keys. The style format is:

```
shape=mxgraph.azure.{shape_name};fillColor=#0078D4;fontColor=#FFFFFF;strokeColor=none;
```

To **discover the exact style** for any Azure shape:
1. Drag the shape from the Draw.io sidebar onto the canvas
2. Use `#tool:mcp_drawio-mcp-se_get-selected-cell` to capture its full style string
3. Record the `shape=mxgraph.azure.*` value for reuse

### Step 4: Build the Diagram

Use the Draw.io MCP tools to construct the architecture diagram.

#### Adding Azure Components

Use `#tool:mcp_drawio-mcp-se_add-rectangle` with the `mxgraph.azure.*` shape style. Example for Azure App Service:

```
Use #tool:mcp_drawio-mcp-se_add-rectangle with:
  - text: "App Service"
  - x, y: position on the canvas
  - width: 50, height: 50
  - style: "shape=mxgraph.azure.app_service;fillColor=#0078D4;fontColor=#FFFFFF;strokeColor=none;"
```

Example for Azure Kubernetes Service (AKS):

```
Use #tool:mcp_drawio-mcp-se_add-rectangle with:
  - text: "AKS Cluster"
  - x, y: position on the canvas
  - width: 50, height: 50
  - style: "shape=mxgraph.azure.kubernetes_service;fillColor=#0078D4;fontColor=#FFFFFF;strokeColor=none;"
```

#### Adding Generic Components

For components without Azure shapes, use rectangles with custom styling:

```
Use #tool:mcp_drawio-mcp-se_add-rectangle with:
  - text: the label
  - x, y: position
  - width, height: size
  - style: "whiteSpace=wrap;html=1;fillColor=#dae8fc;strokeColor=#6c8ebf;"
```

#### Connecting Components

Create edges between components to show data flow or relationships:

```
Use #tool:mcp_drawio-mcp-se_add-edge with:
  - source_id: ID of the source cell (returned when creating cells)
  - target_id: ID of the target cell
  - text: (optional) label for the connection (e.g., "HTTPS", "gRPC")
  - style: edge style string
```

#### Grouping with Layers or Parent Cells

Use layers to organize logical groups (e.g., "Frontend", "Backend", "Data"):

```
Use #tool:mcp_drawio-mcp-se_create-layer with name for the group
```

Or use a rectangle as a container and set `parent_id` on child cells.

#### Layout Guidelines

- Place components left-to-right or top-to-bottom following the data flow
- Use consistent spacing (e.g., 80-100px between components)
- Group related services visually (e.g., all data services in one area)
- Use labels on edges to describe the protocol or data flow (e.g., "HTTPS", "gRPC", "Events")
- Use Azure-branded colors when not using Azure shapes:
  - Azure Blue: `#0078D4`
  - Light Blue fill: `#E6F2FF`
  - Border: `#0078D4`

### Step 5: Export the Diagram as an Image

Export the completed diagram as a PNG image to the architecture document folder:

```
Use #tool:mcp_drawio-mcp-se_export-diagram with:
  - format: "png"
  - output_path: absolute path to the target image file (e.g., "c:\path\to\architectures\{app-name}\architecture-diagram.png")
  - dpi: 150 (for good quality in documents)
  - crop: true
  - background: "#ffffff"
  - border: 10
```

For SVG export (preferred for scalability):

```
Use #tool:mcp_drawio-mcp-se_export-diagram with:
  - format: "svg"
  - output_path: absolute path to the target SVG file
  - crop: true
  - background: "#ffffff"
  - border: 10
```

### Step 6: Reference the Image in the Markdown File

Add the exported image to the target markdown file using a relative path:

```markdown
![Architecture Diagram](./architecture-diagram.png)
```

or for SVG:

```markdown
![Architecture Diagram](./architecture-diagram.svg)
```

Insert the image reference in the appropriate section of the architecture document (typically in `overview.md` under the "Architecture Overview" heading, or in the relevant domain-specific file).

## Tips

- **Discover Azure styles** by dragging a shape from the UI sidebar, then using `#tool:mcp_drawio-mcp-se_get-selected-cell` to capture its style.
- **Inspect the model** using `#tool:mcp_drawio-mcp-se_list-paged-model` to review what cells exist and their IDs after adding them.
- **Edit cells** after creation using `#tool:mcp_drawio-mcp-se_edit-cell` to adjust position, size, or style.
- **Edit edges** after creation using `#tool:mcp_drawio-mcp-se_edit-edge` to change routing or labels.
- **Delete mistakes** using `#tool:mcp_drawio-mcp-se_delete-cell-by-id` to remove cells.
- **Use containers** for grouping — create a large rectangle first, then add child cells with `parent_id` set to the container's ID.

## Common Azure Shape Styles

All Azure shapes follow the pattern:
```
shape=mxgraph.azure.{shape_name};fillColor=#0078D4;fontColor=#FFFFFF;strokeColor=none;
```

| Azure Service | Style |
|---|---|
| App Service | `shape=mxgraph.azure.app_service` |
| Function App | `shape=mxgraph.azure.function_apps` |
| Kubernetes Service (AKS) | `shape=mxgraph.azure.kubernetes_service` |
| Container Apps | `shape=mxgraph.azure.container_apps` |
| SQL Database | `shape=mxgraph.azure.sql_database` |
| Cosmos DB | `shape=mxgraph.azure.cosmos_db` |
| Cache for Redis | `shape=mxgraph.azure.cache_redis` |
| Front Door | `shape=mxgraph.azure.front_door` |
| Application Gateway | `shape=mxgraph.azure.application_gateway` |
| Azure Firewall | `shape=mxgraph.azure.firewall` |
| Virtual Network | `shape=mxgraph.azure.virtual_network` |
| Private Endpoint | `shape=mxgraph.azure.private_endpoint` |
| Key Vault | `shape=mxgraph.azure.key_vaults` |
| Monitor | `shape=mxgraph.azure.monitor` |
| Log Analytics | `shape=mxgraph.azure.log_analytics_workspaces` |
| Event Hubs | `shape=mxgraph.azure.event_hubs` |
| Service Bus | `shape=mxgraph.azure.service_bus` |
| Storage Account | `shape=mxgraph.azure.storage` |
| Entra ID | `shape=mxgraph.azure.active_directory` |
| AI Foundry | `shape=mxgraph.azure.ai_studio` |

> **Note:** These shape names are based on known Draw.io `mxgraph.azure` conventions. If a shape doesn't render, drag it from the sidebar and use `get-selected-cell` to capture the exact style.
