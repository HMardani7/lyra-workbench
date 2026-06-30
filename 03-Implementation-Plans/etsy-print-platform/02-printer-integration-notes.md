# Bambu P1S Printer Integration Notes

Sources checked:
- Prior Hammad conversation: printer is Bambu P1S with AMS-powered multi-material setup.
- Bambu community/local-control docs and libraries:
  - OpenBambuAPI MQTT notes
  - Bambu JS library README
  - P1S local web app examples
  - Offline P1S/P1P notes using FTPS + MQTT

## Integration stance

Bambu P1S local control is useful but not an official stable public API in the same way Etsy's Open API is. Kiro should design this as an adapter with graceful fallback:

```ts
interface PrinterAdapter {
  getStatus(): Promise<PrinterStatus>;
  uploadFile(localPath: string, remotePath: string): Promise<void>;
  startPrint(request: StartPrintRequest): Promise<void>;
  pausePrint(jobId?: string): Promise<void>;
  resumePrint(jobId?: string): Promise<void>;
  stopPrint(jobId?: string): Promise<void>;
}
```

Then provide two implementations:

1. `ManualPrinterAdapter` — no direct printer control; creates operator instructions.
2. `BambuP1SLocalAdapter` — FTPS upload + MQTT control/status.

The app should be useful even if printer automation is disabled.

## Local connection assumptions

Common P1S local settings:

```text
FTPS:
- host: printer LAN IP
- port: 990
- username: bblp
- password: LAN access code from printer

MQTT:
- host: printer LAN IP
- port: 8883
- TLS: true
- username: bblp
- password: LAN access code from printer
- report topic: /device/{serial}/report
- request topic: /device/{serial}/request
```

Store these in environment/config, not source code:

```env
PRINTER_MODEL=P1S
PRINTER_HOST=192.168.x.x
PRINTER_SERIAL=...
PRINTER_ACCESS_CODE=...
PRINTER_MQTT_PORT=8883
PRINTER_FTPS_PORT=990
PRINTER_USERNAME=bblp
```

Do not commit real IPs/access codes.

## File upload flow

For a pre-sliced file:

1. Validate local file exists.
2. Validate file is linked to an approved `ProductVariant`.
3. Validate extension is allowed.
4. Upload via FTPS to a deterministic path, e.g.:

```text
/etsy-print-platform/{print_job_id}-{sku}.3mf
```

Community command example for manual testing:

```bash
curl --ftp-pasv --insecure \
  -T ./example.gcode.3mf \
  ftps://<printer-ip>:990/etsy-print-platform/example.gcode.3mf \
  --user bblp:<access-code>
```

Kiro should use a library instead of shelling out in production, but keep a small manual connectivity test script.

## Start print flow

Community MQTT examples show two local print styles:

### `project_file` for `.3mf`/`.gcode.3mf`

```json
{
  "print": {
    "sequence_id": "0",
    "command": "project_file",
    "param": "Metadata/plate_1.gcode",
    "project_id": "0",
    "profile_id": "0",
    "task_id": "0",
    "subtask_id": "0",
    "subtask_name": "",
    "url": "file:///mnt/sdcard/etsy-print-platform/example.gcode.3mf",
    "md5": "",
    "timelapse": true,
    "bed_type": "auto",
    "bed_levelling": true,
    "flow_cali": true,
    "vibration_cali": true,
    "layer_inspect": true,
    "ams_mapping": "",
    "use_ams": true
  }
}
```

### `gcode_file` for plain `.gcode`

```json
{
  "print": {
    "sequence_id": "0",
    "command": "gcode_file",
    "param": "etsy-print-platform/example.gcode"
  }
}
```

Kiro should not assume these commands work unchanged. It must test against Hammad's actual P1S and capture the confirmed working command shape in code comments/tests.

## Status monitoring

Subscribe to:

```text
/device/{serial}/report
```

Persist raw status events during testing. Build a small mapper:

```ts
type PrinterState =
  | 'unknown'
  | 'offline'
  | 'idle'
  | 'preparing'
  | 'printing'
  | 'paused'
  | 'finished'
  | 'failed';
```

Only allow `startPrint` if mapped state is `idle` and operator approved the job.

## AMS/material handling

For v1, do **not** rely on automatic AMS slot mapping unless Hammad explicitly configures it.

Represent required material plainly in the dashboard:

```text
Job #123
SKU: DRAGON-EGG-BLACK-PLA
Required: PLA, black, 0.4mm nozzle
File: /prints/dragon-egg/black-pla.3mf
Operator check: AMS slot contains black PLA
```

Optional later:

```text
AmsSlot
- printer_id
- slot_index: 1..4
- material: PLA/PETG/TPU
- color_name
- color_hex
- spool_brand
- grams_remaining_estimate
- last_checked_at
```

## Safety rules

Never start a print if:

- printer status is offline/unknown/printing
- product variant mapping is missing
- file does not exist
- file was not pre-approved
- nozzle requirement is not `0.4mm` for MVP
- operator approval is missing
- app cannot record the resulting job state

## Recommended implementation order

1. Build manual queue first.
2. Add printer status read-only connection.
3. Add FTPS upload with a test file.
4. Add MQTT start command behind explicit feature flag.
5. Add status reconciliation.

Feature flags:

```env
PRINTER_AUTOMATION_ENABLED=false
PRINTER_START_PRINT_ENABLED=false
PRINTER_UPLOAD_ENABLED=false
```

Default all to false in `.env.example`.

## Test plan for printer integration

- Unit test adapter command construction.
- Unit test safety gate rejects unsafe jobs.
- Integration test FTPS connection against printer with a harmless small file.
- Integration test MQTT status subscription only.
- Manual test start command using a tiny calibration print after Hammad approves.
- Verify dashboard status transitions.

## Handoff warning for Kiro

Do not build direct cloud control through Bambu account APIs unless Hammad asks. Prefer local LAN control because the Etsy platform is intended to run near the homelab/printer and avoid depending on Bambu Cloud for production workflow.
