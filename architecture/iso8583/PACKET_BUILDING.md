# ISO 8583 sale packet — complete building flow

This document describes how a **sale** authorization message is turned from a normalized transaction model into **framed on-wire bytes**, using the types and stages in the application. It follows the actual pipeline: **logical field list → per-field binary packing → MTI and bitmaps → TPDU and length prefix**.

---

## 1. End-to-end pipeline

```mermaid
flowchart LR
  M[BaseTransactionModel]
  S[SaleISOFieldsBuilder.getSaleIsoFields]
  L[List of ISOFieldModel]
  P[ISO8583PacketBuilder.buildPacket]
  Msg[ISO8583Message]
  H[getPacketHeader]
  Out[ByteArray on socket]
  M --> S --> L --> P --> Msg --> H --> Out
```

1. **Sale orchestration** holds a populated **`BaseTransactionModel`** (amounts, PAN, dates, EMV blobs, merchant data, etc.).
2. **`SaleISOFieldsBuilder.getSaleIsoFields`** returns an ordered **`List<ISOFieldModel>`**, mixing direct constants from **`SaleConstants`** with values wrapped by **`saleFieldsInterface`** (flavor-specific).
3. **`ISO8583PacketBuilder.buildPacket`** constructs an **`ISO8583Message`**, calls **`setBit`** for each retained field, then **`customToByteArray`** to produce the ISO body.
4. **`getPacketHeader`** prepends **two-byte BCD message length** and **TPDU** bytes to that body.

---

## 2. `ISOFieldModel`

**Role:** One logical ISO data element before bitmap/body assembly.

| Property | Meaning |
|----------|---------|
| **`field`** | ISO-style **data element number** as used by **`setBit`**: **0** = MTI only; **2–128** = primary (and extended) bitmap fields. **−1** is used by some flavor helpers to mean “omit this slot” (combined with empty value / zero length so **`buildPacket`** skips it). |
| **`value`** | Payload as a **string**. Numeric and text fields use normal digits/ASCII. **PIN** and **ICC** are carried as **hex** text when binary bodies are needed. |
| **`length`** | Semantics depend on the field: usually **character length** of `value` for ASCII/BCD sources; for **binary PIN/ICC**, `length` is the **byte length** of the original blob (while `value` is hex). |
| **`useCustomConversion`** | If **true**, **`ConversionUtils.stringToByteArray`** parses **`value` as hex** into raw bytes before **`setBit`**. If **false**, `value` is converted with **`String.toByteArray()`** (ASCII octets). |

**Retention rule in `buildPacket`:** A field is written only if **`field > -1`**, **`value` non-empty**, and **`length > 0`**.

---

## 3. `InterfaceInstances`

**Role:** Central holder for flavor-bound singletons used across the app.

Relevant members for ISO:

| Member | Interface type | Role in packet building |
|--------|----------------|-------------------------|
| **`saleFieldsInterface`** | **`SaleFieldsInterface`** | Returns **`ISOFieldModel`** instances for DEs whose **formatting or padding** differs by **host/acquirer flavor** (e.g. local time, date, country code, NII, track 2, merchant name, private 48/49/53/60/120, tip 121/122). Also supplies **POS condition** via **`getConditionCode(SaleType)`** and **POS entry mode** string constants. |
| **`iso8583Interface`** | **`ISO8583Interface`** | Supplies **bitmap width** (8 vs 16 “rows”) and **per-slot `ISO8583DomainModel` overrides** for selected DEs (7, 19, 22, 43, 49, 120) when **`ISO8583DomainConfig`** initializes the **`ISO8583Message`** domain table. |

Implementations are chosen at **compile time** per product flavor (e.g. Paysys vs Euronet vs Sunmi Paysys **`ISO8583InterfaceImpl`** classes under flavor source sets).

---

## 4. `SaleConstants` (field numbers and defaults)

**Role:** Canonical **integer bit numbers** and default strings for sale.

Notable constants: **`FIELD_MTI` = 0**, **`FIELD_PAN` = 2**, **`FIELD_PROC_CODE` = 3**, **`FIELD_AMOUNT` = 4**, **`FIELD_STAN` = 11**, **12–14** time/date/expiry, **19** country, **22–25** entry mode / PAN seq / NII / POS condition, **35** track 2, **41–43** terminal/merchant/name, **48** private, **49** currency, **52–53** PIN / secured control, **55** ICC, **60** reserved, **62** private invoice-style, **120** network private, **121–122** tip and batch when tipping applies.

Default MTI in constants is **`0200`** for sale request (aligned with **`SaleISOFieldsBuilder`** usage).

**Note:** **`POS_ENTRY_MODE_*`** values in **`SaleConstants`** are delegated to **`saleFieldsInterface`** so entry-mode strings stay host-aligned.

---

## 5. `SaleISOFieldsBuilder`

**Role:** Deterministic **ordered list** of `ISOFieldModel` for sale.

**Order of operations (conceptual):**

1. Resolve **POS condition** from **`TransactionManager.transactionType`** (MOTO vs normal) via **`saleFieldsInterface.getConditionCode`**.
2. Append **MTI, PAN, processing code, amount, STAN**.
3. Append **local time** and **local date** through **`saleFieldsInterface`** (padding for date).
4. Append **expiry**, **country code** (flavor), **POS entry mode**.
5. Optionally **PAN sequence** if present.
6. **NII / function code** (flavor), **POS condition**, optional **track 2** (flavor).
7. **Terminal ID** and **merchant ID** with fixed-width padding, then **merchant name**, **DE48**, **currency** (flavor).
8. If PIN bytes exist: **hex** `value`, **`length` = byte size**, **`useCustomConversion = true`**.
9. If secured control data non-empty: flavor wrapper for **DE53**.
10. If ICC bytes exist: same **hex + byte length + custom conversion** pattern for **DE55**.
11. **DE60** (flavor), **DE62** invoice, **DE120** (flavor).
12. If tip amount > 0: **DE121** and **DE122** via flavor helpers.

Each appended model is logged when valid.

---

## 6. `SaleFieldsInterface`

**Role:** Host-specific **wrapping** of `ISOFieldModel` so **`SaleISOFieldsBuilder`** stays a single ordered pipeline.

Methods mirror DE families that often need different **padding, charset, or field suppression** per switch (e.g. **`getField35Track2FromFlavors`**, **`getField120ReservedPrivateFromFlavors`**). Implementations live in flavor **`SaleFieldsInterfaceImpl`** classes.

---

## 7. `ISO8583Constants` and related constants

**`ISO8583Constants`**

| Constant | Role |
|----------|------|
| **`ISO8583_MAX_LENGTH`** | **128** domain slots tracked by **`ISO8583Message`**. |
| **`MAX_BUFFER_LEN`** | **2048** bytes for the internal field payload buffer. |
| **`MSG_ID_LEN`** | **4** — MTI length in characters / encoded half-bytes for BCD MTI. |
| **`BIT_NUM_LEN_8` / `BIT_NUM_LEN_16`** | Number of **bitmap bytes** (one row vs primary+secondary). |

**`ISO8583LengthType`**

| Value | Meaning |
|-------|---------|
| **`FIX_LEN`** | Fixed length; **`setBit`** pads/truncates to **`maxLength`** from domain config. |
| **`LLVAR_LEN`** | One-byte BCD length prefix before value. |
| **`LLLVAR_LEN`** | Two-byte BCD length prefix before value. |

**`ISO8583EncodingType`**

| Value | Meaning in **`setBit`** |
|-------|-------------------------|
| **`L_BCD`** | Left-justified BCD packing. |
| **`L_ASC`** | Left-justified ASCII. |
| **`R_BCD`** | Right-justified BCD (zero-padded on the left for numeric). |
| **`R_ASC`** | Right-justified ASCII (space-padded on the left). |
| **`D_BIN`** | Binary-style path used with variable-length fields (length prefix rules in **`customToByteArray`** adjust stored length when computing LLVAR/LLLVAR). |

**`ISO8583ResponseConstants`** (separate concern) holds unpacker-oriented field numbers and bitmap bit indices for **inbound** responses — not used when **building** the sale request.

---

## 8. `ISO8583DomainModel`

**Role:** Dual use — **static configuration** and **per-message runtime state**.

**Configuration fields** (from **`ISO8583DomainConfig`** / flavor overrides):

- **`maxLength`** — Maximum field length for packing rules.
- **`type`** — One of **`ISO8583EncodingType`**.
- **`flag`** — One of **`ISO8583LengthType`**.
- **`domainName`** — Descriptive label (often Chinese labels in the default table).

**Runtime fields** (per `ISO8583Message` instance, updated in **`setBit`**):

- **`bitF`** — 1 if this DE is present in the current message.
- **`length`** — Actual length used for this occurrence.
- **`startAddress`** — Start offset inside the internal **`dataBuffer`** where the packed value lives.

---

## 9. `ISO8583DomainConfig`

**Role:** Static **`Map<Int, ISO8583DomainModel>`** defining **max length, encoding, and FIX/LLVAR/LLLVAR** for each internal slot.

**Initialization:** On **`ISO8583Message`** construction, **`getAllConfigs()`** is applied so each **`domains[i]`** receives the template for that slot.

**Flavor hooks:** Several **`put`** entries call **`iso8583Interface.getField7DomainConfigFromFlavors`**, **`getField19DomainConfigFromFlavors`**, **`getField22DomainConfigFromFlavors`**, **`getField43DomainConfigFromFlavors`**, **`getField49DomainConfigFromFlavors`**, and **`getField120DomainConfigFromFlavors`** so those DEs’ **encoding/length class** can differ by build flavor without forking the whole table.

**Indexing convention:** Internal map key **`k`** feeds **`domains[k]`**. **`setBit(n)`** uses **`index = n - 1`** for **`n ≥ 2`**, so **`domains[k]`** corresponds to **ISO data element `k + 1`** (e.g. **`domains[1]`** is **DE2 PAN**). **`setBit(0)`** is special: it only copies **MTI** into **`messageId`**, not into **`domains`**.

---

## 10. `ISO8583Message`

**Role:** Bitmap + body builder for one outbound ISO message.

### 10.1 Construction

- Allocates **`domains`** array of **`ISO8583_MAX_LENGTH`** `ISO8583DomainModel` entries.
- Copies each entry from **`ISO8583DomainConfig.getAllConfigs()`** into **`domains`**.

### 10.2 `clear`

Resets all **`bitF`**, **`length`**, **`startAddress`**, clears **`dataBuffer`**, resets **`offset`**.

### 10.3 `setBit(n, src, len)`

- **`n == 0`:** Copies **`src`** into **`messageId`** (MTI), length expected **`MSG_ID_LEN`**. Returns 0.
- **`n ≤ 1` or `n > 128`:** Invalid; returns **−1** or **0** per branch.
- Otherwise **`index = n - 1`**: clamps **`len`** to **`maxLength`**, honors **FIX_LEN** by forcing domain length to **`maxLength`**, sets **`bitF`**, records **`startAddress = offset`**, then writes **`src`** into **`dataBuffer`** according to **`type`**:
  - **L_BCD / R_BCD / D_BIN** paths use **`BCDUtils.fromASCIIToBCD`** with left/right alignment rules.
  - **L_ASC / R_ASC** copy ASCII with space or zero padding to fixed width.
- **`offset`** advances by the packed byte size.

### 10.4 `customToByteArray`

- Reads **`bitNum = iso8583Interface.getBitNumberFromFlavors()`** — **8** for settlement / settlement trailer transactions, **16** for other types (so **secondary bitmap** when needed).
- Encodes **MTI** from **`messageId`** into the output buffer with BCD.
- For each bitmap byte **row** and bit within row, walks **fields 2–127** (skipping unused indices), sets bitmap bits for present **`domains[n]`**, and for **LLVAR/LLLVAR** prepends **BCD length** (1 or 2 bytes), adjusting length when **`D_BIN`**.
- Copies packed field bytes from **`dataBuffer`** in order.
- If **`bitNum == 16`**, applies **special case for field 127** (extended bitmap flag and extra bytes) and sets **high bit of first bitmap byte** for secondary bitmap presence.

Returns a **trimmed** byte array of the ISO body (MTI + bitmaps + fields).

### 10.5 `getBit`

Unpacks a previously set field for debugging or reply handling (symmetric to **`setBit`** encoding rules).

---

## 11. `ISO8583PacketBuilder`

**`buildPacket(isoFields)`**

1. Throws if the list is empty.
2. **`ISO8583Message().apply { clear(); for each field ... }`**
3. For each **`ISOFieldModel`**, if retained by the filter, **`ConversionUtils.stringToByteArray(value, useCustomConversion)`** then **`setBit(field, fieldData, length)`**.
4. **`customToByteArray()`** → **`getPacketHeader(isoByte)`** → returns final **`ByteArray`**.

**`getPacketHeader(isoByte)`**

1. Computes **total length** = **`TPDU.get().size + isoByte.size`**, formats as **4 hex digits**.
2. Converts that hex string to **2-byte BCD length**.
3. Concatenates **`[BCD length] + TPDU + isoBody`**.

---

## 12. `TPDU`

**Role:** Transport routing prefix. **`TPDU.get()`** returns a small fixed **`ByteArray`** (implementation builds **`0x60` + `0x00 0x00`** style prefix via **`ConversionUtils.byteArrayAddTPDU`**, with NII-related composition elsewhere in utilities for other contexts).

---

## 13. Sale DE checklist (what goes on the wire, conceptually)

| DE | Purpose in sale list |
|----|----------------------|
| **0 (MTI)** | Message type; stored separately then BCD-encoded with body. |
| **2** | PAN. |
| **3** | Processing code. |
| **4** | Amount, 12-digit zero-padded. |
| **11** | STAN, 6-digit zero-padded. |
| **12–13** | Local time / date (flavor may alter formatting). |
| **14** | Expiry. |
| **19** | Country code (flavor). |
| **22** | POS entry mode. |
| **23** | PAN sequence, optional. |
| **24** | NII / function code (flavor). |
| **25** | POS condition (normal vs MOTO). |
| **35** | Track 2, optional (flavor). |
| **41–43** | Terminal, merchant, name/location (padding + flavor). |
| **48** | Additional private data (flavor). |
| **49** | Currency (flavor). |
| **52** | PIN block, optional — **hex + byte length + custom conversion**. |
| **53** | Secured control info, optional (flavor). |
| **55** | ICC data, optional — **hex + byte length + custom conversion**. |
| **60** | Reserved private (flavor). |
| **62** | Invoice / private reference. |
| **120** | Network private (flavor). |
| **121–122** | Tip amount and batch-related value when tip > 0 (flavor). |

Exact presence and host-specific layout are always governed by the live **`SaleISOFieldsBuilder`** + **`SaleFieldsInterface`** for your flavor.

---

## 14. Related types (responses)

**`ISO8583ResponseUnPacker`** and **`ISO8583ResponseConstants`** handle **inbound** host messages (bitmap parsing, field extraction). They mirror the same DE numbering philosophy but are not part of **`buildPacket`**.

---

## 15. Summary table

| Type | Responsibility |
|------|----------------|
| **`ISOFieldModel`** | One DE: number, string value, length, hex-vs-ASCII flag. |
| **`SaleConstants`** | DE integer ids and defaults for sale. |
| **`SaleISOFieldsBuilder`** | Builds ordered sale **`List<ISOFieldModel>`** from **`BaseTransactionModel`**. |
| **`InterfaceInstances.saleFieldsInterface`** | Host-specific **`ISOFieldModel`** factories and POS constants. |
| **`InterfaceInstances.iso8583Interface`** | Bitmap width and selected **domain config** overrides. |
| **`ISO8583DomainConfig` + `ISO8583DomainModel`** | Per-DE max length, encoding, FIX/LLVAR/LLLVAR; runtime placement in buffer. |
| **`ISO8583Message`** | **`setBit` / `customToByteArray`** — actual bitmap and body bytes. |
| **`ISO8583PacketBuilder`** | Orchestrates message build + **length + TPDU** framing. |
| **`ISO8583Constants` / length / encoding objects** | Sizes and enums driving **`ISO8583Message`**. |

This is the full sale **request** packet path from model to socket-ready bytes.
