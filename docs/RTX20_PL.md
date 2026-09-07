# Wsparcie dla kart NVIDIA RTX serii 20 (architektura Turing) w PenguinBurner

Ten fork rozszerza aplikację **PenguinBurner** o pełną obsługę kart graficznych **NVIDIA GeForce RTX z serii 20 (architektura Turing)**, takich jak RTX 2060, RTX 2070 oraz RTX 2080 (w tym wersji laptopowych / Mobile).

---

## 1. Przyczyna braku działania w wersji oficjalnej

W oficjalnej wersji PenguinBurner próba uruchomienia procedury Auto-UV na kartach z rodziny Turing kończyła się błędem:
```text
Auto-UV initial check failed.
Detected GPU: NVIDIA GeForce RTX 2060 (driver 610.xx, architecture unknown (6))
Errors:
- Invalid V/F curve points were reported: The V/F curve contains zero or negative voltage/frequency points: index=0 voltage=450mV freq=405MHz base=0MHz...
- V/F curve does not look usable for Auto-UV: PenguinBurner did not find enough plausible voltage/frequency points in the 600-1300mV range.
```

### Przyczyny techniczne:
1. **Brak zwracania krotki `vf_tuple_base` przez sterownik dla Turinga:**
   - Na architekturach Ampere (RTX 30), Ada Lovelace (RTX 40) oraz Blackwell (RTX 50) sterownik NVIDIA przekazuje zarówno aktualny zegar/napięcie punktu, jak i osobną bazę fabryczną (`vf_tuple_base`).
   - Na architekturze Turing sterownik zwraca w pełni poprawną krzywą V/F (128 punktów w zakresie od 450 mV do 1243 mV), ale pole `vf_tuple_base` pozostawia wyzerowane (`base_freq_khz = 0`).
   - Oryginalny kod walidacji PenguinBurner sztywno wymagał `base_freq_khz > 0`, co powodowało fałszywe odrzucanie w pełni sprawnej krzywej.
2. **Brak mapowania architektury Turing:**
   - Sterownik NVML raportuje dla Turinga identyfikator `6`. W kodzie brakowało definicji `NVML_DEVICE_ARCH_TURING = 6`, co skutkowało komunikatem `architecture unknown (6)`.
3. **Brak tabeli docelowych profili dla serii 20:**
   - Tabela celów Auto-UV (`_UV_LIMIT_TARGETS`) zawierała wyłącznie wpisy dla serii 30, 40 i 50.
4. **Przysłanianie modułów przez katalog roboczy (Module Shadowing):**
   - Podprocesy uruchamiane przez interfejs graficzny (`python -m runtime.daemon_client`) dziedziczyły katalog roboczy w `sys.path[0]`. Uruchomienie programu z katalogu zawierającego lokalny plik `profiles.py` powodowało konflikt importu z wewnętrznym pakietem `profiles`.

---

## 2. Wprowadzone rozwiązania

1. **Obliczanie parametrów bazowych dla kart Turing:**
   - W plikach `drivers/nvidia/daemon_gpu.py`, `runtime/support/nvidia_runtime_defaults.py` oraz `auto_uv/initial_check/auto_uv_hardware_initial_check.py`: jeśli `base_freq_khz <= 0`, przyjmuje się rzeczywisty zegar bazowy punktu:
     $$\text{base\_freq\_khz} = \max(\text{freq\_khz} - \text{current\_offset\_khz}, 0)$$
     a `base_voltage_uv` przyjmuje wartość `voltage_uv`.
2. **Rozpoznawanie architektury Turing:**
   - Dodano stałą `NVML_DEVICE_ARCH_TURING = 6` oraz nazwę `"Turing"` do bazy `NVML_DEVICE_ARCH_NAMES`.
3. **Predefiniowane profile Auto-UV dla serii RTX 20:**
   - Zdefiniowano profile celów dla RTX 2060, RTX 2070 i RTX 2080:
     - **RTX 2060:** Efficiency: 750 mV / 1425 MHz | Balanced: 800 mV / 1575 MHz | Performance: 875 mV / 1725 MHz
     - **RTX 2070:** Efficiency: 750 mV / 1450 MHz | Balanced: 800 mV / 1620 MHz | Performance: 875 mV / 1750 MHz
     - **RTX 2080:** Efficiency: 750 mV / 1500 MHz | Balanced: 800 mV / 1680 MHz | Performance: 875 mV / 1800 MHz
     - Limit mocy: 100% (bezpieczny dla zablokowanych TGP w laptopach).
4. **Izolacja importów Pythona:**
   - Dodano flagę `-P` do podprocesów oraz usuwanie katalogu roboczego z `sys.path[0]` na wejściu demona klienta.

---

## 3. Wyniki testów rzeczywistych (RTX 2060 Mobile 80W)

Testy przeprowadzone na **NVIDIA GeForce RTX 2060 Mobile (TGP 80W)**, Linux Mint 22.3 (sterownik 610.43.02):

| Parametr | Ustawienia fabryczne | Profil Performance (762 mV) | Profil Efficiency (750 mV) |
| :--- | :--- | :--- | :--- |
| **Napięcie pod obciążeniem** | ~830 – 868 mV (skoki) | **762 mV (stabilne)** | **750 mV (stabilne)** |
| **Pobór mocy** | 78.0 – 80.5 W (uderza w limit) | **71.1 W (-11.1%)** | **69.9 W (-12.2%)** |
| **Taktowanie rzeczywiste** | 1500 – 1560 MHz | **1660.88 MHz (+100 MHz)** | **1664.64 MHz (+104 MHz)** |
| **Temperatura maksymalna** | 76 – 82°C | **68 – 70°C** | **65 – 68°C** |
| **Test stabilności** | — | **PASS (300 s Quake II RTX + CUDA)** | **PASS (300 s Quake II RTX + CUDA)** |

*Wniosek: Obniżenie napięcia na kartach mobilnych eliminuje zjawisko power-throttlingu, dzięki czemu GPU osiąga wyższy stały zegar niż fabrycznie przy znacznie niższych temperaturach.*

---

## 4. Instalacja ze źródeł

```bash
git clone https://github.com/Kaspral1/PenguinBurner-RTX20.git
cd PenguinBurner-RTX20
pipx install --force .
```

Uruchomienie:
```bash
penguin-burner
```
