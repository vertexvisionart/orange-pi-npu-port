# 05 — Userspace runtime bringup

## Components

The vendor userspace stack consists of:

- `libnoe` — C++ NPU runtime, talks to `/dev/aipu`
- `NOE_Engine` — Python wrapper, `EngineInfer(...)` / `forward()`
- `onnxruntime_zhouyi` — ONNX Runtime build with a Zhouyi EP
- `ZhouyiOperators_x2` — operator pack for the Zhouyi EP
- `.cix` — vendor's compiled graph format (binary, not ONNX)

Wheels available (all `aarch64`):

- `libnoe-2.0.0-py3-none-manylinux2014_aarch64.whl` (any cpy)
- `NOE_Engine-2.0.0-py3-none-manylinux2014_aarch64.whl` (any cpy)
- `onnxruntime_zhouyi-1.20.0-cp311-cp311-linux_aarch64.whl` (cp311 hard pin)
- `ZhouyiOperators_x2-25.4.23-py3-none-any.whl` (any)

## Python version

ALT Linux 11.1 ships Python 3.12 by default. `libnoe` and
`NOE_Engine` are `manylinux2014_aarch64`, no cpy tag — they install
fine under 3.12. Only `onnxruntime_zhouyi` is hard-pinned to
cp311.

For this bringup the NPU path goes through `NOE_Engine` directly,
so cp312 is sufficient. If you need ORT, install Python 3.11
alongside and use a separate venv for the ORT path.

## Bringup commands

```bash
mkdir -p /opt/cix-runtime/{lib,wheels,models}
# (place vendor lib*.so and wheels under /opt/cix-runtime — these are
# vendor binaries, not redistributed here)

python3 -m venv /opt/cix-runtime/venv312
. /opt/cix-runtime/venv312/bin/activate

python -m pip install --upgrade pip setuptools wheel
python -m pip install /opt/cix-runtime/wheels/libnoe-*.whl
python -m pip install /opt/cix-runtime/wheels/NOE_Engine-*.whl
python -m pip install numpy torch tqdm matplotlib streamlit pillow

export LD_LIBRARY_PATH=/opt/cix-runtime/lib:$LD_LIBRARY_PATH
```

## Smoke test — driver visible from userspace

```python
import os
fd = os.open('/dev/aipu', os.O_RDWR)
print(fd)
os.close(fd)
```

If this raises, the kernel side is wrong. If it prints an FD, the
device node and permissions are right.

## Smoke test — graph load

```python
from NOE_Engine import EngineInfer

for model_path in [
    "/opt/cix-runtime/models/sd-demo-streamlit/encoder.cix",
    "/opt/cix-runtime/models/sd-demo-streamlit/unet.cix",
    "/opt/cix-runtime/models/sd-demo-streamlit/decoder.cix",
]:
    print("loading:", model_path)
    engine = EngineInfer(model_path)
    print("loaded successfully")
    engine.clean()
```

A clean load + `clean()` cycle here confirms the runtime is talking
to the driver and graphs are valid for this hardware.

## Smoke test — forward()

Input shapes for the SD-style pipeline graphs (read from each `.cix`
tensor descriptor):

- `unet.cix`:
  - `x`: `(2, 4, 64, 64)` float32
  - `t`: `(1,)` float32
  - `c`: `(2, 77, 768)` float32
- `encoder.cix`:
  - tokens: `(77,)` int32
- `decoder.cix`:
  - latent: `(1, 4, 64, 64)` float32

```python
import numpy as np
from NOE_Engine import EngineInfer

engine = EngineInfer("/opt/cix-runtime/models/sd-demo-streamlit/unet.cix")
x = np.zeros((2, 4, 64, 64), dtype=np.float32)
t = np.array([1.0], dtype=np.float32)
c = np.zeros((2, 77, 768), dtype=np.float32)
out = engine.forward([x, t, c])
print("outputs:", len(out))
print("output[0] elems:", out[0].size)
engine.clean()
```

If the cluster-core enable from `03_cluster_cores.md` is missing,
the load step succeeds but `forward()` blocks/fails with
`the NPU hw is not available now`. If the runtime PM fix from
`04_pm_runtime.md` is missing, `forward()` may succeed once and then
flap.
