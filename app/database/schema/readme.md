# DATABASE SCHEMA


# 1️⃣ `mst_asset` — ASSET MASTER

### 🔹 What this table represents

This table represents **real-world assets that have been observed at least once**.

An asset **does not exist in the system** until:

* A human scans its QR/barcode with a wand.

This table answers:

> “What is this thing that was scanned?”

---

### 🔹 Core philosophy

* **Auto-created on first scan**
* Asset identity comes **only** from QR/barcode
* Metadata is **opaque** to the system (we store it, not interpret it)

---

### 🔹 Conceptual fields (and meaning)

| Field          | Meaning                                                                                     |
| -------------- | ------------------------------------------------------------------------------------------- |
| `asset_id`     | Unique identifier extracted from QR/barcode. This is the **primary identity** of the asset. |
| `raw_metadata` | Full raw payload from QR/barcode (JSON or text). Stored as-is, no assumptions.              |
| `created_at`   | Timestamp when asset was first ever scanned (first observation).                            |
| `last_seen_at` | Timestamp of the most recent scan of this asset.                                            |
| `status`       | Logical state of the asset (`active`, `removed`).                                           |

---

### 🔹 Rules for `mst_asset`

* `asset_id` is **immutable**
* Asset is **auto-created** if not found during an observation
* `raw_metadata` may be updated if a newer scan provides updated metadata
* Asset is **never hard-deleted**
* `status = removed` means:

  * Asset is no longer present in warehouse
  * History is preserved

---

### 🔹 What this table is NOT

* ❌ It does NOT store position
* ❌ It does NOT track movement
* ❌ It does NOT know about UWB

It is purely **identity + description**.

---

# 2️⃣ `mst_wands` — WAND MASTER

### 🔹 What this table represents

This table represents **scanning devices**.

Each row corresponds to:

> One physical wand = (human + Pi Zero + UWB tag)

---

### 🔹 Core philosophy

* Wands are **trusted observers**
* Wands are **pre-registered**
* Assets are unknown; wands are controlled

This table answers:

> “Who observed the asset?”

---

### 🔹 Conceptual fields (and meaning)

| Field         | Meaning                                                    |
| ------------- | ---------------------------------------------------------- |
| `wand_id`     | Logical identifier for the wand (you assign this).         |
| `tag_id`      | UWB tag ID attached to this wand.                          |
| `description` | Optional human-readable info (operator name, device note). |
| `active`      | Whether this wand is allowed to send data.                 |
| `created_at`  | When this wand was registered in the system.               |

---

### 🔹 Rules for `mst_wands`

* Wands **must exist** before sending observations
* If `active = false`, master should reject events
* Tag replacement = update `tag_id`
* One wand = one active tag at a time

---

### 🔹 What this table is NOT

* ❌ It does NOT store asset info
* ❌ It does NOT store position history
* ❌ It does NOT store scan events

It represents **observers**, not observations.

---

# 3️⃣ `observation_events` — THE SOURCE OF TRUTH

### 🔹 What this table represents

This is the **most important table**.

Each row represents:

> **One human-verified observation of an asset at a specific place and time**

Everything else is derived from this table.

---

### 🔹 Core philosophy

* **Append-only**
* **Never updated**
* **Never deleted**
* This table is your **audit log**

If something goes wrong, this table is how you recover.

---

### 🔹 Conceptual fields (and meaning)

| Field           | Meaning                                                             |
| --------------- | ------------------------------------------------------------------- |
| `event_id`      | Unique identifier for the observation event.                        |
| `asset_id`      | Asset that was scanned (FK → mst_asset).                            |
| `wand_id`       | Wand that performed the scan (FK → mst_wands).                      |
| `tag_id`        | Tag ID at time of scan (copied for traceability).                   |
| `x`             | X coordinate of wand at scan time.                                  |
| `y`             | Y coordinate of wand at scan time.                                  |
| `z`             | Z coordinate of wand at scan time.                                  |
| `quality`       | Optional RTLS quality/confidence value.                             |
| `scan_action`   | Semantic meaning of scan (`placed`, `moved`, `removed`, `checked`). |
| `event_time`    | Time when scan actually happened (from wand).                       |
| `received_time` | Time when master received the event.                                |

---

### 🔹 Rules for `observation_events`

* Every scan → exactly **one row**
* Rows are **never modified**
* `event_time` can be earlier than `received_time`
* `scan_action` is **explicit**, never inferred
* Asset is auto-created **before** inserting event if missing

---

### 🔹 Why `tag_id` is duplicated here

Even though `wand_id → tag_id` exists:

* Tag may change later
* This preserves historical correctness
* Makes debugging much easier

---

### 🔹 What this table is NOT

* ❌ It is NOT “current position”
* ❌ It is NOT optimized for fast UI queries
* ❌ It is NOT editable

It is **raw truth**.

---

# 4️⃣ `asset_latest_state` — CURRENT SNAPSHOT (DERIVED)

### 🔹 What this table represents

This table represents:

> “What is the most recently known state of each asset?”

It exists purely for **performance and convenience**.

---

### 🔹 Core philosophy

* Derived from `observation_events`
* Can be rebuilt anytime
* Never trusted over history

---

### 🔹 Conceptual fields (and meaning)

| Field             | Meaning                                     |
| ----------------- | ------------------------------------------- |
| `asset_id`        | Asset identifier (PK, FK → mst_asset).      |
| `x`               | Latest known X coordinate.                  |
| `y`               | Latest known Y coordinate.                  |
| `z`               | Latest known Z coordinate.                  |
| `last_event_time` | Event time of most recent observation.      |
| `status`          | Current asset status (`active`, `removed`). |
| `wand_id`         | Wand that last scanned the asset.           |

---

### 🔹 Rules for `asset_latest_state`

* Updated on every new observation
* One row per asset
* Overwritten, not appended
* Can be dropped and recomputed

---

### 🔹 What this table is NOT

* ❌ Not audit-safe
* ❌ Not authoritative
* ❌ Not immutable

It is a **cache**.

---

# 🔗 RELATIONSHIP SUMMARY (MENTAL MAP)

```
mst_wands  ──┐
             ├── observation_events ──► asset_latest_state
mst_asset ───┘
```

* `mst_asset` → identity
* `mst_wands` → observer
* `observation_events` → truth
* `asset_latest_state` → convenience

---
