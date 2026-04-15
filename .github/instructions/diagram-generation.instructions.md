---
description: "Use when creating architecture diagrams, Draw.io diagrams, generating visual diagrams, exporting diagrams as images (PNG, SVG), embedding diagrams in markdown, using Azure shapes in diagrams, or any diagram-related task."
---
# Diagram Generation

When generating diagrams, follow the full procedure in the diagram generation skill file:

- [Diagram Generation Skill](../../skills/diagram-generation/SKILL.md)

## Quick Reference

1. **Open** the Draw.io editor at `http://localhost:3000/` using the browser tool
2. **Create** a new empty diagram by importing blank XML with mode `replace`
3. **Azure shapes** are not in the MCP shape API — use `add-rectangle` with `mxgraph.azure.*` styles: `shape=mxgraph.azure.{shape_name};fillColor=#0078D4;fontColor=#FFFFFF;strokeColor=none;`
4. **Build** the diagram using `add-rectangle` with `mxgraph.azure.*` styles, connect with `add-edge`
5. **Discover unknown styles** by dragging a shape from the UI sidebar, then calling `get-selected-cell`
6. **Export** as PNG or SVG using `export-diagram` with an absolute `output_path`
7. **Reference** the exported image in the target markdown file
