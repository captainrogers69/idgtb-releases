# idgtb

Front-desk guest check-in for Windows. You scan an ID card on a Thales/3M QS2000 reader, the guest form fills itself in, and you check the guest in.

## Architecture

```mermaid
flowchart LR
  R[QS2000 reader<br/>USB VID_17B9] -->|WinUSB| SDK[MMMReaderDotNet40.dll<br/>Thales SDK 9.9]
  SDK --> H[IdgtbScanner.exe<br/>32-bit C# helper<br/>scanner_helper/Program.cs]
  H -->|"stdout JSON lines<br/>status / image / log"| A[idgtb.exe<br/>Flutter app<br/>lib/scanner.dart]
  A -->|"stdin closed = exit"| H
  A --> S[(AppData\Roaming\idgtb<br/>guests.json<br/>images\&lt;id&gt;_front/back.jpg<br/>idgtb.log)]
```

The SDK is 32-bit .NET, so it runs in a separate helper process. The app starts the helper and reads one JSON message per line from it:

| `type` | Meaning | App reaction |
|---|---|---|
| `status` | `connected` / `disconnected` / `error` + message | Status pill bottom-right |
| `image` | Saved card side + PDF417 text (or null) | Feeds the scan session |
| `log` | Anything worth keeping | Appended to `idgtb.log` |

## Scan → check-in

```mermaid
flowchart TD
  I([Card placed on reader]) --> D[SDK: DOC_ON_WINDOW<br/>captures IR / VIS / photo]
  D --> B{Helper: PDF417<br/>found in VIS image?}
  B -- no --> F[image msg, barcode = null<br/>= FRONT]
  B -- yes --> K[image msg, barcode = text<br/>= BACK]
  F --> P{Guest form open?}
  K --> P
  P -- yes --> X[Ignored<br/>pill shows Paused]
  P -- no --> SS[ScanSession.addImage]
  SS --> C{front AND back?}
  C -- no --> W[Scanning ID popup<br/>Waiting for other side…]
  W --> I
  C -- yes --> PA[parseAamva barcode]
  PA --> E{findByCard<br/>idNumber + state}
  E -- returning guest --> M[Merge: keep phone/email/remarks,<br/>refresh card fields + history]
  E -- new --> N[New draft guest]
  M --> G[New Guest form]
  N --> G
  G -- Cancel --> Z([Nothing saved])
  G -- Check-In --> SV[Copy images to images/&lt;id&gt;_front/back.jpg<br/>append CheckIn now + room remark<br/>write guests.json atomically]
  SV --> L([Guest listed, Today's Check-ins +1])
```

Notes:
- The app tells the sides apart only by the barcode: the side with the PDF417 is the back. If a second side arrives with no barcode, the popup shows a red "retry back" hint.
- Scanning the same card again updates the existing guest (matched on `idNumber` + `state`) and adds a new check-in, so no duplicate guest is created.
- **View details** on a listed guest opens the same form with Save in place of Check-In. It edits fields and the last check-in remark.

## Scanner status pill

```mermaid
stateDiagram-v2
  [*] --> starting
  starting --> disconnected: helper up, reader initialising
  disconnected --> connected: PLUGINS_INITIALISED / Initialise OK
  connected --> error: SDK error (e.g. UV not supported)
  error --> connected: END_OF_DOCUMENT / state change
  connected --> disconnected: USB unplugged
  disconnected --> connected: Recover() every 5 s
  connected --> Paused: guest form open (display only)
  Paused --> connected: form closed
  starting --> helperMissing: IdgtbScanner.exe failed to start
```

Click the pill to open **Scanner diagnostics**: it shows the current status, the last 400 log lines, **Copy log** and **Open log folder**.
