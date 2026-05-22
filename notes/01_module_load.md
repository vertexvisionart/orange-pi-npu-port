# 01 — Module load, vermagic, headers

## Symptom

Even after a clean compile, the first `insmod` failed with:

```
Invalid module format
disagrees about version of symbol module_layout
```

## Cause

Two kernels were in play:

- The kernel actually running at the time: `6.12.41-6.12-alt1`
- The kernel whose headers were installed:  `6.12.74-6.12-alt1`

The module was compiled with vermagic `6.12.74-6.12-alt1` and the
running kernel was the `.41` one.

## Resolution

Two valid paths:

1. Reboot into `6.12.74-6.12-alt1` (the version matching the headers).
2. Install matching headers for `6.12.41-6.12-alt1` and rebuild.

Both worked. Final state: kernel and module both at
`6.12.74-6.12-alt1`.

## Commands used

```bash
# verify vermagic of the built module
modinfo ./aipu.ko | grep -E 'filename|name|vermagic'

# verify running kernel
uname -r

# install matching headers, if going that route
apt-get install -y kernel-headers-modules-6.12

# check what /usr/src has
ls /usr/src | grep linux

# install module to the correct path for the running kernel
mkdir -p /lib/modules/$(uname -r)/extra
cp -a /usr/src/aipu-5.11.0/aipu.ko /lib/modules/$(uname -r)/extra/aipu.ko
depmod -a

rmmod aipu 2>/dev/null || true
modprobe aipu || insmod /lib/modules/$(uname -r)/extra/aipu.ko

lsmod | grep aipu
ls -l /dev/aipu
dmesg | grep -i aipu | tail -n 100
```

## `KMD_VERSION undeclared` aside

When building `aipu.c` out-of-tree without the vendor's wrapper
script, the macro `KMD_VERSION` is never defined and the build dies.
Workaround inside `armchina-npu/aipu.c`:

```c
#include "aipu_priv.h"

#ifndef KMD_VERSION
#define KMD_VERSION "5.11.0"
#endif
```

That gives `dmesg` the version string the vendor expects:

```
AIPU KMD (v5.11.0) probe start...
```
