# Wsparcie dla kart NVIDIA RTX serii 20 oraz GTX serii 16 w PenguinBurner

Ten fork rozszerza aplikację **PenguinBurner** o pełną obsługę kart graficznych **NVIDIA GeForce RTX z serii 20** oraz **GTX z serii 16 (architektura Turing)**, a także układu **RTX 2050 (architektura Ampere GA107)**.

---

## 1. Dlaczego oficjalna wersja nie działała?

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
3. **Brak tabeli docelowych profili dla serii 20 i 16:**
   - Tabela celów Auto-UV (`_UV_LIMIT_TARGETS`) zawierała wyłącznie wpisy dla serii 30, 40 i 50.
4. **Przysłanianie modułów przez katalog roboczy (Module Shadowing):**
   - Podprocesy uruchamiane przez interfejs graficzny (`python -m runtime.daemon_client`) dziedziczyły katalog roboczy w `sys.path[0]`. Uruchomienie programu z katalogu zawierającego lokalny plik `profiles.py` powodowało konflikt importu z wewnętrznym pakietem `profiles`.

---

## 2. Obsługiwane modele

W forku dodano zoptymalizowane profile dla całej generacji Turing i pokrewnych kart:
- **RTX 2080 Ti:** Efficiency: 775 mV / 1600 MHz | Balanced: 825 mV / 1750 MHz | Performance: 875 mV / 1850 MHz
- **RTX 2080 Super:** Efficiency: 750 mV / 1530 MHz | Balanced: 800 mV / 1700 MHz | Performance: 875 mV / 1815 MHz
- **RTX 2080:** Efficiency: 750 mV / 1500 MHz | Balanced: 800 mV / 1680 MHz | Performance: 875 mV / 1800 MHz
- **RTX 2070 Super:** Efficiency: 750 mV / 1470 MHz | Balanced: 800 mV / 1650 MHz | Performance: 875 mV / 1770 MHz
- **RTX 2070:** Efficiency: 750 mV / 1450 MHz | Balanced: 800 mV / 1620 MHz | Performance: 875 mV / 1750 MHz
- **RTX 2060 Super:** Efficiency: 750 mV / 1440 MHz | Balanced: 800 mV / 1600 MHz | Performance: 875 mV / 1740 MHz
- **RTX 2060 (Desktop & Laptop):** Efficiency: 750 mV / 1425 MHz | Balanced: 800 mV / 1575 MHz | Performance: 875 mV / 1725 MHz
- **RTX 2050 (Mobile, rdzeń Ampere GA107):** Efficiency: 725 mV / 1350 MHz | Balanced: 775 mV / 1475 MHz | Performance: 825 mV / 1600 MHz
- **GTX 1660 Ti & 1660 Super:** Efficiency: 750 mV / 1500 MHz | Balanced: 800 mV / 1650 MHz | Performance: 875 mV / 1770 MHz
- **GTX 1660 & 1650 Super:** Efficiency: 750 mV / 1450 MHz | Balanced: 800 mV / 1600 MHz | Performance: 875 mV / 1725 MHz
- **GTX 1650:** Efficiency: 750 mV / 1400 MHz | Balanced: 800 mV / 1550 MHz | Performance: 850 mV / 1665 MHz

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
