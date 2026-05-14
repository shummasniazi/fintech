# Data and storage (structure)

This page outlines how **persistence** and **storage health** participate in payments, without referencing a concrete schema file or module layout.

---

## Transaction log

Each sale attempt is backed by a **transaction log row** created early in the pipeline. The row carries serialized request payloads, status transitions (draft, ready to send, sent, host success or failure), and fields populated as the host responds (response code, retrieval reference, authorization code, EMV diagnostics where applicable). The same concept extends to voids, settlements, and related operations depending on product configuration.

---

## Access style

The product combines:

- **Structured entities** for well-known tables and migrations.
- A **dynamic query** layer for flexible inserts, updates, and reporting-style reads built from maps of column names to values—useful where merchant-specific columns or legacy shapes evolve without rewriting every DAO method.

---

## Storage gates

Before starting certain flows (including sale entry paths that share risk with heavy I/O), the application checks **free space** on internal storage and on app-scoped external storage. If space is below a configured threshold, the operator sees a blocking message until space is recovered—reducing failures for database writes and for **fallback file logging** used when remote logging is unavailable.

---

## Post-authorization persistence

After the host responds, the completion path **merges** host and EMV fields into the existing row, may trigger **asynchronous transaction logging** with timeout and local encrypted fallback, and updates print-related state on the row as receipts are generated.

---

## Related topics

- **Sale flow** — when rows are inserted and updated.
- **ISO 8583** — what gets stored as request bytes versus parsed response fields.
