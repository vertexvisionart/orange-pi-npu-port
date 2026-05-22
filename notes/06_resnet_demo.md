# 06 — ResNet-50 demo: real workload, proof of NPU execution

## Goal

Run a real public model end-to-end on the NPU, on the ALT Linux 6.12
build of `aipu.ko`, and prove it's actually executing on the NPU and
not falling back to CPU.

## Demo source

Vendor's public AI Model Hub:

- GitHub:    `https://github.com/cixtech/ai_model_hub`
- ModelScope (Q3 release with real binaries):
  `https://modelscope.cn/models/cix/ai_model_hub_25_Q3`

The GitHub copy ships Git LFS pointers, not real files. The actual
`.cix` blobs and `utils/` package live on ModelScope.

```bash
mkdir -p ~/models/resnet50 && cd ~/models/resnet50
git init
git remote add origin https://www.modelscope.cn/cix/ai_model_hub_25_Q3.git
git sparse-checkout init
git sparse-checkout set models/ComputeVision/Image_Classification/onnx_resnet_v1_50
git pull origin master
```

## Rescuing a real `.cix` from a Git LFS pointer

Some installs end up with `resnet_v1_50.cix` as a 100-byte LFS
pointer file. Verify and replace:

```bash
ls -l resnet_v1_50.cix
file resnet_v1_50.cix
sed -n '1,20p' resnet_v1_50.cix
# version https://git-lfs.github.com/spec/v1
# oid sha256:<HEX>
# size <N>

OID=$(grep -E '^oid sha256:' resnet_v1_50.cix | sed 's/^oid sha256://')
REAL_CIX=$(find /path/to/real-checkout/.git/lfs/objects -type f \
            | grep "$OID" | head -n 1)
cp -f "$REAL_CIX" resnet_v1_50.cix
```

## Running it

```bash
. /opt/cix-runtime/venv312/bin/activate
export LD_LIBRARY_PATH=/opt/cix-runtime/lib:${LD_LIBRARY_PATH:-}
export PYTHONPATH=/opt/utils:${PYTHONPATH:-}
cd /opt/cix-runtime/demos/onnx_resnet_v1_50
python inference_npu.py --images test_data --model_path resnet_v1_50.cix
```

Python deps for the demo: `opencv-python`, `torchvision`, `imageio`,
`pillow`. The vendor's `utils/` (with `NOE_Engine.py`,
`image_process.py`, `tools.py`, `label/imagenet_classes.py`) goes on
`PYTHONPATH`.

## Output

```
npu: noe_init_context success
npu: noe_load_graph success
npu: noe_create_job success
npu: noe_clean_job success
npu: noe_unload_graph success
npu: noe_deinit_context success

image path : test_data/ILSVRC2012_val_00021564.JPEG -> coucal
image path : test_data/ILSVRC2012_val_00002899.JPEG -> rock python
image path : test_data/ILSVRC2012_val_00045790.JPEG -> Yorkshire terrier
image path : test_data/ILSVRC2012_val_00037133.JPEG -> ice bear
image path : test_data/ILSVRC2012_val_00024154.JPEG -> Ibizan hound
```

All five test images classify correctly.

## Proving it's the NPU, not CPU

`inference_npu.py` goes through `NOE_Engine` and a `.cix` blob. There
is no ONNX CPU path inside it (the CPU equivalent is a sibling script
`inference_onnx.py`). Concretely:

1. With `aipu.ko` loaded: demo runs to completion, classifications
   succeed, `dmesg` logs `noe_create_job success` /
   `noe_clean_job success`.
2. `rmmod aipu`. `/dev/aipu` disappears.
3. Run the same `inference_npu.py` again: it fails — `NOE_Engine`
   cannot open `/dev/aipu`.

That A/B rules out CPU fallback. The classifications come from the
NPU.

## Performance

Earlier reference run on the vendor-supplied Ubuntu kernel
(`6.6.89-cix`) timed at ~2.91 ms per ResNet-50 inference on this NPU.
Hardware is rated 28.8 TOPS. The 6.12 build is in the same regime;
exact head-to-head numbers under ALT Linux were not the goal of this
bringup.
