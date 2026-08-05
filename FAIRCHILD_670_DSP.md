# Fairchild 670 DSP — Notas Técnicas de Referencia Rápida

## Valores Clave del Circuito Original

### Tubos
| Tubo | Qty | Función | Mu | Rp | Notas |
|------|-----|---------|-----|-----|-------|
| 6386 | 8 | Variable-Mu (core) | ~17 | 100kΩ | Remote-cutoff, x=1.0 |
| 12AX7 | 2 | Pre-amp / Sidechain | 100 | — | High-mu twin triode |
| 6973 | 4 | Output push-pull | — | — | Beam tetrode |
| 12BH7 | 2 | Driver | ~20 | — | Medium-mu dual triode |
| E80F | 1 | Sidechain amp | — | — | Low-noise pentode |
| 5651 | 1 | Voltage reference | — | — | Gas regulator |
| EL34 | 1 | Power / B+ reg | — | — | Power tetrode |
| GZ34 | 1 | Rectifier | — | — | Full-wave, 5AR4 |

### Valores Eléctricos por Canal
```
B+ Voltage:     250V DC (Pin 8 de V302/EL34)
Plate Resistor: 100kΩ (varía por etapa)
Cathode Res:    1.5kΩ (varía por etapa)
Cathode Bypass: Variable (0-47µF en some versions)
```

### Transformadores (por canal)
```
T101/T201 Input:        600Ω → 50kΩ,    Ratio 1+1:9+9
T102/T202 Output:       60kCT → 600Ω,   Ratio 9+9:1+1
T103/T203 Control In:   600Ω → 170kΩ,   Ratio 17+17:1+1
T104/T204 Control Out:  10kCT → 600Ω,   Ratio 4:1
```

### Alimentación
```
T301 Bias Supply:    26.8V (tapped 24V for Si) @ 200mA
T302 Mains:          375-0-375V @ 200mA, 6.3VCT @ 5A, 5V @ 2A
L301 HT Choke:       10H, 200mA, 71Ω
L302 Bias Choke:     5H, 200mA, 85Ω
```

## Time Constants (Valores exactos del manual)

| Pos | Kind | Attack | Release | Notas |
|-----|------|--------|---------|-------|
| 1 | Fixed | 0.2ms | 0.3s | Fastest |
| 2 | Fixed | 0.2ms | 0.8s | Fast |
| 3 | Fixed | 0.2ms | 2.0s | Medium |
| 4 | Fixed | 0.2ms | 5.0s | Slow |
| 5 | Auto | 0.2ms | Fast/Slow | Program-dependent |
| 6 | Auto | 0.2ms | Fast/Slow | Slowest, most transparent |

### Auto-Release Dual Branch Model
```
Pos 5: fast_τ = 0.3s, slow_τ = 8.0s
Pos 6: fast_τ = 0.5s, slow_τ = 15.0s

CV_output = max(cv_fast, cv_slow)

Brief transients → fast branch decays → quick recovery
Sustained loud → slow branch persists → slow recovery
```

## Especificaciones de Audio

```
Input Impedance:    600Ω
Output Impedance:   600Ω
Input Level:        0 dBm to +16 dBm
Output Level:       +4 or +8 VU (+27 dBm clipping)
Gain:               +7 dB (no limiting)
Freq Response:      40 Hz - 15 kHz ±1 dB
Noise:              -70 dB below +4 dBm
IM Distortion:      <1% any level up to +18 dBm out (no limiting)
                    <1% at +12 dBm out (limiting)
Attack Time:        0.2-0.4ms (adjustable, extremely fast)
Release Time:       0.2s to 25s (6 positions)
Compression Ratio:  1:2 to 1:30 (variable, function of threshold)
Separation:         60 dB (A-B position)
```

## Modelo del 6386 (Koren adapted)

```
Para remote-cutoff (6386), usar exponente x = 1.0:
  Ip = k * (E₁)^1.0 * ln(1 + exp(1/E₁^1.0))
  donde E₁ = mu * Vg + Vp

  x = 1.0 → Ip ∝ E₁ → soft, gradual gain reduction
  x = 1.4 → Ip ∝ E₁^1.4 → sharp cutoff (NOT what we want)

LUT mapping:
  CV_norm = 0 → gain = 1.0 (unity, no compression)
  CV_norm = 1 → gain ≈ 0.05 (heavy compression)
  Curve: gain = 1 / (1 + |Vg|^x * k)
```

## Sidechain Detector

```
Full-wave rectifier (6AL5 twin diode):
  Vforward ≈ 0.8V (soft onset)
  
  output = abs(input) > 0.8 ? abs(input) - 0.8 : 0;

Envelope follower (RC):
  attack_coeff = exp(-1 / (srate * 0.0002))  // 0.2ms
  release_coeff = exp(-1 / (srate * release_time))

  env = coeff * env + (1 - coeff) * target;
  
  if target > env: use attack_coeff (fast)
  if target < env: use release_coeff (slow)
```

## Transformer Emulation (Simplified)

```
Each transformer modeled as:
  1. 2nd-order biquad cascade (HPF + LPF)
  2. Soft saturation (tanh with drive)

Input transformer:
  HPF: 30 Hz, Q=0.707
  LPF: 30 kHz, Q=0.707
  Drive: 1.2

Output transformer:
  HPF: 25 Hz, Q=0.707
  LPF: 22 kHz, Q=0.707
  Drive: 1.1
  Saturator: tanh

Interstage:
  HPF: 25 Hz
  LPF: 22 kHz
  Drive: 1.1
```

## L/V Matrix (Mid/Side for Stereo Disc Cutting)

```
Encoding:
  Lateral (Mid)  = (L + R) / 2
  Vertical (Side) = (L - R) / 2

Decoding:
  L = (Lat + Vert) / 2
  R = (Lat - Vert) / 2

Processing:
  Lat_out = compress(lateral, cv_lat)
  Vert_out = compress(vertical, cv_vert)
```

## Plugin Parameter Mapping

```
Original Hardware → JSFX Mapping:
  Input Level Control    → slider1/5 (Input Gain dB)
  Limiting Threshold     → slider2/6 (Threshold %)
  Time Constant Switch   → slider3/7 (1-6 enum)
  Metering Switch        → (automatic in @gfx)
  AC Threshold (front)   → slider2/6 (exposed as Threshold)
  DC Threshold (internal) → fixed at factory default (or new slider)
  L/V Switch             → slider9 (Mode enum)
  Bypass                 → slider11 (On/Off)
  
Modern Extensions (not in original):
  Output Gain Makeup     → slider4/8 (dB)
  Mix/Dry-Wet            → slider10 (%)
  Sidechain HPF          → future addition
```

## CPU Budget (per instance @ 48kHz/512)

```
Variable-Mu stage:     ~1.0%  (LUT lookup + tanh)
Sidechain detector:    ~0.5%  (rect + envelope)
Time constants:        ~0.1%  (coeff selection)
Transformers (×4):     ~1.0%  (biquads + saturation)
Stereo matrix:         ~0.1%  (simple math)
UI (@gfx):             ~0.5%  (when visible)
─────────────────────────────
Total estimated:       ~3.2%  (target: <3%)

With 8 instances:      ~25%   (acceptable)
```

## Reference: phu-fair-kid-67 Plugin Parameters

Del repositorio GitHub del modelo DSP de referencia:
```
VariableMuPushPullStage:
  Vcc = 250V
  Rp = 100kΩ
  Rk = 1.5kΩ
  Koren model x = 1.0 (remote-cutoff)

Sidechain:
  SoftRectifierDetector: Vf ≈ 0.8V
  One-pole IIR smoothing
  cvMaxV = 6V (estimated)

Transformers:
  Input:  HPF=30Hz, LPF=30kHz, drive=1.2
  Interstage: HPF=25Hz, LPF=22kHz, drive=1.1
  Output: 2nd-order biquad, Q=0.7071, tanh saturator
```

---
*Referencia rápida para desarrollo — Agosto 2026*
