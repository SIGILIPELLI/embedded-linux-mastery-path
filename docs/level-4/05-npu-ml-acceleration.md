# 05 · NPU & ML Acceleration (eIQ)

Running an ML model on an embedded SoC's CPU is usually a nonstarter for
anything real-time — object detection at even modest resolution can take
hundreds of milliseconds on an ARM Cortex-A core. The NPU (i.MX8M
Plus/i.MX9's integrated neural processing unit, accessed through NXP's
eIQ software stack) exists to run the same model in single-digit
milliseconds, but only if the model, the toolchain, and the runtime all
agree on what the NPU can actually execute.

## The pipeline: from trained model to running inference

```
┌───────────────┐   ┌──────────────┐   ┌────────────────┐   ┌──────────┐
│ Trained model  │──▶│ Quantization │──▶│ NPU-compatible  │──▶│ Delegate  │
│ (TF/PyTorch,   │   │ (INT8, per-  │   │ .tflite / ONNX │   │ (VX/NPU)  │
│  FP32)         │   │  channel)    │   │                │   │  runtime  │
└───────────────┘   └──────────────┘   └────────────────┘   └──────────┘
```

Every stage can silently degrade accuracy or fall back to CPU execution
without raising an obvious error — this module is mostly about detecting
those silent failures.

## Quantization: the step that determines whether the NPU can run it at all

The i.MX8M Plus NPU executes **INT8** operations; a model that stays FP32
either doesn't run on the NPU at all or runs a subset of ops there with
the rest falling back to CPU. Post-training quantization with a
representative dataset is the standard path:

```python
import tensorflow as tf

def representative_dataset():
    for data in calibration_images[:200]:
        yield [data.astype("float32")]

converter = tf.lite.TFLiteConverter.from_saved_model("saved_model/")
converter.optimizations = [tf.lite.Optimize.DEFAULT]
converter.representative_dataset = representative_dataset
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
converter.inference_input_type = tf.int8
converter.inference_output_type = tf.int8
tflite_model = converter.convert()

with open("model_int8.tflite", "wb") as f:
    f.write(tflite_model)
```

**Trap**: a representative dataset that doesn't actually represent
production input distribution (calibrated on daytime images, deployed
against a camera that also sees low-light frames) produces a model that
quantizes "successfully" — it converts with no errors — but loses far
more accuracy in the deployed distribution than the calibration accuracy
numbers suggested. The conversion succeeding is not evidence the
quantization was good; only an accuracy comparison against a held-out set
drawn from real deployment conditions tells you that.

## Checking what actually landed on the NPU

```console
$ python3 -c "
import tflite_runtime.interpreter as tflite
interp = tflite.Interpreter(
    model_path='model_int8.tflite',
    experimental_delegates=[tflite.load_delegate('libvx_delegate.so')])
interp.allocate_tensors()
"
VX_DELEGATE: allocate tensors
VX_DELEGATE: unsupported op RESIZE_BILINEAR, falling back to CPU
VX_DELEGATE: unsupported op FULLY_CONNECTED (dynamic shape), falling back to CPU
```

That delegate log is the single most important artifact in this entire
module. A model report saying "compiled successfully" tells you nothing
about performance; the per-op delegate placement log tells you exactly
which ops execute on the NPU and which silently fall back to CPU. A model
that's 95% NPU-resident by op count but has one CPU-fallback op sitting
between two NPU stages can lose most of its speed advantage to the
data-marshaling overhead of bouncing between NPU and CPU memory domains
for that one op — the fix is usually restructuring the model graph to
avoid the unsupported op, not just accepting the fallback.

## Benchmarking correctly

```console
$ ./benchmark_model --graph=model_int8.tflite \
    --external_delegate_path=libvx_delegate.so \
    --num_runs=100 --warmup_runs=10
Inference timings in us: Init: 421882, First inference: 18204, Warmup (avg): 4102, Inference (avg): 3891
```

**Always discard the first several inferences** from any latency claim —
NPU driver/firmware initialization, delegate graph compilation, and
memory-map setup happen lazily on first use, and "first inference" being
4-5x the steady-state average is normal and expected on this class of
hardware, not a sign of a problem. Report steady-state average and,
ideally, tail latency (p99) — a real-time perception pipeline cares about
the worst case, not the average, exactly as in Module 6's PREEMPT_RT
material.

## Memory and zero-copy: where a "fast" model still bottlenecks

```c
/* NNAPI/VX-style zero-copy input, avoiding a memcpy per frame */
VxDelegate_SetTensorBuffer(interp, input_tensor_idx,
			    dma_buf_fd, VX_MEMORY_TYPE_DMA_BUF);
```

A pipeline that captures a camera frame into a DMA-BUF (Module 7's
graphics stack territory) but then copies it into a plain heap buffer
before handing it to the NPU delegate pays a full frame-sized memcpy on
every inference — invisible in the delegate's own inference-time
benchmark (which only measures the NPU compute, not the copy before it),
but very visible in end-to-end pipeline latency. Profiling the model
alone and profiling the full camera-to-inference-result pipeline can
disagree by a wide margin for exactly this reason.

## Model updates and versioning in the field

Treat a deployed `.tflite`/ONNX model file as a versioned artifact with
the same rigor as firmware — it directly affects product behavior and
safety-relevant decisions in some product categories:

```console
$ sha256sum model_int8.tflite
a3f9c21e8b4d5f0e1a2b3c4d5e6f7089abcdef0123456789abcdef012345678  model_int8.tflite
$ cat model_manifest.json
{"model_version": "2.3.1", "sha256": "a3f9c21e...", "min_runtime": "eiq-2024.1", "trained_on": "dataset-v14"}
```

A model shipped over the same OTA mechanism as Module 3 must go through
the same A/B/rollback discipline — a model update that regresses accuracy
in a way that isn't caught by a "did it load" health check is exactly the
kind of failure Module 3's mark-good discipline is meant to catch, applied
here to model quality metrics instead of boot success.

## Traps

- **Trusting "conversion succeeded" as evidence of acceptable accuracy.**
  It is not — only a held-out accuracy comparison against real deployment
  data conditions is.
- **Not reading the per-op delegate placement log** and assuming "used
  the VX delegate" means "ran entirely on the NPU."
- **Benchmarking the model in isolation** and reporting that number as
  the product's end-to-end latency, ignoring capture/copy/post-processing
  overhead that can dominate the actual user-visible delay.
- **Including cold-start latency in a steady-state performance claim** —
  or the reverse, promising cold-start latency the hardware can't deliver
  because you only ever measured warmed-up runs.

## Cheat sheet

| Step | What to check |
|---|---|
| Quantize (INT8, representative dataset) | Calibration data matches real deployment distribution |
| Convert to `.tflite`/ONNX | Conversion succeeding is not an accuracy guarantee |
| Load with NPU delegate | Read the per-op fallback log, not just "delegate loaded" |
| Benchmark | Discard warmup runs; report steady-state avg and p99 |
| Full pipeline | Profile capture→inference→postprocess, not the model alone |
| Field model updates | Version, hash, and roll back exactly like firmware |

!!! note "On verification"
    The eIQ/TFLite quantization and delegate workflow, and the
    representative-dataset calibration pattern, are checked against
    documented TensorFlow Lite and NXP eIQ conventions; no model was
    actually trained, quantized, or run against real or emulated NPU
    hardware from this machine — output shown is illustrative.

## Exercise

(1) Given a delegate placement log showing 40 NPU-resident ops and 2
CPU-fallback ops sitting in the middle of the graph, explain why op count
alone doesn't tell you whether this is a performance problem, and what
additional measurement would. (2) Design the model manifest (version,
hash, minimum runtime, calibration dataset reference) you'd require before
allowing a model update to ship via OTA, and explain what health check
you'd run post-update before `mark-good`. (3) One paragraph: a colleague
reports "the NPU benchmark shows 4ms but our product feels laggy at
200ms+ per frame." List the three most likely places to look, in the
order you'd check them.
