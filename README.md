# libframeutil
Some DMD frame utilities used by libzedmd, libdmdutil and libserum

Header-only: everything lives in `include/FrameUtil.h`.

## Frame scaling

`FrameUtil::ScalingAlgorithm` is the shared selector for how a frame is scaled
up:

- `Scale2x` (`0`, **default**) — Scale2x / AdvMAME2x edge-preserving pixel-art
  upscaling
- `LineDoubling` (`1`) — replicate every source pixel into an NxN block

Scale2x is the zero value on purpose: anything that has not made an explicit
choice resolves to it, which is the look this stack has always produced for
upscaled frames. Line doubling is an opt-out.

The numeric values are persisted in libserum's `cROMc` header and exposed
through its public C API as `SERUM_SCALING_*`. **Never renumber them**; new
algorithms append.

Two equivalent forms are provided, and they must always produce identical
output:

- `ScaleUpBy(algorithm, dst, src, w, h, bits)` / `ScaleUpIndexedBy(...)` —
  whole-frame. The canonical entry points; prefer these over calling `ScaleUp`,
  `Scale2XIndexed` or `ScaleDouble*` directly, so a new algorithm only has to be
  wired up in one place.
- `SampleUpscaled2x<T>(src, w, h, destX, destY, algorithm)` — per-pixel,
  templated over the pixel type. For callers that need only some destination
  pixels, whose source frame changes between nested render passes, or whose
  pixels are not `uint8_t`. libserum uses it to sample both 8-bit shade indices
  and 16-bit RGB565.

The per-pixel form is inherently 2x only: Scale4x is Scale2x applied twice and
needs the intermediate frame, so a 4x addition belongs on the whole-frame API.

### Who selects it

libserum stores the algorithm per colorization and reports it through
`Serum_GetScalingAlgorithm()`. Consumers that scale libserum's output further
should adopt that value, so that in-frame scaling and display scaling agree.
