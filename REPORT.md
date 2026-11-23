# eSim Installation Report: Ubuntu 25.04+ (Task 4)

**Submission:** eSim Semester Long Internship, Autumn 2026, Task 4
**Author:** siyana8921
**Fork:** https://github.com/siyana8921/eSim (branch: `installer`)

---

## 1. Environment

| Item | Value |
|---|---|
| Host OS | Windows 11 |
| Hypervisor | Oracle VirtualBox |
| Guest OS | Ubuntu 26.04.1 LTS "resolute" (checked with `lsb_release -a`) |
| VM resources | 2 CPUs, 4096 MB RAM, 16 MB video memory (VMSVGA) |
| eSim version | 2.6 (latest `master`, commit `d86c1891` "Add release notes for version 2.6") |
| KiCad | 9.0.8 (kicad-symbols 9.0.7) |

> **Note:** the task asks for eSim 2.5 on Ubuntu 25.04. I tested **eSim 2.6 on Ubuntu 26.04.1 LTS**, because that is the current eSim release and the Ubuntu version I had installed. (VirtualBox's VM label still says "Ubuntu 25.04"; that is only the VM type setting, not the installed OS.) The issues below are still the kind that affect Ubuntu 25.04 and newer.

## 2. Method

1. Forked the eSim repository and checked out the `installer` branch.
2. Ran the installer with output captured: `./install-eSim.sh 2>&1 | tee install.log`.
3. For each failure: recorded the error, identified the root cause, patched the script or data files, and re-ran.
4. Launched eSim after installation and tested the KiCad schematic editor with the eSim symbol libraries.

## 3. Issues Found and Fixed (committed to the repo, commit `8f7d2664`)

### Bug 1: Missing header in `eSim_PSpice.lib`

- **Symptom:** KiCad reports "not a valid symbol library" when loading `eSim_PSpice`.
- **Root cause:** The library file is missing the required header line, so KiCad's parser rejects it.
- **Fix:** Added the header line `EESchema-LIBRARY Version 2.3` as the first line of the file.
- **Commit:** `8f7d2664` ("Fix legacy Kicad libs: add missing header to eSim_PSpice.lib, fill blank reference fields")
- **Files changed:** `library/kicadLibrary/eSim-symbols/legacy/eSim_PSpice.lib` and the converter copy `src/converter/schematic_converters/ConvertedLibs/eSim_PSpice.lib`

### Bug 2: Blank reference field in `eval.lib` and other legacy libraries

- **Symptom:** KiCad reports an "invalid pin offset" or parse error when loading `eval` and similar legacy libraries.
- **Root cause:** Legacy symbol definitions contain an empty reference (`F0`) field that newer KiCad versions cannot parse.
- **Fix:** In each affected symbol, the `DEF` line and the `F0` field were changed from an empty reference to `U`. Example (`HDR20_PSPICE`):
  ```
  -DEF HDR20_PSPICE 0 30 Y Y 1 F N
  -F0 "" 0 0 30 H V L CNN
  +DEF HDR20_PSPICE U 0 30 Y Y 1 F N
  +F0 "U" 0 0 30 H V L CNN
  ```
  The same change was applied to `vdb_PSPICE`, `idb_PSPICE`, `vphase_PSPICE` and others.
- **Commit:** `8f7d2664` (same commit as Bug 1; 7 files changed, 150 insertions, 148 deletions)
- **Files changed:** `eSim_PSpice.lib`, `eval.lib`, `marker.lib`, `port.lib`, `special.lib`, `wizard.lib` (all under `library/kicadLibrary/eSim-symbols/legacy/`) and the converter copy of `eSim_PSpice.lib`

## 4. Observations

### VM freezing

During testing, the VM froze or became very slow, particularly when opening KiCad's symbol chooser.

- **Likely cause:** low resources (4 GB RAM, 16 MB video memory) combined with KiCad's first-run library indexing.
- This is an environment limitation and not a bug in the installer.

## 5. Open Issues (not resolved in the repo)

### Bug 4: KiCad symbol libraries fail to load in the schematic editor

- **Symptom:** After the fixes above, the retest of the schematic editor still could not load or use the eSim symbol libraries.
- **Investigation:** `~/.config/kicad/9.0/sym-lib-table` contained entries pointing to files that do not exist (`pspice` and `MCU_ST_STM32L4+`). A manual workaround was to back up the file and delete these two lines with `sed`.
- **Status:** **Unresolved.** The retest did not confirm that the libraries work, so this is reported as an open issue and not as a fix. No change for this issue is included in the repository.
- **Suggested next steps:**
  - Retest with more VM resources (6 GB RAM, 128 MB video memory).
  - Check that the library paths in the sym-lib-table match the installed files.
  - Consider an installer step that validates the table after KiCad's first run.

### Bug 3: Missing `last_project.json` on first launch

- **Symptom:** eSim errors out on first launch because `~/.esim/last_project.json` does not exist.
- **Root cause:** the file is only written after a project is opened, but it is read at startup.
- **Status:** **Reported, not fixed in the repository.** I only worked around it locally on my VM. No commit for this issue exists on the `installer` branch.
- **Suggested fix:** create the file with default contents in the installer, or make the code handle a missing file.

## 6. Summary

| # | Issue | Severity | Status |
|---|---|---|---|
| 1 | Missing header in `eSim_PSpice.lib` | High (breaks library loading) | Fixed, committed |
| 2 | Blank reference field in legacy libs | High | Fixed, committed |
| 3 | Missing `last_project.json` | Medium (first launch) | Reported, local workaround only |
| 4 | KiCad library table / library loading | High | Open, documented |

## 7. How to Reproduce

```bash
git clone https://github.com/siyana8921/eSim.git
cd eSim
git checkout installer
chmod +x install-eSim.sh
./install-eSim.sh 2>&1 | tee install.log
```

## 8. Screenshot

- [ ] `git show --stat 8f7d2664` output

