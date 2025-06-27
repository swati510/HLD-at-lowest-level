# How SSDs Really Work

*Part 1 of the "High Level Design at the Lowest Level" series*

Do you know your SSD uses **quantum tunneling** to store data?

That sounds like sci-fi, but it's literally how information is written onto modern storage devices. The world moved from spinning magnetic disks to solid-state drives (SSDs), but most people—even engineers—still think of SSDs as just "faster storage." In reality, the inner workings of SSDs are filled with clever tradeoffs, physics-level engineering, and architectural patterns that echo all the way into high-level system design.

This post kicks off my new series, "High Level Design at the Lowest Level," where I dive deep into components we usually treat as black boxes. We're starting with SSDs—not just because they're ubiquitous, but because understanding how they work reveals a lot about why certain systems behave the way they do.

## The Basics: What is NAND Flash?

At the heart of an SSD is **NAND flash memory**—a type of non-volatile storage that doesn't lose data when power is off. NAND stores data using floating-gate transistors, which are like tiny switches that can trap or release electrons.

The key idea is this:
Each cell in NAND flash can hold a certain charge (or lack of it), and this charge is interpreted as a binary value. These are not your average transistors—they're designed to trap electrons using a phenomenon called **quantum tunneling**, where electrons "tunnel" through an insulating layer into the floating gate.

Once trapped, the electrons stay there for years—essentially locking in the bit. This ability to store charge without power is what makes NAND ideal for SSDs.

### Deep Dive: Quantum Tunneling in Plain English

Think of an electron as a tiny traveler faced with an **insulating wall** only a few atoms thick. In classical physics that wall is impenetrable—like a concrete barrier. Quantum mechanics, however, says particles are also *waves*. When the wave that represents an electron hits a very thin barrier, a small portion of it *leaks* through. That leakage is **quantum tunneling**.

Why does this matter for storage? In a floating-gate transistor, the "wall" is a thin layer of silicon dioxide (~10 nm). By applying a high electric field, we give electrons enough *probability* to tunnel through that oxide and get trapped on the floating gate. No holes are drilled, no physical gateways are opened—the electrons just *appear* on the other side because quantum mechanics allows it.

A handy mental model:

1. Picture a skateboarder in a half-pipe. Classical physics says they must reach the rim's height to exit. Quantum physics says there's a tiny chance they'll *teleport* over the rim even if they're a little short on speed. That unlikely hop is tunneling.
2. The thinner the rim (oxide), the higher the hop probability. Modern fabs make the oxide thin enough that with the right voltage push, tunneling is reliable and repeatable.

> **Key takeaway:** SSDs rely on *probabilities*, not tiny mechanical doors. Every program/erase cycle is a carefully engineered gamble that overwhelmingly favors the desired outcome.

![Cross-section of a floating-gate transistor showing electrons tunneling through the oxide. Source: Wikimedia Commons, CC BY-SA 4.0](https://upload.wikimedia.org/wikipedia/commons/a/ae/Floating_gate_transistor-en.svg)

## Reading from SSDs: Fast and Efficient

Reading data is relatively simple and fast. The controller checks the voltage level in each cell. If electrons are trapped, the voltage is lower. If not, it's higher. The read operation doesn't disturb the charge state—it just senses it—so reads are fast and have minimal wear.

## Writing to SSDs: A Story of Tunneling and Tradeoffs

Writing is where things get interesting. To write data, electrons are pushed into the floating gate via quantum tunneling—through a process called **Fowler–Nordheim tunneling**. This changes the charge level of the cell, thereby altering the stored bit.

But here's the catch:
**You can't just flip bits randomly.**

NAND cells are organized in **pages** (typically 4–16KB), and pages are grouped into **blocks** (typically 128–256 pages). You can write to a fresh page once, but if you want to update it, you can't overwrite it in place. You have to write the new data to a new page, and mark the old one invalid.

Eventually, when enough pages in a block are marked invalid, the SSD needs to erase the entire block in one go, which is slow and wears down the NAND cells over time.

This is why deletion on SSDs isn't immediate—it's deferred, expensive, and done in batches. It's also why SSDs degrade after thousands of erase cycles.

## The Physics Problem: Why Deletion is Expensive

Erasing a NAND block means resetting all its cells. This involves reversing the quantum tunneling process—pulling electrons out of the floating gate, again by applying high voltages. It's a costly operation, both in terms of time and long-term device health.

If you've heard of "**write amplification**," this is part of it. You may think you're just updating a 1KB file, but under the hood, it could trigger multiple page copies and even full block erases.

## How SSDs Handle All This Complexity

To hide these physical limitations, SSDs rely on smart firmware and controllers that implement:

- **Wear leveling**: Ensures all blocks are used evenly to prevent early failure.
- **Garbage collection**: Reclaims invalid pages and erases blocks in the background.
- **Over-provisioning**: Includes extra storage space invisible to the user to absorb these backend operations.
- **TRIM command**: Allows the OS to tell the SSD which blocks are no longer in use, so it can clean them up proactively.

## Why This Matters for System Design

At a high level, you might be building a caching layer, optimizing a database, or architecting a file system. If you don't understand how SSDs work internally, you may make wrong assumptions about:

- Write performance consistency (hint: it's not always consistent)
- Lifetime of a device under heavy write loads
- Behavior of random writes vs sequential writes
- Cost of deletions and frequent updates

For instance, storing temporary logs in small chunks may seem harmless—until you realize you're triggering tons of write amplification and early SSD wear. Or you might wonder why a query is slow, not realizing it's hitting fragmented storage blocks that the controller is still garbage collecting.

**Understanding the hardware contract helps you make smarter, more resilient design decisions.**

## What's Next?

This was just the beginning. In upcoming posts, I'll go into:

- The different types of NAND (SLC, MLC, TLC, QLC)
- What makes enterprise SSDs different from consumer-grade ones
- How file systems like ext4 or APFS are optimized for flash
- And how distributed systems like Cassandra or RocksDB are flash-aware by design

**The deeper you understand the lowest level, the more confidently you can architect at the highest level.**

Let's go there.

### Pages, Blocks, and the Spatial Hierarchy

Before we dive into writes, it helps to visualize how cells are grouped on silicon:

* **Cell:** One floating-gate transistor that stores 1–4 bits depending on SLC/MLC/TLC/QLC.
* **Page:** Smallest unit you can *read* or *program* in one shot (typically 4–16 KB).
* **Block:** Smallest unit you can *erase* (typically 256 × page size → 1–8 MB).

![NAND flash hierarchy—cells, pages and blocks. Source: STMicroelectronics diagram via phyton.com](https://phyton.com/attachments/docs/ChipProgUSB_HTML/nand_large_page_zoom70.png)

This hierarchy is the root cause of SSD quirks:

* You can write 4 KB, but later you might have to erase 2 MB to modify it (hello, **write amplification**).
* Erase operations are much slower because they must reset *every* cell in the block.

---

*This is Part 1 of the "High Level Design at the Lowest Level" series. [← Back to series overview](./README.md)* 