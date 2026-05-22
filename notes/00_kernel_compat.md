# 00 — Kernel compatibility (6.6 → 6.12)

## Context

Vendor source is `aipu-5.11.0`. It builds against the Orange Pi
custom 6.6 kernel; it does not build against stock 6.12.

Two API breaks hit immediately:

1. `platform_driver::remove` changed from `int (*)(...)` to
   `void (*)(...)` somewhere between 6.6 and 6.12.
2. The vendor relies on out-of-tree SCMI helpers
   (`scmi_device_get_freq`, `scmi_device_set_freq`,
   `scmi_device_opp_table_parse`) that are not in mainline.

## Fix 1 — `platform_driver::remove` signature

`armchina-npu/default/default.c`:

```bash
sed -i 's/static int default_remove(struct platform_device \*p_dev)/static void default_remove(struct platform_device *p_dev)/' \
    armchina-npu/default/default.c

sed -i 's/return armchina_aipu_remove(p_dev);/armchina_aipu_remove(p_dev);/' \
    armchina-npu/default/default.c
```

`armchina-npu/sky1/sky1.c`:

```bash
sed -i 's/static int sky1_npu_remove(struct platform_device \*p_dev)/static void sky1_npu_remove(struct platform_device *p_dev)/' \
    armchina-npu/sky1/sky1.c
```

And drop the dangling `return 0;` from the body. Whatever called
`armchina_aipu_remove` and used the int return needs `return` removed
too.

## Fix 2 — stub the CIX-private SCMI helpers

These three symbols are not in mainline 6.12 and would cause module
load failures (unknown symbols). Stubbed in `armchina-npu/sky1/sky1.c`:

```python
content = open('armchina-npu/sky1/sky1.c').read()

content = content.replace(
    'pre_freq = scmi_device_get_freq(cix_aipu_priv->opp_pmdomain);',
    'pre_freq = 0; /* disabled: not available in kernel 6.12 */')

content = content.replace(
    'ret = scmi_device_set_freq(cix_aipu_priv->opp_pmdomain, *freq);',
    'ret = 0; /* disabled: not available in kernel 6.12 */')

content = content.replace(
    '*freq = scmi_device_get_freq(cix_aipu_priv->opp_pmdomain);',
    '*freq = 0; /* disabled: not available in kernel 6.12 */')

content = content.replace(
    'stat->current_frequency = scmi_device_get_freq(cix_aipu_priv->opp_pmdomain);',
    'stat->current_frequency = 0; /* disabled: not available in kernel 6.12 */')

content = content.replace(
    'ret = scmi_device_opp_table_parse(cix_aipu_priv->opp_pmdomain);',
    'ret = 0; /* disabled: not available in kernel 6.12 */')

open('armchina-npu/sky1/sky1.c', 'w').write(content)
```

Combined with `BUILD_NPU_DEVFREQ=n`, this neutralizes the entire
devfreq path. NPU runs at the firmware-default operating point. Real
fix is to backport the SCMI extensions to mainline; that's a separate
piece of work.

## Build invocation

```bash
cd /usr/src/aipu-5.11.0
export AIPU_INC="-Wno-error -I$(pwd)/armchina-npu/include -I$(pwd)/armchina-npu -I$(pwd)/armchina-npu/zhouyi"

make -C /usr/src/linux-6.12.74-6.12-alt1 M="$(pwd)" clean
make -C /usr/src/linux-6.12.74-6.12-alt1 M="$(pwd)" modules \
  BUILD_AIPU_VERSION_KMD=BUILD_ZHOUYI_V3 \
  BUILD_TARGET_PLATFORM_KMD=BUILD_PLATFORM_SKY1 \
  BUILD_NPU_DEVFREQ=n \
  EXTRA_CFLAGS="$AIPU_INC"
```

Build flags:

- `BUILD_AIPU_VERSION_KMD=BUILD_ZHOUYI_V3` — actual ISA on this part
- `BUILD_TARGET_PLATFORM_KMD=BUILD_PLATFORM_SKY1` — Sky1 SoC
- `BUILD_NPU_DEVFREQ=n` — devfreq off (stubs above)
- `EXTRA_CFLAGS` — driver internal include paths

Output: `aipu.ko` linked against `6.12.74-6.12-alt1`.
