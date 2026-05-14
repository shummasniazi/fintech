# Architecture notes (structure only)

These pages describe **how the system is organized and how major flows connect**. The **search card** and **ISO 8583** entries include implementation-level detail (class names and stages); other pages may stay higher level.

---

## Topics

| Topic | Purpose |
|--------|---------|
| [search-card/CARD_SEARCH_BY_DEVICE.md](search-card/CARD_SEARCH_BY_DEVICE.md) | **Full** search-card + EMV: class-by-class **Topwise** and **Sunmi**, sequence diagrams, CVM/CDCVM, contactless limit, TLV reference, cleanup. |
| [sale/SALE_FLOW.md](sale/SALE_FLOW.md) | End-to-end **sale** from operator action through host response and completion. |
| [iso8583/PACKET_BUILDING.md](iso8583/PACKET_BUILDING.md) | **ISO 8583** sale packet: full pipeline, core types (`ISOFieldModel`, `ISO8583Message`, domain config, constants), flavor interfaces, and every sale DE. |
| [iso8583/SALE_MESSAGE.md](iso8583/SALE_MESSAGE.md) | Short redirect to **PACKET_BUILDING** (kept for old links). |
| [database/DATA_AND_STORAGE.md](database/DATA_AND_STORAGE.md) | Transaction records, dynamic SQL vs ORM-style usage, and storage-related gates. |
| [sdk/EMBEDDED_SDK_AND_HOST_CALLBACKS.md](sdk/EMBEDDED_SDK_AND_HOST_CALLBACKS.md) | Running payments inside a **host app**: entry, callbacks, and return to host. |

---

## How to use this folder

Read top-down for a full picture, or jump to a single topic. Diagrams use **Mermaid** where a sequence or decision tree helps more than prose.
