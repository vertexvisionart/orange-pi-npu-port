# Orange Pi 6 Plus — NPU mainline kernel port

Porting the CIX Sky1 (Zhouyi V3, 28.8 TOPS) NPU driver from Linux 6.6 to
mainline 6.12 on ALT Linux. Driver loads, `/dev/aipu` comes up, and real
inference runs end-to-end: ResNet-50 image classification plus the three
graphs of a Stable-Diffusion-style pipeline (text encoder, UNet, VAE
decoder).

## Hardware

- Board: Orange Pi 6 Plus (CIX CD8160 SoC, 12 ARM cores)
- NPU: CIX Sky1 / Arm China Zhouyi V3, rated 28.8 TOPS
- RAM: 32 GB LPDDR5
- Storage: 128 GB NVMe + 64 GB SD

## Problem

The vendor ships the NPU driver (`aipu.ko`, KMD 5.11.0) pinned to a
custom Linux 6.6 tree. The target distro for deployment is ALT Linux
(included in the Russian software registry, required for legal use in
that environment), which ships stock kernel 6.12. Out of the box the
NPU is invisible: the prebuilt `.ko` won't load against a 6.12
`Module.symvers`, and `aipu.ko` rebuilt as-is fails compilation against
the 6.12 platform-driver and PM APIs. Mainline status of the Zhouyi
upstream submission was still "Planning" at the time.

So: take the vendor source, make it compile and load against an
unmodified 6.12 kernel, and prove it can actually execute a graph.

## Approach

Three orthogonal classes of fixes were needed.

### Kernel API drift, 6.6 → 6.12

- `platform_driver::remove` switched from `int (*)(...)` to
  `void (*)(...)`. Both `default.c` and `sky1.c` had to lose their
  `return 0;` and adjust the prototype.
- `armchina_aipu_remove`'s `int` return became dead and was removed at
  the call sites accordingly.
- A handful of `KMD_VERSION undeclared` failures from out-of-tree
  builds — fixed with a fallback `#define KMD_VERSION "5.11.0"` in
  `aipu.c` so manual `make M=` works without the vendor's wrapper.

### CIX-private SCMI helpers that mainline doesn't have

The vendor tree relies on a handful of CIX-private SCMI hooks for
NPU devfreq:

- `scmi_device_get_freq`
- `scmi_device_set_freq`
- `scmi_device_opp_table_parse`

None of those exist in mainline 6.12. Two options: backport the SCMI
extensions (out of scope) or stub the devfreq path. Stubbed it and
built with `BUILD_NPU_DEVFREQ=n` — the NPU runs at the firmware-default
operating point, which is fine for bringing it up. Frequency scaling
is a separate workstream.

### Runtime bringup — the part that actually matters

Once the module compiled and loaded against 6.12, three real bugs
surfaced that had nothing to do with the kernel jump:

1. **Hardware version misidentification.** `zhouyi_detect_aipu_version`
   returned `5` (`AIPU_ISA_VERSION_ZHOUYI_V3`). The vendor probe path
   matched on `V3_1` and bailed with `unidentified hardware version
   number: 5`. Forced the runtime ops table to `get_v3_priv_ops()` for
   `AIPU_ISA_VERSION_ZHOUYI_V3` so probe completes on this silicon.
2. **Clusters discovered, cores never enabled.** `dmesg` showed
   `configure cluster #0 done: en_core_cnt 0` and the scheduler then
   reported `the NPU hw is not available now`. The vendor's
   `zhouyi_v3_initialize()` set partitions but didn't call
   `zhouyi_v3_enable_core_cnt()`. Added the explicit per-cluster
   enable in the init loop.
3. **NPU sleeping immediately after probe.** `sky1_npu_probe()` called
   `sky1_npu_pm_runtime_put()` right after `armchina_aipu_probe()`,
   which dropped the runtime PM ref and put the NPU back to sleep
   before any userspace job could land. Removed the early
   `pm_runtime_put` so the device stays awake. Idle suspend can be
   reintroduced later with proper resume handling.

After all three the probe sequence is clean:

```
AIPU KMD (v5.11.0) probe start...
AIPU is behind an IOMMU
AIPU detected: zhouyi-v3
sky1_npu_probe: armchina_aipu_probe done
skip pm_runtime_put after probe to keep NPU cores enabled
```

### Userspace bringup

The vendor runtime (`libnoe`, `NOE_Engine`, `onnxruntime_zhouyi`,
`ZhouyiOperators_x2`) ships only as `aarch64` wheels for cp311. ALT
Linux 11.1 ships Python 3.12. Verified the `manylinux2014_aarch64`
wheels for `libnoe`/`NOE_Engine` work under 3.12; only the
`onnxruntime_zhouyi` wheel is hard-pinned to cp311, which is
acceptable because the inference path used here goes through
`NOE_Engine` directly.

## Outcome

End-to-end inference confirmed on stock ALT Linux 6.12.74-6.12-alt1:

- `aipu.ko` builds cleanly against `kernel-headers-modules-6.12`.
- Module auto-loads on boot, `/dev/aipu` is created, opens cleanly
  from userspace.
- `EngineInfer` loads `.cix` graphs for the Stable-Diffusion-style
  pipeline (`encoder.cix`, `unet.cix`, `decoder.cix`); `forward()`
  produces the expected output shapes.
- `inference_npu.py` on the public ResNet-v1-50 demo runs to
  completion and classifies the ImageNet validation set images
  correctly (`coucal`, `rock python`, `Yorkshire terrier`,
  `ice bear`, `Ibizan hound`).
- 28.8 TOPS hardware confirmed working through the kernel path.

A/B isolation rules out CPU fallback: with `aipu.ko` loaded the demo
runs; after `rmmod aipu`, the same `inference_npu.py` invocation
fails (the script links through `NOE_Engine` and a `.cix` blob — no
ONNX CPU path is reached).

## What this took, in retrospect

- The 6.6 → 6.12 surface area for an out-of-tree platform driver is
  small and concentrated in `platform_driver` and PM hooks. The
  port-the-kernel-API part was the easy part.
- The hard part was post-probe runtime: cluster cores not enabled
  and a stray `pm_runtime_put` after probe. Both are silent in the
  sense that the module loads and the device node appears, but
  every job submission fails. Bringup of an embedded NPU is
  bookended by power management state, and getting that wrong is
  invisible from `lsmod`.
- Vendor SCMI extensions (`scmi_device_get_freq` and friends) are
  the cleanest example of why "just rebuild against the new kernel"
  isn't enough for a vendor BSP — half the dependencies are not in
  mainline at all. Devfreq is the right thing to backport properly
  if frequency scaling matters.
- A `.cix` "model" in the vendor SDK is a binary graph; getting an
  honest demo running also meant rescuing the real artifact from a
  Git LFS pointer file (`oid sha256:...`) by walking the LFS object
  store on a working install.

## Notes

`notes/` contains selected command logs and debugging snippets from
the bringup, lightly edited to translate Russian commentary and to
strip internal references. They're organized by phase:

- `00_kernel_compat.md` — initial 6.6 → 6.12 build fixes
- `01_module_load.md` — `vermagic`, headers, kernel match
- `02_runtime_v3.md` — hardware-version detection fix
- `03_cluster_cores.md` — explicit core enable in `v3.c`
- `04_pm_runtime.md` — keep NPU awake after probe
- `05_userspace.md` — `NOE_Engine`, wheels, Python 3.12 bringup
- `06_resnet_demo.md` — public ResNet-50 demo bringup and proof

## Status

This is a writeup of work performed under a paid engagement.
The full vendor driver source is not redistributed here — the patches
above are described at the source-level and referenced against the
vendor's `aipu-5.11.0` tree, which the reader is expected to have
their own access to. The notes document the engineer's process and
findings.

## License

MIT for this writeup. See [LICENSE](LICENSE). The vendor driver and
runtime are not covered by this license.
