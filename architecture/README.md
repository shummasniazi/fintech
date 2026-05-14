# Architecture notes (structure only)

These pages describe **how the system is organized and how major flows connect**. They intentionally avoid source listings, class names tied to a tree, and file paths so this documentation can stay stable while the private codebase evolves.

---

## Topics

| Topic | Purpose |
|--------|---------|
| [search-card/CARD_SEARCH_BY_DEVICE.md](search-card/CARD_SEARCH_BY_DEVICE.md) | How card discovery and EMV differ between **Sunmi** and **Topwise** OEM stacks, and how the shared UI ties in. |
| [sale/SALE_FLOW.md](sale/SALE_FLOW.md) | End-to-end **sale** from operator action through host response and completion. |
| [iso8583/SALE_MESSAGE.md](iso8583/SALE_MESSAGE.md) | Conceptual **ISO 8583** sale message: which data elements matter and how encoding layers relate. |
| [database/DATA_AND_STORAGE.md](database/DATA_AND_STORAGE.md) | Transaction records, dynamic SQL vs ORM-style usage, and storage-related gates. |
| [sdk/EMBEDDED_SDK_AND_HOST_CALLBACKS.md](sdk/EMBEDDED_SDK_AND_HOST_CALLBACKS.md) | Running payments inside a **host app**: entry, callbacks, and return to host. |

---

## How to use this folder

Read top-down for a full picture, or jump to a single topic. Diagrams use **Mermaid** where a sequence or decision tree helps more than prose.
