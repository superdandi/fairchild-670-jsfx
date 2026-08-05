# Fairchild 670 JSFX — Plan de Desarrollo por Fases

## Nombre del Plugin
```
desc:Fairchild 670 Vari-Mu Compressor
tags: compressor limiter tube vari-mu fairchild stereo mastering
author: [Tu Nombre]
// @version 0.1.0
```

## Estructura del Proyecto

```
fairchild-670-jsfx/
├── DEVELOPMENT_PLAN.md          ← Este documento
├── FAIRCHILD_670_DSP.md         ← Notas técnicas DSP del Fairchild
├── Fairchild670.jsfx            ← Plugin principal (fase a fase)
├── fairchild670_dsp.jsfx-inc    ← Librería de DSP reutilizable
├── fairchild670_ui.jsfx-inc     ← Librería de UI @gfx
├── fairchild670_tubes.jsfx-inc  ← Modelado de tubos
├── fairchild670_xforms.jsfx-inc ← Emulación de transformadores
├── presets/                     ← Presets de fábrica
│   ├── Mix_Bus_Glue.fxb
│   ├── Vocal_Warmth.fxb
│   └── Mastering-transparent.fxb
└── ref/                         ← Material de referencia
    ├── Fairchild_670_Manual.pdf
    └── 6386_datasheet.pdf
```

---

## Fase 0: Setup del Proyecto y Research (1-2 días)

### Objetivo
Configurar el entorno de desarrollo, estructura de archivos, y validar que JSFX puede manejar la complejidad del proyecto.

### Tareas

- [ ] Crear estructura de directorios
- [ ] Crear archivo `.jsfx` principal con metadata completa
- [ ] Configurar `options:maxmem=8000000` y `options:prealloc=8000000`
- [ ] Crear librerías `.jsfx-inc` vacías con su cabecera
- [ ] Validar que REAPER carga el plugin (aparece en lista FX)
- [ ] Crear "Hello World" DSP: pass-through con ganancia slider
- [ ] Verificar funcionamiento en múltiples sample rates (44.1k, 48k, 96k)
- [ ] Documentar baseline de CPU Usage con plugin vacío

### Salida Esperada
Plugin pass-through funcional que aparece en REAPER, con estructura de archivos lista para desarrollo iterativo.

### Metadata del Plugin Principal
```javascript
desc:Fairchild 670 Vari-Mu Compressor v0.1
tags: compressor limiter tube vari-mu fairchild stereo mastering
author: [Tu Nombre]
options:maxmem=8000000 prealloc=8000000

import fairchild670_dsp.jsfx-inc
import fairchild670_ui.jsfx-inc
import fairchild670_tubes.jsfx-inc
import fairchild670_xforms.jsfx-inc
```

### Sliders Preliminares (Mapeo del Hardware)
```javascript
// --- Canal Izquierdo ---
slider1:0<-60,12,0.1>Input Gain L (dB)
slider2:50<0,100,0.1>Threshold L (%)
slider3:1<1,6,1{0.3s/0.8s, 0.8s/2s, 2s/5s, 5s/10s, Auto-Fast, Auto-Slow}>Time Constant L
slider4:0<0,100,0.1>Output Gain L (dB)

// --- Canal Derecho ---
slider5:0<-60,12,0.1>Input Gain R (dB)
slider6:50<0,100,0.1>Threshold R (%)
slider7:1<1,6,1{0.3s/0.8s, 0.8s/2s, 2s/5s, 5s/10s, Auto-Fast, Auto-Slow}>Time Constant R
slider8:0<0,100,0.1>Output Gain R (dB)

// --- Modo ---
slider9:0<0,2,1{Stereo,L/R Independent,Lateral-Vertical}>Mode
slider10:0<0,100,0.5>Mix / Dry-Wet %
slider11:0<0,1,1{Off,On}>Bypass
```

### Notas de Implementación
- Los 6 positions de Time Constant del original son:
  - Pos 1: Attack 0.2ms, Release 0.3s
  - Pos 2: Attack 0.2ms, Release 0.8s
  - Pos 3: Attack 0.2ms, Release 2.0s
  - Pos 4: Attack 0.2ms, Release 5.0s
  - Pos 5: Attack 0.2ms, Auto Release (fast/slow dual branch)
  - Pos 6: Attack 0.2ms, Auto Release (slowest, most transparent)
- El slider de "Threshold" es una extensión moderna (el original no tiene threshold, comprime cualquier señal sobre el bias point del tubo)

---

## Fase 1: Variable-Mu Gain Stage — El Corazón (3-5 días)

### Objetivo
Implementar la etapa de ganancia variable que es el corazón del sonido Fairchild. El 6386 remote-cutoff twin triode en configuración push-pull.

### Teoría

El Variable-Mu funciona así:
1. El tubo 6386 tiene una curva de transconductancia suave (remote-cutoff)
2. A medida que la grid se vuelve más negativa, la ganancia disminuye gradualmente
3. Una voltaje de control DC (del sidechain) aplicado a la grid reduce la ganancia
4. La topología push-pull cancela armónicos pares → baja distorsión
5. Nunca hace hard clipping — siempre suave y musical

### Modelo DSP del 6386

Usaremos el **modelo Koren** adaptado para remote-cutoff:

```
El modelo Koren para un triode:
  Ip = (mu * Vg + Vp) / Rp * (1 + k * ln(1 + exp(C * (mu * Vg + Vp) / Vp)))

Para remote-cutoff (6386), usar exponente x = 1.0 en lugar de x ≈ 1.4:
  Ip = k * (E₁)^x * ln(1 + exp(1/E₁^x))
  donde E₁ = mu * Vg + Vp (voltaje de grid efectivo)
  con x = 1.0 → suavidad remota
  con x = 1.4 → cutoff agudo
```

### Tareas

- [ ] Implementar función `variable_mu_stage()` en `fairchild670_tubes.jsfx-inc`
- [ ] Implementar topología push-pull (dos etapas con ±Vin)
- [ ] Crear lookup table para la curva del 6386 (pre-calculada en @init)
- [ ] Implementar gain normalization (unity gain cuando CV = 0)
- [ ] Calcular B+ voltage (250V default), Rp (100kΩ), Rk (1.5kΩ)
- [ ] Probar con señal seno a diferentes niveles de entrada
- [ ] Verificar que la curva de compresión es suave (sin hard knee)
- [ ] Medir CPU usage de la etapa Variable-Mu

### Estructura del Código

```javascript
// fairchild670_tubes.jsfx-inc
@init
// Modelo Koren para 6386 remote-cutoff
// Parámetros tubo
VAR_MU = 17;           // mu del 6386
VAR_RP = 100000;       // plate resistance (100kΩ)
VAR_RK = 1500;         // cathode resistance (1.5kΩ)
VAR_VCC = 250;         // B+ voltage
VAR_REMOTE_CUTOFF = 1; // x exponent (1.0 = remote, 1.4 = sharp)

function koren_tube(Vg, Vp, mu, rp, x) (
  Eg = mu * Vg + Vp;
  Eg > 0.001 ? (
    ip = (1/rp) * Eg * (1 + 0.3 * log(1 + exp(Eg * 10 / max(Vp, 0.1))));
  ) : (
    ip = 0;
  );
  ip;
);

// Lookup table para curva 6386 (pre-calculada en @init)
tube_lut_size = 1024;
tube_lut = 0;
freembuf(tube_lut + tube_lut_size);

// Llenar LUT: grid voltage (-10V a 0V) → ganancia normalizada
function build_tube_lut() (
  i = 0;
  while (i < tube_lut_size) (
    vg_norm = i / tube_lut_size; // 0 a 1
    vg = vg_norm * -10;          // 0V a -10V
    // Curva remote-cutoff: ganancia ~ 1/(1 + |Vg|^x)
    gain = 1 / (1 + abs(vg)^VAR_REMOTE_CUTOFF * 0.1);
    tube_lut[i] = gain;
    i += 1;
  );
);

build_tube_lut();

// Función: Obtener ganancia del tubo desde CV
function tube_gain(cv_norm) (
  // cv_norm: 0 = sin compresión, 1 = compresión máxima
  idx = cv_norm * (tube_lut_size - 1);
  idx < 0 ? idx = 0;
  idx >= tube_lut_size ? idx = tube_lut_size - 1;
  idx_floor = floor(idx);
  idx_frac = idx - idx_floor;
  idx_next = idx_floor + 1;
  idx_next >= tube_lut_size ? idx_next = tube_lut_size - 1;
  // Interpolación lineal
  tube_lut[idx_floor] * (1 - idx_frac) + tube_lut[idx_next] * idx_frac;
);

// Función: Push-pull Variable-Mu stage
// input_l, input_r = señal de entrada
// cv = voltaje de control (0-1, viene del sidechain)
// Retorna: señal procesada
function var_mu_pp(input_l, input_r, cv) (
  // Gain desde LUT
  gain_l = tube_gain(cv);
  gain_r = tube_gain(cv); // stereo linked o independiente

  // Push-pull: diferencia cancela armónicos pares
  out_l = input_l * gain_l;
  out_r = input_r * gain_r;

  // Agregar armónicos impares sutiles (color de tubo)
  // Modelo simplificado de saturación
  drive = 0.3;
  out_l = tanh(out_l * (1 + drive * cv));
  out_r = tanh(out_r * (1 + drive * cv));

  out_l + out_r; // retorno promedio push-pull
);
```

### Pruebas de Fase 1
1. **Señal seno 1kHz, 0 dBV** → Ganancia debe ser unity (sin CV)
2. **CV creciente** → Ganancia debe disminuir suavemente
3. **Señal fuerte + CV alto** → Sin clipping duro, saturación suave
4. **CPU:** Debe ser < 5% en @sample con este stage solo
5. **Verificar:** La curva de compresión tiene "soft knee" (nunca hard)

---

## Fase 2: Sidechain / Detector (2-3 días)

### Objetivo
Implementar el circuito de detección que mide el nivel de la señal de entrada y genera el voltaje de control DC para el Variable-Mu stage.

### Teoría del Sidechain Fairchild

El sidechain del Fairchild 670 usa:
1. **Full-wave rectifier** con tubo 6AL5 twin diode
2. **Soft onset** (voltaje forward ~0.8V antes de que empiece a conducir)
3. **RC envelope follower** para suavizar el voltaje de detección
4. El voltaje suavizado se aplica a la grid del 6386

### Tareas

- [ ] Implementar full-wave rectifier con soft onset
- [ ] Implementar envelope follower (RC smoothing)
- [ ] Implementar attack/release coefficients por posición del Time Constant
- [ ] Crear función de suavizado program-dependent para posiciones 5 y 6
- [ ] Implementar el modelo dual-branch para auto-release
- [ ] Conectar sidechain con Variable-Mu stage
- [ ] Probar con transitorios rápidos (click, snare)
- [ ] Probar con material sostenido (pad, vocal)

### Estructura del Código

```javascript
// fairchild670_dsp.jsfx-inc
@init
// --- Sidechain Detector ---
sc_env_l = 0;      // envelope level canal L
sc_env_r = 0;      // envelope level canal R
sc_cv_l = 0;       // control voltage canal L
sc_cv_r = 0;       // control voltage canal R

// Constantes de los 6 positions
// [attack_ms, release_fixed_s, release_fast_s, release_slow_s, is_auto]
tc_attack_ms = 0.2; // Ataque siempre ~0.2ms

// Posiciones 1-4: fixed release
tc_release[0] = 0.3;   // Pos 1
tc_release[1] = 0.8;   // Pos 2
tc_release[2] = 2.0;   // Pos 3
tc_release[3] = 5.0;   // Pos 4

// Posiciones 5-6: auto release (dual branch)
tc_auto_fast[4] = 0.3;   // Pos 5 fast
tc_auto_slow[4] = 8.0;   // Pos 5 slow
tc_auto_fast[5] = 0.5;   // Pos 6 fast
tc_auto_slow[5] = 15.0;  // Pos 6 slow

// Coefficients calculados en @slider
sc_attack_coeff_l = 0;
sc_release_coeff_l = 0;
sc_auto_fast_coeff_l = 0;
sc_auto_slow_coeff_l = 0;
sc_env_fast_l = 0;
sc_env_slow_l = 0;

// Función: Soft onset rectifier (simula 6AL5)
function soft_rect(x) (
  // Voltaje forward ~0.8V
  x > 0.8 ? x - 0.8 : (x < -0.8 ? -(x + 0.8) : 0);
);

// Función: Full-wave rectifier
function fullwave_rect(x) (
  abs(x);
);

// Función: Envelope follower
function envelope_follow(input, env, attack_c, release_c) (
  target = fullwave_rect(input);
  target = soft_rect(target);
  coeff = target > env ? attack_c : release_c;
  coeff * env + (1 - coeff) * target;
);

// Función: Auto-release dual branch
function auto_release(env_fast, env_slow, input, fast_c, slow_c) (
  target = fullwave_rect(input);
  target = soft_rect(target);

  // Rama rápida
  coeff_fast = target > env_fast ? fast_c * 0.5 : fast_c;
  new_fast = coeff_fast * env_fast + (1 - coeff_fast) * target;

  // Rama lenta
  coeff_slow = target > env_slow ? slow_c * 0.3 : slow_c;
  new_slow = coeff_slow * env_slow + (1 - coeff_slow) * target;

  // Output = max de ambas ramas
  new_fast > new_slow ? new_fast : new_slow;
);

@slider
// Calcular coefficients desde time constant position
tc_pos = slider3 - 1; // 0-indexed

// Attack coefficient (siempre ~0.2ms)
sc_attack_coeff = exp(-1 / (srate * tc_attack_ms / 1000));

// Release coefficient
tc_pos < 4 ? (
  // Fixed release
  sc_release_coeff = exp(-1 / (srate * tc_release[tc_pos]));
  sc_is_auto = 0;
) : (
  // Auto release
  sc_auto_fast_coeff = exp(-1 / (srate * tc_auto_fast[tc_pos]));
  sc_auto_slow_coeff = exp(-1 / (srate * tc_auto_slow[tc_pos]));
  sc_is_auto = 1;
);

@sample
// Detectar nivel de entrada (mono sum para sidechain)
sc_input = 0.5 * (abs(spl0) + abs(spl1));

// Envelope follower
sc_is_auto ? (
  sc_cv_l = auto_release(sc_env_fast_l, sc_env_slow_l,
                         sc_input, sc_auto_fast_coeff, sc_auto_slow_coeff);
  sc_env_fast_l = ... ; // actualizar
  sc_env_slow_l = ... ; // actualizar
) : (
  sc_cv_l = envelope_follow(sc_input, sc_env_l,
                             sc_attack_coeff, sc_release_coeff);
  sc_env_l = sc_cv_l;
);

// Normalizar CV a rango 0-1 para el LUT del tubo
sc_cv_l = min(1, sc_cv_l * 5); // escalar a rango util
```

### Pruebas de Fase 2
1. **Señal constante** → CV debe estabilizarse suavemente
2. **Transitorio rápido (click)** → CV debe subir rápido, bajar según release
3. **Silencio → señal fuerte** → Attack debe ser ~0.2ms (muy rápido)
4. **Posiciones 5-6** → Recovery debe ser rápido para picos breves, lento para material sostenido
5. **Sin señal** → CV = 0 (sin ruido de fondo)

---

## Fase 3: Constantes de Tiempo (1-2 días)

### Objetivo
Implementar el selector de 6 posiciones de attack/release con valores exactos del manual original.

### Mapeo de Posiciones del Original

| Pos | Attack | Release | Modo |
|-----|--------|---------|------|
| 1 | 0.2ms | 0.3s | Fixed |
| 2 | 0.2ms | 0.8s | Fixed |
| 3 | 0.2ms | 2.0s | Fixed |
| 4 | 0.2ms | 5.0s | Fixed |
| 5 | 0.2ms | Fast/Slow (auto) | Auto-program-dependent |
| 6 | 0.2ms | Fast/Slow (auto, slower) | Auto-program-dependent |

### Tareas

- [ ] Mapear las 6 posiciones con valores exactos
- [ ] Implementar interpolación suave entre posiciones (evitar clicks)
- [ ] Implementar dual-branch para posiciones 5 y 6
- [ ] Agregar validación de rango para coefficients
- [ ] Testear cambio de posición en tiempo real (sin glitches)

### Notas de Implementación

```javascript
// Los coefficients se calculan en @slider
// Para evitar glitches al cambiar posición:
// - Usar crossfade suave entre coefficients viejos y nuevos
// - O simplemente recalcular (el envelope follower es inherentemente suave)

// El ataque del Fairchild es extremadamente rápido (0.2ms)
// Esto es ~10 samples a 48kHz
// attack_coeff = exp(-1 / (srate * 0.0002));

// El release del modo auto es dual-branch:
// Rama rápida: recovery rápido para transitorios breves
// Rama lenta: recovery lento para material sostenido fuerte
// Output CV = max(cv_fast, cv_slow)
```

---

## Fase 4: Emulación de Transformadores (3-4 días)

### Objetivo
Emular el color y la respuesta en frecuencia de los transformadores del Fairchild 670.

### Transformadores del Circuito

Cada canal tiene 4 transformadores de señal:
1. **T101/T201 (Input):** 600Ω/50kΩ, Ratio 1+1:9+9 — Acoplamiento de entrada
2. **T102/T202 (Output):** 60kCT/600Ω, Ratio 9+9:1+1 — Acoplamiento de salida
3. **T103/T203 (Control Amp Input):** 600Ω/170kΩ, Ratio 17+17:1+1 — Para el sidechain
4. **T104/T204 (Control Amp Output):** 10kCT/600Ω, Ratio 4:1 — Para el output del control

### Modelo de Transformador Simplificado

```javascript
// fairchild670_xforms.jsfx-inc
@init
// --- Transformadores ---
// Modelo: 2do orden biquad + saturación tanh

// Transformer Input (T101)
function xfmr_input_calc() (
  // HPF: ~30 Hz (acoplamiento)
  // LPF: ~30 kHz (limitación de ancho de banda)
  // Q: 0.707 (Butterworth, respuesta plana)
  // Saturación leve con mu-metal core

  freq_hp = 30;
  freq_lp = 30000;
  Q = 0.707;

  // HPF coefficients
  w0_hp = TWO_PI * freq_hp / srate;
  alpha_hp = sin(w0_hp) / (2 * Q);
  xf_in_b0_hp = (1 + cos(w0_hp)) / 2;
  xf_in_b1_hp = -(1 + cos(w0_hp));
  xf_in_b2_hp = (1 + cos(w0_hp)) / 2;
  xf_in_a0_hp = 1 + alpha_hp;
  xf_in_a1_hp = -2 * cos(w0_hp);
  xf_in_a2_hp = 1 - alpha_hp;
  // Normalizar...

  // LPF coefficients
  w0_lp = TWO_PI * freq_lp / srate;
  alpha_lp = sin(w0_lp) / (2 * Q);
  xf_in_b0_lp = (1 - cos(w0_lp)) / 2;
  xf_in_b1_lp = 1 - cos(w0_lp);
  xf_in_b2_lp = (1 - cos(w0_lp)) / 2;
  xf_in_a0_lp = 1 + alpha_lp;
  xf_in_a1_lp = -2 * cos(w0_lp);
  xf_in_a2_lp = 1 - alpha_lp;
  // Normalizar...
);

// Función: Aplicar transformador con saturación
function apply_transformer(x, b0, b1, b2, a1, a2, drive, z1, z2) (
  // Biquad filter
  y = b0*x + z1;
  z1 = b1*x - a1*y + z2;
  z2 = b2*x - a2*y;

  // Saturación suave (transformer core saturation)
  y * drive > 1 ? y = tanh(y * drive) / drive : 0;

  y;
);

// Estados del filtro (por canal)
xf_in_z1_l = 0; xf_in_z2_l = 0;
xf_out_z1_l = 0; xf_out_z2_l = 0;
xf_in_z1_r = 0; xf_in_z2_r = 0;
xf_out_z1_r = 0; xf_out_z2_r = 0;
```

### Tareas

- [ ] Implementar biquad cascade para HPF + LPF por transformador
- [ ] Agregar saturación suave tipo transformer (tanh con drive)
- [ ] Calibrar frecuencias de corte (30Hz HPF, 30kHz LPF para input)
- [ ] Implementar estados separados por canal (L/R)
- [ ] Añadir color armónico sutil (2do y 3er armónico)
- [ ] Probar respuesta en frecuencia con sweep de seno
- [ ] Medir que el ancho de banda resultante es ~40Hz-15kHz ±1dB

### Respuesta en Frecuencia Esperada
```
Original: 40 Hz to 15 kHz ±1 dB
Nuestro modelo target: 30 Hz to 18 kHz ±1.5 dB (con margen para variación)
```

---

## Fase 5: Stereo Link y Matriz L/V (2-3 días)

### Objetivo
Implementar los modos de operación estéreo del Fairchild.

### Modos del Original

1. **Independent (A-B):** Canales L y R procesados por separado
2. **Linked Stereo:** Ambos canales con el mismo gain reduction
3. **Lateral-Vertical (L-V):** Matriz M/S para corte de discos

### Matriz L/V

```
Lateral (Mid) = (L + R) / 2
Vertical (Side) = (L - R) / 2

L = (Lat + Vert) / 2
R = (Lat - Vert) / 2
```

### Tareas

- [ ] Implementar modo Independent (default)
- [ ] Implementar modo Linked Stereo (CV promediado)
- [ ] Implementar Matriz L/V:
  - Codificación: L,R → Lat,Vert
  - Procesamiento independiente de cada componente
  - Decodificación: Lat,Vert → L,R
- [ ] Agregar selector de modo en slider
- [ ] Testear separación en modo L/V
- [ ] Verificar que el modo linked produce imagen estable

### Código Base

```javascript
@sample
// Modo Independent (default)
mode == 0 ? (
  // Procesar L y R independientemente
  cv_l = sc_cv_l;
  cv_r = sc_cv_r;
);

// Modo Linked Stereo
mode == 1 ? (
  // CV promedio
  cv_avg = (sc_cv_l + sc_cv_r) * 0.5;
  cv_l = cv_avg;
  cv_r = cv_avg;
);

// Modo Lateral-Vertical
mode == 2 ? (
  // Codificar M/S
  lateral = (spl0 + spl1) * 0.5;
  vertical = (spl0 - spl1) * 0.5;

  // Procesar cada componente independientemente
  lat_out = process_channel(lateral, sc_cv_l);
  vert_out = process_channel(vertical, sc_cv_r);

  // Decodificar de vuelta a L/R
  spl0 = lat_out + vert_out;
  spl1 = lat_out - vert_out;
);
```

---

## Fase 6: Interfaz Gráfica @gfx (4-6 días)

### Objetivo
Crear una interfaz visual que capture la estética del Fairchild 670 original con knobs custom, VU meters, y visualización de gain reduction.

### Diseño de UI

```
┌─────────────────────────────────────────────────────────────────────┐
│                     FAIRCHILD 670 VARI-MU                          │
│  ┌─────────────────────────┐  ┌─────────────────────────┐         │
│  │     CANAL IZQUIERDO     │  │      CANAL DERECHO      │         │
│  │                         │  │                         │         │
│  │  [INPUT GAIN knob]      │  │  [INPUT GAIN knob]      │         │
│  │  [THRESHOLD knob]       │  │  [THRESHOLD knob]       │         │
│  │  [TIME CONSTANT 1-6]    │  │  [TIME CONSTANT 1-6]    │         │
│  │  [OUTPUT GAIN knob]     │  │  [OUTPUT GAIN knob]     │         │
│  │                         │  │                         │         │
│  │  ┌─── VU METER ───┐    │  │  ┌─── VU METER ───┐    │         │
│  │  │ ▓▓▓▓▓▓░░░░░░░░ │    │  │  │ ▓▓▓▓▓▓▓░░░░░░ │    │         │
│  │  └────────────────┘    │  │  └────────────────┘    │         │
│  │  ┌─── GR METER ───┐   │  │  ┌─── GR METER ───┐   │         │
│  │  │ ▓▓▓░░░░░░░░░░░ │   │  │  │ ▓▓▓▓░░░░░░░░░ │   │         │
│  │  └────────────────┘   │  │  └────────────────┘   │         │
│  └─────────────────────────┘  └─────────────────────────┘         │
│                                                                     │
│  [MODE: Stereo ▼]  [MIX: ████░░ 75%]  [BYPASS: OFF]              │
│                                                                     │
│  ┌───────────────────── TRANSFER CURVE ─────────────────────┐     │
│  │                                                         │     │
│  │                    ╱                                    │     │
│  │                  ╱                                      │     │
│  │                ╱                                        │     │
│  │              ╱                                          │     │
│  │            ╱                                            │     │
│  │          ╱                                              │     │
│  │        ╱                                                │     │
│  │      ╱                                                  │     │
│  │    ╱                                                    │     │
│  └─────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────┘
```

### Tareas

- [ ] Crear librería `fairchild670_ui.jsfx-inc`
- [ ] Implementar función `draw_knob()` con:
  - Círculo exterior con gradiente
  - Arco de valor (verde → amarillo → rojo)
  - Indicador de posición
  - Label del parámetro
  - Valor numérico
- [ ] Implementar VU Meter con:
  - Escala VU estándar (-20 a +3 dB)
  - Colores: verde → amarillo → rojo
  - Peak hold con decay
- [ ] Implementar GR (Gain Reduction) Meter:
  - Escala invertida (0 dB arriba, -30 dB abajo)
  - Color rojo
- [ ] Implementar Transfer Curve display:
  - Eje X: Input level
  - Eje Y: Output level
  - Curva que cambia con threshold y ratio
- [ ] Implementar knob de Time Constant (6 posiciones stepped)
- [ ] Implementar botón de Bypass con LED indicator
- [ ] Implementar slider de Mix/Dry-Wet
- [ ] Manejo de mouse para knobs (drag up/down)
- [ ] Offscreen rendering para UI estática (fondo, labels)
- [ ] Sync thread-safe entre @gfx y @sample con `atomic_*`

### Funciones de UI

```javascript
// fairchild670_ui.jsfx-inc
@init
// Offscreen buffer para fondo estático
gfx_bg_buffer = 1;

// Variables de estado UI
ui_needs_redraw = 1;
ui_last_mouse_y = 0;

// Función: Draw custom knob
function draw_knob(x, y, radius, value_norm, label, color_r, color_g, color_b) (
  // Fondo del knob
  gfx_r = 0.2; gfx_g = 0.2; gfx_b = 0.2;
  gfx_circle(x, y, radius, 1);

  // Arco de valor (de -135° a +135°)
  start_angle = -0.75 * $pi;
  end_angle = start_angle + value_norm * 1.5 * $pi;

  gfx_r = color_r; gfx_g = color_g; gfx_b = color_b;
  gfx_arc(x, y, radius - 3, start_angle, end_angle, 3);

  // Indicador de posición
  indicator_angle = end_angle;
  ix = x + cos(indicator_angle) * (radius - 6);
  iy = y + sin(indicator_angle) * (radius - 6);
  gfx_r = 1; gfx_g = 1; gfx_b = 1;
  gfx_circle(ix, iy, 3, 1);

  // Label
  gfx_r = 0.8; gfx_g = 0.8; gfx_b = 0.8;
  gfx_x = x - 20; gfx_y = y + radius + 8;
  gfx_printf("%s", label);
);

// Función: Draw VU Meter
function draw_vu_meter(x, y, w, h, level_db) (
  // Fondo
  gfx_r = 0.05; gfx_g = 0.05; gfx_b = 0.05;
  gfx_rect(x, y, w, h);

  // Nivel (normalizado -20 a +3 dB → 0 a 1)
  level_norm = (level_db + 20) / 23;
  level_norm < 0 ? level_norm = 0;
  level_norm > 1 ? level_norm = 1;

  meter_w = level_norm * w;

  // Color según nivel
  level_db < -6 ? (
    gfx_r = 0; gfx_g = 0.8; gfx_b = 0; // verde
  ) : level_db < 0 ? (
    gfx_r = 1; gfx_g = 1; gfx_b = 0; // amarillo
  ) : (
    gfx_r = 1; gfx_g = 0; gfx_b = 0; // rojo
  );

  gfx_rect(x, y, meter_w, h);

  // Marcas VU
  gfx_r = 0.3; gfx_g = 0.3; gfx_b = 0.3;
  // ... dibujar marcas ...
);

// Función: Draw Gain Reduction Meter
function draw_gr_meter(x, y, w, h, gr_db) (
  // Fondo
  gfx_r = 0.05; gfx_g = 0.05; gfx_b = 0.05;
  gfx_rect(x, y, w, h);

  // GR normalizado (0 a -30 dB → 0 a 1)
  gr_norm = abs(gr_db) / 30;
  gr_norm > 1 ? gr_norm = 1;

  meter_h = gr_norm * h;

  gfx_r = 1; gfx_g = 0.2; gfx_b = 0.2;
  gfx_rect(x, y + h - meter_h, w, meter_h);
);
```

### Pruebas de Fase 6
1. **Knobs responden a mouse drag** (up = increase, down = decrease)
2. **VU meters muestran nivel real** del audio
3. **GR meter muestra reducción de ganancia** correctamente
4. **UI no causa glitches en audio** (sync con atomic_*)
5. **Redibujar solo cuando hay cambios** (optimización CPU)
6. **Offscreen rendering funciona** para fondo estático

---

## Fase 7: Serialización y Presets (1 día)

### Objetivo
Persistir el estado completo del plugin más allá de los sliders.

### Tareas

- [ ] Implementar `@serialize` con versionado
- [ ] Guardar/cargar variables de estado interno:
  - sc_env_l, sc_env_r (envelope states)
  - sc_cv_l, sc_cv_r (control voltages)
  - sc_env_fast_l/r, sc_env_slow_l/r (auto-release states)
  - xf_in_z1/z2 (transformer filter states)
- [ ] Implementar guardado de presets de fábrica
- [ ] Manejar compatibilidad hacia atrás (version field)
- [ ] Testear save/load cycle completo

### Código

```javascript
@serialize
preset_version = 1.0;
file_var(0, preset_version);

// Variables de estado interno
file_var(0, sc_env_l);
file_var(0, sc_env_r);
file_var(0, sc_cv_l);
file_var(0, sc_cv_r);
file_var(0, sc_env_fast_l);
file_var(0, sc_env_fast_r);
file_var(0, sc_env_slow_l);
file_var(0, sc_env_slow_r);

// Estados de filtros (transformers)
file_var(0, xf_in_z1_l);
file_var(0, xf_in_z2_l);
file_var(0, xf_out_z1_l);
file_var(0, xf_out_z2_l);
file_var(0, xf_in_z1_r);
file_var(0, xf_in_z2_r);
file_var(0, xf_out_z1_r);
file_var(0, xf_out_z2_r);

// Compatibilidad hacia atrás
preset_version < 1.0 ? (
  sc_env_l = 0; sc_env_r = 0;
  // ... defaults ...
);
```

---

## Fase 8: Optimización y CPU (2-3 días)

### Objetivo
Asegurar que el plugin funcione eficientemente en cualquier sesión de REAPER sin causar glitches o alta carga CPU.

### Estrategias de Optimización

1. **Pre-calculación en @init/@slider:**
   - Lookup tables para curva del tubo
   - Coeficientes de filtros
   - Coeficientes de attack/release

2. **@sample optimizado:**
   - Minimizar ramas condicionales
   - Usar branchless patterns donde sea posible
   - Cache values en variables locales
   - Prevenir denormals con tiny offset

3. **@gfx eficiente:**
   - Offscreen rendering para UI estática
   - Solo redibujar cuando hay cambios
   - Usar `atomic_*` para sync thread-safe

### Tareas

- [ ] Agregar anti-denormal a todas las variables de estado
- [ ] Mover cálculos de transformadores a @slider
- [ ] Verificar que no hay bucles en @sample
- [ ] Medir CPU con plugin completo en diferentes block sizes
- [ ] Agregar `options:prealloc=8000000` para pre-allocar memoria
- [ ] Optimizar @gfx: solo redraw cuando `ui_needs_redraw`
- [ ] Testear con múltiples instancias (4-8 en una sesión)
- [ ] Profile con el medidor de CPU de REAPER

### Target de Performance
```
Single instance: < 3% CPU @ 48kHz, 512 samples block
8 instances: < 15% CPU total
Latencia: 0 samples (sin PDC)
```

### Anti-Denormal

```javascript
@sample
// Anti-denormal: agregar tiny offset antes de procesamiento
tiny = 1e-18;

// En todas las variables de estado de filtros:
sc_env_l += tiny;
sc_cv_l += tiny;
xf_in_z1_l += tiny;
// ... etc ...

// Al final del procesamiento, restar tiny
// (opcional, el offset es tan pequeño que no afecta el audio)
```

---

## Fase 9: Testing y Validación (3-4 días)

### Objetivo
Validar que el plugin suena correctamente y se comporta como se espera.

### Pruebas de Audio

1. **Test de ganancia unity:**
   - Señal seno 1kHz, 0 dBV → Output debe ser ~0 dBV (sin compresión)

2. **Test de compresión:**
   - Señal seno con nivel creciente → GR debe aumentar
   - Verificar curva de compresión suave (soft knee)

3. **Test de attack/release:**
   - Transitorio rápido (click) → debe ser atrapado (~0.2ms attack)
   - Release según posición del Time Constant

4. **Test de auto-release (pos 5-6):**
   - Picos breves → recovery rápido
   - Material sostenido fuerte → recovery lento

5. **Test estéreo:**
   - Señal L/R idéntica → imagen centrada
   - Señal L/R diferente → verificas separación

6. **Test L/V matrix:**
   - Señal mono (L=R) → toda en Lateral, nada en Vertical
   - Señal completamente out-of-phase → toda en Vertical

7. **Test de saturación:**
   - Input gain alto + Output gain bajo → saturación armónica

8. **Test de bypass:**
   - Bypass on/off → sin cambios de nivel ni glitches

### Pruebas Técnicas

1. **CPU:** < 3% por instancia @ 48kHz/512
2. **Memoria:** < 50MB por instancia
3. **Sample rates:** 44.1k, 48k, 88.2k, 96k, 192k
4. **Block sizes:** 64, 128, 256, 512, 1024, 2048
5. **Serialización:** Save/load mantiene estado exacto
6. **Múltiples instancias:** 8+ sin problemas
7. **No clicks/pops** al cambiar sliders en tiempo real

### Comparación con Referencia

Si tenemos acceso a un Fairchild 670 original o una buena emulación (plugin):
1. Comparar curvas de transferencia
2. Comparar respuesta a transitorios
3. Comparar spectrum de salida (armónicos)
4. Comparar comportamiento de auto-release

### Prueba de Spectrum

```javascript
// Test: Medir distorsión armónica
// Señal seno 1kHz → Medir nivel de 2do, 3er, 4to armónico
// El Fairchild debe producir:
// - 2do armónico: -40 a -50 dB bajo fundamental
// - 3er armónico: -50 a -60 dB bajo fundamental
// - Armónicos superiores: por debajo de -60 dB
```

---

## Fase 10: Polish y Distribución (2-3 días)

### Objetivo
Pulir el plugin, documentar, y preparar para distribución.

### Tareas

- [ ] Agregar tooltip descriptions a todos los sliders
- [ ] Agregar nombre descriptivo a cada modo del Time Constant
- [ ] Implementar presets de fábrica:
  - "Mix Bus Glue" (suave, 1-3 dB GR)
  - "Vocal Warmth" (medio, con saturación)
  - "Mastering Transparent" (mínimo color)
  - "Classic Limiting" (agresivo)
  - "Drum Bus Pump" (para batería)
  - "Box Tone Only" (sin compresión, solo color de tubos)
- [ ] Crear README.md con:
  - Descripción del plugin
  - Instrucciones de instalación
  - Guía de uso por modo
  - Créditos y referencias
- [ ] Agregar metadata ReaPack para distribución
- [ ] Testear en REAPER Windows, macOS, Linux
- [ ] Crear video demo (opcional)

### Presets de Fábrica

```javascript
// Mix Bus Glue
slider1: 0       // Input Gain L: 0 dB
slider2: 40      // Threshold L: 40%
slider3: 3       // Time Constant: 3 (2s/5s)
slider4: 0       // Output Gain L: 0 dB
slider5: 0       // Input Gain R: 0 dB
slider6: 40      // Threshold R: 40%
slider7: 3       // Time Constant: 3
slider8: 0       // Output Gain R: 0 dB
slider9: 1       // Mode: Stereo Linked
slider10: 80     // Mix: 80%

// Vocal Warmth
slider1: 3       // Input Gain: +3 dB (empujar tubo)
slider2: 60      // Threshold: 60%
slider3: 2       // Time Constant: 2 (0.8s/2s)
slider4: -2      // Output: -2 dB (compensar)
slider9: 0       // Mode: Independent
slider10: 100    // Mix: 100%
```

---

## Resumen de Fases y Timeline

| Fase | Descripción | Días | Dependencias |
|------|-------------|------|--------------|
| **0** | Setup y Research | 1-2 | Ninguna |
| **1** | Variable-Mu Gain Stage | 3-5 | Fase 0 |
| **2** | Sidechain / Detector | 2-3 | Fase 1 |
| **3** | Time Constants | 1-2 | Fase 2 |
| **4** | Transformers | 3-4 | Fase 1 |
| **5** | Stereo Link / L/V Matrix | 2-3 | Fases 1-3 |
| **6** | UI @gfx | 4-6 | Fases 1-3 |
| **7** | Serialization | 1 | Fases 1-5 |
| **8** | Optimization | 2-3 | Fases 1-7 |
| **9** | Testing | 3-4 | Fases 1-8 |
| **10** | Polish | 2-3 | Fases 1-9 |
| **TOTAL** | | **22-35 días** | |

### Orden de Desarrollo Recomendado

```
Fase 0 ──→ Fase 1 ──→ Fase 2 ──→ Fase 3
                  │              │
                  └──→ Fase 4   └──→ Fase 5
                                    │
                          Fase 6 ←──┘
                           │
                      Fase 7 ──→ Fase 8 ──→ Fase 9 ──→ Fase 10
```

### Próximos Pasos (después de Fase 10)
- **Fase 11 (opcional):** Comparación A/B con Fairchild original/plugin de referencia
- **Fase 12 (opcional):** Añadir sidechain HPF (extensión moderna)
- **Fase 13 (opcional):** Añadir dry/wet por canal independiente
- **Fase 14 (opcional):** Añadir mid/side output independiente
- **Fase 15 (opcional):** Portar a VST3/AU usando ysfx

---

*Documento creado: 5 de Agosto de 2026*
*Basado en documentación técnica del Fairchild 670 original y guías de desarrollo JSFX*
