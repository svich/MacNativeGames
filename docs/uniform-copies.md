# Copies of renamed inline uniform updates

Findings of 2026-10-23 on 1.4.3, Mac only (MacBook Air). Work in progress: see [Open work](#open-work).

## The cost

The game updates small uniform ranges inline between draws (`nvnCommandBufferUpdateUniformBuffer`). The runtime logs
each update (`mtl_sub_update`) instead of copying it on the GPU, and a draw that binds a range overlapping logged
updates gets a private copy of that range (`upd_shadow` → `pend_materialize`): memory plus every pending write.

- Profile of the replay thread: `upd_shadow` 7.3% inclusive, of which memmove 2.6%.
- Game log (`uniform 换副本` line): about 300 updates, 200 new copies and 4.5–5 MB copied per frame, so about 25 KB
  per copy for updates of tens to hundreds of bytes. Each copy is the whole range the game bound, and each new copy
  also walks the whole pending-write table (`pend_overlay_x`).

## Changes on this branch

| Commit | Change | Off switch |
|---|---|---|
| `da73689` | An inline update whose bytes equal the newest logged update covering its whole range is dropped: no log entry, no new copy, argument buffers stay valid. Never across a newer overlapping update or a flush (`upd_guard` clears the log before every other GPU access to the range). | `TOTK_UB_SAME=0` |
| `ee1d8f5` | A copy covers only the bytes the shader reads from the binding (`ubo_window`). `msl_read_extents` scans the MSL once per shader: every reference must be `(*spvDescriptorSet0.<name>)._m0[<integer>]`, anything else (computed index, pointer use, other layout) means the whole range. Updates past the window need no copy for that draw. | `TOTK_UB_WINDOW=0` |
| `a8ecf7a` | Both switches are globals (`g_ub_same`, `g_ub_window`) registered in `ub_var`, so `DM_NVN_UB` can switch them in play. | — |

`ee1d8f5` is a cherry-pick of `3e1752d` on `claude/happy-lehmann-570c81`; only the probe conflicted.

Over the v143 seed (33,264 shader variants), 90% of uniform slots are read only at constant indices, median extent
496 bytes, at most 3,248. A Python scan and the C scanner agree on all 113,922 slots.

The `uniform 换副本` log line now reads `每帧更新 X 次（内容未变跳过 Y 次）`: X counts logged updates only, so the game
issued X + Y updates.

## Verification

- `probe-gpu-ub-rename` under the Metal validation layer: 3,956 checks pass. Negative controls fail at their own
  checks: 1 (memory bound instead of the copy), 2 (no flush before a copy), 3 (scanner ignores computed indices),
  8 (stale shadow index), 9 (equal update compared with any covering update, not the newest).
- `python -m unittest discover -s tests`: 303 tests pass (needs a Python with `cryptography`).
- Built only for the Mac (`build/totk-native-v143`). iPhone / iPad: not built, not run.
- About 15 minutes of play with both changes on, one region, no visual glitches seen, no crash.

## Measurements

### Separate runs: copy counts only

Two runs of 3–5 minutes, both changes on and both off, medians over windows of 2,500 or more draws per frame:

| | Both on | Both off |
|---|---|---|
| KB copied per frame | ~745 | ~4,845 (6.5×) |
| New copies per frame | ~202 | ~270 |
| Updates skipped as equal | ~46% of ~351 | 0 |

A third run with the skip only copied ~4,177 KB and made ~171 copies per frame. So the read window removes most of
the bytes and the skip most of the copies.

CPU time from separate runs is **not usable**: the runs had different scenes (3,050 vs 3,600 draws per frame) and
clocks (median 1.7, 1.9 and 3.3 GHz on the replay thread, depending on how hot the Mac was). An early estimate of
−17 to −26% replay CPU per draw came from such a comparison and was wrong.

### One alternating run: CPU per change

One 10,800-frame session with `DM_NVN_UB_N=300`: even phases both on, odd phases in turn window only, skip only,
both off. Each odd phase is compared with the mean of its two both-on neighbours, per draw, over phases of 1,500 or
more draws (4 pairs per configuration; clocks within 0.06 GHz inside each triple). + means it costs more than both on:

| Switched off | CPU time / draw (mean, median) | Instructions / draw | Cycles / draw (mean, median) | KB / frame | Copies / frame |
|---|---|---|---|---|---|
| Skip | +5.5%, +2.8% | +7.3% | +4.3%, +1.2% | ~0 | +93 |
| Read window | +3.3%, +3.7% | +0.4% | +5.7%, +5.0% | +3,100 | −83 |
| Both | +4.6%, +5.1% | +6.0% | +1.3%, +0.4% | +3,960 | +31 |

- Together: about 4–5% less replay-thread CPU per draw, about 0.5–0.7 ms per frame in busy scenes (replay thread
  10–15 ms per frame). That is up to ~2–4% more FPS where the replay thread limits the frame, nothing visible where
  the GPU or the 30/60 cap does.
- The skip mostly saves instructions (fewer copies to build); the window mostly saves memory traffic (cycles, not
  instructions). The window adds ~80 small copies per frame: shaders reading different amounts of one bound block
  each need their own copy.
- Single pairs range from −4% to +18%: four pairs give the size, not a reliable ranking of the two. "Both off" being
  less than the sum of the singles is within that noise.

### How to repeat

Launch the app from Terminal (Finder / Dock start it without the environment):

```bash
open --env DM_NVN_UB="win_only=@set:g_ub_same=0;skip_only=@set:g_ub_window=0;off=@set:g_ub_same=0,@set:g_ub_window=0" --env DM_NVN_UB_N=300 "<app>"
```

Phases and the 300-frame statistics lines share boundaries, so the window ending at frame N is phase N / 300 − 1.
Per window, read `翻译` (instructions per draw, CPU ms, draws per frame), `翻译细项` (cycles per frame, clock) and
`uniform 换副本` (updates, skipped, copies, KB) from `~/Library/Logs/MacNativeGames/totk.log`. Stay in one busy place:
draw counts that swing between neighbouring phases make the per-draw ratios noisy.

## Risks

- **Read window (low–moderate).** A window smaller than what a shader reads makes the shader read unrelated arena
  bytes past its copy: wrong transforms or colours, flicker, only when an update overlaps that binding in that frame,
  so intermittent. Mitigated by the conservative scanner, the cross-check and negative control 3. A read past the end
  of the arena buffer itself would be an out-of-bounds GPU read (unlikely). Shaders loaded without their MSL source
  get no window (whole range).
- **Skip (low).** Stale constants if a GPU write to a uniform range bypassed `upd_guard`. None found; the
  "overlap found after the encoder opened" counter stayed 0 in every log.
- **Copy table.** ~80 more copies per frame wrap the 512-entry shadow list more often: CPU only, not correctness.
- Players starting from Finder cannot set the off switches: turning a change off for them means a rebuild.

## Open work

1. Play-check other regions with both changes on: the Depths, sky islands, a town, cutscenes, menus.
2. A longer alternating run standing in one busy spot (~10 min) to separate the two shares more reliably.
3. Build and run on iPhone / iPad.
4. Merge with the other unmerged perf branches that touch `nvn_mtl.m` (`perf/replay-draw-overhead`,
   `perf/arena-out-params`, `perf/border-key-cache`, `feat/timeline-recording-copies`): on those sources only the
   probe conflicted.
5. The rest of `upd_shadow`: each new copy still walks the whole pending-write table (`pend_overlay_x`); copying from
   the previous copy of the same range plus only newer updates would avoid that walk.
