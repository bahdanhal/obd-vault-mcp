# OBD Vault MCP Tools Reference

Endpoint: `https://mcp.obd-vault.com/mcp` (`streamable-http`, protocol `2025-11-25`, OAuth 2.0 PKCE S256, scope `mcp:read`).

Machine-readable definitions: [`spec/tools.json`](spec/tools.json).

## Recommended Diagnostic Workflow

1. **Discover vehicles:** Call `list_vehicles` if `vehicle_id` is not known.
2. **Health snapshot:** Start with `get_latest_status(vehicle_id)` for active/pending DTCs, latest scan metrics, latest completed trip, and recent anomalies.
3. **Session details & summaries:** Use `list_scan_sessions` and `get_scan_session` for pre-computed telemetry summaries (`min`/`max`/`mean`/`count`), readiness monitors, and manufacturer control units.
4. **Fault codes & freeze frames:** Use `get_dtc_history` and `get_dtc_context` for Mode 02 freeze frames and surrounding sensor windows.
5. **Peaks & raw timeseries:** Prefer `aggregate_observations` for peaks/averages; use `get_observations` with `next_cursor` for granular raw waveforms.
6. **Export:** Use `create_vehicle_export` to generate a bounded JSON or CSV download reference.

---

## `list_vehicles`

**Title:** List vehicles  
**Read-only:** Yes (`readOnlyHint: true`)

List vehicles owned by the authenticated user using an opaque cursor for pagination. Vehicles appear only after the user has scanned a car with the OBD Vault mobile app; when there are none, the result carries a notice explaining what the user has to do.

| Parameter | Type | Required | Description |
| --- | --- | :---: | --- |
| `cursor` | string | No | Pagination cursor. |
| `limit` | integer (1–200) | No | Maximum number of vehicles to return. |

---

## `get_vehicle_profile`

**Title:** Get vehicle profile  
**Read-only:** Yes (`readOnlyHint: true`)

Get factual metadata and coverage for one vehicle. VIN is excluded unless explicitly requested.

| Parameter | Type | Required | Description |
| --- | --- | :---: | --- |
| `include_vin` | boolean, default `false` | No | Include full VIN when set to `true`. |
| `vehicle_id` | string | No | Optional vehicle ID. Defaults to the primary vehicle if omitted. |

---

## `get_latest_status`

**Title:** Get latest vehicle status  
**Read-only:** Yes (`readOnlyHint: true`)

Get a comprehensive snapshot of the vehicle in a single call: latest diagnostic scan with key metrics (coolant, RPM, voltage), active/pending DTCs, the most recent completed driving trip (km, duration, fuel L/100km with its source: recorded, or the app's after-trip estimate with its coverage, speed, RPM), recent driving anomalies from the last 30 days, and active connection status.

| Parameter | Type | Required | Description |
| --- | --- | :---: | --- |
| `vehicle_id` | string | No | Optional vehicle ID. Defaults to the primary vehicle if omitted. |

---

## `list_scan_sessions`

**Title:** List scan sessions  
**Read-only:** Yes (`readOnlyHint: true`)

List diagnostic scan and driving sessions for a vehicle, including duration, distance, observation count, and trip indicators. Filterable by time or cursor. Vehicle ID defaults to primary vehicle.

| Parameter | Type | Required | Description |
| --- | --- | :---: | --- |
| `cursor` | string | No | Pagination cursor. |
| `from` | string | No | Start timestamp (ISO-8601). |
| `limit` | integer (1–200) | No | Maximum number of sessions to return. |
| `to` | string | No | End timestamp (ISO-8601). |
| `vehicle_id` | string | No | Optional vehicle ID. Defaults to the user's primary vehicle if omitted. |

---

## `get_scan_session`

**Title:** Get scan session  
**Read-only:** Yes (`readOnlyHint: true`)

Get one bounded scan session including pre-computed telemetry metrics summary (peaks/averages for speed, coolant, voltage, RPM), diagnostic trouble codes, monitor readiness, the control units that answered a manufacturer dialogue (type, software and part numbers, their own fault memory), and sample observations.

| Parameter | Type | Required | Description |
| --- | --- | :---: | --- |
| `scan_session_id` | string | Yes | The unique scan session identifier. |

---

## `get_dtc_history`

**Title:** Get DTC history  
**Read-only:** Yes (`readOnlyHint: true`)

List stored and pending diagnostic trouble code events for a vehicle, newest first, using an opaque cursor for pagination. Each event carries `detected_at` (start of the session that read it); `code_summary` gives each code's first and last sighting, so a code that has not been read for a long time is not mistaken for a current fault.

| Parameter | Type | Required | Description |
| --- | --- | :---: | --- |
| `cursor` | string | No | Pagination cursor. |
| `limit` | integer (1–500) | No | Maximum number of DTC events to return. |
| `vehicle_id` | string | No | Optional vehicle ID. Defaults to the primary vehicle if omitted. |

---

## `get_dtc_context`

**Title:** Get DTC context  
**Read-only:** Yes (`readOnlyHint: true`)

Return the Mode 02 freeze frame stored with a DTC and the readings within ±30s (or custom) of it. The window is centred on the freeze frame capture when one exists, otherwise on the start of the session that recorded the DTC (see `target_time_source`). Vehicle and DTC code default to the latest if omitted.

| Parameter | Type | Required | Description |
| --- | --- | :---: | --- |
| `dtc_code` | string | No | Optional diagnostic trouble code (e.g. `P0101`). If omitted, defaults to the latest recorded DTC. |
| `timestamp` | string | No | Optional ISO-8601 timestamp to center the analysis window around. |
| `vehicle_id` | string | No | Optional vehicle ID. Defaults to the user's primary vehicle if omitted. |
| `window_seconds` | integer (5–120) | No | Time window in seconds before and after the event (default 30s). |

---

## `aggregate_observations`

**Title:** Summarize telemetry observations  
**Read-only:** Yes (`readOnlyHint: true`)

Calculate statistical aggregates (min, max, mean, count) for an observed metric. Supports filtering by vehicle, metric, `scan_session_id`, or time range. Use this tool instead of pulling raw observation lists when looking for peaks, extremes, or averages. Only measured values are aggregated: the app's after-trip fuel estimates are left out (`get_latest_status` and `get_scan_session` report them, labelled).

| Parameter | Type | Required | Description |
| --- | --- | :---: | --- |
| `bucket` | string (`all` \| `hour` \| `day` \| `week` \| `month`) | No | Time bucket for aggregation. Default is `"day"`, or `"all"` to aggregate the entire queried range or session into one summary. |
| `from` | string | No | Optional start timestamp (ISO-8601). |
| `metric_key` | string | Yes | The observed telemetry metric to aggregate (e.g. `vehicle_speed`, `coolant_temperature`, `adapter_atrv_voltage`, `engine_rpm`). Battery voltage measured at the adapter (ATRV) is `adapter_atrv_voltage`; `control_module_voltage` is the ECU's own supply reading (PID 42), which many cars do not report. |
| `scan_session_id` | string | No | Optional scan session ID to aggregate observations for a specific trip or scan. |
| `timezone` | string | No | IANA time zone that defines hour, day, week and month boundaries (e.g. `Europe/Warsaw`). Default `UTC`. |
| `to` | string | No | Optional end timestamp (ISO-8601). |
| `vehicle_id` | string | No | Vehicle ID. Optional if `scan_session_id` is provided. |

---

## `get_observations`

**Title:** Get telemetry observations  
**Read-only:** Yes (`readOnlyHint: true`)

Return raw sensor timeseries observations (RPM, coolant, trims, voltages, speed). Sorted DESC (newest-first) by default; supports ASC. Use cursor for pagination. Ideal for waveform analysis or detailed moment-by-moment debugging.

| Parameter | Type | Required | Description |
| --- | --- | :---: | --- |
| `cursor` | string | No | Pagination cursor from `next_cursor`. |
| `from` | string | No | Start timestamp (ISO-8601). |
| `limit` | integer (1–500) | No | Maximum number of observations to return. |
| `metric_key` | string | No | Filter by specific telemetry metric (e.g. `vehicle_speed`, `engine_rpm`, `coolant_temperature`, `adapter_atrv_voltage`). Battery voltage measured at the adapter (ATRV) is `adapter_atrv_voltage`; `control_module_voltage` is the ECU's own supply reading (PID 42), which many cars do not report. |
| `order` | string (`asc` \| `desc`) | No | Chronological order: `"desc"` (newest first, default) or `"asc"` (oldest first). |
| `scan_session_id` | string | No | Filter observations by scan session ID. |
| `to` | string | No | End timestamp (ISO-8601). |
| `vehicle_id` | string | No | Vehicle ID. Optional if `scan_session_id` is provided. |

---

## `get_data_quality`

**Title:** Get data quality  
**Read-only:** Yes (`readOnlyHint: true`)

Return factual coverage counts and observed metric keys for a vehicle.

| Parameter | Type | Required | Description |
| --- | --- | :---: | --- |
| `vehicle_id` | string | No | Optional vehicle ID. Defaults to the primary vehicle if omitted. |

---

## `create_vehicle_export`

**Title:** Create vehicle data export  
**Read-only:** No (`readOnlyHint: false`, `destructiveHint: false` — creates a temporary export reference without modifying vehicle records)

Create a bounded read-only JSON or CSV export for an owned vehicle, with up to 1,000 observations per session. Use `get_observations` with `next_cursor` to retrieve complete sensor histories.

| Parameter | Type | Required | Description |
| --- | --- | :---: | --- |
| `format` | string (`json` \| `csv`) | No | Export format (`json` or `csv`). |
| `vehicle_id` | string | Yes | Target vehicle ID. |
