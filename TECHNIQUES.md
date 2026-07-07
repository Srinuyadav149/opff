# Workarounds & Application-Specific Techniques

OPFF’s core is frozen and devoid of implicit logic. If your hardware architecture—such as FPGA alignment constraints or SIMD vectorization targets—demands specific memory layouts, you must enforce them at the application layer.

## Accountability: The "Clean Room" Guarantee

When you debug a memory dump, you have absolute certainty that any logic modifying the raw data was explicitly defined in your own code. The format does not perform padding, alignment, or metadata injection behind your back. 

* **Zero Hidden Variables:** If a buffer is misaligned, you know exactly which line of your application code added the padding.

* **Deterministic Verification:** Because the core parser is restricted to raw, unpadded memory blocks, you can verify your application's output against the disk state without the noise of format-specific transformations.

* **Format Neutrality:** The format acts as a passive observer. It does not "know" if it contains a bitmap, a tensor, or a LiDAR point cloud; it only maps the physical geometry.

---

## 1. Explicit Padding (Memory Alignment)

* **The Constraint:** Your pipeline requires buffers to be aligned to a 64-byte boundary or a modulo-8 row stride for SIMD efficiency.

* **The Application Logic:** Manually insert dummy bytes or pad your width dimensions in your application code *before* writing the buffer. You effectively bake the alignment requirement directly into the file geometry, ensuring the OPFF file remains a faithful 1:1 mirror of your memory.

## 2. Bitmaps & Modulo-8 Stride

* **The Constraint:** Your data is sub-byte (e.g., 1-bit or 4-bit) and the scanline width does not map cleanly to a byte boundary.

* **The Application Logic:** Define the scanline as an array of the smallest possible byte-aligned unit. For example, if a 1-bit image width is 13 pixels, define the width as a 2-byte (16-bit) array in the header. Your parser logic remains O(1) by consistently reading a byte-aligned stride.

## 3. Endianness & Byte Order (The Rare Exception)

* **The Constraint:** OPFF captures a direct mirror of RAM, meaning it captures the host architecture's native endianness. Because modern hardware (x86, ARM) is overwhelmingly Little-Endian, endian mismatch is a rare exception.

* **The Application Logic:** If you are operating in a cross-architecture environment (e.g., sharing data with legacy Big-Endian mainframes or specific network hardware), you must handle the byte order manually. Either pre-swap your multi-byte data (`u16`, `f32`) to a standardized Little-Endian format before triggering the memory dump, or use a custom footer to embed an endianness flag for the receiving parser.

## 4. Custom Footers (`OPFX` & `OPF*`)

* **The Constraint:** You need to attach metadata (calibration matrices, timestamps, color spaces) or define experimental, application-specific data types (e.g., swapping a standard 8-bit float for Bfloat8) without breaking the zero-copy core parser.

* **The Application Logic:** Append arbitrary bytes immediately after the contiguous data block. The core parser stops at the defined dimensions. You can use the namespace byte in the magic number (`OPFX` for personal hacks, `OPF*` for domain standards) to signal to your application layer how to parse this trailing metadata. This enables seamless type-swaps: the core header maps the physical byte footprint, while the footer instructs your application logic to cast that exact footprint into a custom or experimental data type.
