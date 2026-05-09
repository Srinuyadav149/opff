# Open Pixmap File Format (OPFF)
**Revision 1.0**

OPFF is a fast, dead-simple file format for multidimensional arrays (like images, video frames, AI tensors, or point clouds). 
It is designed strictly for **memory-mapped, zero-copy parsing**. It has no compression, no variable-length metadata, and no implicit padding. The parser simply reads the fixed 64-byte header, calculates the memory bounds, and hands a raw pointer directly to the application. 
The format guarantees the structural size of the memory block. It does *not* care what the data means (e.g., color spaces, coordinate systems). That is up to your application.
---
## 1. The Header
The header is exactly **64 bytes** to guarantee perfect alignment with modern CPU cache lines (like AVX-512). 
* The first 16 bytes contain the layout. 
* The next 48 bytes are reserved and must be filled with zeros (`0x00`).
* Everything is strictly **Little Endian (LE)**.

| Offset | Size | Field | Description |
| :--- | :--- | :--- | :--- |
| `0x00` | 4B | Magic Number | `b'OPFF'` (See Section 6 for extensions) |
| `0x04` | 1B | Version | `0x01` |
| `0x05` | 1B | Depth | Specifies data type and byte size (see below). |
| `0x06` | 1B | Config | Bit 7: Planar flag. Bits 0-6: Channel count. |
| `0x07` | 1B | Frames | Number of frames (`1` to `255`). |
| `0x08` | 4B | Start ID | Starting frame index or time coordinate (`u32`). |
| `0x0C` | 2B | Width | Spatial width (`u16`). |
| `0x0E` | 2B | Height | Spatial height (`u16`). |
| `0x10` | 48B | Reserved | Must be `0x00`. Strict cache alignment padding. |

## 2. Data Types (Offset `0x05`)
This byte tells the hardware how big a single element is, and whether to send it to the integer unit (ALU) or float unit (FPU). By default, the core treats these as unsigned (`u`) or standard floats (`f`). If you need signed integers (`i`), handle that in your app.
Only **one** bit can be set:
* **Integers:** `0x01` (1-byte), `0x02` (2-byte), `0x04` (4-byte), `0x08` (8-byte)
* **Floats:** `0x10` (FP8), `0x20` (f16), `0x40` (f32), `0x80` (f64)
* **Bitmap:** `0x00` (1-bit data, packed into bytes)
## 3. Memory Layout (Offset `0x06`)
This byte tells the parser how many channels there are, and how they are arranged.
* **Bits 0-6 (Count):** Number of channels/values per coordinate (1 to 127).
* **Bit 7 (Layout):** * `0`: Interleaved (e.g., RGB RGB RGB)
  * `1`: Planar (e.g., RRR GGG BBB)
## 4. Size Validation
To prevent integer overflow exploits, you **must** cast the header variables to 64-bit integers (`u64`) before doing this math.
* **Standard calculation:** `Expected_Size = 64 + (Frames * Width * Height * Channels * Depth_Bytes)`
* **If Depth is `0x00` (Bitmap):**
  `Expected_Size = 64 + (Frames * Channels * ((Width * Height) / 8))`
## 5. Rejection Rules
A valid parser must immediately abort if it sees any of the following:
1. **Invalid dimensions:** Width, Height, or Frames are `0`.
2. **Invalid channels:** Channel count is `0`.
3. **Invalid bitmap layout:** Depth is `0x00` but the layout is set to Interleaved (`0`).
4. **Invalid bitmap size:** Depth is `0x00` but `Width * Height` is not a clean multiple of 8.
5. **File too small:** The actual file size on disk is smaller than `Expected_Size`.
## 6. Custom Metadata & Ecosystem
OPFF headers are completely locked. If your application needs to store metadata (like JSON dictionaries, camera parameters, or color profiles), you append it to the *end* of the file.
* If `Actual_File_Size == Expected_Size`: It's just raw data.
* If `Actual_File_Size > Expected_Size`: There is custom metadata sitting right after the memory block.
To avoid format collisions, the community can swap the 4th byte of the Magic Number to define specific footer standards:
* `b'OPFF'`: The strict core standard.
* `b'OPFX'`: The sandbox. Use this for local testing and experimental features.
* `b'OPF[A-E, G-W, Y-Z]'`: Open namespace for custom community formats.
## 7. Applications & Limitations
**Where OPFF shines:**
* **Local AI/ML Inference:** Routing heavily quantized FP8 or f16 tensor weights directly to mobile hardware or GPUs with zero deserialization overhead.
* **3D Geospatial & LiDAR:** Memory-mapping massive point clouds instantly without the chunking or decompression overhead of formats like LAS.
* **Systems Engines:** Loading raw video frames, spatial maps, or deterministic physics buffers directly into C or Rust engine structs.
**Where OPFF fails (Limitations):**
* **Heterogeneous Data:** It only supports arrays of the exact same data type. It cannot mix floats and text in the same contiguous block.
* **General-Purpose Storage:** It is terrible for web APIs, text documents, or nested relational databases. If you need dynamic or variable-length data, use JSON, Protobuf, or FlatBuffers instead.
---
## License
This format and reference implementation are licensed under the Apache License, Version 2.0.

Copyright 2026 @Srinuyadav149
Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at
    [http://www.apache.org/licenses/LICENSE-2.0](http://www.apache.org/licenses/LICENSE-2.0)
Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.