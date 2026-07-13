# TDS Automation Project
 
Automate service address synchronization and fiber infrastructure associations for TDS telecommunications infrastructure. This project replaces broken FME (Feature Manipulation Engine) workflows that stopped functioning after November 7, 2024. The system synchronizes data across three critical platforms for 987 plan areas in Wisconsin:
 
- **AGOL (ArcGIS Online)** - Source of truth for field-collected service addresses

- **SDE (PostgreSQL)** - Enterprise geodatabase for internal workflows

- **Vetro FiberMap** - Fiber network management platform
 
**Last Updated:** March 17, 2026 | See `memory-bank/CHANGELOG.md` for full change history.
 
---
 
## Quick Start
 
### Prerequisites
 
- GIS AVD (Azure Virtual Desktop) - Required for database access (10.0.20.9 not accessible from local machines)

- ArcGIS Pro Python 3.11 environment
 
### Setup
 
1. Connect to GIS AVD

2. Navigate to project:
 
   ```powershell

   cd C:\Users\gluke\Documents\tds_automation

   ```
 
3. Verify configuration:
 
   ```powershell

   python -m issue1_service_address_sync.sync_service_addresses status

   ```
 
### Interactive Menu (Recommended for Operators)
 
```powershell

# From dev machine

python tds_menu.py
 
# From production network drive (double-click or run directly)

E:\08_Scripts\PythonScripts\tds_automation\Run_TDS_Menu.bat

```
 
The menu provides:
 
- Issue 1 through Issue 7 pipeline execution with guided prompts

- Vetro rate limit detection and automatic countdown (Issue 4)

- Full 8-step orchestrated plan sync with halt-on-failure

- Dry-run by default; explicit confirmation required for execute mode
 
See `docs/TDS_Automation_Menu_User_Guide_v2_1.docx` for full operator documentation.
 
---
 
## Pipeline Overview
 
| Issue | Pipeline | Purpose | Status |

| ------- | ---------- | --------- | -------- |

| 1 | Service Address Sync | AGOL -> SDE -> Vetro (3-step) | Production |

| 2 | Drop Fiber Association | Match drops to service addresses (10m spatial) | Production |

| 3 | Splice Closure Association | Match addresses to splice closures (polygon) | Production |

| 4 | Vetro to SDE Sync | Sync 13 Vetro feature classes to SDE | Production |

| 5 | SDE to AGOL Sync | Sync 9 feature layers + 3 related tables + spatial_fiber_drop | Production |

| 6 | Conduit Footage Reports | Generate Excel reports with conduit totals | Production |

| 7 | Drop Cable Routing | Route drops from SAs to splice enclosures | Production |
 
### Core Pipeline Execution Order
 
When processing a new plan area, execute pipelines in this order:
 
```text

1. Issue 1 Step 4 (AGOL Change Detection)

2. Issue 1 Step 5 (AGOL -> SDE SA Sync)

3. Issue 1 Step 6 (SDE -> Vetro SA Sync)

4. Issue 4 (Vetro -> SDE) -- RATE LIMITED

5. Issue 3 (Splice Closure Association)

6. Issue 3 Fiberdrop Enrichment (Derive placement_type and conduit_path)

7. Issue 2 (Drop Fiber Association)

8. Issue 5 (SDE -> AGOL)

```
 
**Notes:**
 
- **Issue 7 (Drop Cable Routing)** is now **standalone and must be run manually before** the full 8-step pipeline if drop routing is needed. It is no longer part of the automated Option 8 sequence.

- Issue 6 (Conduit Footage Reports) is standalone and can run anytime.

- **Equipment Count Report** (menu option 12) is a standalone Vetro-only report — splitter and splice-enclosure counts grouped by size, supports single / multiple / all / cached-only DFNs.

- **Delete Operations:** Run `tools\admin\delete_sde_plan_records.py` and `tools\admin\delete_agol_by_spatial_boundary.py` as needed before re-syncing a plan.
 
**Data Flow Summary:**
 
- **Service Addresses:** AGOL -> SDE -> Vetro (Issue 1)

- **Network Features:** Vetro -> SDE -> AGOL (Issues 4, 5)

- **Fiber Drops:** Created from routes (Issue 7), matched to SAs (Issue 2), synced to AGOL via Issue 5

- **Associations:** Computed in SDE (Issues 2, 3), pushed to AGOL via Issue 5
 
---
 
## Issue 1: Service Address Sync (AGOL -> SDE -> Vetro)
 
**Status:** Complete & Operational
 
### Step 4: AGOL Change Detection
 
```cmd

python -m issue1_service_address_sync.sync_service_addresses step4-agol-changes --plan 0872TE_05

python -m issue1_service_address_sync.sync_service_addresses step4-agol-changes --plan 0872TE_05 --dry-run

```
 
### Step 5: AGOL to SDE Sync
 
```cmd

python -m issue1_service_address_sync.sync_service_addresses step5-agol-to-sde

python -m issue1_service_address_sync.sync_service_addresses step5-agol-to-sde --plan 0872TE_05

python -m issue1_service_address_sync.sync_service_addresses step5-agol-to-sde --plan 0872TE_05 --force-initial

python -m issue1_service_address_sync.sync_service_addresses step5-agol-to-sde --plan 0872TE_05 --dry-run

```
 
### Step 6: SDE to Vetro Sync
 
```cmd

python -m issue1_service_address_sync.sync_service_addresses step6-sde-to-vetro --plan 0902DA_05

python -m issue1_service_address_sync.sync_service_addresses step6-sde-to-vetro --plan 0902DA_05 --force

python -m issue1_service_address_sync.sync_service_addresses step6-sde-to-vetro --plan 0902DA_05 --dry-run

```
 
### Check Status
 
```cmd

python -m issue1_service_address_sync.sync_service_addresses status

python -m issue1_service_address_sync.sync_service_addresses status --plan 0902DA_05

```
 
---
 
## Issue 2: Drop Fiber Association
 
Matches fiber drop endpoints to service addresses using spatial proximity (haversine distance, 10m threshold). Updates `fiberdrop.service_address_gid`.
 
```cmd

python -m issue2_drop_association.sync_drop_associations associate --plan 0902DA_05 --dry-run --limit 10

python -m issue2_drop_association.sync_drop_associations associate --plan 0902DA_05

python -m issue2_drop_association.sync_drop_associations status --plan 0902DA_05

```
 
---
 
## Issue 3: Splice Closure Association
 
Matches service addresses to splice closures using OFDC polygon containment (SQL spatial join). Updates `serviceaddress_tds_agol.fiber_splice_enclosure_gid`.
 
```cmd

python -m issue3_splice_closure_association.sync_splice_closures associate --plan 0902DA_05 --dry-run --limit 10

python -m issue3_splice_closure_association.sync_splice_closures associate --plan 0902DA_05

python -m issue3_splice_closure_association.sync_splice_closures status --plan 0902DA_05

```
 
---
 
## Issue 3: Fiberdrop Enrichment
 
Derives `placement_type` and `conduit_path` on fiberdrop records where those fields are NULL. This step runs automatically after Issue 3 (Splice Closure Association) in the full 8-step pipeline.
 
```cmd

python -m issue3_splice_closure_association.sync_fiberdrop_enrichment --plan 0902DA_05 --dry-run

python -m issue3_splice_closure_association.sync_fiberdrop_enrichment --plan 0902DA_05

```
 
---
 
## Issue 4: Vetro to SDE Sync
 
Syncs 13 feature classes from Vetro FiberMap to SDE. Rate-limited; use tds_menu.py for automatic countdown handling.
 
**Status:** Complete & Operational
 
**Target Feature Classes (13 Vetro layers):**
 
- **Parent Layers:** Plan Boundary, Drop Cluster (OFDC Polygon), Pole, Handhole, Pedestal, Splice Closure (Aerial)

- **Infrastructure Layers:** Fiber Cable Lateral, Fiber Cable Drop, Duct, Strand, Slack Loop

- **Child/Equipment Layers:** Splice Closure (Equipment, Underground), Splitter
 
**Note:** Backbone and Pigtail cable types are excluded (span multiple plan areas)
 
```cmd

python -m issue4_vetro_to_sde_sync.vetro_to_sde_cli list-layers
 
python -m issue4_vetro_to_sde_sync.vetro_to_sde_cli sync-plan --plan 0872TE_05 --dry-run query

python -m issue4_vetro_to_sde_sync.vetro_to_sde_cli sync-plan --plan 0872TE_05 --dry-run off
 
python -m issue4_vetro_to_sde_sync.vetro_to_sde_cli sync-plan --plan 0872TE_05 --layer 44 --dry-run query

python -m issue4_vetro_to_sde_sync.vetro_to_sde_cli sync-plan --plan 0872TE_05 --layer conduit --dry-run off
 
python -m issue4_vetro_to_sde_sync.vetro_to_sde_cli sync-plan --plan 0872TE_05 --exclude-layers "17,1" --dry-run query

python -m issue4_vetro_to_sde_sync.vetro_to_sde_cli sync-plan --plan 0872TE_05 --exclude-layers "17" --dry-run off
 
python -m issue4_vetro_to_sde_sync.vetro_to_sde_cli status --plan 0872TE_05

```
 
**Layer Filtering:** `--layer` and `--exclude-layers` are mutually exclusive. No flags syncs all 13 layers.
 
---
 
## Issue 5: SDE to AGOL Sync
 
Syncs feature layers and related tables from SDE to AGOL. Handles geometry transformation, field mapping, and GlobalID-based record matching.
 
**Status:** Complete & Operational
 
**Supported Layers (14 total):**
 
| Layer Key | AGOL Layer | SDE Table | Type |

| --------- | ---------- | --------- | ---- |

| fiber_splitter | 0 | fiberequipment | Feature |

| fiber_splice_enclosure | 1 | fiber_splice_enclosure | Feature |

| access_point | 2 | accesspoint | Feature |

| pole | 5 | pole | Feature |

| pole_support | 6 | pole_support | Feature |

| slack_loop | 8 | slackloop | Feature |

| fiber_cable | 9 | fibercable | Feature |

| conduit | 10 | conduit | Feature |

| strand | 11 | strand | Feature |

| serving_area | 13 | servingarea | Feature |

| port_connection | 18 | port_connection | Related Table |

| riser | 19 | riser | Related Table |

| fiber_drop | 23 | fiberdrop | Related Table |

| spatial_fiber_drop | 9 | fiberdrop | Feature |
 
```cmd

python -m issue5_sde_to_agol_sync.orchestrator --list-layers
 
python -m issue5_sde_to_agol_sync.orchestrator --plan 0872TE_05

python -m issue5_sde_to_agol_sync.orchestrator --plan 0872TE_05 --apply
 
python -m issue5_sde_to_agol_sync.orchestrator --plan 0872TE_05 --layer fiber_cable

python -m issue5_sde_to_agol_sync.orchestrator --plan 0872TE_05 --layer conduit --apply
 
python -m issue5_sde_to_agol_sync.orchestrator --plan 0872TE_05 --exclude-layers "fiber_drop,riser"

python -m issue5_sde_to_agol_sync.orchestrator --plan 0872TE_05 --skip-inserts --apply

```
 
### Business Key Matching (match_field / agol_match_field)
 
Some layers use `match_field` (SDE column) and `agol_match_field` (AGOL field) for hybrid business-key matching. This prevents duplicate GlobalID creation when AGOL auto-generates new GlobalIDs. Records with NULL `match_field` values fall back to GlobalID matching via state manager mappings.
 
---
 
## Issue 6: Conduit Footage Reports
 
Generates Excel reports with conduit footage calculations by querying Vetro API directly.
 
**Status:** Complete & Operational
 
```cmd

python -m issue6_calculate_conduit_footages.conduit_footage_report --plan 0872TE_05

python -m issue6_calculate_conduit_footages.conduit_footage_report --plan 0872TE_05 --output report.xlsx

```
 
---
 
## Issue 7: Drop Cable Routing
 
Routes fiber drop cables from service addresses to splice enclosures using NetworkX shortest path. Creates fiberdrop records in both SDE and Vetro (Layer 17).
 
**Status:** Complete & Operational
 
**Routing Strategy:**
 
- **Primary:** Conduit network routing (underground)

- **Fallback:** Aerial routing via strand network (overhead)

- **Last Resort:** Direct-line fallback when no network path exists
 
```cmd

python -m issue7_drop_cable_routing.sync_drop_cables route --plan 0902KA_04

python -m issue7_drop_cable_routing.sync_drop_cables route --plan 0902KA_04 --max-drop-feet 800

python -m issue7_drop_cable_routing.sync_drop_cables route --plan 0902KA_04 --max-drop-feet 0

python -m issue7_drop_cable_routing.sync_drop_cables route --plan 0902KA_04 --no-dry-run

python -m issue7_drop_cable_routing.sync_drop_cables status --plan 0902KA_04

```
 
**Default:** `--max-drop-feet 600` (routes exceeding 600 ft are flagged and skipped). Use `--max-drop-feet 0` to disable.
 
---
 
## Equipment Count Report (Menu Option 12)
 
Standalone Vetro-only report. For each DFN, counts splitters and splice enclosures grouped by size, and writes an Excel workbook with two sheets (`splitters`, `splice_enclosures`) to `reports/equipment_count_report_<scope>_<ts>.xlsx`.
 
**Vetro layers / size fields:**
 
- Layer 36 (Splitter) — size from `Splitter Configuration`

- Layer 19 (Splice Closure, Aerial) — size from `Size`

- Layer 658 (Splice Closure Equipment, Underground) — size from `Size`, falling back to `splice_enclosure_type`
 
**Data path:** Cache-first per plan; falls back to Vetro `/v3/features/query` when a cache file is missing or `--refresh` is set. `--all` uses 3 paginated `/features/query` calls (one per layer) and groups locally — bypasses the `/plans` 20/day cap.
 
```cmd

# Single DFN

python -m tools.analysis.equipment_count_report --plan 0872TE_05
 
# Multiple DFNs (repeatable or comma-separated)

python -m tools.analysis.equipment_count_report --plan 0872TE_05 --plan 0902DA_05

python -m tools.analysis.equipment_count_report --plan 0872TE_05,0902DA_05
 
# Every plan in reference/All_ServingAreas.csv

python -m tools.analysis.equipment_count_report --all
 
# Only DFNs that already have a Vetro export cache file

python -m tools.analysis.equipment_count_report --cached-only
 
# Force a fresh API pull instead of reading the cache

python -m tools.analysis.equipment_count_report --plan 0872TE_05 --refresh

```
 
**Menu (option 12) scopes:** Single DFN / Multiple DFNs (comma-separated) / All DFNs / Cached-only.
 
---
 
## Configuration
 
### Environment Variables (.env)
 
See `.env` file for required variables including database credentials and API tokens.
 
### Application Settings (config/config.yaml)
 
Contains field mappings, sync settings, and test configuration. See `docs/architecture/configuration_notes.md` for details.
 
---
 
## System Connections
 
| System | Endpoint | Purpose |

| -------- | ---------- | --------- |

| SDE | 10.0.20.9:5432/tds_3gis_design | Enterprise geodatabase |

| Vetro | api.vetro.io/v3 | Fiber network management |

| AGOL | tilsontech.maps.arcgis.com | Field data collection |
 
---
 
## Key Technical Details
 
- **Coordinate Systems:** EPSG:4326 (WGS84) for AGOL/Vetro, SRID 3857 (Web Mercator Auxiliary Sphere) for SDE storage

- **Transform Patterns:** Reading from SDE: sde.ST_Transform(shape, 4326)  -- converts SRID 3857 to WGS84; Writing to SDE: sde.ST_Transform(sde.ST_Geometry('POINT(%s %s)', 4326), 3857)  -- converts WGS84 to SRID 3857

- **Database:** All PostgreSQL column names are lowercase

- **Vetro IDs:** String format `"SL-########"`

- **State Files:** JSON-based per-plan tracking with SHA256 hashes in `state/` directory

- **Spatial Matching (Issue 2):** Haversine distance with 10-meter threshold

- **Polygon Matching (Issue 3):** SQL ST_Contains spatial join

- **Related Tables (Issue 5):** Non-spatial tables with foreign key relationships

- **Vetro Rate Limit:** Issue 4 is rate-limited; tds_menu.py handles countdown automatically
 
---
 
## Tools Directory
 
The `tools/` directory is organized into functional subdirectories:
 
| Subdirectory | Purpose |

| ------------ | ------- |

| `tools/admin/` | Destructive admin scripts (delete records, fix assignments, backfill operations) |

| `tools/analysis/` | Read-only analysis and export scripts (state auditing, plan analysis) |

| `tools/diagnostics/` | Health checks and schema inspection (AGOL/SDE validation) |

| `tools/query/` | Direct query scripts for SDE, AGOL, and Vetro data exploration |

| `tools/llm/` | Context file generators for AI assistants |
 
Each subdirectory contains a `README.md` with available tools and usage.
 
---
 
## Troubleshooting
 
**Query SDE directly:**
 
```cmd

python tools/query/query_sde.py

```
 
**Count records for a plan:**
 
```cmd

python tools/query/count_sde_records.py --plan 0902DA_05

```
 
**Export drop associations:**
 
```cmd

python tools/analysis/export_drop_associations.py --plan 0902DA_05 --output results.csv

```
 
**Audit sync state:**
 
```cmd

python tools/analysis/audit_sync_state.py

```
 
**Verify Vetro IDs:**
 
```cmd

python tools/diagnostics/check_vetro_id_status.py

```
 
**Check AGOL layer fields:**
 
```cmd

python tools/diagnostics/check_agol_layer_fields.py

```
 
**Force initial sync (delete state files):**
 
```cmd

del state\agol_sync_state.json

del state\agol_to_sde_sync_state.json

python -m issue1_service_address_sync.sync_service_addresses step6-sde-to-vetro --plan 0902DA_05

```
 
---
 
## Known Limitations
 
**PortConnection Orphans:** Deleting Splitter features from AGOL (via spatial boundary deletion or manual cleanup) leaves orphaned PortConnection records (Layer 18) in AGOL. PortConnection is a related table without geometry and cannot be targeted by spatial queries. After deleting Splitters, manually verify and clear orphaned PortConnections in AGOL if needed.
 
---
 
## Development Notes
 
- Must run on GIS AVD (database not accessible from local machines)

- Files in Documents folder persist across AVD sessions

- Test plans: `0872TE_05` (Issue 1), `0902DA_05` (Issues 2-3), `0902KA_02` (Issue 5), `0902DA_06` (Issues 7 + fiber_drop)

- Never commit `.env` file

- Package installation: use pip from ArcGIS Pro Python environment
 
---
 
## Documentation
 
| File | Purpose |

| ---- | ------- |

| `docs/TDS_Automation_Menu_User_Guide_v2_1.docx` | Operator guide for the interactive menu system |

| `docs/architecture/directory_structure.md` | Project file structure |

| `docs/architecture/configuration_notes.md` | Detailed configuration reference |

| `docs/reference/field_mapping_table.md` | Field mappings between all three systems |

| `docs/reference/code_patterns.md` | Client code, SQL, and CLI examples |

| `docs/reference/agol_layers.md` | AGOL layer indices and schemas |

| `docs/reference/vetro_layers.md` | Vetro layer IDs and schemas |

| `docs/reference/sde_tables.md` | SDE table schemas |

| `memory-bank/CHANGELOG.md` | Full dated change history |

| `memory-bank/activeContext.md` | Current project status and next actions |
 
---
 
## Contact
 
- **Developer:** Jerry Luke
 
