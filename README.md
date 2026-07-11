# Open Pixmap File Format (OPFF)

**Revision 1.0**

The Open Pixmap File Format (OPFF) is a deterministic, hardware-first file format for multidimensional arrays (images, video frames, AI tensors, and point clouds). 

It is designed strictly for **memory-mapped, zero-copy parsing**. It physically prohibits data compression, variable-length metadata trees, and implicit scanline padding. The parser simply reads a fixed 64-byte header, calculates the mathematical memory bounds in `O(1)` time, and hands a cache-aligned raw pointer directly to the application. 

### Motivation: Radical Accessibility & Debugging
OPFF was not engineered to compete with enterprise serialization frameworks, nor is it designed to perfectly align with every proprietary hardware accelerator on the market. It was built for **hackers, researchers, and engine developers**. 

The primary goal of OPFF is to drop cognitive load to zero during algorithmic experimentation and debugging. By stripping out variable-length metadata, compression, and nested schemas, OPFF provides a format so mechanically transparent that a developer can open it in a hex editor and instantly verify its spatial bounds. High-performance traits—like zero-copy memory mapping and native CPU cache alignment—are not the goal; they are simply the natural byproducts of keeping the data structure aggressively simple.


### Architectural Boundaries (Limitations)

* **Hardware Agnosticism over Hyper-Optimization:** While the 64-byte header provides a clean baseline for most modern processors, OPFF does not attempt to chase proprietary alignments (e.g., 128-byte Tensor Core boundaries). Simplicity and structural predictability will always take precedence over hardware-specific micro-optimizations.

## Documentation & Architecture

For the historical context, design rules, and strict boundaries regarding this format, refer to the supplementary documentation:

* [Origin Story: The Netpbm & PNG Dilemma](ORIGIN.md)
* [Architectural Philosophy & Extension Rules](PHILOSOPHY.md)
* [Techniques: Workarounds & Application-Specific Extensions](TECHNIQUES.md)
* [AI Usage Statement](AI_USAGE.md)


---
## 1. The Header & Cache Alignment

The header is exactly **64 bytes** to guarantee that the contiguous payload starts perfectly aligned with modern CPU L1 data cache lines (e.g., AVX-512, ARM SVE). 

* The first 16 bytes contain the structural layout geometry. 
* The next 48 bytes are reserved and MUST be filled with zeros (`0x00`) to force the 64-byte boundary.
* Both the header and the payload are strictly **Little Endian (LE)** to eliminate dynamic byte-swapping overhead on modern execution architectures.

| Offset | Size | Field | Description |
| :--- | :--- | :--- | :--- |
| `0x00` | 4B | **Magic Number** | `b'OPFF'` (See Section 7 for Extensions) |
| `0x04` | 1B | **Version** | `0x01` |
| `0x05` | 1B | **Depth** | Specifies physical byte size and hardware routing. |
| `0x06` | 1B | **Config** | Bit 7: Memory Layout. Bits 0-6: Channel Count. |
| `0x07` | 1B | **Frames** | Number of sequential frames (`1` to `255`). |
| `0x08` | 4B | **Start ID** | Starting frame index or temporal coordinate (`u32`). |
| `0x0C` | 2B | **Width** | Spatial X limit (`u16`). |
| `0x0E` | 2B | **Height** | Spatial Y limit (`u16`). |
| `0x10` | 48B | **Reserved** | Must be `0x00`. Strict cache alignment padding. |

---
## 2. Hardware Routing & Data Types (Offset `0x05`)

This byte physicalizes the execution pathway. It tells the hardware the scalar size and whether the data routes to the Arithmetic Logic Unit (ALU) or the Floating Point Unit (FPU). By default, the core treats these as Unsigned Integers (`u`) or Standard Floats (`f`).

**Constraint:** Only exactly one bit can be set (`popcount <= 1`). This allows parsers to validate the execution path with a single branchless CPU instruction.

* **ALU Routing (Integers):**
  * `0x01` (1-byte / `u8`)
  * `0x02` (2-byte / `u16`)
  * `0x04` (4-byte / `u32`)
  * `0x08` (8-byte / `u64`)
* **FPU & Tensor Routing (Floats):**
  * `0x10` (FP8 - Maps natively to Hopper/SME2 Tensor Cores)
  * `0x20` (f16)
  * `0x40` (f32)
  * `0x80` (f64)
* **Bitmap Fallback:**
  * `0x00` (1-bit spatial data, packed into modulo-8 bytes)

---
## 3. Memory Layout & SIMD Stride (Offset `0x06`)

This byte dictates the vector depth and how the channels are physically laid out in memory, directly impacting cache-hit rates during SIMD operations.

* **Bits 0-6 (Channel Count):** Number of values per coordinate (Valid: `1` to `127`).
* **Bit 7 (Layout Flag):**
  * `0` = **Interleaved / Array of Structures (AoS):** Channels are stored sequentially per pixel (e.g., `RGB RGB RGB`).
  * `1` = **Planar / Structure of Arrays (SoA):** Channels are separated into contiguous memory blocks (e.g., `RRR GGG BBB`). Guarantees perfectly coalesced memory access for vector processing.

---
## 4. Size Validation & The 64-Bit Mandate

To mathematically prevent 32-bit heap buffer overflows during allocation, parsers **MUST** cast the 8-bit and 16-bit header variables to 64-bit unsigned integers (`u64`) *before* executing the expected size multiplication.

**Standard Calculation (`Depth != 0x00`):**
```text
Expected_Size = 64 + (Frames * Width * Height * Channel_Count * Depth_Bytes)
```
**Bitmap Calculation (`Depth == 0x00`):**
```text
Expected_Size = 64 + (Frames * Channel_Count * ((Width * Height) / 8))
```

---
## 5. The Atomic Rejection Matrix

A valid parser must instantly abort and reject the payload if it encounters any of these hardware contradictions:

1. **The Interleaved Bitmap Contradiction:** Depth is `0x00` AND Layout is `0`. Hardware cannot efficiently interleave fractional byte boundaries across channels.
2. **The Channel Count Out-of-Bounds Contradiction:** The extracted Channel Count is `0` (void), or the raw byte attempts to declare a magnitude `> 127` without respecting the Layout bitmask. Hardware limits this to a strict `u7` logical bound.
3. **The Zero-Dimension Contradiction:** Width, Height, or Frames are `0`. Prevents undefined behavior during `mmap` calls.
4. **The Bitmap Modulo Contradiction:** Depth is `0x00` but the spatial area (`Width * Height`) is not a clean multiple of 8. Prevents implicit scanline padding.
5. **The Geometric Bounds Contradiction:** The physical file size on disk is strictly less than the `Expected_Size`.

---
## 6. Custom Metadata (The Schema-in-the-Footer Pattern)

OPFF system headers are mathematically locked and physically prohibit variable-length metadata trees. If an application needs to store semantics (e.g., JSON dictionaries, camera intrinsic parameters, or signedness overrides), it appends them to the *end* of the file.

> If `Actual_File_Size == Expected_Size` ➔ The payload is a raw, transient memory block.

> If `Actual_File_Size > Expected_Size` ➔ Custom metadata immediately follows the valid memory block. 

The parser maps the geometry, hands the `pixel_buffer` up to the application, and the application simply advances its pointer to `Expected_Size` to read its own footer schema.

---
## 7. Extensions & The Decentralized Namespace

Because OPFF physically isolates structural bounds from semantic interpretation, communities can build independent formats on top of the OPFF core. To avoid format collisions, ecosystems mutate the 4th byte of the Magic Number at offset `0x00`:

* `b'OPFF'`: The strict, foundational core standard.
* `b'OPFX'`: The Sandbox. Designated for unformalized local testing and experimental systems.
* `b'OPF[A-E, G-W, Y-Z]'`: The Decentralized Ecosystem. Open namespace characters for communities to claim and define their own standardized semantic footers.

---
## 8. Applications & Limitations

### Core Application: Debugging & Experimental Verification

OPFF is fundamentally a diagnostic tool. Its primary purpose is to rapidly dump, map, and verify multidimensional memory states without the interference of implicit padding, text-parsing overhead, compression, or variable-length header bloat.

* **Graphics & Rendering:** Dumping raw framebuffers, raytracer outputs, or intermediate pipeline states to verify mathematical logic.

* **AI/ML Development:** Inspecting intermediate neural network states or isolating tensor mathematics during model training.

* **Physics & Spatial Systems:** Verifying the exact state of deterministic physics grids or 3D LiDAR point clouds without the friction of sequential chunking.

### Secondary Applications: Zero-Overhead Pipelines

Because OPFF is mathematically rigid and completely devoid of bloat, its raw throughput makes it highly effective for specialized local pipelines, even outside of debugging contexts:

* **Hardware Sensor Dumps:** Capturing high-frequency, raw physical sensor arrays (e.g., FPGA to NVMe) straight to disk with zero CPU processing overhead.

* **Local Engine Assets:** Loading uncompressed textures or spatial arrays into custom systems engines via instant zero-copy memory mapping.

* **Embedded Inference:** Routing quantized ML weights directly to hardware FPU registers on local edge devices where deserialization lag is unacceptable.

### Architectural Boundaries (Limitations):

* **Heterogeneous Data:** OPFF enforces strict multidimensional arrays of a single data type. It cannot mix floats and variable strings in the contiguous payload.

* **General-Purpose Storage:** It is inherently hostile to web APIs, network distribution, text documents, or sparse data graphs. Use Protobuf, FlatBuffers, or JSON for dynamic schema evolution.


---
## License

This format and reference implementation are dual-licensed under either the Apache License, Version 2.0, or the MIT License, at your option.

Copyright 2026 @Srinuyadav149

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   [http://www.apache.org/licenses/LICENSE-2.0](http://www.apache.org/licenses/LICENSE-2.0)

Alternatively, this project may be used under the MIT License:

   [http://opensource.org/licenses/MIT](http://opensource.org/licenses/MIT)

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the respective licenses.
