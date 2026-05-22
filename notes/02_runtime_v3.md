# 02 — Hardware version detection (Zhouyi V3)

## Symptom

After fixing kernel-API drift and the vermagic mismatch, `aipu.ko`
loaded but probe failed with:

```
unidentified hardware version number: 5
```

## Cause

`zhouyi_detect_aipu_version()` reads `version=5`, which corresponds to
`AIPU_ISA_VERSION_ZHOUYI_V3`. The vendor probe path in `aipu_priv.c`
matches against `V3_1` and other variants but the runtime ops table
gets selected through a chain that, for this build, doesn't pick
`get_v3_priv_ops()` for plain V3.

Result: `aipu->ops` stays NULL, probe takes the
`unidentified hardware version` error path.

## Fix

Force `aipu->ops = get_v3_priv_ops()` for `AIPU_ISA_VERSION_ZHOUYI_V3`.
This build is hard-pinned to V3 anyway via
`BUILD_AIPU_VERSION_KMD=BUILD_ZHOUYI_V3`.

`armchina-npu/aipu_priv.c`, in `armchina_aipu_probe()` (or wherever
the version-detect block lives in this vendor revision):

```c
zhouyi_detect_aipu_version(p_dev, &version, &config, &revision);
dev_dbg(aipu->dev, "AIPU core0 ISA version %d, configuration %d\n",
        version, config);
aipu->version  = version;
aipu->revision = revision;

if (version == AIPU_ISA_VERSION_ZHOUYI_V3) {
    aipu->ops = get_v3_priv_ops();
} else {
    ret = -EINVAL;
    dev_err(aipu->dev,
            "unsupported runtime hardware version for this build: %d\n",
            version);
    goto finish;
}
```

The original block was located by anchoring on the call site
`zhouyi_detect_aipu_version(p_dev, &version, &config, &revision);`
and on the trailing error block:

```c
if (!aipu->ops) {
    ret = -EINVAL;
    dev_err(aipu->dev, "unidentified hardware version number: %d\n", version);
    goto finish;
}
```

The whole region between (inclusive of the trailing `if (!aipu->ops)`)
is replaced by the V3-only block above.

## After the fix

```
AIPU detected: zhouyi-v3
```

Probe proceeds. Next problem surfaces immediately: the cluster comes
up with zero enabled cores. Continued in `03_cluster_cores.md`.
