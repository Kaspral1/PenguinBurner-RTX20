# NVIDIA RTX 20 & GTX 16 Series (Turing) Support in PenguinBurner

This repository extends **PenguinBurner** to fully support **NVIDIA GeForce RTX 20-series** and **GTX 16-series (Turing architecture)** graphics cards, including mobile/laptop variants, as well as the **RTX 2050 (Ampere GA107)**.

---

## 1. Background & Root Cause Analysis

In upstream PenguinBurner, running Auto-UV on Turing cards resulted in the following initial check failure:
```text
Auto-UV initial check failed.
Detected GPU: NVIDIA GeForce RTX 2060 (driver 610.xx, architecture unknown (6))
Errors:
- Invalid V/F curve points were reported: The V/F curve contains zero or negative voltage/frequency points: index=0 voltage=450mV freq=405MHz base=0MHz...
- V/F curve does not look usable for Auto-UV: PenguinBurner did not find enough plausible voltage/frequency points in the 600-1300mV range.
```

### Technical Root Causes:
1. **Unpopulated `vf_tuple_base` in NVAPI on Turing:**
   - On Ampere (RTX 30), Ada Lovelace (RTX 40), and Blackwell (RTX 50), NVIDIA drivers populate both current clock/voltage and a separate baseline tuple (`vf_tuple_base`) in `ClockClientClkVfPointsStatusV3`.
   - On Turing (RTX 20 / GTX 16), the driver returns valid live points (128 voltage/frequency points spanning 450 mV to 1243 mV), but leaves `vf_tuple_base` zeroed (`b_vf_tuple_base_supported == 0`).
   - Upstream PenguinBurner strictly asserted `base_freq_khz > 0`, causing it to reject otherwise completely healthy Turing V/F curves.
2. **Missing Turing Architecture Mapping:**
   - NVML reports architecture code `6` for Turing. The codebase lacked `NVML_DEVICE_ARCH_TURING = 6`, causing the initial check to label the GPU as `architecture unknown (6)`.
3. **Missing Target Ladders in `_UV_LIMIT_TARGETS`:**
   - Pre-optimized voltage/frequency ladders existed only for 30, 40, and 50 series cards.
4. **Working Directory Module Shadowing:**
   - Subprocesses launched with `python -m runtime.daemon_client` inherited the parent working directory in `sys.path[0]`. If launched from a folder containing a local file named `profiles.py`, Python shadowed the internal `profiles` package with `error: No module named 'profiles.uv'; 'profiles' is not a package`.

---

## 2. Implemented Modifications

### A. Fallback Baseline Derivation for Turing
- **Files:** `drivers/nvidia/daemon_gpu.py`, `runtime/support/nvidia_runtime_defaults.py`, `auto_uv/initial_check/auto_uv_hardware_initial_check.py`
- If `base_freq_khz <= 0`, it is dynamically calculated as:
  $$\text{base\_freq\_khz} = \max(\text{freq\_khz} - \text{current\_offset\_khz}, 0)$$
- `base_voltage_uv` defaults to `voltage_uv`.
- `_validate_vf_curve` accepts valid points using this effective base clock calculation.

### B. Turing Architecture Recognition
- **File:** `auto_uv/initial_check/auto_uv_hardware_initial_check.py`
- Defined `NVML_DEVICE_ARCH_TURING = 6` and added `"Turing"` to `NVML_DEVICE_ARCH_NAMES`.

### C. Pre-optimized Auto-UV Targets for RTX 20 & GTX 16 Series
- **File:** `auto_uv/scan_mode/uv_limits.py`
- Added target profiles tailored for the entire generation:
  - **RTX 2080 Ti:** Efficiency: 775 mV / 1600 MHz | Balanced: 825 mV / 1750 MHz | Performance: 875 mV / 1850 MHz
  - **RTX 2080 Super:** Efficiency: 750 mV / 1530 MHz | Balanced: 800 mV / 1700 MHz | Performance: 875 mV / 1815 MHz
  - **RTX 2080:** Efficiency: 750 mV / 1500 MHz | Balanced: 800 mV / 1680 MHz | Performance: 875 mV / 1800 MHz
  - **RTX 2070 Super:** Efficiency: 750 mV / 1470 MHz | Balanced: 800 mV / 1650 MHz | Performance: 875 mV / 1770 MHz
  - **RTX 2070:** Efficiency: 750 mV / 1450 MHz | Balanced: 800 mV / 1620 MHz | Performance: 875 mV / 1750 MHz
  - **RTX 2060 Super:** Efficiency: 750 mV / 1440 MHz | Balanced: 800 mV / 1600 MHz | Performance: 875 mV / 1740 MHz
  - **RTX 2060 (Desktop & Mobile):** Efficiency: 750 mV / 1425 MHz | Balanced: 800 mV / 1575 MHz | Performance: 875 mV / 1725 MHz
  - **RTX 2050 (Mobile, Ampere GA107):** Efficiency: 725 mV / 1350 MHz | Balanced: 775 mV / 1475 MHz | Performance: 825 mV / 1600 MHz
  - **GTX 1660 Ti & Super:** Efficiency: 750 mV / 1500 MHz | Balanced: 800 mV / 1650 MHz | Performance: 875 mV / 1770 MHz
  - **GTX 1660 & 1650 Super:** Efficiency: 750 mV / 1450 MHz | Balanced: 800 mV / 1600 MHz | Performance: 875 mV / 1725 MHz
  - **GTX 1650:** Efficiency: 750 mV / 1400 MHz | Balanced: 800 mV / 1550 MHz | Performance: 850 mV / 1665 MHz

### D. Subprocess Path Isolation
- **Files:** `ui/commands.py`, `runtime/daemon_client.py`
- Added `-P` (safe path) to all internal `python -m` invocations.
- Stripped current working directory from `sys.path[0]` on entry to `runtime/daemon_client.py`.

---

## 3. Real-World Test Results

Tested on **NVIDIA GeForce RTX 2060 Mobile (80W TGP)** running on **Linux Mint 22.3 (driver 610.43.02)**:

| Metric | Stock (Unmodified) | Auto-UV Performance (762 mV) | Auto-UV Efficiency (750 mV) |
| :--- | :--- | :--- | :--- |
| **Voltage Under Load** | 830 – 868 mV (bouncing) | **762 mV (stable)** | **750 mV (stable)** |
| **Power Consumption** | 78.0 – 80.5 W (power throttling) | **71.1 W (-11.1%)** | **69.9 W (-12.2%)** |
| **Effective Clock** | 1500 – 1560 MHz | **1660.88 MHz (+100 MHz)** | **1664.64 MHz (+104 MHz)** |
| **Max Temperature** | 76 – 82°C | **68 – 70°C** | **65 – 68°C** |
| **Stability Test** | N/A | **PASS (300s Quake II RTX + CUDA)** | **PASS (300s Quake II RTX + CUDA)** |

*Observation: Lowering the voltage eliminates power-budget throttling on laptop GPUs, allowing the GPU core to sustain higher effective boost clocks than stock settings.*

---

## 4. Installation from Source

To install this modified version from this repository:

```bash
# Clone the repository
git clone https://github.com/Kaspral1/PenguinBurner-RTX20.git
cd PenguinBurner-RTX20

# Install using pipx (recommended on modern distros)
pipx install --force .

# Or using user pip:
python3 -m pip install --user --upgrade .
```

To run the application:
```bash
penguin-burner
```
