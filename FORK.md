# Gradient Control Laboratories' fork of Fidget

This is a fork of [mkeeter/fidget](https://github.com/mkeeter/fidget). The
`omega` branch is upstream commit `1879743d` ("Return an `i64` for Rhai interop
(#518)", 2026-11-17) plus the commits listed below:

```sh
git log --oneline 1879743d..omega
```

It exists to carry the changes that
[omega-core](https://github.com/Gradient-Control-Laboratories/omega-core), a
compiler for the GSL implicit-modeling language, needs from Fidget: it evaluates
and meshes GSL fields with Fidget's VM, JIT, dual-contouring mesher and GPU
renderer. All files stay under MPL-2.0 (see `LICENSE.txt`).

Commits with a `Co-Authored-By: Claude` trailer were prepared with Claude Code.
Upstream's README sets a no-LLM policy for Fidget and for contributions to it, so
this fork is not an upstream contribution and none of it is offered as a pull
request.

Each commit is one change and its message says why. They fall in two groups.

## Self-contained fixes and additions

None of these needs `MeshOracle` or the channel pass.

- **sqrt of a gradient with no derivative has none**, on the CPU
  (`core, jit: ...`) and on the GPU (`wgpu: sqrt of an argument ...`).
  `Grad::sqrt` divided by `2 sqrt(v)`, which is 0 / 0 where both are zero, as
  `length(max(d, 0))` is deep inside an exact box.
- **Progress from the mesher**: `Settings::progress`, for the multithreaded build
  (`mesh: an optional progress callback ...`) and for a single-threaded one
  (`mesh: progress for a single-threaded octree build`).
- **Mesher output changes**, each in `fidget-mesh`:
  - `mesh: triangulate a dual quad without folding around the edge intersection`
  - `mesh: merge the face and center QEFs of eight leaves from the right edges`
  - `mesh: a zero gradient at a min/max tie adds no plane, and its leaf is kept`

## Specific to a host that embeds its own fields

These add machinery upstream may not want to carry.

- **`MeshOracle`**, an op whose value comes from a host-supplied callback
  (`core: a MeshOracle op, a leaf the host evaluates`, and
  `core: a mesh oracle's coordinate that folded to a constant ...`). The VM
  evaluates it; the JIT and the bytecode compiler refuse a tape that contains
  one.
- **A GPU channel pass**, N unclamped floats per pixel
  (`wgpu: a channel pass, N unclamped floats per pixel`). It is written against
  the `fidget-wgpu` architecture of upstream's #510 and modeled on the color
  pass.
- **`core: getrandom 0.4 without wasm_js`**, a build setting for a wasm32 host
  that supplies its own randomness.

## Tests

The commits that add behavior with a unit test carry it: `quad()`, the merging of
leaves' QEFs, a mesh oracle's constant coordinates, and the channel pass (its
shader compiles, its config matches the Rust struct, and `voxel_channels` renders
a sphere on the GPU and reads the channels back). The regression tests for
the sqrt and min/max-tie problems are in omega-core's own suite, not here: they
mesh shelled and unshelled boxes with round dimensions through GSL.

## Known issue

`core: a MeshOracle op, ...` says in its message that `test_vm_op_size` fails on
its `SsaOp` assertion (`SsaOp` is 24 bytes with the op, not 16). The assertion has
not been updated on this branch.
