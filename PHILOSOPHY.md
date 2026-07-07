# Architectural Philosophy

OPFF is governed by strict architectural rules designed to protect the integrity of the data layer, prevent feature creep, and allow infinite extensibility without breaking the core parser.

## 1. The "Implementation Friction" Heuristic

OPFF enforces a strict boundary between the data layer and the application layer using a simple heuristic: **If a feature creates implementation friction for the parser, it does not belong in the data layer.**

The data layer is a static map of physical geometry, not a processing engine.

* **Compression:** Requires dictionary matching and sliding windows. This belongs in an external OS pipeline.

* **Dynamic Metadata Trees:** Requires schema parsing and heap allocation. This belongs in the application's appended footer.

* **Concurrency and File Locks:** Requires OS scheduler management. This belongs entirely in the application layer.

Because OPFF relies strictly on $O(1)$ arithmetic for boundary calculation, a developer can write a perfectly safe, zero-copy parser from scratch in any systems language in a matter of minutes.

## 2. The Frozen Core

The core 64-byte specification of OPFF is **permanently frozen**. It will never receive new features, dynamic flags, or structural updates. 

In debugging and rapid experimentation, your file format must act as a strict scientific control group. If an AI tensor outputs unexpected values or a physics grid desyncs, you need absolute certainty that the file format did not silently misalign the memory or implicitly pad a scanline. By permanently freezing the core geometry, OPFF removes itself as a variable in your debugging process. 

## 3. Mechanism, Not Policy (The Extension Model)

While the 64-byte core is strictly frozen, the format provides an isolated escape hatch for metadata via the 1-byte namespace character in the Magic Number (`OPF*`). This allows infinite flexibility at the application layer without requiring a single change to the core specification.

* **`OPFX` (Personal Extension):** This namespace is strictly for rapid, personal experimentation. A developer can append whatever they want to the footer—an unstructured JSON string, a raw casted C struct, or a FlatBuffer. The core parser simply bypasses it, giving hackers complete freedom to mutate metadata on the fly.

* **`OPF*` (Domain-Specific Standards):** For domains that require strict typing, communities can claim other namespaces (e.g., `OPFI` for Imaging, `OPFL` for LiDAR, `OPFT` for Tensors). Those communities can mandate and freeze their own custom footer schemas. If a sensor manufacturer wants to mandate a hardcoded parsing logic for an `OPFL` footer, they can do so entirely within their own ecosystem.

The core parser remains completely ignorant of these footers, ensuring the fundamental format never bloats while supporting both chaotic experimentation and rigid domain standards.
