# ISO 8583 sale message (conceptual)

The host integration speaks **binary ISO 8583**-style messages for authorization. This page describes the **shape** of a sale request: what categories of data appear, how they are layered, and how **host flavors** differ, without enumerating implementation types.

---

## Two-stage assembly

1. **Logical field list** — From the normalized sale model, the application builds an ordered list of **data elements** (bit numbers, values, lengths). Some bits are optional depending on card path, PIN presence, and tips.
2. **Wire encoding** — A message builder applies per-bit rules (fixed vs variable length, BCD vs ASCII, binary handling for PIN and ICC), builds the **bitmap**, prepends message type and any **routing / length** prefix required by the switch, and hands bytes to the transport layer.

```mermaid
flowchart LR
  M[Normalized sale model]
  L[Logical ISO field list]
  E[Bitmap and body encoding]
  F[Framed bytes on socket]
  M --> L --> E --> F
```

---

## Representative data elements (sale)

The exact set is configuration- and host-dependent; typical groups include:

| Group | Role |
|--------|------|
| Message type and processing | Financial request indicator, processing code. |
| Amounts and trace | Transaction amount, system trace number, local time and date, expiry. |
| Geography and entry | Country code, POS entry mode, optional PAN sequence, NII or function code, POS condition (normal vs MOTO). |
| Card data | Primary account number, optional track 2, optional PIN block, optional EMV **ICC** container. |
| Acceptor | Terminal id, merchant id, merchant name / location, currency. |
| Private use | Host-specific private fields, invoice or reference, network private blocks. |
| Optional tip | Tip amount and related batch fields when the product enables tipping and rules allow. |

**Host A vs Host B:** A **host field adapter** concept wraps many bits so the same orchestration can emit different formatting or subfield layouts per switch without duplicating the ordered list logic.

---

## Binary fields (PIN and EMV)

PIN and ICC data are **binary** at the protocol level. The logical list typically carries them in a **hex representation** for logging and builder convenience while the encoder uses the **byte length** and a conversion flag so variable-length binary fields pack correctly into the message body.

---

## Response (conceptual)

The application parses the reply into a small **response model** (approval or decline code, identifiers, optional second-generation AC data for EMV). That model drives database updates, UI, and whether reversal or advice flows run.

---

## Related topics

- **Sale flow** — when the message is built and sent.
- **Data and storage** — where request and response metadata land.
