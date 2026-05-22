# 03 — Cluster cores not enabled

## Symptom

Probe completes, `/dev/aipu` exists, userspace can `open()` it.
Loading a `.cix` graph through `NOE_Engine` works. But when an actual
job is submitted, kernel logs show:

```
configure cluster #0 done: en_core_cnt 0
the NPU hw is not available now
```

`en_core_cnt 0` — partition is configured but no cores are enabled
in it. Scheduler then refuses to dispatch.

## Cause

In `armchina-npu/zhouyi/v3.c`, `zhouyi_v3_initialize()` calls
`zhouyi_v3_set_partition()` for each cluster but does not call
`zhouyi_v3_enable_core_cnt()`. On this silicon / this firmware combo,
cores aren't enabled by default — they need an explicit enable.

## Fix

Inside `zhouyi_v3_initialize()` after partitions are assigned, walk
the partition's clusters and enable all cores:

```c
for (iter = 0; iter < partition->cluster_cnt; iter++) {
    u32 cid = partition->clusters[iter].id;
    u32 core_cnt = partition->clusters[iter].core_cnt;
    zhouyi_v3_set_partition(partition, cid);
    zhouyi_v3_enable_core_cnt(partition, cid, core_cnt);
}
```

Both helpers are already defined in `v3.c`. The vendor calls
`zhouyi_v3_enable_core_cnt()` from a different code path (probably
exposed through the SCMI/devfreq glue that we stubbed in
`00_kernel_compat.md`), so on a no-devfreq build the explicit call is
needed here.

## After the fix

Cluster comes up with the expected number of cores enabled, scheduler
accepts jobs, real `forward()` calls land on the NPU. Without this
fix the graph loads but every forward fails with
`the NPU hw is not available now`.

This was the second of three "looks loaded, isn't actually working"
issues. The third — runtime PM — is in `04_pm_runtime.md`.
