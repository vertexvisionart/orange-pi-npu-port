# 04 — Runtime PM: keep NPU awake after probe

## Symptom

Even with the V3 ops fix and the cluster cores enabled, jobs still
flapped: a forward might succeed once and then fail, or just fail
silently. Sometimes `dmesg` showed the NPU dropping back into a
low-power state moments after probe.

## Cause

`armchina-npu/sky1/sky1.c::sky1_npu_probe()` ends with:

```c
#ifdef CONFIG_PM
    sky1_npu_pm_runtime_put(&p_dev->dev);
#endif
```

This drops the runtime PM ref the probe path took, which on this
platform causes the NPU to go to runtime suspend almost immediately.
For a freshly probed device with no userspace clients holding it
awake yet, this means cores get powered down before the first job
ever lands.

The vendor's normal stack apparently keeps the device pinned awake
through some other reference path (devfreq, SCMI hooks, runtime usage
counter from a userspace daemon). With devfreq stubbed and no
`pm_runtime_get` from userspace pre-job, nothing was holding the ref.

## Fix

Replace the `pm_runtime_put` at end of probe with a log line and
leave the device awake:

```c
#ifdef CONFIG_PM
    dev_info(&p_dev->dev,
             "skip pm_runtime_put after probe to keep NPU cores enabled\n");
#endif
```

Probe still does `pm_runtime_get_sync()` at the start, so the device
ends up with an outstanding ref that keeps it awake.

This is the minimum-change fix for bringup. The proper long-term fix
is to wire runtime suspend/resume through the userspace open/close
on `/dev/aipu` (so the device sleeps when no client is holding it,
and wakes on `open()`), or to drive it through devfreq when the SCMI
extensions land in mainline. Neither is necessary to demonstrate the
NPU works under 6.12.

## After the fix

`dmesg` after probe:

```
AIPU KMD (v5.11.0) probe start...
AIPU is behind an IOMMU
AIPU detected: zhouyi-v3
sky1_npu_probe: armchina_aipu_probe done
skip pm_runtime_put after probe to keep NPU cores enabled
```

Forward calls now run reliably — see `06_resnet_demo.md` for the
end-to-end demo and the proof-of-NPU A/B test.

## Auto-load on boot

```bash
printf 'aipu\n' > /etc/modules-load.d/aipu.conf
```

After reboot:

```bash
uname -r           # 6.12.74-6.12-alt1
lsmod | grep aipu  # aipu loaded
ls -l /dev/aipu    # device node present
```
