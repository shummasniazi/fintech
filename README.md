# Loop NextGen POS — Android Payment Terminal Application

**Portfolio / work showcase.** This document describes **Loop NextGen POS** — a production-style Kotlin Android payment terminal application. **Application source is not in this public repository**; paths below use `...` within `app/src/main/java/...` to avoid tying the narrative to internal product or host codenames.

**Author:** [shummasniazi](https://github.com/shummasniazi)

**Neutral glossary (this README only)**

- **Acquirer A / Acquirer B** — Two white-label **client** product lines in Gradle (branding, resources). Not bank names.
- **Host A / Host B** — Two **payment switch** implementations behind the same domain APIs (ISO 8583 field layout and socket behavior differ per host flavor).
- **OEM stack A / OEM B** — Two **device** integrations (card reader, printer, EMV lifecycle).
- **Environment** — Non-production vs production **config endpoints** (Gradle dimension); values are not listed here.

---

## GitHub

This file is the default **`README.md`** on [github.com/shummasniazi/fintech](https://github.com/shummasniazi/fintech). Edit in the private repo and push to `main`, or edit on GitHub.

This README intentionally **does not** embed proprietary URLs, keys, merchant data, or real institution names.

---

## Tech stack (summary)

Kotlin, Jetpack Compose, Hilt, Room, Retrofit, OkHttp, raw TCP **ISO 8583**, JSch SFTP, Detekt, Spotless, Sonar wiring.

---

## Part 1 — Product shape (no host or bank names)

The app is built as **one application module** with Gradle **flavor dimensions** so one codebase ships:

- **Two acquirer product lines** (Acquirer A / B)
- **Two OEM terminal stacks** (OEM A / B)
- **Two payment hosts** (Host A / B)
- **Several environments** (dev / QA / UAT / prod style labels in Gradle)

That yields many **assemble** task combinations; each variant selects the right **host field adapter** and **device service** implementations at compile time.

---

## Part 2 — End-to-end sale flow (user journey)

High-level path from idle terminal to receipt:

```mermaid
flowchart TD
  subgraph preDash [Dashboard and entry]
    D[Dashboard]
    S[Sale amount entry]
  end
  subgraph card [Card and EMV]
    C[Search card]
    P[PIN if required]
  end
  subgraph online [Host]
    X[Process screen send]
    H[Host authorization]
  end
  subgraph finish [Complete and print]
    U[Update DB host fields]
    L[Async transaction log API]
    R[Print receipts]
    OK[Success UI]
  end
  D --> S
  S --> C
  C --> P
  P --> X
  C --> X
  X --> H
  H --> U
  U --> L
  L --> R
  R --> OK
```

---

## Part 3 — Checks before the operator presses **Sale**

These run **before** navigation to the amount / card flow (order is conceptually: debounce, print busy, retail USB, storage, printer paper, then business rules like forced settlement).

```mermaid
flowchart TD
  Tap[Operator taps Sale or Punch order]
  Debounce[Debounce rapid taps]
  PrintBusy{Another print in progress}
  Retail{Retail mode and USB disconnected}
  Storage{Free space below threshold}
  Paper{Printer paper out}
  Settle{Forced settlement required}
  Allow[Proceed to sale entry or retail amount]
  Tap --> Debounce
  Debounce --> PrintBusy
  PrintBusy -->|yes| Block1[Block]
  PrintBusy -->|no| Retail
  Retail -->|blocked path| Block2[Block]
  Retail -->|ok| Storage
  Storage -->|low| DialogStorage[Alert Retry after cleanup]
  Storage -->|ok| Paper
  Paper -->|out| DialogPaper[Alert Retry after load paper]
  Paper -->|ok| Settle
  Settle -->|yes| ToastSettle[Block with settle message]
  Settle -->|no| Allow
```

**What each gate does (conceptually)**

- **Print in progress** — Global print status prevents overlapping receipt jobs.
- **Retail** — If integrated retail mode expects a USB peripheral, connectivity is checked before continuing.
- **Storage** — Before actions that share the same gate as printing, the app requires **minimum free space** on the internal data partition and on app-scoped external files (so Room writes and log fallbacks are less likely to fail). Implemented in `.../utils/StorageHealth.kt` and invoked from `.../presentation/features/dashboard/viewModel/DashboardViewModel.kt`.
- **Printer paper** — Paper-out sensor path; **Retry** re-checks before continuing (`ReceiptPrintManager` from the dashboard VM).
- **Forced settlement** — Business policy can block new sales until batch settlement (same VM layer as sale click).

**Code anchors (private tree)**

- Dashboard click wiring: `.../presentation/features/dashboard/view/DashboardScreen.kt`
- Debounce, settlement, storage + printer orchestration: `.../presentation/features/dashboard/viewModel/DashboardViewModel.kt`

---

## Part 4 — After amount and card: building the **domain packet** (not wire yet)

When the card flow finishes, `ProcessSaleTransaction.initialize` runs in the sale pipeline:

1. **Allocate** next invoice and STAN from the database helper.
2. **`generateSalePacket`** — Validates EMV / card state from `TransactionHandler`, then builds a `BaseTransactionModel` via `SaleModelCreation` (PAN, amount, dates, entry mode, ICC, track2, PIN block if present, merchant/terminal IDs, NII, private fields, discount and billing-related fields, etc.).
3. **`Validator.validate(saleModel)`** — Internal consistency checks; on failure the flow stops with a terminal-side error path.
4. **`finalIsoPacket`** — See Part 5.

**Code anchors**

- Orchestration: `.../domain/transactions/sale/ProcessSaleTransaction.kt`
- Model factory: `.../domain/transactions/sale/utils/SaleModelCreation.kt`
- In-flight card and amount state: `.../domain/transactions/sale/handlers/TransactionHandler.kt`

---

## Part 5 — Which **ISO 8583 fields** go into the wire message (and how Host A/B matters)

`SaleISOFieldsBuilder.getSaleIsoFields` turns the `BaseTransactionModel` into an ordered list of `ISOFieldModel` entries. **Common bits** include MTI, PAN, processing code, amount (12-digit), STAN, local time/date, expiry, country code, POS entry mode, optional PAN sequence, NII/function code, **POS condition** (differs for normal vs MOTO sale via `SaleType`), terminal ID, merchant ID, merchant name, private field 48, currency, optional PIN (52), optional security-related field 53, optional ICC (55), field 60, invoice (62), reserved private (120), and optional tip / batch extension fields (121–122) when tip is non-zero.

**Host-specific shaping** — Many positions call `saleFieldsInterface` (implemented in the **Host A / Host B** source sets): examples include local date/time presentation, country code, NII, track 2, merchant name, field 48/49/53/60/120, tip and batch-related fields. That is how the **same** `BaseTransactionModel` produces **different** on-wire layouts without forking the whole sale orchestrator.

**Code anchors**

- Field list and conditionals: `.../domain/transactions/sale/utils/SaleISOFieldsBuilder.kt`
- Host-facing interface: `.../flavors/interfaces/` (implementations live under host-specific source roots bundled by Gradle)

```mermaid
flowchart LR
  M[BaseTransactionModel]
  B[SaleISOFieldsBuilder]
  I[saleFieldsInterface Host A or B]
  L[List of ISOFieldModel]
  M --> B
  B --> I
  I --> L
```

---

## Part 6 — How the **binary ISO packet** is produced

1. **`ISO8583PacketBuilder.buildPacket`** walks each `ISOFieldModel`. Non-empty fields with positive length are written into an `ISO8583Message` bit map using per-field length and optional hex/binary conversion flags.
2. **`getPacketHeader`** prepends a **length prefix** and **TPDU** header bytes to the ISO body so the raw socket layer sends a single framed message.

**Code anchors**

- `.../services/iso8583/packetBuilder/ISO8583PacketBuilder.kt`
- `.../services/iso8583/packetBuilder/ISO8583Message.kt`

```mermaid
flowchart LR
  L[ISOFieldModel list]
  Msg[ISO8583Message setBit per field]
  Bytes[ISO body bytes]
  H[TPDU plus length prefix]
  Out[Final ByteArray for socket]
  L --> Msg --> Bytes --> H --> Out
```

---

## Part 7 — **Persisting** the sale in Room **before** the host responds

After `finalIsoPacket()`:

- `insertTransactionLogWithReadyToSendStatus` maps the model to a `TransactionLogEntity`, adds logger columns (`if_transaction_guid`, `if_transaction_logged`, etc.), and **`insertAndGetId`** into the `TRANSACTION_LOG` table.
- Status is **ReadyToSend**; the serialized ISO bytes live in the row for audit and retransmit patterns.
- On success the UI navigates to the **process** screen, which drives the socket send and host wait.

```mermaid
sequenceDiagram
  participant PS as ProcessSaleTransaction
  participant DB as Room DatabaseManager
  participant UI as ProcessScreenActivity
  PS->>PS: finalIsoPacket
  PS->>DB: insertAndGetId TRANSACTION_LOG
  DB-->>PS: transaction id
  PS->>UI: navigate ProcessScreen
```

**Code anchors**

- Insert and status: `.../domain/transactions/sale/ProcessSaleTransaction.kt` (`insertTransactionLogWithReadyToSendStatus`, `initialize`)

---

## Part 8 — **Online** phase: send, update status, interpret host reply

After insert, the process screen sends the byte array over TCP, then:

- Updates **SentToHost** and persists STAN-related fields via callbacks registered in `setupAfterSendCallback`.
- Parses the host response into a `SocketResponse` (RRN, auth code, response code, host date/time, optional ICC payload).
- **Success path** (response code rules include an optional certification mode for EMV testing): updates model to **HostSuccess**, hands off to completion.
- **Failure path** — Reversal / error UI depending on policy.

**Code anchors**

- `.../domain/transactions/sale/ProcessSaleTransaction.kt` (`handleSaleSuccess`, `afterSendMethodSale`)
- `.../remote/socket/` (socket client and response parsing)

---

## Part 9 — **Completing** the sale: DB merge, logging, **printing**

`CompleteSaleTransaction` coordinates post-host work:

- **Persist host fields** — Updates the same `TRANSACTION_LOG` row with response time/date, DE39, RRN, auth code, EMV TVR/TSI if updated, QR/barcode value, billing columns when enabled, etc.
- **Transaction logger** — Asynchronous REST call with timeout; on failure the app appends an **encrypted JSON line** to a local fallback file (rolled by record count), uploaded later over SFTP during settlement. See `.../remote/network/repositories/DataLoggerRepository.kt` and `.../services/fileLoggingService/FileLoggingService.kt` and internal doc `docs/TransactionLogging.md` in the private repo.
- **Printing** — Merchant/customer receipt generation through the print stack for the active **OEM** integration; print state transitions (e.g. in progress / success) are written back to the transaction row.

```mermaid
flowchart TD
  HS[HostSuccess]
  DBU[Update row host and EMV fields]
  LOG[Send transaction log API or fallback file]
  PR[Print merchant then customer if configured]
  SU[Success screen]
  HS --> DBU
  DBU --> LOG
  LOG --> PR
  PR --> SU
```

**Code anchors**

- `.../domain/transactions/sale/CompleteSaleTransaction.kt`
- Printing managers under `.../printing/` and OEM-specific implementations under OEM-flavored source sets

---

## Part 10 — **DMS**, **settlement SFTP**, **crash**, **SDK** (short pointers)

- **Discount / DMS** — Update API; on failure a **CSV line** is appended under app external files for later SFTP (`.../presentation/features/dms/service/DMSService.kt` plus file service above).
- **Settlement** — Batch close and reporting; also triggers ordered **SFTP upload** of pending discount and logger fallback files before follow-up UI (`.../domain/transactions/settlement/helpers/SettlementHelper.kt`).
- **Crash** — Encrypted external log with internal-storage fallback (`.../presentation/features/crashScreen/crashHandler/CrashHandler.kt`).
- **Embeddable SDK** — Same activities invoked from a host app: pay, keyed pay, void, settle, print, configure, settings, logon, history (`.../sdk/SdkManager.kt`, `.../sdk/SdkManagerImp.kt`). Low-storage guard applies on SDK entry paths as well.

---

## Part 11 — Build and quality (commands only)

```bash
./gradlew :app:assemble<Acquirer><OEM><Host><Env>Debug
```

APKs appear under `app/build/outputs/apk/<variant>/debug/` with a generated filename pattern from Gradle.

Quality tooling: Spotless, Detekt, Sonar properties in `app/build.gradle.kts`.

---

## Part 12 — Further reading (files exist only in the private repo)

- Transaction logging and SFTP: `docs/TransactionLogging.md`
- TMS / app update: `docs/TMS_APP_UPDATE_FLOW.md`
- MOTO: `docs/MOTO_SALE_DOCUMENTATION.md` and related MOTO docs
- SDK JSON: `docs/SDK_RESPONSES_README.md`
- Obfuscation: `docs/OBFUSCATION_GUIDE.md`, `docs/OBFUSCATION_SUMMARY.md`
- Code quality: `docs/code_quality.md`
- High-level map: `skill.md`

---

## Closing

This README is a **structured portfolio narrative** focused on **sale flow**, **preconditions**, **ISO construction**, **persistence**, and **completion**. For exact field matrices per host, see the private repository’s host-specific `SaleFields` implementations and internal integration documents.
