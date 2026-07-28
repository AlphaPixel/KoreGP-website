# KoreGP (KGP): A Proposal and Implementation Plan

A minimal General Processing API, built from the guts of Mesa's Vulkan drivers

Draft 0.2 — for discussion. Nothing here is built yet, the point of this document is to find out whether it should be. I'm probably not even the person to build it. I'm just a guy going to market across the valley on my bicycle. I can see where a bridge should be built, but I'm not qualified as a structural engineer, architect or builder.

Yes, some of this document was partially written using AI tools. That doesn't make it wrong. I just didn't have the effort necessary to flesh it out, so I gave it my notes and asked it to complete the work. If that morally offends you, move along, but AI tools ARE a force multiplier, if used responsibly.

Reference target: AMD RDNA2+ through Mesa RADV internals, NVidia NVK through Mesa, on Linux.

---

## 1. KGP WTF?

In December 2025, Sebastian Aaltonen published "No Graphics API," arguing that DirectX 12, Vulkan, and Metal — ten years old, designed for GPUs that are now thirteen years old — carry an API surface that modern hardware no longer needs. His prototype replacement fits in 150 lines: memory is `gpuMalloc` returning CPU-mapped GPU pointers, shader inputs are a single 64-bit root pointer cast to a user struct, textures are 32-bit indices into a heap that is just memory, barriers are producer/consumer stage masks with no resource lists, and the pipeline object shrinks to shader code plus target formats. The current Vulkan header is around 20,000 lines. The differece between those two numbers is the cost of abstracting 2013 hardware APIs on 2026 hardware designs.

Within weeks, at least three independent implementations appeared. All three are layers on top of the Vulkan API, and all three run today, which settles one question immediately: the programming model works on shipping drivers. But all three also pay a penalty for sitting at the wrong layer, and one of them documents the cost explicitly (Section 3). This proposal is about removing the tax by implementing the API below the Vulkan entry points instead of above them, inside Mesa, where the driver internals already speak in pointers and packets, and where the Vulkan object model is itself just a translation layer that can be deleted rather than doubled.

Why bother, when the layered versions work? Three reasons, in order of weight. First, correctness of the argument: Aaltonen's claim is that the abstraction is unnecessary, and a layered implementation cannot test that claim, it can only hide the abstraction, still paying for it (but maybe reducing the cost a little). A native implementation either proves or disproves it, either result is useful, but I think we know the result in advance. Second, CPU cost: the layered versions translate pointers to VkBuffer handles so the driver can translate them back to pointers, a native version does neither. Third, leverage for standardization: "here is a 150-line API and a paper design" persuades nobody anywhere. "Here is a conformant driver on Mesa, benchmarked against RADV on the same silicon" is a whole different beast.

Why Mesa? Well, because it's open source. Nobody will give me the source to the Windows or Mac NVidia or AMD drivers, so I gotta work with what I have. Mesa is
actually a phenomenally great piece of code, and it being open means it's very transparent and easy to validate and easy to fork. Nobody can say "no you can't",
thank you FSF and Stallman for the Free Software Principles. They give us enough rope to vkHangOurselves() if we want, but that's also a good thing.

## 2. What KoreGP is, and what's with the name?

The idea is that KoreGP is a cross-vendor GPU API with the feature set of Aaltonen's prototype: compute pipelines, vertex+pixel and mesh+pixel raster pipelines, pointer-based memory, a writable descriptor heap, stage-mask barriers, timeline semaphores, indirect everything. It is not a rendering engine, not a shader language (v0.1 consumes SPIR-V), and not a Vulkan competitor in scope — it omits ray tracing, video, sparse residency, and Windows (for now at least). More could be done later if it proves useful in the short run.

The name is rather clever, if I do say so myself. *Kore* is the layer beneath the mantle, which is a dig at AMD's Mantle, the 2013 API that begat this whole generation, with the nudge that we are now going one layer deeper (GPU-ception!). The K follows house tradition: Khronos is Chronos with a K, Vulkan is Vulcan with a K, so a pastiche of both keeps the K. *GP* drops the U from GPU the way GL once shed letters into a two-character signature, and it expands to General Processing rather than Graphics — an acknowledgment that somewhere between the mining boom and the AI boom, graphics became a minority workload on the "graphics" processor. Nvidia's gaming revenue is around 10% of the total; the hardware evolved accordingly, and the hardware won the argument. An API named for general processing, with graphics as one profile among several, describes what these chips actually are in 2026. OPenGL kinda went this way around OpenGL 3, and then backtracked some in 4 and 4.5. It's fine. KoreGP might be a bit of a competitor to generic CUDA, but without the CUDA API compatability, nobody will be much moved.
If major portable GPU libraries supported KoreGP in addition to CUDA and OpenCL and such, maybe someone would care.

Function prefix: `kgp`. `kgpMalloc`, `kgpDispatch`, `kgpBarrier`. Reads like an instruction mnemonic, which is about right.

## 3. Prior art: three layered implementations and what each one teaches

*This section is wholly AI written. Live with it. -Xenon*

**sgpu** (roeyb1, Jai, ~160 commits) collapses Vulkan's descriptor model into one unified pipeline layout: a single push-constant range holding two 64-bit device addresses, plus one bindless heap indexed from shaders. `gpu_malloc` returns a CPU/GPU pointer pair. Shaders are Slang, so the root struct is a real pointer in the source: `[[vk::push_constant]] VertexData* params;`. Runs on Windows, Linux, and macOS via MoltenVK, with RenderDoc capture. sgpu's lesson is that the programming model needs no driver changes at all, and that Slang already provides the pointer-capable shader language Aaltonen's examples assume — no need to invent one.

**no_api** (UnNabbo, Jai) is the closest transliteration of the blog's prototype header, and its README contains the most valuable sentence in this whole body of work, filed under "Problems": since Vulkan is buffer-centric, most memory-touching calls must run a tree lookup converting a raw GPU address back into the VkBuffer + offset the API demands — after which the driver converts the handle back into the address the user started with. The author also notes the texture heap can't be laid out freely; it must match Vulkan's descriptor stride rules. Both problems are artifacts of the API boundary, not the hardware. no_api's lesson is the tax, measured and named by someone who paid it.

**loon_gpu** (rkevingibson, C++17, CMake, MIT) is the most production-shaped of the three: a self-contained library with vendored dependencies, CI, doctests, natvis/LLDB debugger visualizers, and Doxygen. It targets a Vulkan 1.3 baseline with exactly four required features — dynamic rendering, buffer device addresses, timeline semaphores, descriptor indexing — and adds a Metal backend, with macOS Vulkan available through LunarG's KosmicKrisp layer. Its README describes itself as the blog design "adapted to the realities of what is possible in Vulkan today," which is a precise statement of the ceiling: adaptation to those realities is the layer's job, and the realities are the cost. loon_gpu's lessons are two. The four-feature baseline is the exact capability set a portability backend needs (Section 9 uses it verbatim). And the Metal backend proves the API maps onto a second, quite different driver stack, which matters for the claim that KoreGP's conventions are vendor-neutral rather than secretly RDNA-shaped.

Read together: the model works everywhere (sgpu, loon_gpu), the layering tax is real and located (no_api), and the portability floor is known (loon_gpu). What none of them can do is delete the tax, because the tax lives on the far side of the API boundary they sit on. That is the job of a native implementation, and Mesa is the only production driver codebase open enough to host one.

## 4. Why Mesa (what did it do to deserve this?)

*Also, significantly AI written, but generally correct. -Xenon*

Mesa's Vulkan drivers, RADV for AMD, NVK for Nvidia, Turnip for Qualcomm, ANV for Intel, are MIT-licensed and structured as a thin common runtime over per-vendor hardware layers. The hardware layers *already* operate in KoreGP's terms. RADV descriptor sets are plain GPU memory the driver memcpys 256-bit hardware descriptor blobs into. Buffer device address is not an extension on GCN/RDNA so much as the native addressing mode; the VkBuffer handle is the fiction and the 64-bit VA is the fact. Barriers compile to a short sequence of cache-flush and wait packets that mostly ignore the resource list. Timeline semaphores are DRM syncobjs. The Vulkan surface is a translation layer inside Mesa from retained-mode objects to the pointer-and-packet model underneath — the same translation the layered projects reimplement in reverse, one level up. So, we want to take a scalpel and remove all the fat at the interface boundary.

An obvious plan is to fork RADV and strip out what KoreGP doesn't need. That plan sounds like subtraction and isn't. RADV's Vulkan bookkeeping — dynamic state dirty bits, pipeline bind fast paths, render pass compatibility checks, descriptor set machinery — is threaded through command recording, not stacked on top of it. Stripping means months of deleting code while keeping a broken build alive, and the endpoint converges on the alternative anyway. The alternative, which this proposal specifies, is extraction: a new `src/koregp/` frontend that calls the hardware truth directly — ACO as the compiler, addrlib for surface layouts, the descriptor blob writers from `radv_image.c`, the flush logic from `si_cmd_buffer.c`, libdrm_amdgpu for submission — and leaves the Vulkan bookkeeping where it is. Perhaps a fifth of RADV is hardware truth; the frontend that binds it to the KoreGP header is an estimated 15–25k lines of new-plus-extracted C.

One complication up front: RADV implements copies, blits, clears, and DCC fast-clear-eliminate as internal Vulkan pipelines ("meta" operations). KoreGP needs equivalents for `kgpMemCpy`, `kgpCopyToTexture`, and render-target clears. A possible plan is to rewrite them as internal KoreGP compute pipelines — they become the first real KoreGP programs, which is convenient dogfood, except where a dedicated hardware path exists (SDMA for bulk copies, CP DMA for small fills), used directly.

## 5. Architecture

* This is entirely outside of my domain expertise, I'm an above-the-API guy, so I'll let the AI speak its opinion and smarter people can weigh in. A great way to get to the truth is to post something wrong online because the people who know better can't resist correcting you. -Xenon*

```
mesa/src/
├── compiler/          # NIR, spirv_to_nir              reused unchanged
├── amd/compiler/      # ACO                             reused unchanged
├── amd/common/        # ac_surface, addrlib, ac_* regs  reused unchanged
├── amd/vulkan/        # RADV                            source of extracted code
├── vulkan/wsi/        # window system integration       reused via shim (5.6)
└── koregp/                                              NEW
    ├── kgp_public.h       # the API, C99, ~150 declarations
    ├── kgp_device.c       # device/queue, winsys init
    ├── kgp_memory.c       # kgpMalloc/kgpFree, VA management
    ├── kgp_texture.c      # ac_surface layouts, descriptor blob writers
    ├── kgp_pipeline.c     # SPIR-V → NIR → ACO, fixed root conventions
    ├── kgp_cmd.c          # recording, PM4 emission
    ├── kgp_barrier.c      # stage+hazard → flush packets
    ├── kgp_sync.c         # semaphores → drm syncobj
    ├── kgp_present.c      # swapchain shim
    └── backends/amd/      # RDNA2+ packet emission, extracted from RADV
```

Extraction means copying a function and cutting its Vulkan-object plumbing, not forking files wholesale. Every `kgp_*` file should remain diffable against its RADV ancestor, both for review and for tracking upstream fixes.

### 5.1 Memory

`kgpMalloc(dev, size, align, type)` allocates a GEM buffer object through the amdgpu winsys, assigns a GPU virtual address from a per-device allocator, maps it, and for the default memory type also CPU-maps the BO and hands back both pointers. The default type is the ReBAR heap: VRAM, host-visible, write-combined (`AMDGPU_GEM_DOMAIN_VRAM | AMDGPU_GEM_CREATE_CPU_ACCESS_REQUIRED`), falling back to GTT on machines without ReBAR, with the fallback reported as a device property. `KGP_MEMORY_GPU` is VRAM without CPU mapping, eligible for DCC. `KGP_MEMORY_READBACK` is cached GTT.

Here is the single kernel-interface fact the whole pointer-only design hangs on. The amdgpu kernel driver historically wanted a buffer-object list attached to every command submission, for residency. If that were still mandatory, every `kgpSubmit` would need a pointer→BO lookup over everything the frame touches — no_api's tree, relocated to the submit path. It isn't mandatory: per-VM always-valid registration (`AMDGPU_GEM_CREATE_VM_ALWAYS_VALID`, the same mechanism RADV uses for its global BO list and for descriptor buffers) makes every allocation resident for the process lifetime, and submits carry an empty list. Every KoreGP allocation registers this way. If this mechanism regressed or a target kernel lacked it, per-submit BO lists come back, and with them one lookup per allocation per submit. That is the failure mode to check first on any new kernel.

`kgpHostToDevicePointer` consults an interval tree of live allocations — once per allocation, cached by the user's own `{cpu, gpu}` wrapper per the blog's pattern, never on the command path. `kgpFree` defers actual destruction against the device timeline; the timeline value at free time is the only refcount.

### 5.2 Textures and the heap

`kgpTextureSizeAlign` runs addrlib (`ac_surface`) for the requested dims/format/usage, DCC metadata included. `kgpCreateTexture` allocates nothing — it computes the surface layout against a VA the caller already owns, which is the allocation/resource separation Vulkan gets backwards and Metal only fixed in Metal 2.

`kgpTextureViewDescriptor` is the function that justifies the project: RADV's descriptor-packing code from `radv_image.c`, called directly, writing the 256-bit RDNA image descriptor into a caller-visible `uint64[4]`. The user memcpys it into their heap at any index, from the CPU or from a compute shader. No pool, no set layout, no vkUpdateDescriptorSets. The heap is a `kgpMalloc` array and nothing else; `kgpSetActiveTextureHeapPtr` records the base VA the compiled shaders index from. Samplers get the same treatment with a 128-bit blob writer, plus Metal-style embedded samplers compiled from shader-side literals.

The retained-mode residue: render targets. Color and depth block programming (CB/DB registers, DCC/HTILE metadata addresses) is CPU-emitted packet state on every current GPU, so `kgpBeginRenderPass` takes `KgpTexture` handles. Aaltonen concedes this in the blog; the hardware imposes it.

### 5.3 Pipelines

Compile path: spirv_to_nir, RADV's NIR pass list minus the descriptor-set lowering passes, ACO, ISA blob uploaded to GPU-only memory. Three fixed conventions replace the pipeline-layout apparatus. Root convention: user SGPRs statically carry two 64-bit root pointers for graphics (one for compute), the heap base VA, and the stage's system values; a new ~300-line NIR pass rewrites root-struct access into SGPR-relative loads and marks the preloadable prefix, so the register-preload optimization the blog describes falls out of ACO's existing machinery. Heap convention: image ops on descriptors from the runtime array lower to RDNA image instructions taking the descriptor from SGPRs when the index is uniform, or through ACO's existing waterfall loop when NonUniform-decorated. Specialization convention: spec constants from a caller struct, extended with 64-bit pointer-valued constants patched as literals before optimization — the hardcoded-address specialization no current API can express.

No pipeline layout means PSO creation is one ACO invocation. There is deliberately no driver-side pipeline cache in v0.1; creation is cheap and cache policy belongs to applications. Depth-stencil and blend state objects compile once into register-write packet snippets; the set-state commands memcpy a snippet into the stream.

Shader authoring for v0.1 is Slang (recommended — sgpu demonstrated the ergonomics), DXC, or GLSL with BDA extensions, all producing SPIR-V 1.6 with PhysicalStorageBufferAddresses, RuntimeDescriptorArray, and Int64. A ~40-line Slang prelude ships with the SDK so user shaders read like the blog's examples. A dedicated language (KSL) is a separate future document; designing one before the driver exists would be decorating a house with no foundation.

### 5.4 Commands

Command buffers are transient by construction: recording pulls a PM4 chunk from a per-queue ring recycled on timeline retirement, and there is no reset or reuse API to misuse. Recording keeps almost no shadow state — the bound pipeline and three state snippets. `kgpDispatch` is a SET_SH_REG for the root pointer plus DISPATCH_DIRECT. `kgpDrawIndexedInstanced` is root SGPRs for both stages, index base/type from the VA, DRAW_INDEX_2. Indirect variants swap in the *_INDIRECT packets — and because root arguments are just SGPR values, GPU-generated root data (which no PC API supports; the blog calls this out) works via indirect SGPR loads on GFX10.3+. Submission is one amdgpu CS ioctl, chained IBs, empty BO list, syncobj arrays.

### 5.5 Barriers

`kgpBarrier(cb, src, dst, hazards)` compiles the stage pair to the minimal packet sequence for the ASIC, extracted from RADV's flush logic but keyed on the enum instead of reconstructed from a resource list. Compute→compute: L0/K$ invalidate plus CS_PARTIAL_FLUSH. Raster-out→pixel adds the CB/DB flush and wait. `KGP_HAZARD_DRAW_ARGUMENTS` adds the command-processor prefetch stall Vulkan derives from INDIRECT_ARGUMENT transitions; `KGP_HAZARD_DESCRIPTORS` invalidates sampler descriptor caches; `KGP_HAZARD_DEPTH_STENCIL` handles compute-written depth. Split barriers emit RELEASE_MEM and WAIT_REG_MEM against a raw user VA with the caller's atomic op and compare — no event objects. RDNA2+ samplers and the display engine read DCC directly, so there are no layout transitions anywhere in the implementation. On this hardware, the blog's central barrier claim is simply true, and the code will be short enough to prove it by inspection.

### 5.6 Presentation

The wart, quarantined: Mesa's window-system integration speaks Vulkan, and reimplementing Wayland/X11 presentation protocols is not a good use of anyone's year. `kgp_present.c` keeps a hidden minimal RADV Vulkan device alive solely to own a VkSwapchainKHR, sharing swapchain images into KoreGP through their underlying BOs and modifiers — same codebase, the handle crosses cleanly. About 800 lines, invisible in the API, replaceable later with direct wsi_common use or KMS. It is the only Vulkan dependency in the stack, and an audit rule enforces that no KoreGP object outside this file touches it. It's the best I can come up with right now unless someone else has a brilliant brainwave.

## 6. The API

*Mostly AI, it's just renaming Sebatian's API entrypoints. -Xenon*


Entry points follow the blog's prototype with `kgp` prefixes. Two deviations from the blog: descriptor writers take an out-pointer instead of returning a 256-bit struct by value (keeps the header clean for FFI), and every call takes an explicit `KgpDevice` (multi-GPU is real; the parameter costs nothing). Representative subset:

```c
void*  kgpMalloc(KgpDevice, size_t bytes, size_t align, KGP_MEMORY type);
void   kgpFree(KgpDevice, void* ptr);
void*  kgpHostToDevicePointer(KgpDevice, void* cpuPtr);

KgpTexture kgpCreateTexture(KgpDevice, const KgpTextureDesc*, void* gpuPtr);
void   kgpTextureViewDescriptor(KgpTexture, const KgpViewDesc*, KgpDescriptor256* out);

KgpPipeline kgpCreateComputePipeline(KgpDevice, KgpByteSpan spirv, const void* constants);
KgpPipeline kgpCreateGraphicsPipeline(KgpDevice, KgpByteSpan vs, KgpByteSpan ps,
                                      const KgpRasterDesc*, const void* constants);

KgpCommandBuffer kgpStartCommandRecording(KgpQueue);
void kgpBarrier(KgpCommandBuffer, KGP_STAGE src, KGP_STAGE dst, KGP_HAZARD flags);
void kgpDispatch(KgpCommandBuffer, void* dataGpu, KgpUvec3 groups);
void kgpDrawIndexedInstanced(KgpCommandBuffer, void* vsData, void* psData,
                             void* indices, uint32_t idxCount, uint32_t instCount);
void kgpSubmit(KgpQueue, KgpCommandBuffer const*, uint32_t n,
               KgpSemaphore signal, uint64_t value);
```

The full header — enums, descs, the remaining draw/copy/sync commands — tracks the blog's 150-line prototype closely enough that reproducing it here adds nothing.

## 7. Validation and tooling

An API whose failure mode is a GPU page fault needs a validation framework before it needs a logo (no I haven't made a logo yet). `KGP_DEBUG=1` interposes the dispatch table: allocation liveness on every pointer argument (the interval tree, consulted only in debug), root-pointer alignment, heap-index bounds against the active heap size, stage-mask sanity, and submit-time detection of freed-but-referenced VAs. Debug shaders get NIR-injected bounds checks on heap indexing, borrowing Mesa's existing robustness lowering.

Profiling comes cheap: Radeon GPU Profiler consumes SQTT streams the backend can emit with the same hooks RADV uses, so RGP works nearly from day one. Frame capture is harder — teaching RenderDoc a new API is unrealistic short-term — so the debug build records a KoreGP-level capture (command stream, referenced memory ranges, VA layout) replayable deterministically using amdgpu's fixed-VA replay allocation, with a standalone replayer. RGP first, capture second, interactive debugger someday.

In a perfect world we could replicate the great validation and debug layers Vulkan has. And, it's Mesa, we CAN do that, it just takes more time.

## 8. Milestones

M0: compute-only bring-up. Device, queues, `kgpMalloc` on ReBAR, SPIR-V→ACO with the root convention, dispatch, compute barriers, timeline semaphores, headless. Exit test: saxpy and a prefix sum at dispatch-cost parity with RADV, empty-BO-list submission confirmed on a real kernel.

M1: textures. Surface layouts, blob writers, the heap convention in the compiler, SDMA copy-to-texture, `KGP_HAZARD_DESCRIPTORS`. Exit test: a stress case writing descriptors from a compute shader and sampling them next dispatch.

M2: raster. Graphics pipelines through NGG, render passes, state objects, draws, indirect draws, the WSI shim. Exit test: the no_api example set ported and running natively, plus a GPU-driven multi-draw demo with per-draw root data.

M3: mesh shaders, multi-draw-indirect, split barriers, the validation layer, RGP hooks, SDK packaging.

M4: a second backend. NVK internals (NAK compiler, nouveau winsys) is the hard core test of whether the conventions are vendor-neutral or two RDNAs in a trenchcoat. Turnip after that would test the blog's tiler claims, blend baked into the PSO, framebuffer fetch, on hardware that actually behaves that way.

## 9. Distribution and the portability backend

Mesa builds with Meson, and there's no reason to change that. The developer-facing SDK is `kgp_public.h`, `libkoregp.so`, the Slang prelude, and the validation layer, packaged independently of a Mesa build, resolved at runtime through a small ICD loader (Vulkan's loader design, sized down). Loon_gpu's build discipline is the model here: self-contained, dependencies fetched or vendored, a CMake config so consumers write `find_package(koregp CONFIG)` and link `koregp::koregp`, plus a vcpkg port. Debugger visualizers (natvis, LLDB) for the handle types ship in the SDK.

Applications built against KoreGP should not be Mesa-only, so a second ICD, `libkoregp_vk.so`, implements the same header as a layer over any Vulkan 1.3 driver exposing loon_gpu's four-feature baseline: dynamic rendering, buffer device addresses, timeline semaphores, descriptor indexing. This is sgpu's architecture rebuilt in C against the KoreGP header, and it exists so the native path can be uncompromising, the fallback absorbs the portability burden, at the layered cost the fallback's users already accept on, say, Nvidia's proprietary driver. This really is basically Loon in a trenchcoat, wearing sunglasses. Loon has done good work, and should be recognized. It might be possible to make a Loon based on OpenGL 4.5 as well, and Loon already knows how to handle Metal as a backend. This would provide a great way to benchmark the performance gains of KoreGP versus Loon.

## 10. Risks and don't-do-it

Yeah, all kinds of risks all over the place.

What is interesting is that an initial prototype of this could probably actually be "vibe-coded" from Loon and Mesa by tag-teaming coding agentic models like Claude (Code) and OpenAI (Codex), maybe with Qwen Coder in the peanut gallery. Would the result be worth keeping around, at least in part? Maybe. It could prove the central tenet early on, that the existence of a driver-based API bypassing Vulkan or Metal, can actually work, and can provide performance benefits (versus Loon). If that was proven practical and performant, then maybe the project can be taken more seriously. 

When I first read Sebastian's blog post, I actually considered asking Claude Code to make essentially what Loon turned out to be, just to see if the idea had practical merit. Loon did the heavy lifting and proved it already. This is just the next logical stage.

## The Future ('s so bright, I gotta wear kgp_shades)

If this worked, then there would be a rational reason for an organization like Khronos to actually legitimize KoreGP by adopting it and promoting development of vendor-optimized official first-party drivers from NVidia, AMD, Intel, Qualcomm, MediaTek, etc. That's a long ways out, but since Sebastian showed us the Emperor has no clothes, we might as well get right into building a nudist camp. The simplicity and performance of a KoreGP driver, compared to the monstrosity of modern OpenGL/Vulkan/DirectX driver codebases also promotes it as a target for embedded and safety-critical domains. Fewer lines of code means fewer places for bugs.


