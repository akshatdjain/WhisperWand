
#  WhisperWand

> *A device that listens to space and tells you where things are.*

**WhisperWand** is a spatial tracking system that turns the physical world into a living coordinate map.

Using **Ultra-Wideband (UWB)** ranging, WhisperWand measures the precise **XYZ position** of physical assets in real time. A human operator carries a handheld device - the **Wand** - which acts as the bridge between the physical object and the digital world.

The system works like this:

1. A human **scans an asset** using a QR code or barcode
2. The **Wand** quietly measures its **position in space** using UWB
3. The Wand sends **(asset ID + XYZ coordinates)** to a **Master node** (Raspberry Pi or server)
4. The Master builds a **live spatial database of reality**

Every scanned object becomes a **named point in space** - a coordinate that knows what it is.

You are no longer just tracking inventory.
You are **teaching the room to remember**.

---

## ✨ The Wand

The **Wand** is a handheld UWB tracker carried by a human operator.
It listens to radio pulses bouncing through the environment and uses time itself to determine where it is.

When an asset is scanned, the Wand binds:

* **Identity** (QR / barcode)
* **Position** (X, Y, Z from UWB)
* **Time**

…into a single spatial record and transmits it to the Master.

The human provides intent.
The Wand provides truth.
The system records reality.

---

## 🛰️ The Master

The **Master** (Raspberry Pi or server) receives data from all Wands and maintains a **global coordinate system** of the space. It becomes the source of truth for:

* Where every asset is
* When it was last seen
* How the space is changing

A UI can later render this into maps, search, history, and automation - but at its core, WhisperWand is about **making the invisible geometry of the world visible**.

---
