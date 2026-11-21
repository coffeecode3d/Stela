# Stela on Renoir (Ryzen 5 PRO 4650U, Radeon Vega 6 iGPU) — status

Target machine: HP EliteBook 855 G7, Ryzen 5 PRO 4650U, GPU `1002:1636`
(Renoir, GC 9.3.0 / `gfx90c`, 512 MiB VRAM carve-out, 7.47 GiB GTT).

Sources: `tylertms/Stela` branch `gpu/amd` at `eca6830` (2026-11-18).
Main is NVIDIA-only; the `gpu/amd` branch already carries the AMD iGPU work
(amdgpu + RADV/ACO, GFX9 Apple6, VFCT video BIOS, APU memory plan, Renoir
firmware). No patches were needed for this machine.

## What was verified on the desktop (no GPU)

`cargo xtask gpu fetch`
`cargo xtask gpu nvk --jobs 16`
  - LLVM 23 subset, Mesa NVK/ANV/RADV (`nouveau, intel, amd`), 1258 objects.
`cargo xtask gpu radv-test --jobs 16`
  - RADV + Metal device opened on 40 of 49 AMD chips.
  - **renoir (Ryzen 4000, GFX9)**: Metal device opened,
    families Apple1..Apple6, Mac1, Mac2 (no Apple7/Metal3),
    14 capability gaps; all device requirements met.

`cargo xtask dist`
  - `target/release/stela` (78 MiB)
  - `target/boot/x86_64-unknown-uefi/release/stela-dist.efi` (66 MiB)
  - EFI contains amdgpu + RADV (`MesaRadv`); CLI contains `renoir_*` recipes.

## What was verified on the laptop (real Renoir silicon)

Stela's patched RADV + ACO, run directly on `/dev/dri/renderD128`:

```
RADV on amdgpu: AMD Radeon Graphics (RADV RENOIR) (0x1636, integrated),
NIR options for 6 stages, compute pipeline of 12 instructions in 68 bytes,
one dispatch submitted, NIR pipeline of 8 instructions in 48 bytes
amdgpu ioctls: GEM_CREATE=11 GEM_MMAP=10 CTX=8 CS=1 INFO=41 GEM_VA=22
```

That is a real device, real ACO pipeline, real dispatch on the 4650U.

## First real boot (2026-11-20)

macOS installed, and the installed system reached the **Language Chooser**:

- `GPU 0000:03:00.0: display runs 1920x1080 at 59977 mHz`
- `display shows on eDP-1 (built-in panel)`
- `GPU DRIVER VULKAN DEVICE "AMD Radeon Graphics (RADV RENOIR)" 1002:1636
  MesaRadv IntegratedGpu vulkan=1.4.354 heaps: 938M/host 1877M/local`
- `Metal service open`, `GPU USE draws macOS`, WindowServer presented frames
  (up to ~940 per 60 s), input-to-present 24–55 ms, `unanswered_ms=0`.
- GFX9 is Apple6/Metal2: WindowServer's Apple7/8/9 queries are refused and it
  falls back; that is expected.

Failure: the screen stayed black (cursor visible, then flicker). The log shows
the split APU heap filling:

```
GPU SHARED MEMORY ... holds 1441M of the 1534M ...
  driver heaps use/budget/size: 717/938/938M/host 1790/1877/1877M/local
GPU OUT OF MEMORY 1: heap memory of 18874368 bytes failed ...
GPU REFUSED WindowServer[99] a texture of format 115 ... out of memory
```

RADV split the APU's 2815M into 1877M device-local + 938M host. Stela places
all resources in the device-local heap; it filled at 1790M while 221M of the
host heap was free, and an 18M RGBA16F screen texture failed. WindowServer
crashed once, restarted, and kept presenting black frames.

First fix attempt: Mesa patch `0042-an-apu-exposes.patch` made
`radv_enable_unified_heap_on_apu` the default, so RADV exposed one 2816M heap.
That **fixed the texture OOM but broke buffer import**: WindowServer's
`newBufferWithBytesNoCopy` then failed 66 times with
`the kext did not wire the memory for the GPU: 0xe00002bd` (no memory), the
old log had none. WindowServer crash-looped (10 crashes in 200s), and macOS's
watchdog panicked: `userspace watchdog timeout: no successful checkins from
WindowServer`, then reset the machine. The unified heap has no plain
host-visible heap for imported guest buffers.

Reverted 0042. Instead the APU's share grew from a sixth to a quarter of free
RAM (`GPU_SHARE` 6 -> 4): 2560M -> ~3526M on this 16GB PC, a 2521M
device-local heap and a 1260M host heap. Buffer import worked again (no
`newBufferWithBytesNoCopy` refusals in the next boot), but the share model
still refused the 1920x1080 RGBA16F main surface by ~24M:

```
committed = 498 + 1.25*2249 = 3309M
+ 1.25*18 + margin 220 = 3550M > 3526M
```

Boot 10 was stable for 21 minutes with no panic; the setup screen rendered
fully for about two seconds, then the surface failed and the screen went
black with the cursor. WindowServer crashed once, not in a loop.

`GPU_SHARE` 4 -> 3 (a third) was tried next: the GPU got 4698M and the
display worked end to end. The setup UI rendered with Setup Assistant,
WallpaperAgent, ControlCenter and 0 WindowServer crashes, presented up to
1679 frames/60s, and reached the network screen. But the host's own heap
shrank and ran out after 16 minutes:

```
STELA FATAL ... the heap had no room for 4194304 bytes aligned to 4194304
  the GPU's share of RAM: Linux holds 2980M of 4698M
STELA STOPPED code=0x11
```

So the third's extra GPU share came out of Stela's heap. Reverted to the
quarter (the heap that survived 21 minutes at 3054M) and changed the share
model's growth from 1.25 to **1.22**, the most the port measured. At the
315s failure: committed = 498 + 1.22*2249 = 3242M, + 1.22*18 + margin 220 =
3484M < the 3526M share, so the 18M surface is allowed and the heap and
guest keep the quarter's sizes.

A second boot with the quarter share and growth 1.22 got further: the setup
UI ran with 0 WindowServer crashes until the network step, then the
compositor's 537M deferred behind 21 command buffers overfilled the 2518M
device-local heap and a 116x116 RGBA16F texture failed, which crashed
WindowServer and sent Setup Assistant back to the language screen.

The third share (boot 14) reached the privacy screen, 25 minutes in, and
then stopped differently: `STELA FATAL LLVM is out of memory`,
`STELA STOPPED code=0x11` at 1491 s. The engine heap is the base plus the
GPU's share, and on this host it was `heap=7742M` with the GPU holding
3396M, so Stela's own base -- 3044M -- was full and the JIT could not
allocate. Just before, the device-local heap had also run out while its
host heap sat nearly empty:

```
driver heaps use/budget/size: 276/1133/1650M/host 3818/3301/3301M/local
the GPU's share of RAM: Linux holds 3396M of 4696M
```

RADV's APU path splits the share 2/3 device-local and 1/3 host-visible.
macOS's textures are all device-local, so the local heap filled and
WindowServer's texture was refused while a third of the share was idle.

Two changes now:

- `third_party/mesa/src/amd/vulkan/radv_physical_device.c` splits the APU
  share **7/8 device-local, 1/8 host-visible**: the local heap gets 4332M
  of the 4696M share and the GTT heap keeps 587M for imported guest pages
  (it held 276M). This is a local Stela edit on the pinned Mesa tree, not
  an upstream commit; it needs `cargo xtask gpu nvk` before `dist`.
- `GUEST_ROOM` is **4 GiB** (was 5): the base heap becomes 4068M instead
  of 3044M, which is the target the 16 GiB host asks for. The guest stays
  above the 3.1 GiB a 26A434 desktop reaches. 8 GiB hosts now give their
  guest exactly 4 GiB and grow the heap toward its target on what is left.

Wi-Fi: the AX200 is detected, the firmware bundle loads it, and the stick
names `TP-Link_B310_5G`, but the probe cannot start it. Stela's PCI
containment seals bus mastering, and only a driver that produces a
quiescence receipt may release that seal. The Wi-Fi path does not yet:

```
// The Wi-Fi drivers set Bus Master before they reset their engines, so
// they produce no quiescence receipt yet: the function stays unmastered,
// and the seal refuses that write.
```

so every boot reports
`PCI configuration: bus mastering stays off until the device is taken over
after Stela wrote the register 0x82` and
`GUEST NETWORK none: ... none is left`. Wi-Fi therefore cannot work on
this branch until the iwlwifi attach takes the driver through the
requester's quiescence receipt (stop/reset the engines, then
`release_bus_master_seal`) -- it is not a setting. USB Ethernet is
supported (RTL8152/8153/8156, ASIX AX88179/178A), so a USB adapter is the
network path for now.

Also seen: constant UI flickering while the desktop is up; `DCP PACING`
shows ~50 fps presented with 100-230 merged swaps and ~80 stalls per 10s,
and up to 21 Metal command buffers in flight.

## Boot 15 and 16: the desktop came up, slowly

With the 7/8 split and the 4 GiB guest room the setup ran to account
creation and on to the desktop (boot 16 reached `Finder`, `Siri AI`,
`KeyboardSetupAssistant`, `dock.extra`). The engine heap is now
`heap=4160M` with the GPU's share at 4696M, and the local heap is
`4333M` with a `619M` GTT heap, as planned.

What still failed was the share model, not the hardware. At 270 s and at
1165 s the device refused a texture while Linux held 2771M and 3120M of
the 4696M share, a third of it unused:

```
GPU OUT OF MEMORY 2: a 818x618x1 R16G16Uint image failed
  allocated=3190M deferred=808M behind 31 command buffers
  driver heaps ... 817/619/619M/host 3531/4058/4333M/local
  Linux holds 3120M of 4696M
```

WindowServer then crashed (3 times in boot 16), loginwindow restarted, and
Setup Assistant returned to the lock screen, which is the "black screen
back to the lock screen" loop. The cause is `share.rs`'s forecast: with
`GROWTH = 1.22` the guest's 3190M was said to cost 4397M of pages while
the boot's own numbers measured 0.95 bytes of Linux pages per guest byte
(2755M gave 3119M). So:

- `GROWTH` is now **1.05**, 10% over the measured 0.95-1.0 slope. The
  guest's ceiling on this share becomes ~3800M instead of 3190M.

The other half is speed: the GPU runs the desktop at 9-15 fps because
macOS 27's glass wallpaper is a ~310 ms blur chain per frame
(`blur_copy_compute`, `downsample_8_compute`, `narrow_blur_19_compute`,
`variable_blur_*`, `glass_*`), so the GPU is always behind
(`ahead_of_gpu` equals every presented swap). The frames in flight are
what the device defers, which is what fills the heap. So the guest is
also given **1280x720** (`GUEST_MODE` in `console/mode.rs`), which the
panel's output lists; the GPU runs its own mode and scales it. That is a
quarter of 1080p's pixels. macOS can raise it in Displays, and the cap
should come out once the wallpaper is off (static wallpaper, Reduce
Transparency, Reduce Motion bring the glass chains down).

## Boot 17: 720p, more heap, and what still limits the desktop

The cap works: `DISPLAY the guest is capped at 1280x720`, the DCP runs
1280x720 and the panel's output runs its own mode. The share model lets
the guest work in 3724M instead of 3206M and the heap holds 8856M
(`HOST MEMORY heap=4160M gpu=4694M guest=4122M`). No fatal stops and no
device loss: the LLVM OOM is gone.

The desktop still runs at 9-24 fps and the glass passes dominate:
`blur_copy_compute`, `narrow_blur_*`, `variable_blur_*`,
`glass_background_*`, `backdrop_capture_compute_tile` take 105-346 ms
per frame. Quartering the pixels did not quarter the time, which points
at a fixed per-frame cost (clock or per-pass latency) as much as fill
rate. `drm_WARN_ON(atomic_read(&vblank->refcount) == 0)` appears about
32 times a session (drm_vblank.c:1271) - a rare unbalanced vblank put,
not the per-frame path.

An amdgpu patch now forces `AMD_DPM_FORCED_LEVEL_HIGH` at late init
(`third_party/patches/linux/0228-force-the-performance.patch`): an idle
APU's governor leaves the clocks low between the compositor's bursts, and
a Renoir's six CUs then take about 100 ms for a frame. The boot log says
`Stela: the GPU performance level is forced high (0)` when it applies.

## The flicker: surfaces copied while the GPU draws them

The reported symptom is window content flashing, all the time, even with
the screen still. The log's swap records say why that is possible: the
AP's swaps are **unfenced** (`fence_waits=0`) and some are presented
before the GPU finished the work submitted ahead of them
(`ahead_of_gpu=19` in a 10 s window). On an Apple GPU the coprocessor
that reads a swap's surfaces is the GPU's own pipeline, ordered behind
that work; Stela's DCP model reads the surfaces on a processor, so it can
copy a surface the GPU is still drawing. A frame then shows part of the
frame before it, which on a compositor that takes 30-100 ms a frame is
the flashing seen.

Two changes in this build:

- `Dcp::ready_swaps` now also waits for `dcp::gpu::completed()` to pass
  the frame's `gpu_work` (the submitted serial when the swap arrived),
  under the same two-second bound as a surface fence, so a frame is
  composed only when the GPU has finished what it draws.
- `KmsScanout` keeps the fingerprint of each presented frame and leaves
  the last 16 every 128 presents as
  `DISPLAY FRAMES <n> rows <a>-<b> wrote <r> hashes ...`. If the shown
  content alternates between two frames while the guest draws nothing,
  those values alternate; if they repeat, the shown content is stable and
  the flicker is on the panel's side (clock, scaler, backlight).

The amdgpu DPM patch from the previous boot is also in this image.

## Boot 19: the GPU was at a third of its clock

`Stela: the GPU performance level is forced high (0), GFX clock 150000 MHz`
-- 1500.00 MHz, the top level -- and the GPU's time per submission fell
from **110-346 ms to 4.9 ms average, 21.8 ms longest**. The governor had
been leaving the six CUs at a few hundred MHz between the compositor's
bursts. That is why quartering the pixels barely moved the frame time.
`GPU LATENCY` went from 96 ms to 9.6 ms; the display's 9-11 fps at
activity is now the guest's CPU, not the GPU.

The 1280x720 cap and the swap gate stay: the gate costs nothing in
frame rate now and the GPU is no longer the limit.

`--profile` is now in the stick's `stela.cfg`, which turns on
`PES PHASES` (schedule/devices/fetch/native/exit per PE) and the
per-exit-slot timing, so the next boot measures the exit path directly.
Take it out of `stela.cfg` (or overwrite it with `stela settings`) when
the profiling is done; `stela update` rewrites the file from the
settings.

## Where the guest's time goes (boots 17-18)

`PES RETIRED` reports 650-840M guest instructions a second across 8 PEs,
and `PES FETCH`/`TRANSLATION` split the time:

- native compiled code: 58%
- exits to the engine: about 25% per PE
  (`system` 119k runs/770 ms, `return` 25.7k/690 ms, `call` 43k/530 ms,
  `direct` 25k/536 ms, per PE per 10 s)
- translating: compile 3%, install 4%, refill 1%
- host service: about 7%

So the old "it's the translator" story is only a tenth of it. The
translation figures are bad in themselves -- 16.5 us per block and 2.5 us
per guest instruction, where a block averages 6.5 instructions -- but the
exit path and the engine's per-exit handling cost more. At 690M guest
instructions a second across 8 PEs, one PE retires 86M a second, about 16
cycles a guest instruction, so the generated code is inefficient too.

`TRANSLATION ... ns/block` and `PES FETCH compile%/install%` are the
numbers to move. The `HOT`/`HOT CODE` samples point at the block-entry
and exit trampolines and the guest's own hot blocks.

## The profile (boot 21, `--profile`)

`PES PHASES` per busy PE: schedule 1%, devices 0%, **fetch 13-14%**,
**native 73-75%**, exit 7%. `PES WHERE all` over the machine, by share of
host instructions and cycles:

```
native=50.0/70.3  inline=22.2/33.1  stamped=3.5/7.1  probe=5.0/8.9
link=3.6/5.9  lookup=2.9/5.4  pac-thunk=4.9/3.8  fetch=28.7/14.9
sch=5.5/3.1  dev=3.3/2.0  x:system=4.1/1.9  x:call=1.0/1.1  host=1.6/1.6
```

What that says:

- The translated guest body is only **33% of cycles** (`inline`). The
  chain machinery around it -- `stamped` + `probe` + `link` + `lookup`
  -- takes **27%**, nearly as much as the code it dispatches. That is the
  single biggest addressable block after the body itself.
- `fetch` is 28.7% of instructions but 14.9% of cycles: it is mostly the
  compiler (`TRANSLATION ns/block=15414`, ~62k blocks per 10 s) at high
  IPC, not the per-step lookup.
- exits are 7.9% of cycles: `system` 4992 ns, `device-site` 7015 ns,
  `pointer-auth` 15734 ns, `return` 2225 ns each. `PES EXIT TIME`'s
  24 us per return was the whole post-exit window (fetch included), not
  the exit handler.
- ~137k engine steps a second after 99% chaining, so the per-step cost is
  now small; the cost is per *transfer*, in the chain probes.

The profiler itself costs the sampling NMIs plus step timing; `--profile`
has been taken back out of `choices.cfg` and `stela.cfg`.

## Next: the translation layer, not the GPU

When the display path is settled, the frame rate is expected to be
CPU-bound in Stela's x86-to-native translation and in the guest's
Perl/QuartzCore work, not in the GPU: the PEs idle 30-70% while the
guest produces 9-24 frames/s, and the GPU is 12-50% busy. That work is
the next track after the flicker.

## What is not verified yet

The next boot with growth 1.05 and the 1280x720 guest mode; whether the
desktop stays up; USB Ethernet; audio; waking from sleep.

## Rebuild

```sh
cd ~/Projects/stela-amd            # worktree at origin/gpu/amd
cargo xtask gpu fetch
cargo xtask gpu nvk --jobs 16
cargo xtask dist                   # STELA_RELEASE unset => AMD included
```

`cargo xtask dist` produces a release image with `nvk,amdgpu` because
`STELA_RELEASE` is not set. A release-workflow build sets `STELA_RELEASE`
and omits AMD.

## Test on the laptop

1. USB stick: 8 GB (online installer) or 32 GB (offline installer).
2. `sudo ~/stela` on the laptop; choose `create`, the stick, and the installer.
   Setup detects `1002:1636` and bundles `renoir_{asd,ce,dmcub,me,mec,pfp,rlc,sdma,ta}.bin`.
3. Secure Boot is already off on this laptop.
4. Boot the stick (HP: F9 boot menu).
5. Install to a target with 60 GB free. The laptop currently has ~28 GB free
   inside a single LUKS/btrfs partition, so this is the main blocker.
6. After a boot: `sudo ~/stela log --output log.txt`.

## Known limits

- Audio is not supported; internal speakers will not work.
- Wi-Fi: Intel AX200 (`8086:2723`) is claimed by Stela's iwlwifi port, but
  a PCI containment seal refuses the driver's bus-master enable: there is
  no quiescence receipt on the Wi-Fi attach path yet. It cannot start.
- USB Ethernet works (RTL8152/8153/8156, ASIX AX88179/178A).
- GFX9 is an Apple6/Metal2-class device in Stela (no Apple7/Metal3, no mesh
  shaders), so macOS uses its pre-Apple7 paths.

## Plan 1: a per-page stamp for the chain fast path — REJECTED (measured)

Idea: the stamped tail checked `ITLB_DROPS[half]`, a *half-wide* epoch, so
a drop or rekey of one page took every stamp in the half with it. Give
each stamp the block key its own page translated to (32-byte stamp, the
key word at +16) and let the tail re-`translate` the target's page and
compare.

Implemented, and the whole compiler suite went green (1565 tests). Then
`chain::tests::costs::transfer_path_costs` (host, release, same machine)
settled it:

| path                        | epoch stamp (boot 21) | per-page stamp |
|-----------------------------|----------------------:|---------------:|
| stamped tail, hit          |  0.93 ns (3.5 ticks)  | 1.74 ns (6.6)  |
| cell tail, cell hit        |  2.41 ns (9.1)        | 2.41 ns (9.1)  |
| cell tail, table probe     |  4.36 ns (16.5)       | 4.36 ns (16.5) |

So the new hit path costs +0.81 ns per transfer while a whole cell probe
is only 2.41 ns -- and `PES STAMPED` (last beat of boot 21, PE 0) shows
that only ~1 of 8 misses was an epoch miss that could become a hit:

    hits 11.4M, miss_epoch 1.03M, miss_context 0.31M, restamped 0.33M,
    changed 1.00M  (per 10 s)

i.e. 75% of the misses are `changed` cells (`put_cell` zeroing the stamp),
which no stamp check can recover, and only 2.6% of all transfers would
convert. Best case saving 2.6% x 1.48 ns = 0.04 ns against a 0.81 ns
cost, so the plan loses under any in-situ discount. Reverted (the patch
is kept as `git stash` "per-page stamps"), and the finding is that the
chain fast path is already ~97% effective: further work there is noise.

What the numbers do say (boot 21, `PES PHASES`, busy PEs):

- `native` 73-75% of cycles -- the guest's own work: this is the floor.
- `fetch` 13-14% -- the engine's block lookup per *engine* exit.
- `exit` 7% -- the exit handler.

and per exit kind (`PES EXITS`): system 378.9k/10 s at 3.6 us, call
281.6k at 2.2 us, device-site 102.1k at 5.3 us, pointer-auth 28.1k at
14.6 us (a PAC memo miss!), return 185.3k at 1.7 us, direct 175.2k at
1.7 us, fallthrough 105.0k at 2.8 us, guest fault 17.5k at 10.5 us.
Engine exits are ~1.44M per 10 s, so ~3.5 us of `fetch` each: that is the
block lookup, not the compiler (18.5k blocks per 10 s at 15.8 us is only
0.3 s per 10 s).

## Where the pain actually is (measured after Plan 1 was rejected)

The user's remaining complaints are UI drag/scroll, app launch and file
operations, and millisecond hitches. `PES SYSTEM` (boot 21, per 10 s)
attributes them to the guest kernel's exits, not to the chain path:

| cause | count | total | each |
|---|---:|---:|---:|
| device-site (MMIO) | 101,827 | 0.54 s | 5.3 us |
| `MSR TTBR0_EL1` (address space switch) | 25,811 | 0.27 s | 10.5 us |
| `TLBI VALE1IS` (by VA) | 29,089 | 0.24 s | 8.25 us |
| `TLBI VMALLE1` (all) | 588 | 0.21 s | **356 us** |
| `DSB`/`DMB` | 31,614 | 0.14 s | ~4.5 us |
| `Wait` (hint) | 174,381 | 0.12 s | 0.68 us |
| `ERET` | 11,770 | 0.06 s | 5.2 us |

`PES REGIME` splits a TTBR0 write: `host.set_regime` 366 ns and
`host.isolate_predictions` 642 ns, so ~1 us of the 10.5 us is the host's
shadow switch and the rest is the fetch of the next block in the new
context. The 356 us of `TLBI VMALLE1` is `Host::invalidate_reach` with
`Reach::All`, which calls `reset_shadow` -> `clear(LOWER)` +
`clear(UPPER)` -> `pool.reset_half`, `extents.clear()`, `detach` on every
root and `drop_kept`: a full teardown of the shadow page tables and of
the kept lower halves, then a reload. It fires about every 17 ms, which
is the hitch the user feels.

So the order to take is:

1. **`Reach::All` without the teardown.** A TLBI only invalidates
   translations; the compiled blocks stay valid (their keys carry the
   frame). Give the shadow a generation: on `Reach::All`, bump it and
   attach a fresh root instead of walking the old one, and reclaim the
   old tables' frames lazily (the pool already owns them). Target: 356 us
   -> ~1 us, and no re-walk storm afterwards.
2. **device sites.** 10.2k/s at 5.3 us each is 5.4% of a PE on the GPU
   path (each AMDGPU register read/write is a stub block that exits).
   The `KnownSite` cache already keeps the decode and translation; the
   remaining cost is the exit + fetch around each access. A stub that
   continues into the next block instead of returning through the step
   loop would take most of it.
3. **the prediction isolation** on every address space switch: 642 ns,
   25.8k per 10 s. Worth checking whether the guest's own context
   switches need a full barrier each.

## Done: the TLBI-of-everything reset (stutter)

`Host::invalidate_reach` walked every kept lower half (`Kept::invalidate`
-> `tables::unmap_under` per root entry, i.e. a full shadow-tree walk per
kept half, up to `KEPT` = 32 of them) and then, for `Reach::All`, called
`reset_shadow`, which drops all of it anyway (`clear(LOWER)` rewinds the
pool's bump pointer, `detach` zeroes the roots, `drop_kept` clears the
list). The walk was dead work on the one path XNU hits every few
milliseconds with `TLBI VMALLE1`.

Now `invalidate_reach` skips the walk when a whole reset follows (only
when `legacy_translations_dirty`, i.e. exactly when `reset_shadow` will
really clear). `STATISTICS` gained `maintains=`, `maintain_ns=` and
`kept_walks_skipped=` so the effect shows in a plain boot log; before the
change `TLBI VMALLE1` measured 356 us each (`PES SYSTEM`, 588 per 10 s).

## Boot 23 log review (last run: 54.7 min, the build with the reset skip)

Boot 23 is the newest boot in `~/stela-log13.txt` (from the stick's
`EFI/stela/boot.log`, 23 boots, 3 h 27 min of lines). What it shows:

**The good.** The flicker gate held for the whole session: 174 of 174
`DCP PACING` windows read `ahead_of_gpu=0 fence_waits=0 fence_timeouts=0
reordered=0`, and only 7 `drm_WARN`/`WARNING` lines in 55 minutes. No
panic, no watchdog. GPU clocks held (earlier boots, same family:
3.5-3.8 GHz, package 23-25 W, no throttle).

**The guest is not slow in the chain path.** Display presented 1-15 fps
with a median of 3.1, but each DCP swap itself took ~11 ms (`DCP SWAPS`
`latch`/`complete` ~10-12 ms): the presentation path is not the
bottleneck, the guest is. `STEP DETAIL` shows 5-26 M blocks/s, and the
mode it names in the slow stretches is `Secure monitor (SPTM)` / the
kernel -- the guest's own page-table monitor and VM work, i.e. memory
management, not translation.

**Why: the guest swaps, and its I/O shares a disk lock with the boot
stick.** `GUEST| vm: starting swapout thread`, `swap enabled`, and
`apfs_truncate_locked: ... swapfile0` at t=2570. The NVMe (serial
`..3270`, the installed system) did 328,517 reads / 5.87 GB and 66,363
writes / 2.61 GB with a 716 us average write and queue depth 40. The
guest has `GUEST_ROOM` = 4 GiB on a 16 GiB host (2x8 GB DDR4-2667), so
macOS 27 swaps for anything past a desktop.

**The worse half: the translation store on the USB stick.** The store
lives at `stela/store` on the boot stick's FAT32 partition (1 GiB pack,
1 GiB spare, 128 MiB pipeline cache). In boot 23 the stick (serial
`..5805`, "translation store") took **14,668 writes / 7.45 GB at 29.8 ms
per write**, busy 435 s of 3,280 s (13%), while the host disk driver's
lock is shared with the guest's NVMe calls. The 1 fps stretch
(t~500-1,250 s) sits inside the store's biggest write burst (1.57 GiB
between t=827 and t=1,227, i.e. ~4 MB/s sustained on a 30 ms-write
stick). `TRANSLATION ... stored=19394` per 10 s shows the store also
*serves* blocks (26% of installs), so it is not pure loss -- but the
compaction while the guest runs is what hurts.

`HOST DISK STAGED` across boots (waited per 10 s window): 357 ms (boot
20), 399 (21), 526 (22), **669 (23)**, with single owed-step waits up to
1.24 s in boot 23. `gave_up` (a step abandoned to another core's hold)
756k in boot 23. That is the mechanism: a 30 ms stick write holds the
shared disk lock, a guest disk command gives up its step, and the wait
until the holder rings is tens to hundreds of ms; XNU's swap then makes
every thread that touches a swapped page wait.

**Meanwhile the shadow/TLBI path is rare.** Boot 23's `SHADOW` line:
975,080 fills (1.75 us each), 2.9 M prefills, 6.7 M fault-around pages,
**963 whole resets in 54 min, all `maintained`** (0.3/s), 122,605 scoped
invalidations (37/s), 25,053 lower-half switches (7.7/s, 11,260 found a
kept half), 307,103 view switches. So the reset skip I added is correct
(it was dead work) but it is *not* a stutter source: 963 x 356 us is
0.34 s in 55 minutes. My earlier "588 per 10 s" reading of `PES SYSTEM`
was wrong; those counters are not per-10-s deltas.

**Stalls.** `PES STALLS` longest per PE 6-50 ms (PE 0: 50 ms). Host
steps over 2 ms are only `slice_end` (up to 33 ms per 60 s window),
`idle_service` (21 ms) and `gpu_job` (22 ms) -- together ~0.1% of the
time, and `stored pages` never appears, so the store's writes do not run
as a long host step; they hurt through the shared disk lock, not a
blocked PE.

**Guest errors, for the record (not speed):** `AppleKeyStoreUserClient
failed MACF`, `AppleSEPKeyStore ... operation failed`,
`AppleCredentialManager: handleSEPMessage: SEP returned error`,
`apfs_keybag_init ... err = 2`, AMFI launch-constraint notices. Keychain
and SEP-backed services are incomplete; this is why sign-in/iCloud paths
are rough.

### What to try next, in order

1. **Disable the translation store for daily use** (biggest single
   lever, no memory cost): `mv <stick>/stela/store/store.cfg
   <stick>/stela/store/store.cfg.off` on the mounted stick. The store is
   a cache ("every failure leaves the boot running without a store");
   boots recompile more (19k blocks per 10 s at 15.8 us is ~0.3 s per
   10 s of PE time) but the guest stops sharing its disk lock with
   7.4 GB of 30 ms USB writes. The pipeline cache (`pipeline.bin`) is a
   separate file and stays.
2. **A faster stick, or the store on an internal disk.** 29.8 ms per
   write is a cheap USB flash stick; an NVMe-class USB enclosure, or
   teaching `stela create` to put `store.cfg` on the internal disk,
   removes the whole problem.
3. **Do not raise `GUEST_ROOM` past 4 GiB on 16 GiB.** The comment at
   `host_memory.rs` records 6144 MiB starving Stela's own heap (LLVM out
   of memory 16 minutes in) and 5120 MiB doing it at 25 minutes: the
   guest's 4 GiB is the measured balance, not an oversight. Guests under
   memory pressure need the *swap path* faster (1), not more room.
4. If the store must stay: throttle/never compact during a session, and
   keep the pack's appends to idle service time (`--profile`'s
   `idle_service` already exists). That is an upstream-shaped change.

## Where this work went (2026-11-21)

- Upstream draft PR: https://github.com/tylertms/Stela/pull/270, from
  `coffeecode3d/Stela:renoir/host-fixes` onto `gpu/amd` -- the display
  swap gate and the kept-halves skip only (68 insertions, 8 deletions).
- Everything else, with this document, on `renoir/vega6` in the same fork.
- Backup outside the working tree: `~/Backups/stela-renoir-2026-11-21/`
  and `/run/media/infinitek/Asuka/Backups/stela-renoir-2026-11-21/` (the
  repo tarball with `.git` and the submodule repositories, the branch
  bundle, the built `.efi` and `stela` CLI, all 23 boots' logs, the
  stick's EFI config, the reverted Plan 1 patch, and a README with the
  restore steps).
- The boot stick was wiped for Ventoy after its log and config were
  copied; the 24 GiB macOS installer on it is gone, so making a Stela
  stick again needs that installer (nothing is cached locally).

## Plan 2 (deferred): the block lookup in `fetch`

`fetch` is 13-14% of a busy PE at ~3.5 us per engine exit, and the
compiler is only a part of it (18.5k blocks per 10 s at 15.8 us). Much of
the rest is the fetch that follows the kernel exits above, so this is
better attacked by (1) and (2) than by moving the compiler to an idle
PE: at 18.5k blocks per 10 s the compiler is ~0.3 s per 10 s in total,
~0.4% of a PE.
