# One-Pass Upscale + Interpolate Plan

## Overview

**Goal**: let a single `video2x` invocation both upscale and frame-interpolate a video, decoding and encoding once (no intermediate file).

**Recommended order**: **upscale first, then interpolate.** This is the default and the only order this feature implements (see rationale below).

**Approach**: keep the interpolator (RIFE) as the primary processor and add the upscaler as an additive **pre-stage**. Each decoded frame is upscaled, then the upscaled frame flows into the *existing, unmodified* `process_interpolation` path. `prev_frame` naturally becomes the upscaled previous frame, so RIFE interpolates between upscaled frames. The encoder already accepts upscaled dimensions and `frm_rate_mul` together, so no encoder change is needed for the common path.

**Key invariant**: with the upscaler bypassed, behavior is byte-for-byte identical to today. The chain is opt-in: it activates only when a scaler `-p` is combined with RIFE options.

---

## Why upscale before interpolate

Researched from the space-time super-resolution literature and practitioner sources.

- **Compute (decisive, high confidence)**: with `m` input frames and factor `n`, upscale-first runs the heavy SR network on `m` frames; interpolate-first runs it on `m × n` frames. SR (Real-ESRGAN ≈ 9,300 GFLOPs for one 540p→4K frame) is ~5× heavier per output pixel than RIFE (~0.2 TFLOPs at 720p, real-time), and far heavier than that versus diffusion/transformer SR. Run the heavy stage on fewer frames; the saving grows with `n`.
- **Quality (small, contested margins)**: only three papers compare the two orders head-to-head and they split by resolution regime. STARnet (CVPR 2020) finds SR→VFI better by up to 1 dB for genuinely low-res input — video2x's typical case. FISR (AAAI 2020) and Choi & Oh (Sensors 2023) narrowly favor VFI→SR, but their inputs are already 2K. The dominant STVSR line (Zooming Slow-Mo, TMNet, VideoINR, RSTT) does not test ordering; it argues any two-stage cascade is worse than a jointly-trained model, which is not an option here.
- **Mechanism**: interpolate-first feeds synthetic, sometimes-ghosted frames into a stage whose job is to sharpen, and SR amplifies pre-existing artifacts into convincing wrong detail. RIFE also estimates flow on an internally downsampled copy, so the usual "large motion at high res" objection to upscale-first is largely neutralized.
- **Practitioner "interpolate-first is faster" advice** (common in Topaz forums) is tool-specific: Topaz Chronos is a *heavy* VFI model, so it benefits from running on fewer frames. RIFE is light, so that advice does not transfer.

Interpolate-first is therefore out of scope. It is also architecturally costlier here (every interpolated *and* original frame would need upscaling before encode, and the upscaler buffering described below complicates frame pairing). It can be added later behind an order flag if a use case appears.

---

## Current single-processor model (what we build on)

- `process_frames` ([src/libvideo2x.cpp](../src/libvideo2x.cpp)) runs exactly one processor, dispatched by `get_processing_mode()` to either `process_filtering` (1→1) or `process_interpolation` (1→N).
- `ProcessorConfig` ([include/libvideo2x/processor.h](../include/libvideo2x/processor.h)) carries one `processor_type` + one `variant config`. `VideoProcessor` takes one `ProcessorConfig`.
- The interpolation orchestration (scene detection, `prev_frame`, PTS math, writing original + interpolated frames) lives in `VideoProcessor`, **not** in `InterpolatorRIFE`. This is why the interpolator must be the *primary* processor: keeping it primary means the dispatch and `process_interpolation` stay untouched, and we only insert an upscale step ahead of them.

Two facts that make the pre-stage design clean:

1. **The encoder already supports both at once.** `Encoder::init` ([src/encoder.cpp](../src/encoder.cpp)) sets `width`/`height` from the (upscaled) output dimensions and derives the frame rate from `input_rate × frm_rate_mul`. No change needed for the happy path.
2. **`get_output_dimensions` composes.** The scaler reports `in × factor`; RIFE is identity. So output dims = scaler dims.

Two facts that need care:

3. **PTS time base.** Both `Interpolator::normalize` ([src/interpolator.cpp:34](../src/interpolator.cpp)) and `InterpolatorRIFE::interpolate` ([src/interpolator_rife.cpp:140](../src/interpolator_rife.cpp)) rescale `in_frame->pts` from **input** time base to encoder time base. But the scaler's `filter()` already rescales its output PTS to encoder time base. The pre-stage must therefore **restore the decoded frame's input-time-base PTS** onto the upscaled frame before handing it to `process_interpolation`, otherwise PTS is rescaled twice.
4. **Upscaler buffering asymmetry.** Real-ESRGAN and Real-CUGAN are synchronous 1→1 (one output per input, never `EAGAIN`, nothing held at flush). **libplacebo wraps an FFmpeg filtergraph** that may return `EAGAIN` and hold trailing frames until `flush()` ([src/filter_libplacebo.cpp](../src/filter_libplacebo.cpp)). The pre-stage must treat the upscaler as a stream of 0-or-1 frames per input and drain trailing frames at flush.

---

## Architecture

```
decode ─▶ [upscaler.filter()] ─▶ restore input-tb PTS ─▶ process_interpolation ─▶ encode
              (pre-stage)                                  (existing, unchanged)
```

- **Primary processor** = interpolator (RIFE), `ProcessingMode::Interpolate`. Dispatch and `process_interpolation` are unchanged; they simply receive upscaled frames.
- **Pre-stage** = the scaler (`Filter`), held as an optional second processor on `VideoProcessor`.
- The CLI's `-p <scaler>` is mapped into the pre-stage slot; RIFE options build the primary config. The "swap" is internal to argument parsing.

---

## Concrete implementation

### Step 1 — Carry an optional upscaler config

**`include/libvideo2x/libvideo2x.h`** — add a defaulted constructor parameter so every existing caller is unaffected:

```cpp
VideoProcessor(
    const processors::ProcessorConfig proc_cfg,
    const encoder::EncoderConfig enc_cfg,
    const uint32_t vk_device_idx = 0,
    const AVHWDeviceType hw_device_type = AV_HWDEVICE_TYPE_NONE,
    const bool benchmark = false,
    const std::optional<processors::ProcessorConfig> upscaler_cfg = std::nullopt  // pre-stage
);
```

Store `upscaler_cfg_` as a member. When it is set, `proc_cfg_` is the interpolator and `upscaler_cfg_` is the scaler.

No new fields are needed on `ProcessorConfig`: the scaler reuses the existing `processor_type` + `width/height/scaling_factor/noise_level` + variant config; the interpolator reuses `frm_rate_mul`, `scn_det_thresh`, `vfr*`, and the `RIFEConfig` variant.

### Step 2 — `process()`: create the pre-stage and size the encoder from it

In `process()` ([src/libvideo2x.cpp](../src/libvideo2x.cpp)), after creating the primary processor:

```cpp
std::unique_ptr<processors::Processor> upscaler;  // null unless chaining
if (upscaler_cfg_.has_value()) {
    upscaler = processors::ProcessorFactory::instance().create_processor(
        *upscaler_cfg_, vk_device_idx_
    );
    if (upscaler == nullptr) {
        return handle_error(-1, "Failed to create upscaler instance");
    }
}
```

Compute output dimensions from the scaler when present (it determines the final size; the interpolator is identity):

```cpp
const processors::Processor& dim_source = upscaler ? *upscaler : *processor;
const processors::ProcessorConfig& dim_cfg = upscaler ? *upscaler_cfg_ : proc_cfg_;
dim_source.get_output_dimensions(dim_cfg, dec_ctx->width, dec_ctx->height, output_width, output_height);
```

The encoder is then initialized exactly as today with `output_width/height` (upscaled) and `proc_cfg_.frm_rate_mul` (interpolation) — both already supported.

Initialize both processors with the same `(dec_ctx, enc_ctx, hw_ctx)` signature:

```cpp
if (upscaler) {
    ret = upscaler->init(dec_ctx, encoder.get_encoder_context(), hw_ctx.get());
    if (ret < 0) return handle_error(ret, "Failed to initialize upscaler");
}
```

`InterpolatorRIFE::init` captures color and time bases from `dec_ctx` independent of frame size, so it remains correct even though it will receive upscaled frames.

### Step 3 — `process_frames`: insert the upscale pre-stage

Pass the upscaler into `process_frames` (new nullable `Processor*` parameter). In the decode loop, after the existing PTS normalization and **before** the mode switch, when an upscaler is present, replace the direct call to `process_interpolation` with a helper:

```cpp
// Pre-stage: upscale, then feed the upscaled frame into the existing interpolation path.
// upscaler->filter() may yield 0 frames (EAGAIN, e.g. libplacebo latency) or 1 frame.
int VideoProcessor::upscale_then_interpolate(
    processors::Processor* upscaler,
    std::unique_ptr<processors::Processor>& interpolator,
    encoder::Encoder& encoder,
    std::unique_ptr<AVFrame, decltype(&avutils::av_frame_deleter)>& prev_frame,
    AVFrame* decoded_frame
) {
    processors::Filter* filter = static_cast<processors::Filter*>(upscaler);
    AVFrame* up = nullptr;
    int ret = filter->filter(decoded_frame, &up);
    if (ret == AVERROR(EAGAIN)) {
        return 0;  // upscaled frame will arrive later or at flush
    }
    if (ret < 0 || up == nullptr) {
        return ret < 0 ? ret : 0;
    }
    auto up_ptr = std::unique_ptr<AVFrame, decltype(&avutils::av_frame_deleter)>(
        up, &avutils::av_frame_deleter
    );
    // Restore input-time-base PTS: process_interpolation/normalize rescale in_tb -> enc_tb,
    // but filter() already rescaled to enc_tb. Copy the decoded PTS so the rescale is applied once.
    up_ptr->pts = decoded_frame->pts;
    return process_interpolation(interpolator, encoder, prev_frame, up_ptr.get());
}
```

The decode-loop dispatch becomes:

```cpp
case processors::ProcessingMode::Interpolate: {
    if (upscaler != nullptr) {
        ret = upscale_then_interpolate(upscaler, processor, encoder, prev_frame, frame.get());
    } else {
        ret = process_interpolation(processor, encoder, prev_frame, frame.get());  // unchanged
    }
    break;
}
```

`process_interpolation`, `process_filtering`, scene detection, and all VFR paths are untouched.

### Step 4 — Flush the pre-stage (libplacebo trailing frames)

In the flush section of `process_frames`, before/after flushing the primary processor, drain the upscaler and route each trailing frame through the interpolation path:

```cpp
if (upscaler != nullptr) {
    std::vector<AVFrame*> trailing;
    ret = upscaler->flush(trailing);  // default Filter::flush() returns none (ncnn scalers)
    if (ret < 0) { /* log + return */ }
    for (AVFrame* raw : trailing) {
        auto up = std::unique_ptr<AVFrame, decltype(&avutils::av_frame_deleter)>(
            raw, &avutils::av_frame_deleter
        );
        // No decoded source frame here; reconstruct input-tb PTS from the enc-tb value
        // the filter's flush already assigned. Only libplacebo reaches this path.
        up->pts = av_rescale_q(up->pts, enc_ctx->time_base, in_time_base_for_flush);
        ret = process_interpolation(processor, encoder, prev_frame, up.get());
        if (ret < 0) return ret;
    }
}
```

For Real-ESRGAN/Real-CUGAN this loop is empty (synchronous, nothing buffered). `in_time_base_for_flush` is `dec_ctx->time_base`, already cached as `in_time_base_` in the VFR setup; cache it unconditionally when chaining so it is available in CFR mode too.

### Step 5 — CLI: layer RIFE onto a scaler

In `parse_args` ([tools/video2x/src/argparse.cpp](../tools/video2x/src/argparse.cpp)):

- After parsing `-p` and the per-type config (the existing switch), detect chaining: `-p` resolved to a scaler (`libplacebo`/`realesrgan`/`realcugan`) **and** RIFE options are present (`--rife-model` together with `--frame-rate-mul >= 2` or `--vfr-min-fps`).
- When chaining, move the scaler config into a new out-parameter `upscaler_cfg`, and build the primary `proc_cfg` as a `RIFEConfig` from the RIFE flags (reusing the existing RIFE validation block).
- `-p rife` alone → interpolation only (unchanged). `-p <scaler>` with no RIFE flags → upscale only (unchanged).
- Update help text: combined usage example and a one-line note that the order is fixed (upscale → interpolate). Example:

```
video2x -i in.mp4 -o out.mp4 -p realesrgan --realesrgan-model realesr-animevideov3 -s 4 \
        --rife-model rife-v4.6 -m 2
```

`video2x.cpp` ([tools/video2x/src/video2x.cpp](../tools/video2x/src/video2x.cpp)) threads the optional `upscaler_cfg` from `parse_args` into the `VideoProcessor` constructor.

### Step 6 — Validation and guardrails

- Reject combinations that cannot work: libplacebo-as-scaler still requires `--width/--height`; RIFE still requires a rate. Reuse the existing per-type validation; just run both halves when chaining.
- Warn (don't block) on likely-OOM combos: two ncnn models resident plus RIFE running at the upscaled resolution. Mention `--rife-uhd` and that smaller `--scaling-factor` reduces VRAM.

---

## Files modified summary

| File | Change |
|---|---|
| `include/libvideo2x/libvideo2x.h` | Optional `upscaler_cfg` ctor param + member; `upscale_then_interpolate` decl; cache `in_time_base_` unconditionally when chaining |
| `src/libvideo2x.cpp` | `process()` creates pre-stage, sizes encoder from scaler, inits both; `process_frames` inserts pre-stage + flush drain; new `upscale_then_interpolate` |
| `tools/video2x/include/argparse.h` | `parse_args` gains an `upscaler_cfg` out-parameter |
| `tools/video2x/src/argparse.cpp` | Detect scaler+RIFE, split into two configs, combined validation, help text |
| `tools/video2x/src/video2x.cpp` | Thread `upscaler_cfg` into the `VideoProcessor` constructor |

No changes required: `encoder.*` (already accepts upscaled dims + `frm_rate_mul`), `processor.h` config struct, the processor/filter/interpolator implementations, `processor_factory.*`.

---

## Testing

- **Core chain**: Real-ESRGAN ×4 + RIFE ×2 on a short clip. Assert output dimensions = 4×, frame count = 2× input, PTS strictly increasing, smooth playback.
- **Buffering path**: libplacebo + RIFE. Confirm `EAGAIN` is tolerated and trailing frames from `flush()` are emitted (no dropped tail, correct count).
- **VFR**: chain under both `--vfr` (proportional) and `--vfr-min-fps`. Timing should match interpolation-only output at the same settings, just upscaled.
- **Scene cuts**: clip with a hard cut. Scene detection should still clone across the cut (now on upscaled frames) rather than blend.
- **Regression**: upscale-only and interpolate-only invocations must be byte-identical to pre-change output (the `upscaler == nullptr` path).
- **Benchmark mode**: `--benchmark` skips writing; verify the chain still runs both stages.

---

## Risks / open questions

- **VRAM**: scaler + RIFE models resident simultaneously, with interpolation at the upscaled resolution, is the main practical limit on small GPUs. Mitigations: warn at parse time, document `--rife-uhd` / lower scaling factor. A future memory-cheaper variant could tile, but is out of scope.
- **libplacebo flush PTS**: trailing frames reconstruct input-tb PTS by rescaling back from encoder tb (no decoded source frame is available). Minor rounding, only at end-of-stream, only for libplacebo. ncnn scalers never hit this path.
- **Color metadata**: the scaler's output frame must carry the decoder's color properties so RIFE and `normalize` pick the same conversion matrix. Both stages capture color from `dec_ctx`; verify the upscaled frame's color fields survive into `process_interpolation` (spot-check HDR/BT.2020 input).
- **Interpolate-first**: deliberately unimplemented. Revisit only with evidence of a real use case (very-low-res, large-motion content), behind an explicit order flag.
</content>
</invoke>
