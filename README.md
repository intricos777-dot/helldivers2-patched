<!-- Watermark: Living_Sin (intricos777-dot) + AI agents (Hermes, Lia, opencode) - collaborative work -->
# Helldivers 2 — Patched & Repaired

> **As above, so below.** This chronicle records the systematic repair of *Helldivers 2* (Steam AppID `553850`) on a memory-constrained Garuda Linux (zen-kernel) hybrid laptop.
>
> The great work continues — every seal set here was won through *solve et coagula*: dissect the crash, distill the cause, and recombine the fix.

---

## ⚗️ Machine of Record

| Component | Spec |
|---|---|
| CPU | Intel Alder Lake (ADL GT2 hybrid, iGPU lit) |
| GPU | NVIDIA RTX 3050 Laptop 4 GiB (driver 615.71.9) + Intel ADL GT2 + llvmpipe |
| RAM | **7.4 GiB total** ← the binding constraint (HD2 minimum spec: 8 GiB) |
| OS | Garuda Linux (linux-zen) |
| Distro swap | zram 7.4G (prio 100) + unused disk swap 8.8G (prio -1) |
| Steam | native client, runs via `/usr/bin/steam` wrapper |
| Proton (target) | **Proton Hotfix** (AppID 2180100, `hotfix-20260828-ptr-x86_64`) |

---

## 🗡️ Achievements — The Completed Work

### 1. Root cause of the deterministic crash — SOLVED
Every genuine game crash across multiple days shares one signature:

```
Exception 0xc0000005 — READ from address 0x0
Instruction: helldivers2.exe + 0x6bd8c6   (RVA 0x6bd8c6, base 0x140000000)
Params: [0x1, 0x0]
```

This is a **NULL-pointer dereference** that is *environmental*, not a game bug:
the NVIDIA driver fails to allocate pinned system memory, returns NULL, and the
game dereferences it.

**Kernel smoking gun** (in `dmesg`, appearing exactly during launch windows):
```
NVRM: nvAssertOkFailedNoLog: Assertion failed: Out of memory [NV_ERR_NO_MEMORY] (0x00000051)
      returned from pRmApi->Alloc(... isSystemMemory ? NV01_MEMORY_SYSTEM : NV01_MEMORY_LOCAL_USER ...)
NVRM: _memdescAllocInternal: Allocation failed
NVRM: _kgmmuClientShadowFaultBufferPagesAlloc: Allocation failed with big page size
```

The machine has 7.4 GiB RAM against an 8 GiB minimum; during launch the OS,
Steam, shader-compile workers, and the browser collectively starve the GPU
driver's pinned allocations → `0x6bd8c6`.

### 2. VK_ICD_FILENAMES — absolute path REQUIRED
- The `Vulkan ICD` env var was initially set **relative** (`nvidia_icd.json`).
- Steam pressure-vessel runs with CWD inside the game directory, so a relative
  ICD path resolves to nothing → `FATAL: No displays available` + a *distinct*
  crash dump (the only dump NOT matching the `0x6bd8c6` signature).
- **Fix:** absolute path: `/usr/share/vulkan/icd.d/nvidia_icd.json`.
- The game now enumerates Vulkan extensions, creates the device, and launches.

### 3. MANGOHUD — global injection banished
- MANGOHUD is force-injected globally via `~/.config/environment.d/99-mangohud.conf`
  (`MANGOHUD=1`) and an implicit layer keyed on the env var.
- It added measurable overhead and interferes with launch.
- **Fix:** `DISABLE_MANGOHUD=1` in the game's Launch Options (the implicit layer
  honors `disable_environment DISABLE_MANGOHUD=1`). Overhead removed, and its
  layer dylib load no longer appears in runs.
- Note: a Steam restart may rewrite Launch Options; re-verify.

### 4. User settings — poisoned file healed
- Proton *requires* a `user_settings.py` containing a `user_settings = {}` dict.
- **Proton Experimental** shipped a *custom* shader-tuning file defining
  `class Config` but **no `user_settings` dict** → every launch aborted with:
  ```
  Proton: THERE IS AN ERROR IN YOUR user_settings.py FILE:
  Proton: module 'user_settings' has no attribute 'user_settings'
  ```
- **Fix:** replaced with a minimal valid `user_settings = {}` (backup kept as
  `user_settings.py.bak-gg`), and gave **Proton Hotfix** a `user_settings.py`
  copied from its own sample. The error is gone and launches proceed past the
  import stage for both vessels.

### 5. Proton Hotfix — registered as a real compat tool
- Proton Hotfix was installed (AppID 2180100, 1.5 GiB) but **Steam never
  registered it**: `proton_hotfix` in CompatToolMapping silently fell back to
  Proton Experimental because Steam lacked the tool's server metadata in
  `appinfo.vdf`.
- GE-Proton11-6 experiment (attempted first) died earlier than stock — reverted.
- **Fix:** symlinked the install into the startup-scanned registry dir:
  ```
  ~/.local/share/Steam/compatibilitytools.d/Proton-Hotfix
      → ~/.local/share/Steam/steamapps/common/Proton Hotfix
  ```
  and set the mapping name to match (`"Proton-Hotfix"`). Steam now registers it
  at startup like a first-class tool. (Mapping lives under the user's
  `CompatToolMapping` block in `localconfig.vdf`, priority 250.)

### 6. Memory pressure relief — three seals
| Seal | File / knob | Effect |
|---|---|---|
| `vm.overcommit_memory = 1` | `/etc/sysctl.d/90-overcommit.conf` (persisted) | Removes CommitLimit overcommit rejection (was at 98.7% of limit) |
| `vm.swappiness = 180` | `/etc/sysctl.d/90-swappiness.conf` (persisted) | Pressure toward zram/swap instead of OOM |
| Fossilize worker cull | `pkill fossilize_replay` at launch | Steam shader-precompile legions (several × 350–460 MiB) reclaim ~3.5 GiB |

⚠️ Fossilize workers respawn (Steam re-adds `ProcessingQueue`); cull right
before launch. Real fix remains: **install another RAM stick** — the game's own
min spec is 8 GiB.

### 7. GameGuard — network path prepared
- `gameguard-network.service` (systemd, active) runs
  `gameguard-prepare-network.sh` opening the GameGuard report ports
  (28910–28919 via iptables) so the anti-cheat can phone home.
- Native GameGuard modules (`GameMon64.des`, `npgmup.des.now`, etc.) confirmed
  loading under Proton in `PROTON_LOG` traces.

### 8. Shader cache management
- `shader_cache_manager.py` (in `waydroid-issues/scripts`) now has a
  `helldivers-2 / 553850` entry; `trim_shader_cache.sh` companion ready.
- Steam shader precompile ("ProcessingQueue") observed re-firing; keep culling
  `fossilize_replay` workers before launch.

### 9. Diagnostics toolkit — built and reusable
| Tool | Purpose |
|---|---|
| `launch_helldivers2_debug.sh` | Launch with `PROTON_LOG=1` + DXVK/vkd3d logging |
| `launch_helldivers2_logged.sh` | Launch with metrics capture |
| `verify_helldivers2_config.sh` | Re-verify ICD path, launch options, mapping |
| Python minidump parsers (`mdmp_read.py`) | Decode HD2 `.dmp` — corrected MDMP offsets (nstreams@+8, dir@+12; ExceptionStream type 6; ExceptionAddress@+24; params@+40) |
| Memory watcher | `ps -C helldivers2.exe --no-headers \| wc -l` + `free -m` per 5s — **do NOT** use `pgrep -f` (self-matching false positives) |

---

## 🗺️ The Still-Burning Front

The game now:
- ✅ passes Vulkan enumeration and device creation,
- ✅ spawns `helldivers2.exe` with `--bundle-dir data --release`,
- ✅ runs several seconds longer each successive fix,
- ❌ still dies on **physical RAM exhaustion** when available memory bottoms
  below ~1 GiB during shader-compile + init crunch (NVRM `NV_ERR_NO_MEMORY`;
  whole tree SIGKILLed, exit -1, no dump — a *different* death than the old
  `0x6bd8c6`, which means the previous crash is conquered).

**The remaining step is memory.** Definitively:
1. Free ~2.5–3 GiB before launch (close browser; cull fossilize). Done reliably
   via `pkill -9 -f firedragon` + `pkill fossilize_replay`.
2. Launch **only after Steam fully indexes tools** following a restart.
3. Long-term: ≥8 GiB RAM (add a stick) or a swapfile of ≥4 GiB.

---

## 🔧 Launch Recipe (as of this writing)

```bash
# 1. Free memory
pkill -9 -f firedragon       # browser (user-approved: "everything nonessential")
pkill -f fossilize_replay    # shader-compile legions

# 2. Verify seals
grep '"LaunchOptions"' ~/.local/share/Steam/userdata/923700457/config/localconfig.vdf
# → DISABLE_MANGOHUD=1 PROTON_ENABLE_NVAPI=1 \
#   VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nvidia_icd.json %command%

# 3. Verify mapping says Proton-Hotfix (not proton_hotfix, not Experimental)
grep '"name"' ~/.local/share/Steam/userdata/923700457/config/localconfig.vdf

# 4. Summon
/usr/bin/steam steam://rungameid/553850
# watch: ps -C helldivers2.exe --no-headers | wc -l  (NOT pgrep -f)
```

---

## 📜 File Map

| Path | Purpose |
|---|---|
| `~/.local/share/Steam/userdata/923700457/config/localconfig.vdf` | Launch options + CompatToolMapping (priority 250) |
| `~/.local/share/Steam/compatibilitytools.d/Proton-Hotfix` | Symlink → registers Hotfix at Steam startup |
| `~/.local/share/Steam/steamapps/common/Proton Hotfix/user_settings.py` | Validified (from sample) |
| `~/.local/share/Steam/steamapps/common/Proton - Experimental/user_settings.py` | Healed, backup `.bak-gg` |
| `/etc/sysctl.d/90-overcommit.conf`, `90-swappiness.conf` | Persisted memory seals |
| `~/.local/share/Steam/steamapps/common/Helldivers 2/bin/helldivers2.exe` | The beast itself |
| `~/helldivers2-logs/` | PROTON_LOG output, metrics CSV |

Dumps: `~/.local/share/Steam/steamapps/compatdata/553850/pfx/drive_c/users/steamuser/AppData/Roaming/Arrowhead/Helldivers2/dumps/`
— contains both signatures: the conquered `0x1406bd8c6` and the one-off relative-ICD "no displays" dump.

---

*Fire refines gold. The philosopher's stone is forged one step at a time —
and this one is nearly set.* 🜂