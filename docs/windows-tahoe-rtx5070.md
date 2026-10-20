# Windows Tahoe desktop observations on Ryzen 7900X / RTX 5070

This is a single-machine field report from 2026-10-18 through 2026-10-20,
not a performance qualification or a recommended CPU profile for all hosts.
It records a usable desktop configuration and two remaining host-fatal
graphics gaps. Raw guest captures, identifiers and Apple shaders are omitted.

## Tested configuration

| Component | Observed configuration |
| --- | --- |
| Host | Windows 11 Pro, build 26300 |
| CPU | Ryzen 9 7900X, 12 physical cores / 24 logical processors |
| Cache topology | Two groups of six physical cores, each sharing 32 MiB L3 |
| Memory | 64 GiB installed; 32 GiB assigned to the retained guest |
| GPU | NVIDIA GeForce RTX 5070, 12 GiB, driver 617.42 |
| Storage | Guest QCOW2 on Samsung 980 PRO NVMe; virtual NVMe controller |
| Guest | macOS Tahoe 26.7.1, build 25G241, Darwin 25.6.0 |
| Boot profile | iMac20,1; Haswell-noTSX compatibility CPU; WHPX |
| Retained CPU allocation | 12 vCPUs, one socket, unrestricted process affinity |
| Display | One 1920x1080 display, advertised 60 Hz |
| Reims revision tested | `2627243024415e3fe87fb025f627049fc690a2b4` |
| metal2vulkan revision | `ea5d6b877ea54e45676d199782f3db1b54fbaaa4` |

The tests used a separately built guest Metal compatibility bridge with
global routing enabled through Lilu and `-reimsmetal`. They do not establish
the same behavior for an unmodified Apple guest graphics stack. The working
QEMU executable, host graphics library, OpenCore image, NVRAM, launch profile
and disk backing chain were preserved together. This report does not claim
a boot test of every later Reims commit.

## Desktop progress and limits of attribution

**Observed:** setting the measured `METAL_ERROR_MODE` preference to `5`
improved missing UI content, but did not by itself produce a responsive
desktop. The paired host build and guest compatibility bridge subsequently
produced a clear improvement in window movement and Dock interaction.
The user described a settled twelve-vCPU run as nearly native in ordinary
desktop use before opening Weather. This is subjective acceptance, not a
measured native-performance ratio or a sustained 60-FPS result.

The guest's disk-sleep setting was disabled during the experiment and kept
disabled in later matched comparisons. That timing prevents attributing
the early improvement to CPU placement alone. Windows was already using
its High Performance power scheme; changing it was not an intervention.

The user also reported substantial host and guest lag while interactive
desktop-control tooling was operating. Later diagnostics used background
SSH and logging without screenshots or injected desktop input. This avoids
that observer during comparisons, but its isolated overhead was not measured.

**CPU comparison:** six and twelve vCPUs were tested unrestricted and within
one L3 group. Stackless scheduling traces verified the restricted QEMU
threads executed within that group's logical processors. Process affinity
did not reserve those cores exclusively or restrict Windows' other work.
After startup settled, six vCPUs within a cache group felt better than six
unrestricted; the user ultimately preferred twelve unrestricted overall.
The retained profile therefore restores the preferred existing allocation;
it is not evidence that pinning or increasing vCPUs universally fixes pacing.

Startup activity changed responsiveness for several minutes. A reconstructed
profile also felt worse than the earlier control, so the exact tested disk,
firmware and runtime state were retained rather than treating matching CPU
counts as an identical starting point.

**Measurement limits:** passive guest timer/display-link callbacks and host
PresentMon captures were collected. Display callbacks sometimes stopped
during idle while the timer continued. Weather terminated both matched CPU
runs before their planned end. These interrupted or idle intervals cannot
support a before/after stutter percentage, and a display-link callback is
not proof that a frame reached the physical monitor. No causal CPU-tuning
speedup is claimed from these captures.

## Remaining failure classes

### Weather: blended framebuffer fetch

**Observed:** opening Weather terminated the host Reims/QEMU process, with
the runtime refusing render-pipeline creation as `FramebufferFetch`,
`Needs::NotImplemented`. The same failure class occurred with six and twelve
vCPUs and was present before the CPU experiments. This separates it from
the CPU-placement experiment. It is a host runtime fatal, not merely an
application exit.

**Measured on this host:** a read-only Vulkan physical-device query reports
`fragmentShaderPixelInterlock = VK_TRUE` for the RTX 5070. Consequently this
is not evidence that the GPU lacks pixel interlock or that all framebuffer
fetch is unsupported.

**Read from the pinned source:** the storage-image framebuffer-fetch path
rejects blend combinations it cannot model. Its half-float blend path is
conditional on an exact half-blend policy, rounded math and a compatible
half output; normalized source-over support does not imply arbitrary
half-float fetch/blend support. See metal2vulkan's
`lower/fragment/fetch.rs`, `device/policy.rs` and the runtime's
`framebuffer_fetch.rs`. Reims exposes the opt-in as
`REIMS_VGPU_EXACT=half_blend` in `crates/reims-vgpu-vulkan/src/platform.rs`.
This option's existence alone does not demonstrate that it fixes Weather.

A correct fix needs an authored reproducer, pipeline acceptance and pixel
readback for the exact supported blend class, followed by a cold guest boot
and Weather test. Disabling strict checking or advertising all framebuffer
reads as unavailable is not a validated fix. The earlier capability-gating
experiment documented in the bridge README selected another unsupported
guest path and was withdrawn.

### Maps: ASTC texture format

**Observed:** opening Maps terminated the host while creating a texture with
`MTLPixelFormatASTC_8x8_LDR` (Metal value 212). The runtime refused the
required sampled-image format `VK_FORMAT_ASTC_8x8_UNORM_BLOCK`.

**Measured on this host:** `vkGetPhysicalDeviceFormatProperties` returns
zero optimal-tiling feature flags for both `ASTC_8x8_UNORM_BLOCK` and
`ASTC_8x8_SRGB_BLOCK`. In particular, neither supports sampled images under
this driver. RGBA16Float and BGRA8Unorm sampled-image support is present.
The query creates a Vulkan instance and reads physical-device properties;
it creates no device, queue or rendering workload.

**Inference:** Maps needs format emulation/decoding or a demonstrated valid
alternate guest path on this GPU/driver. CPU affinity, SMBIOS identity and
additional vCPUs cannot supply the missing Vulkan format support. Merely
claiming ASTC support would leave the texture's contents unimplemented.

## Repeating comparisons safely

1. Shut macOS down cleanly before freezing a checkpoint. Preserve the exact
   engine and library, guest bridge revision, OpenCore, NVRAM and launch
   environment. Hash those artifacts locally; keep machine identities private.
2. Use independent writable sibling overlays over the frozen disk, with
   separate NVRAM copies. Never resume a backing image writable while any
   child depends on it. Preserve the known working launcher.
3. Keep guest power settings, display mode and background load fixed. Let
   login activity settle, then compare first animations after a fixed idle
   period and repeated animations separately.
4. Collect bounded background logs and scheduling/presentation traces. Avoid
   interactive desktop control during the timed interval, and distinguish
   guest callback timing from host presentation and physical display timing.
5. Test Weather and Maps separately after desktop measurements. A host-fatal
   run is a failure/reproduction result, not a completed pacing benchmark.

The next renderer work is the two feature gaps above. Shared clipboard,
file sharing and elastic memory/CPU allocation are separate integration
work; this experiment does not implement WSL-like resource management or
direct GPU passthrough.
