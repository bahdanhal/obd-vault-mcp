# OBD Vault MCP Server — Public Specification

Public specification, tool schemas, and connection guide for the **[OBD Vault](https://obd-vault.com/)** remote **Model Context Protocol (MCP)** server.

Connect AI assistants (ChatGPT, Claude, Cursor, Windsurf, or any MCP-compatible client) to your own vehicle's **OBD2 / OBD-II** diagnostic scans, trouble codes (DTCs), Mode 02 freeze frames, live sensor telemetry, and recorded driving trips captured with the OBD Vault mobile app and an ELM327 adapter.

- **Website:** [obd-vault.com](https://obd-vault.com/)
- **MCP Endpoint (Streamable HTTP):** `https://mcp.obd-vault.com/mcp`
- **Claude Official Listing:** [OBD Vault on Claude Directory](https://claude.ai/directory/obd-vault)
- **ChatGPT Official Listing:** [OBD Vault on ChatGPT](https://chatgpt.com/plugins/plugin_asdk_app_6ade5eeb9e248191a75c718cc4c5cc20)
- **Mobile App:** [Google Play (Android)](https://play.google.com/store/apps/details?id=com.obdvault.app) · [App Store (iPhone)](https://apps.apple.com/app/obd-vault/id6808813416)

> **Scope of this repository:** This repository publishes the public MCP server manifest (`server.json`), tool schemas (`spec/tools.json`), and integration documentation. The mobile application and backend source code remain private.

---

## How It Works

1. Record vehicle scans, live sensor sessions, or driving trips with the **OBD Vault** mobile app and a compatible **ELM327 OBD2** adapter (Bluetooth LE, Bluetooth Classic on Android, or Wi-Fi).
2. Enable optional cloud sync (`cloud_mcp` plan) in the mobile app so your vehicle history is available remotely.
3. Connect your MCP client to `https://mcp.obd-vault.com/mcp` via **OAuth 2.0 (Authorization Code + PKCE S256)** with scope `mcp:read`.
4. Ask your AI assistant questions about your car's health, fault codes, freeze frames, sensor peaks, and fuel economy — without being plugged into the car.

---

## Connecting Your Client

### Claude
Use the official connector listing directly: [OBD Vault on Claude Directory](https://claude.ai/directory/obd-vault).

For **Claude Code**, you can also add it from the terminal:

```bash
claude mcp add --transport http obd-vault https://mcp.obd-vault.com/mcp
```

### ChatGPT
Use the official listing directly: [OBD Vault in ChatGPT](https://chatgpt.com/plugins/plugin_asdk_app_6ade5eeb9e248191a75c718cc4c5cc20).

### Cursor / Windsurf / Generic Streamable HTTP MCP Client
Configure a remote Streamable HTTP MCP server with OAuth 2.0 discovery:

```json
{
  "mcpServers": {
    "obd-vault": {
      "url": "https://mcp.obd-vault.com/mcp"
    }
  }
}
```

- **Protocol Version:** `2025-11-25`
- **Transport:** `streamable-http`
- **Authentication:** OAuth 2.0 Authorization Code + PKCE (`S256`), scope `mcp:read`
- **OAuth Discovery:** Standard RFC 9728 Protected Resource Metadata and RFC 8414 Authorization Server Metadata at `https://mcp.obd-vault.com`

---

## Safety & Privacy Model

- **No vehicle control or ECU writes:** The MCP server only queries historical and synced diagnostic records stored in your OBD Vault account. It cannot send OBD commands, clear fault codes, or modify ECU settings.
- **Read-only by default:** 10 of the 11 tools are strictly read-only (`readOnlyHint: true`, `destructiveHint: false`). The remaining tool (`create_vehicle_export`) generates a bounded, temporary download reference for your own records (`readOnlyHint: false`, `destructiveHint: false`).
- **VIN protection:** Vehicle Identification Numbers (VINs) are excluded from `get_vehicle_profile` unless `include_vin: true` is explicitly requested (exports created via `create_vehicle_export` can include the full VIN).
- **Revocable access:** You authorize access at the account level and can revoke connected clients at any time.
- **Policies & Support:** [Privacy Policy](https://obd-vault.com/privacy) · [Terms of Service](https://obd-vault.com/terms) · `support@obd-vault.com`

---

## Available Tools (11)

Full parameter schemas and workflow notes are documented in [TOOLS.md](TOOLS.md) and machine-readable JSON in [spec/tools.json](spec/tools.json).

| Tool | Title | Read-Only | Purpose |
| --- | --- | :---: | --- |
| `list_vehicles` | List vehicles | Yes | List vehicles owned by the authenticated user |
| `get_vehicle_profile` | Get vehicle profile | Yes | Get vehicle metadata, coverage, and diagnostic guidance (VIN excluded by default) |
| `get_latest_status` | Get latest vehicle status | Yes | One-call health snapshot: latest scan metrics, active/pending DTCs, latest trip, recent anomalies |
| `list_scan_sessions` | List scan sessions | Yes | List diagnostic scans and recorded driving trips with time/cursor pagination |
| `get_scan_session` | Get scan session | Yes | Detailed session view: pre-computed metrics summary, DTCs, readiness monitors, OEM control units |
| `get_dtc_history` | Get DTC history | Yes | Paginated history of stored and pending trouble codes with first/last sighting summary |
| `get_dtc_context` | Get DTC context | Yes | Mode 02 freeze frame and surrounding sensor readings (±30s or custom window) for a DTC |
| `aggregate_observations` | Summarize telemetry observations | Yes | Server-side statistical aggregates (`min`, `max`, `mean`, `count`) for any metric |
| `get_observations` | Get telemetry observations | Yes | Paginated raw sensor timeseries (RPM, coolant, fuel trims, voltage, speed) |
| `get_data_quality` | Get data quality | Yes | Factual coverage counts and list of observed metric keys for a vehicle |
| `create_vehicle_export` | Create vehicle data export | No* | Create a bounded JSON or CSV export download reference for an owned vehicle |

*\*`create_vehicle_export` does not modify vehicle or diagnostic records; it creates a temporary export artifact (`readOnlyHint: false`, `destructiveHint: false`).*

---

## Example Prompts

- *"Summarize the latest diagnostic status for my vehicle."*
- *"Show my recent fault codes, freeze-frame data, and explain what they mean."*
- *"Did my battery voltage drop below 13 V or coolant temperature exceed 100 °C on my last drive?"*
- *"Compare fuel economy and distance across my recent recorded trips."*
- *"Export my vehicle's diagnostic history as JSON."*

---

## Files in This Repository

- [`server.json`](server.json) — Official MCP Registry manifest (`io.github.bahdanhal/obd-vault`)
- [`spec/tools.json`](spec/tools.json) — Complete JSON Schema definitions for all 11 MCP tools
- [`TOOLS.md`](TOOLS.md) — Human-readable tool reference and recommended diagnostic workflow
- [`LICENSE`](LICENSE) — MIT License (applies to this specification and documentation)
