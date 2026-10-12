# VFR Interpolation Support Plan

## Overview

**Approach**: Approach 1 — always emit exactly `m × n` output frames (m input frames, n = `frm_rate_mul`), with each input gap `(prev, curr)` subdivided into `n` equal temporal slices. RIFE's `time_step` already spans `0→1` per gap correctly; what changes is how output PTS values are computed and how the encoder is configured.

**Key invariant**: the output video's timing faithfully mirrors the input VFR timing, just with each gap subdivided by `n`. No target framerate is assumed.

**Assumed VFR source model**: VFR content in practice is almost always produced by applying variable-speed effects (slow motion, speed ramps) to a CFR-captured source in an editor. This means large timestamp gaps between frames do not imply large motion differences — the frames are still temporally close in content, just displayed more slowly. This assumption is documented here because it directly affects the risk assessment of issues 5 and 7 below.

---

## Core Change (what the rest of the issues orbit around)

In `process_interpolation` ([src/libvideo2x.cpp](../src/libvideo2x.cpp)), after each `interpolator->interpolate()` call, override the returned frame's PTS:

```cpp
int64_t prev_pts_out = av_rescale_q(prev_frame->pts, in_time_base, out_time_base);
int64_t curr_pts_out = av_rescale_q(frame->pts,      in_time_base, out_time_base);

// Interpolated position: prev + (i+1)/n * (curr - prev), computed with av_rescale for precision
int64_t interp_pts = prev_pts_out
    + av_rescale(i + 1, curr_pts_out - prev_pts_out, proc_cfg_.frm_rate_mul);
proc_frame->pts = interp_pts;
```

The `normalize()` path already emits `av_rescale_q(in_frame->pts, in_time_base_, out_time_base_)` for the original frame — correct as-is.

In `process_frames`, do not overwrite `frame->pts` before entering the `Interpolate` path (see issue #3 below).

---

## Issues to Resolve

### Issue 1: Output time base too coarse to represent `Δt / n`

**Problem**: `av_rescale(i+1, curr_out - prev_out, n)` is integer arithmetic. If the output time base has only 1 tick per input frame (e.g., input `time_base = 1/24`, `frm_rate_mul = 4`), then `curr_out - prev_out = 1` tick, and `av_rescale(1, 1, 4) = 0`. Three of the four interpolated frames land on the same PTS. Encoders reject or silently drop duplicates; muxers may produce a corrupt file.

**Fix**: When initializing the encoder for interpolation mode, multiply the input time base denominator by `frm_rate_mul`:

```cpp
enc_ctx_->time_base = {in_tb.num, in_tb.den * frm_rate_mul};
```

This is exact, minimal, and guaranteed to provide sufficient ticks for every interpolated position. Change lives in `Encoder::init` ([src/encoder.cpp](../src/encoder.cpp)) inside the `frm_rate_mul > 0` branch.

Overflow guard: `in_tb.den` is an `int` (32-bit). Typical values are `≤ 90000`; multiplied by a realistic `frm_rate_mul` of `≤ 16` the product stays well within int range. Add a static assert or runtime check and fall back to `{1, 90000 * frm_rate_mul}` if overflow would occur.

---

### Issues 2 + 3: Session-level PTS mode selection (merged)

**Fix**: VFR mode is an explicit user opt-in flag — no auto-detection. The user passes `vfr = true` (added to `RIFEConfig` or `ProcessorConfig`) to declare that the input is variable frame rate. This sidesteps the auto-detection heuristic entirely and makes the contract unambiguous.

The flag is propagated into `process_frames` and `Encoder::init` before the encoder is initialized, governing:
- Whether `enc_ctx->framerate` is set to `avg_fps * n` (CFR, default) or `0/1` (VFR)
- Whether `recalculate_pts` is applied in `process_frames` (CFR only)
- Whether `process_interpolation` uses PTS-delta math (VFR) or `frame_idx_`-based spacing (CFR)

**CFR behavior is completely unchanged** when `vfr = false` (the default). No regression risk.

**PTS resolution in VFR mode**: use `best_effort_timestamp` as the primary source, with `pts` as fallback. FFmpeg's own decode examples normalize decoded frames with `frame->pts = frame->best_effort_timestamp` immediately after `avcodec_receive_frame()`, treating `best_effort_timestamp` as the authoritative decoded-video timestamp. `pkt_dts` is omitted — it reflects decode order, not display order, and is not appropriate for PTS delta computation. `best_effort_timestamp` survives `av_frame_clone()` (clone copies all frame properties).

```cpp
int64_t resolve_pts(const AVFrame* frame) {
    if (frame->best_effort_timestamp != AV_NOPTS_VALUE)
        return frame->best_effort_timestamp;
    if (frame->pts != AV_NOPTS_VALUE)
        return frame->pts;
    return AV_NOPTS_VALUE;
}
```

**If `resolve_pts` returns `AV_NOPTS_VALUE` during a VFR session**: log a warning and skip interpolation for that gap — write only the original frame, no interpolated frames. Clean degradation: the output remains a valid VFR file.

---

### Issue 4: Audio sync on inputs with PTS drift or discontinuity

**Analysis**: For well-formed VFR inputs, video and audio PTS are already in sync in the container; preserving video PTS maintains that sync. The only risk is inputs with broken video PTS (drift or discontinuity) where `recalculate_pts` was silently fixing video timing — but those inputs had incorrect audio sync before this change too, just less visibly.

CFR path is completely unaffected (VFR mode is opt-in).

**Fix**: In VFR mode, detect large unexpected `Δt` jumps relative to the stream's `avg_frame_rate` and emit a warning:

```
Warning: large PTS gap detected at frame N (Δt=Xms, expected ~Yms). Audio sync may be affected if input has PTS discontinuities.
```

No corrective action — just visibility. The user opted in to VFR mode and is responsible for providing a well-formed input.

---

### Issue 5: Scene detection threshold not normalized by `Δt`

**Previously identified concern**: large `Δt` gaps in VFR content would accumulate more motion and produce larger frame diffs, causing false-positive scene cuts at exactly the pairs that most need interpolation.

**Resolved by source model assumption**: under the assumed VFR model (slow-motion from CFR source), large `Δt` does not imply large inter-frame pixel difference — the frames are content-adjacent, just time-stretched. `get_frame_diff` measures a static pixel-space property of the two frames themselves, independent of how far apart they are in time. A slow-motion segment will produce the same small diff as the equivalent CFR frames would.

**Conclusion**: no normalization needed. No code change. `get_frame_diff` ([src/avutils.cpp:140](../src/avutils.cpp)) converts both frames to RGB24 and computes mean absolute pixel difference as a percentage — purely pixel-space, no motion vectors, no temporal weighting. The result is identical whether two frames are 16ms or 500ms apart. Issue fully closed.

---

### Issue 6: Use `best_effort_timestamp` for PTS delta source

**Resolved**: `frame->pts` is not guaranteed to be a reliable display-order timestamp after `avcodec_receive_frame` — FFmpeg's own decode examples immediately normalize with `frame->pts = frame->best_effort_timestamp`, which is the authoritative field set by libavcodec for decoded frames. `av_frame_clone` preserves `best_effort_timestamp` (it copies all frame properties).

`resolve_pts()` in issues 2+3 already uses `best_effort_timestamp` as the primary source with `pts` as fallback, which is correct. No additional changes needed.

---

### Issue 7: RIFE artifact risk on very large `Δt`

**Previously identified concern**: RIFE is trained on frames at typical frame rates; very large `Δt` might present motion magnitudes outside its training distribution, causing artifacts.

**Resolved by source model assumption**: under the assumed VFR model (slow-motion from CFR source), large `Δt` means the two frames are temporally close in content, not that they have large motion between them. RIFE sees the same inter-frame pixel difference it would see on the original CFR source frames — which is exactly what it was trained on. The `time_step` parameter continues to correctly span `0→1` across whatever the gap duration is.

**Conclusion**: no special handling needed for large gaps. No code change, no warning threshold. The existing `scn_det_thresh` mechanism already handles the only genuinely problematic case (actual scene cuts with large pixel differences).

---

### Issue 8: `total_frames_` accuracy for VFR containers

**Analysis**: Pre-existing limitation, not introduced by VFR support. FFmpeg documents `AVStream::nb_frames` as "number of frames if known or 0" — so 0 is a valid upstream value meaning unknown, not a bug.

**Current caller behavior** ([tools/video2x/src/video2x.cpp](../tools/video2x/src/video2x.cpp) ~line 161):
- `percentage` is correctly guarded by `total_frames > 0` — no divide-by-zero
- `frames_remaining = total_frames - processed_frames` goes negative when `total_frames == 0`, but is clamped by `std::max(..., 0)` to 0
- Net result: output reads `frame=N/0 (0.00%); remaining=00:00:00` — no crash, but `remaining=00:00:00` is actively misleading (suggests the job is almost done rather than signaling unknown total)

**Fix**: In the `total_frames == 0` branch, skip the ETA calculation and omit `remaining=` from the output line entirely, replacing it with something like `remaining=?`. This is a pre-existing bug made more visible by VFR inputs; fix it in the same PR to avoid shipping a confusing UX regression.

---

## Concrete Implementation

Issues 5, 6, and 7 require no code changes. All other issues map to specific diffs below.

---

### Step 1 — New fields and helpers

**`include/processor.h`** — add to `ProcessorConfig`:
```cpp
bool vfr = false;  // if true, use PTS-delta interpolation; requires frm_rate_mul > 0
```

**`include/encoder.h`** — add to `EncoderConfig`:
```cpp
bool vfr = false;  // if true, encoder uses framerate=0/1 and fine time base
```

**`include/video_processor.h`** — add private members to `VideoProcessor`:
```cpp
bool is_vfr_ = false;            // derived from proc_cfg_.vfr && frm_rate_mul > 0
AVRational in_time_base_{0, 1};  // dec_ctx->time_base, cached for process_interpolation
int64_t avg_in_frame_duration_ = 0;  // in input time base ticks; for large-gap warning
```

**`include/avutils.h`** — declare:
```cpp
// Returns the best available display-order PTS for a decoded frame.
// Prefers best_effort_timestamp (set by libavcodec) over pts.
// Returns AV_NOPTS_VALUE if neither field is valid.
int64_t resolve_pts(const AVFrame* frame);
```

**`src/avutils.cpp`** — implement:
```cpp
int64_t resolve_pts(const AVFrame* frame) {
    if (frame->best_effort_timestamp != AV_NOPTS_VALUE)
        return frame->best_effort_timestamp;
    if (frame->pts != AV_NOPTS_VALUE)
        return frame->pts;
    return AV_NOPTS_VALUE;
}
```

---

### Step 2 — `VideoProcessor::process()` session setup

In `process()` ([src/libvideo2x.cpp](../src/libvideo2x.cpp)), after the decoder is initialized and before the encoder is initialized:

```cpp
// Derive session VFR flag
is_vfr_ = proc_cfg_.frm_rate_mul > 0 && proc_cfg_.vfr;

if (is_vfr_) {
    // Cache input time base for use in process_interpolation
    in_time_base_ = dec_ctx->time_base;

    // Cache avg frame duration for large-gap warnings (issue 4)
    AVRational avg_fps = av_guess_frame_rate(ifmt_ctx, video_stream, nullptr);
    if (avg_fps.num > 0 && avg_fps.den > 0) {
        avg_in_frame_duration_ = av_rescale_q(1, av_inv_q(avg_fps), dec_ctx->time_base);
    }

    // Propagate to encoder config — disables recalculate_pts and enables VFR time base
    enc_cfg_.vfr = true;
    enc_cfg_.recalculate_pts = false;
}
```

---

### Step 3 — `Encoder::init()` VFR time base (issue 1)

In `Encoder::init` ([src/encoder.cpp](../src/encoder.cpp)), replace the `frm_rate_mul > 0` branch:

```cpp
if (frm_rate_mul > 0) {
    if (enc_cfg.vfr) {
        // VFR: framerate is informational only; time base must be fine enough
        // to represent (frm_rate_mul - 1) interpolated positions within each input gap.
        enc_ctx_->framerate = {0, 1};
        AVRational in_tb = dec_ctx->time_base;
        if (in_tb.den <= INT_MAX / frm_rate_mul) {
            enc_ctx_->time_base = {in_tb.num, in_tb.den * frm_rate_mul};
        } else {
            logger()->warn(
                "Time base denominator overflow ({}*{}); falling back to 1/{}",
                in_tb.den, frm_rate_mul, 90000 * frm_rate_mul
            );
            enc_ctx_->time_base = {1, 90000 * frm_rate_mul};
        }
    } else {
        // CFR: existing behavior, unchanged
        AVRational in_frame_rate = avutils::get_video_frame_rate(ifmt_ctx, in_vstream_idx);
        enc_ctx_->framerate = {in_frame_rate.num * frm_rate_mul, in_frame_rate.den};
        enc_ctx_->time_base = av_inv_q(enc_ctx_->framerate);
    }
}
```

---

### Step 4 — `process_frames()` PTS normalization

In `process_frames` ([src/libvideo2x.cpp](../src/libvideo2x.cpp)), replace the existing `recalculate_pts` block:

```cpp
// Before: if (enc_cfg_.recalculate_pts) { frame->pts = ...; }
// After:
if (is_vfr_) {
    // Normalize to best available display-order timestamp.
    // AV_NOPTS_VALUE is left in place; process_interpolation detects and skips.
    int64_t resolved = avutils::resolve_pts(frame.get());
    if (resolved != AV_NOPTS_VALUE) {
        frame->pts = resolved;
    }
} else if (enc_cfg_.recalculate_pts) {
    // CFR: existing behavior, unchanged
    frame->pts = av_rescale_q(
        frame_idx_, av_inv_q(enc_ctx->framerate), enc_ctx->time_base
    );
}
```

---

### Step 5 — `process_interpolation()` VFR PTS-delta path (core change)

In `process_interpolation` ([src/libvideo2x.cpp](../src/libvideo2x.cpp)), add a VFR branch around the interpolation loop. The scene-detection block above the loop is shared and unchanged.

```cpp
// --- after scene detection, before the interpolation loop ---

if (is_vfr_ && prev_frame != nullptr) {
    int64_t prev_pts = prev_frame->pts;
    int64_t curr_pts = frame->pts;

    if (prev_pts == AV_NOPTS_VALUE || curr_pts == AV_NOPTS_VALUE) {
        logger()->warn(
            "Unresolvable PTS at frame {}; skipping interpolation for this gap",
            frame_idx_.load()
        );
        // Fall through: loop below is skipped, original frame is still written
        goto write_original;
    }

    // Large-gap warning (issue 4)
    if (avg_in_frame_duration_ > 0) {
        int64_t delta = curr_pts - prev_pts;
        if (delta > 4 * avg_in_frame_duration_) {
            logger()->warn(
                "Large PTS gap at frame {} ({:.1f}ms, expected ~{:.1f}ms). "
                "Audio sync may be affected if input has PTS discontinuities.",
                frame_idx_.load(),
                delta * av_q2d(in_time_base_) * 1000.0,
                avg_in_frame_duration_ * av_q2d(in_time_base_) * 1000.0
            );
        }
    }

    // Convert boundary PTS to output time base once, outside the loop
    AVRational out_tb = encoder.get_encoder_context()->time_base;
    int64_t prev_pts_out = av_rescale_q(prev_pts, in_time_base_, out_tb);
    int64_t curr_pts_out = av_rescale_q(curr_pts, in_time_base_, out_tb);

    for (int i = 0; i < proc_cfg_.frm_rate_mul - 1; i++) {
        float ts = static_cast<float>(i + 1) / static_cast<float>(proc_cfg_.frm_rate_mul);

        if (!skip_frame) {
            ret = interpolator->interpolate(prev_frame.get(), frame, &proc_frame, ts);
        } else {
            ret = 0;
            proc_frame = av_frame_clone(prev_frame.get());
        }

        if (ret < 0 && ret != AVERROR(EAGAIN)) {
            av_strerror(ret, errbuf, sizeof(errbuf));
            logger()->critical("Error interpolating frame: {}", errbuf);
            return ret;
        } else if (ret == 0 && proc_frame != nullptr) {
            // Override PTS: prev + (i+1)/n * (curr - prev)
            proc_frame->pts = prev_pts_out
                + av_rescale(i + 1, curr_pts_out - prev_pts_out, proc_cfg_.frm_rate_mul);

            auto pf = std::unique_ptr<AVFrame, decltype(&avutils::av_frame_deleter)>(
                proc_frame, &avutils::av_frame_deleter
            );
            ret = write_frame(pf.get(), encoder);
            if (ret < 0) return ret;
        }
        frame_idx_.fetch_add(1);
    }
}
// CFR path: existing loop runs unchanged when !is_vfr_
// ...

write_original:
// normalize + write original frame — unchanged
```

Note: the `goto` can be replaced with a flag variable (`bool pts_valid`) if the coding style prefers it; semantics are identical.

---

### Step 6 — Progress display fix for unknown total (issue 8)

In `tools/video2x/src/video2x.cpp` (~line 172), guard the ETA block:

```cpp
// Before:
int64_t frames_remaining = total_frames - processed_frames;
double processing_rate = ...;
int time_remaining = static_cast<int>(frames_remaining / processing_rate);
time_remaining = std::max<int>(time_remaining, 0);
// ... print remaining=HH:MM:SS

// After:
if (total_frames > 0) {
    int64_t frames_remaining = total_frames - processed_frames;
    double processing_rate = time_elapsed > 0
        ? static_cast<double>(processed_frames) / time_elapsed : 0.0;
    int time_remaining = processing_rate > 0
        ? static_cast<int>(frames_remaining / processing_rate) : 0;
    time_remaining = std::max<int>(time_remaining, 0);
    auto [hr, min, sec] = calculate_time_components(time_remaining);
    std::cout << "; remaining=" << std::setw(2) << std::setfill('0') << hr << ":"
              << std::setw(2) << std::setfill('0') << min << ":"
              << std::setw(2) << std::setfill('0') << sec;
} else {
    std::cout << "; remaining=?";
}
```

---

### Files modified summary

| File | Change |
|---|---|
| `include/processor.h` | `ProcessorConfig::vfr` field |
| `include/encoder.h` | `EncoderConfig::vfr` field |
| `include/video_processor.h` | `is_vfr_`, `in_time_base_`, `avg_in_frame_duration_` members |
| `include/avutils.h` | `resolve_pts` declaration |
| `src/avutils.cpp` | `resolve_pts` implementation |
| `src/encoder.cpp` | VFR branch in `frm_rate_mul > 0` section |
| `src/libvideo2x.cpp` | `process()` session setup; `process_frames()` PTS normalization; `process_interpolation()` VFR path |
| `tools/video2x/src/video2x.cpp` | Progress display guard for `total_frames == 0` |
