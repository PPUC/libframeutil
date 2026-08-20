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

### Frame borders

Outside the frame is **black**, not a copy of the edge pixel. Clamping an
out-of-bounds neighbour to the centre makes content touching an edge see its own
colour beyond it — a false edge that trips Scale2x's rounding branch and shaves
pixels off. DMD scores drawn on row 0 lost the tops of `8`, `S`, `0`, `9` and
`3` for exactly this reason, and only there, because the same glyphs were intact
lower down the frame.

Because outside is a real value, the selection can land on it.
`SelectUpscaled2xSourceIndex()` returns `Helper::kUpscaleSourceOutside` for that
case rather than an index — **callers that index parallel planes with the result
must test for it**. `SampleUpscaled2x()` and the whole-frame forms resolve it to
black themselves.

The per-pixel form is inherently 2x only: Scale4x is Scale2x applied twice and
needs the intermediate frame, so a 4x addition belongs on the whole-frame API.

### Performance

The scalers are header-only templates specialized on the pixel size, so they
are compiled as part of whatever consumer includes them and pick up that
build's optimization flags. Nothing here hardcodes an optimization level or a
`-march`/`-mcpu`, deliberately: a target that knows its CPU (zedmdos does) gets
the benefit by passing flags at its own build, and no consumer has to fight a
setting baked into the library.

What matters most is that the consumer builds with `-O3`. GCC vectorizes at
`-O2` only under a very-cheap cost model; `-O3` switches to the dynamic model,
which is what lets these loops reach NEON. Measured on aarch64 GCC 16, that is
worth about 5x on the formats with a native pixel type. `-mcpu=<cpu>` is worth
adding on top where the CPU is known, and the choices this header makes hold
across Cortex-A53, A72 and A76 alike, so one build of the header suits all of
them.

The scalers are tuned for GCC/aarch64, clang/aarch64 and clang/x86_64, which
covers Raspberry Pi, macOS and Windows between them. Two decisions differ by
compiler and are selected at compile time rather than picked once:

- how a neighbour is bound (`Bound<T>`), by pixel type;
- whether the four outputs are computed branchlessly
  (`FU_VECTORIZES_MASKED_SELECTS`), by whether the compiler will vectorize the
  loop at all. GCC does and is 2.4x faster branchless; clang does not, and is
  1.4x faster keeping the branch that skips uniform areas. MSVC is grouped with
  clang as the weaker auto-vectorizer.

Both are documented at their definitions with the measurements behind them. If
you retune, keep a bit-identity check against the previous implementation: every
one of these choices is a pure performance trade and must not change output.

### Who selects it

libserum stores the algorithm per colorization and reports it through
`Serum_GetScalingAlgorithm()`. Consumers that scale libserum's output further
should adopt that value, so that in-frame scaling and display scaling agree.
