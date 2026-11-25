# KoreGP on NVK: what a test implementation found

*Status on 2026-11-24. Written with AI tools from the implementation's measurement logs. Every number below comes from `docs/results-n1.md` in the code repository.*

The [proposal](../README.md) says nothing is built yet. A test implementation now exists: KoreGP as a native frontend inside Mesa 26.3.1, on NVK's internals (the NAK compiler, NIL image layouts, the nouveau winsys), running on one RTX 2070 SUPER (TU104). It was built on NVIDIA first because that was the card on hand. That makes it the proposal's M4 question asked before M0: are the conventions vendor-neutral, or "two RDNAs in a trenchcoat"?

They carried over. Compute, textures, raster and presentation run on the GPU below the Vulkan entry points. The pointer-only memory model needed no per-submit bookkeeping. Recording a dispatch costs about a fifth of the CPU time it costs through NVK's Vulkan path on the same card; recording a draw costs between two fifths and two thirds on the card and a third offline. Several things in the proposal took a different form on NVIDIA; three of them change the header. They are listed under [Suggested edits](#suggested-edits-to-the-proposal).

Code: <https://github.com/AlphaPixel/KoreGP>, branch `nvk-test-implementation`.

## What runs

| Proposal milestone | Built | Checked on the GPU |
|---|---|---|
| M0, compute | devices, `kgpMalloc`, SPIR-V through NAK with the root convention, dispatch and indirect dispatch, barriers, timeline semaphores | Yes. Saxpy and prefix sum are correct, and both halves of the exit test are met (cost below). |
| M1, textures | texture layouts over caller memory, descriptor writers, texture and sampler heaps, GPU-side copies, `KGP_HAZARD_DESCRIPTORS` | Yes, including the exit test: descriptors written into the heap by a compute shader and used by the next dispatches. |
| M2, raster | graphics pipelines, render passes, depth and stencil, blend and depth-stencil state objects, direct, indirect and multi draws, presentation | Yes: triangles, a heap texture sampled in a pixel shader, depth, stencil, a multi-draw whose arguments and per-draw root pointers are read on the GPU, and 120 frames on a monitor. The exit test as written (the no_api examples ported) was not attempted. |
| M3 | the validation mode; multi-draw-indirect; mesh shaders (without task shaders); split barriers | Yes, the shader-side heap bounds checks included. A command buffer waiting on a value in memory stays blocked until the CPU writes it. |
| beyond the proposal | MSAA at 2, 4 and 8 samples | Yes. There is no resolve call: a shader reads the samples through the texture heap. |
| not built | task shaders, Wayland and X11 surfaces, the SDK and loader, the layered Vulkan backend, an AMD backend | |

The API is 50 entry points in a 680-line header. The frontend is about 7,000 lines of C; it links NVK and calls into it where it can, so it copies little. 81 tests run without hardware on Mesa's DRM shim. The hardware suite ran in thirteen sessions between 2026-11-21 and 2026-11-24, and everything in the tree has passed on the GPU. Beyond the table, that covers depth bias, blend constants, colour write masks, a second colour target, render targets in sRGB and three float formats, copies to mip levels, array layers and 3D textures, and draw arguments written by a compute shader in the same command buffer.

## What it costs

CPU time to record, in release builds of the same Mesa tree on the same machine. The Vulkan side is the cheapest Vulkan spelling of the same work: the pipeline bound once, then `vkCmdPushConstants` and `vkCmdDispatch` (or `vkCmdDrawIndexed`) per call.

| | KoreGP | NVK through Vulkan | Measured |
|---|---|---|---|
| Record a dispatch | 83 – 140 ns | 429 – 838 ns | on the card |
| Record a dispatch | 41 – 53 ns, 240 instructions | 330 – 350 ns, 3308 instructions | offline, on the DRM shim |
| Record a draw with its own root pointers | 58 – 73 ns | 99 – 110 ns | on the card, 100 draws per command buffer |
| Record a draw with its own root pointers | 27 ns | 63 ns | on the card, 1000 draws per command buffer (one run) |
| Record a draw with its own root pointers | 19 ns, 218 instructions | 59 ns, 660 instructions | offline, 1000 draws per command buffer |
| Submit a command buffer | 8 – 13 µs | the same | on the card |

The dispatch saving is one thing. NVIDIA's compute launch takes a 256-byte descriptor (the QMD) naming the shader, the group counts and the constant buffers. NVK packs it field by field on every dispatch, because which constant buffers it names depends on the descriptor state bound at that moment. Under KoreGP's fixed conventions everything in it except the group counts and one address is known when the pipeline is created, so `kgpDispatch` copies a template and patches five words.

The draw saving is smaller and has a different cause. Per draw, both drivers send the GPU almost the same words. What NVK also does, and KoreGP does not, is bookkeeping: it shadows the push constants and asks on every draw whether any dynamic state, descriptor set or vertex buffer is dirty. KoreGP's draw has one flag to look at. On the card both pay about 35 to 40 ns more per draw for writing commands into GPU-visible memory, so the difference stays and the ratio shrinks.

In both cases the saving is not cleverness in the frontend. It is state the API no longer has.

Submission is the kernel's path and KoreGP does not change it. What the proposal's §5.1 hangs on was checked with `strace`:

| | `VM_BIND` calls | `EXEC` calls |
|---|---|---|
| 8 command buffers | 12 | 27 |
| 800 command buffers | 12 | 819 |

Memory is bound once, when it is allocated. A submit carries address ranges and nothing else.

What these numbers do not cover:

- They are recording cost. GPU time was not compared; both sides run the same shader through the same compiler.
- One shader and one shape each. No application has been ported.
- Other jobs were running on the machine during the hardware sessions, so the spread between runs is wide there. The offline instruction counts are the stable figure.
- NVK could cache its QMDs too. The comparison is against the driver as it is.
- Nothing was compared with a layered implementation such as Loon or sgpu, which is the comparison the proposal's §9 asks for.

## How the architecture translated

Labels D1 to D10 refer to `docs/nvk-mapping.md` in the code repository, which has the detail.

| Proposal | On NVIDIA | Outcome |
|---|---|---|
| Submits carry an empty buffer list (§5.1) | nouveau's submit call has no buffer list to leave empty. It takes address ranges and sync objects; residency comes from `VM_BIND`. | Holds, more strongly than on AMD. Needs kernel 6.6 or later. |
| Default memory is CPU-mapped VRAM through ReBAR (§5.1) | On this card CPU-mapped VRAM goes through a 256 MB aperture. | The device property has to be a size, not a yes or no (D1). |
| The texture heap is a plain array the shader indexes (§5.2) | This is the hardware's own model: a register points at a table of 32-byte entries in memory, and the shader's texture handle is the index. | Fits better than on AMD. The heap setter is one register write, the descriptor is exactly 256 bits, and non-uniform indexing needs no special path. |
| Sampler descriptors are 128-bit (§5.2) | Samplers live in a separate table of at most 4096 32-byte entries, and the hardware is told how many entries each table has. | API change: a sampler heap setter, an entry count on both setters, 256-bit sampler descriptors (D2, D3). |
| A texture is a layout over memory the caller already owns (§5.2) | NVIDIA tags page-table entries with a surface "kind". Colour and 32-bit float depth work at an ordinary address. D24S8 does not. | No API change so far: for that format the driver maps the same pages a second time under the kind. Multisampled textures are mapped under their kind without trying the ordinary address first. |
| Root pointers live in SGPRs (§5.3) | There are no SGPRs. For compute, the driver writes a 64-byte root record per dispatch and binds it as a constant buffer; the shader reads the pointer from it. For raster, the two pointers go into a hardware root table written inline from the command stream. | Works. Compute has one more indirection than on AMD; raster has a direct counterpart (D5, D10). |
| Root fields are preloaded into registers (§5.3) | The counterpart is binding the caller's root struct itself as a constant buffer. It was built, was correct on the card and was measured in three sessions. | Not adopted, and removed. It was 5 to 9 % faster on saxpy and 25 to 81 % slower on a shader written to favour it. It also needs 64-byte aligned roots and a new hazard bit. |
| A dispatch is a register write and a packet (§5.4) | A dispatch writes 320 bytes of memory: the 256-byte QMD and the root record. | This is the hardware's model, not a layering cost, and the Vulkan driver pays more of it (D6). |
| GPU-generated root data through indirect SGPR loads (§5.4) | The 3D engine has a macro processor that can read memory and write root tables. One KoreGP macro reads the draw count and each draw's arguments, and writes each draw's root pointers. | Works on the card, with no CPU work per draw. |
| Barriers keyed on a stage pair plus hazards (§5.5) | NVK already reduces every Vulkan barrier to ten flags. KoreGP sets them from the enum. | Compute to compute emits exactly the three methods NVK emits for a shader write to read barrier. There are no layout transitions. Split barriers are the channel's semaphore methods, which NVK does not use: a release stores a 64-bit value at an address, or the maximum of it and what is there, and an acquire holds the channel until the memory equals or reaches a value. |
| A hidden Vulkan device owns the swapchain (§5.6) | Built as written, in 363 lines. The images cross as dma-bufs. | Works on a monitor (through `VK_KHR_display`) and headless. Wayland and X11 are not built. |
| `KGP_DEBUG=1` interposes the dispatch table (§7) | There is no dispatch table; the entry points are plain functions. Each has one branch on a flag. | Offline, a dispatch goes from 49 to 60 ns with validation on. |
| Each file stays diffable against its ancestor (§5) | Where it could, the frontend calls NVK's own functions instead of copying them: the draw macros, the 3D engine setup, image layout, the compiler. | Every command path has a Vulkan twin and an automated comparison with NVK's command stream. From render pass to first draw, for example, 338 methods are identical. |

## Suggested edits to the proposal

1. **Front matter.** "Nothing here is built yet" is out of date.
2. **§5.1.** The mapped-VRAM fallback is reported as a size (`mappedVramBytes`), not a flag.
3. **§5.2 and §6.** Add `kgpSetActiveSamplerHeapPtr`. Both heap setters take an entry count. The sampler descriptor is a `KgpDescriptor256`; AMD uses half of it.
4. **§5.3.** Pin down how SPIR-V spells the conventions, because the proposal leaves it open and the layered backend of §9 needs the same shaders to run on both paths:
   - Root: a push-constant block whose first member is a `PhysicalStorageBuffer` pointer. Compute reads it at offset 0. For raster, the vertex stage reads offset 0 and the pixel stage offset 8.
   - Heaps: `RuntimeDescriptorArray` at set 0, binding 0 for textures and binding 1 for samplers.
   - Specialization: a list of `{id, 64-bit value}` pairs, so a pointer-valued constant is an ordinary 64-bit one.

   The test shaders run unchanged through Vulkan with these, which is how each test gets its Vulkan twin.
5. **§5.3 and §5.4.** State the consequence that produced the largest measured saving: with fixed conventions, the driver can build per-dispatch state once, at pipeline creation.
6. **§5.6 and §6.** Name the presentation calls. The implementation has four: `kgpCreateSwapchain`, `kgpAcquireSwapchainTexture`, `kgpPresent`, `kgpDestroySwapchain`.
7. **§6.** Choices the header had to make where the sketch is silent: `kgpSubmit` and `kgpWaitSemaphore` return `bool`; `kgpCreateDevice` takes a device index and each device has one queue; `kgpSetPipeline` is a separate call, as in the blog; destroying a pipeline is deferred against the device timeline, as `kgpFree` is.
8. **§5.5 and §6.** Name the split-barrier calls and their operations. The implementation has `kgpSignalAfter` (set, maximum) and `kgpWaitBefore` (equal, greater or equal) on a 64-bit value; on NVIDIA the address must be below 2^40.
9. **§6.** Say where MSAA lives: here a sample count on the texture and on the raster state, with standard sample locations and no resolve call.
10. **§7.** "Root-pointer alignment" needs a number (4 bytes here). The heap bounds check takes its limits from the counts of edit 3. "Interposes the dispatch table" can be a flag.
11. **§8.** M4 is partly answered. What remains open is the other half: the same conventions on RADV.

## What this does not show

- It is one card, one kernel driver (nouveau with GSP firmware) and one machine.
- It says nothing about AMD, the proposal's reference target.
- There is no ZCULL, which NVK has, so GPU time on depth-heavy scenes is not comparable yet.
- A wait that is never satisfied was not tried; the longest a channel was held was 200 ms.
- It is a test implementation. It refuses GPUs older than Turing, and its presentation path has only been exercised without a compositor.

## Where to look

In the code repository:

- `docs/nvk-mapping.md`: the proposal restated section by section in NVIDIA terms.
- `docs/results-n1.md`: every measurement, by hardware session, with method and caveats.
- `docs/plan.md`: what is done, what is next, and in what order.
