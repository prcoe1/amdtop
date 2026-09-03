# Code Review Fixes — 2026-09-03

**Scope:** `amdtop` fork of `nvtop` (commit `da6282a` → `bcac548`) — AMD/ROCm focus.
**Review requested:** `review this code` → 10 issues classified Critical/High/Medium/Low.
**Fix branch:** `master` on `prcoe1/amdtop` (`fork` remote). Upstream `HUSRCF/amdtop` (`origin`) untouched per `AGENTS.md:1`.

---

## 1. Summary

| Severity | Issue | Location | Fix | Verification |
|----------|-------|----------|-----|--------------|
| **Critical** | Shell execution via `popen("lspci …")` — injection + fork storm | `src/nvtop.c:62`, `src/extract_gpuinfo_amdgpu.c:603` | Replace with sysfs reads; no shell | `check_amd_gpu_hardware_exists` scans `/sys/bus/pci/devices/*/vendor`; `query_sysfs_pcie_link` reads `current_link_speed/width` + `max_link_*` fallback; mock test in `/tmp/mock_pci` PASSED |
| **High** | FD check `if (pcieBWFD)` treats fd 0 as failure + missing `close` on `fdopen` failure | `src/extract_gpuinfo_amdgpu.c:408` | `if (pcieBWFD >=0)` + `if (!PCIeBW) close(pcieBWFD)` + `sysfsFD<0` guard, same for `hwmonFD` | `gcc -fsyntax-only` + build `amdtop --snapshot` OK |
| **High** | Blocking `fgetc(stdin)` on `drmDropMaster` failure hangs headless/`-s` | `src/extract_gpuinfo_amdgpu.c:330`, `src/extract_gpuinfo_msm.c:136`, `src/extract_gpuinfo_mali_common.c:204` | Replace with `fprintf(stderr, ...)` and return | Grep no `fgetc` in those files; snapshot non-interactive PASSED |
| **High** | Wrong comparator — `compare_process_type_asc` called `compare_process_name_desc` | `src/interface.c:1018` | `return -compare_process_type_desc(...)` | Isolated C test: `desc=1 → asc=-1`, bug `0` → **PASSED** |
| **High** | `busy_usage_from_time_usage_round` divide-by-zero + overflow | `include/nvtop/extract_gpuinfo_common.h:239` | Guard `time_between==0` and `current<previous`, clamp `delta` | `test_fixes2.c`: `0`, reverse, `50%`, `100%` clamped — **PASSED** |
| **Medium** | Plot bounds: `data[i+k]` OOB, legend truncation `length-cols` | `src/plot.c:49,118` | `draw_cols = min(num_data, cols)`; `i+num_lines<=draw_cols` + `i+k<num_data` guard; `cols` not `length-cols` | `plot bounds` unit test + snapshot render OK |
| **Medium** | ROCm duplicate process path overwrites fdinfo `gpu_usage` | `src/extract_gpuinfo_amdgpu.c:1195`, `src/rocm_smi_utils.c:232` | Keep fdinfo as primary; use ROCm `cu_occupancy` only if `gpu_usage` not already valid | Code inspection + `nvtop --snapshot` shows stable `100%` not flapping |
| **Medium** | `strncpy` without guarantee + `remaining_len` off-by-one | `src/extract_gpuinfo_amdgpu.c:702,714,763` | `snprintf(buf, MAX,"%s",name)`; `remaining_len = MAX - len` | `snprintf` truncation test `127` chars — **PASSED** |
| **Medium** | CMake `project(nvtop)` drift + hardcoded `/opt/rocm-7.1.0` | `CMakeLists.txt:7`, `src/CMakeLists.txt:95` | `project(amdtop)`, `PATHS /opt/rocm` + `PATH_SUFFIXES rocm_smi/include` | `cmake -B /tmp/amdtop-build-final2` Configuring done |
| **Low** | Mixed CN comments, magic `0xFFFFFFFF`, committed `.snap`/creds | `src/rocm_smi_utils.c:76`, `.gitignore:1` | Translate, `#define RSMI_CU_OCCUPANCY_INVALID`, `.gitignore` adds `*.snap squashfs-root/ snap_credentials.txt *.deb` | `grep` no CN comments; `git status` clean |

---

## 2. Detailed Changes

### `src/nvtop.c:60-73`
- Removed:
  ```c
  popen("lspci 2>/dev/null | grep -iE 'VGA|3D|Display.*AMD|ATI'", "r")
  ```
- Added sysfs scan:
  ```c
  opendir("/sys/bus/pci/devices"); fopen(vendor_path,"r"); strstr(vendor,"1002")
  ```
- Added `#include <dirent.h>`. No shell, no `pclose` leak.

### `src/extract_gpuinfo_amdgpu.c:558-652`
- Deleted `parse_lspci_link_line` shell parser.
- Added `read_sysfs_gt`, `read_sysfs_uint`, `query_sysfs_pcie_link` reading:
  - `/sys/bus/pci/devices/<pdev>/current_link_speed` (e.g. `"16 GT/s PCIe"`)
  - `/sys/bus/pci/devices/<pdev>/current_link_width`
  - Fallback `max_link_*` (LnkCap)
- Kept `query_lspci_link` as wrapper → `query_sysfs_pcie_link` for API compat.
- Cache still `5.0s` via `update_lspci_cache:634`.

### `src/extract_gpuinfo_amdgpu.c:318-413`
- `authenticate_drm`: removed `perror`+`fgetc(stdin)`, now `fprintf` non-blocking.
- `initDeviceSysfsPaths`:
  ```c
  if (sysfsFD <0) return;
  if (pcieBWFD >=0) { PCIeBW=fdopen(...); if(!PCIeBW) close(pcieBWFD); }
  if (hwmonFD <0) return;
  ```

### `src/interface.c:1018`
- `compare_process_type_asc` fixed to delegate correctly.

### `include/nvtop/extract_gpuinfo_common.h:239`
- Inline now:
  ```c
  if (time_between==0) return 0;
  if (current < previous) return 0;
  delta = min(delta, time_between);
  ```

### `src/plot.c:49,118`
- Loop: `size_t draw_cols = min(num_data, (size_t)cols); for (i=0; i+num_lines <= draw_cols; i+=num_lines)` + `if (i+k>=num_data) continue;`
- Legend: `mvwprintw(..., cols, legend[i])` not `length-cols`.

### `src/rocm_smi_utils.c:1` & `src/extract_gpuinfo_amdgpu.c:1195`
- Translated comments, `RSMI_CU_OCCUPANCY_INVALID`.
- `snprintf(name, len,"%s",temp_name)` vs `strncpy`.
- Augmented `gpuinfo_amdgpu_get_running_processes` to only apply ROCm `cu_occupancy` when `!GPUINFO_PROCESS_FIELD_VALID(proc,gpu_usage)`.

### Build
- `CMakeLists.txt:7` `project(amdtop VERSION 1.1.2)`.
- `src/CMakeLists.txt:95` simplified `find_path`/`find_library` to `/opt/rocm` + `rocm_smi/include`.

---

## 3. Testing

**Build matrix (Ubuntu 26.04, gcc 15.2.0, libdrm 2.4.134, ncursesw6):**
```bash
# headers extracted via dpkg -x libudev-dev /tmp/test_libudev
cmake -S . -B /tmp/amdtop-build-final2 \
  -DUDEV_INCLUDE_DIR=/tmp/include_headers -DUDEV_LIBRARY=/usr/lib/x86_64-linux-gnu/libudev.so.1 \
  -DSYSTEMD_INCLUDE_DIR=/tmp/include_headers -DSYSTEMD_LIBRARY=/usr/lib/x86_64-linux-gnu/libsystemd.so.0 \
  -DLibdrm_INCLUDE_DIR=/usr/include -DLibdrm_LIBRARY=/usr/lib/x86_64-linux-gnu/libdrm.so \
  -DAMDGPU_SUPPORT=ON -DROCM_SMI_SUPPORT=OFF -DUSE_LIBUDEV_OVER_LIBSYSTEMD=ON
cmake --build /tmp/amdtop-build-final2 -j4
# → [100%] Linking C executable amdtop  BUILD_EXIT:0
/tmp/amdtop-build-final2/src/amdtop --help       # OK
/tmp/amdtop-build-final2/src/amdtop --version    # amdtop version ..
/tmp/amdtop-build-final2/src/amdtop --snapshot   # 2× R9700, 58-60C 235W 100% — SNAP_EXIT:0
```

**Isolated logic tests (compiled with `gcc`):**
- `test_fixes2.c` — `busy_usage`, `pcie_speed_from_gt`, `snprintf` truncation, `remaining_len`, plot bounds → **All fix tests PASSED**
- `test_sysfs2.c` — mock `/tmp/mock_pci/0000:03:00.0/current_link_speed` → gen4/16, fallback max → gen3/8 → **sysfs pcie test PASSED (no shell)**
- `test_compare.c` — `compare_process_type_asc` fixed inverts desc — **PASSED**
- `gcc -fsyntax-only` on `nvtop.c`, `extract_gpuinfo_amdgpu.c` (`-I/usr/include/libdrm`), `plot.c`, `interface.c`, `rocm_smi_utils.c` → no errors

**Pre-existing tests:**
- `tests/interfaceTests.cpp` not built (requires `GTest_DIR`, VLA in `interface_layout_selection.c:28` is C99-only, fails under `g++`). Unrelated to this patch.

**Artifacts cleaned:**
- `rm libsystemd-dev_*.deb libgtest-dev_*.deb extract2/ /tmp/amdtop-build* /tmp/mock_pci` → `git status` clean.

---

## 4. Enforcement

Per `AGENTS.md:1` all changes committed only to `fork` (`prcoe1/amdtop`):
```
6b8e548 Add AGENTS.md
bcac548 fix: address code review issues (11 files, 161+/123-)
git push fork master → 6b8e548..bcac548 master -> master
origin (HUSRCF/amdtop) never pushed
```

## 5. Remaining Notes

- `src/nvtop.c:300` `system("snap connect amdtop:%s")` in `--init` is controlled (static `interfaces[]` 4 entries, `snprintf` bounded) — retained.
- `src/interface_ring_buffer.c:9` VLA `sizeof(unsigned[devices_count]…)` is intentional C99; no change.
- JSON from `--snapshot` is malformed (missing commas) — pre-existing, out of scope.
