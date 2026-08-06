# Bugfix v1.0.3 - Fairchild 670 Vari-Mu Compressor

## Resumen

Corrección de un bug crítico que causaba buzz, hiss y ruido continuo en el plugin, incluso al 1% de mezcla húmeda.

**Fecha:** 2026-08-06
**Versión anterior:** v1.0.2
**Versión corregida:** v1.0.3

---

## Bug Crítico: Filtro LPF Inestable (Buzz/Hiss/Ruido)

### Síntoma

El plugin producía **muchísimo buzz, hiss y ruido**, incluso con el Mix/Dry-Wet al 1%. El audio procesado era inutilizable.

### Root Cause

El filtro LPF del transformer de salida tenía un cutoff de **30000 Hz**, que **EXCEDE la frecuencia de Nyquist** (22050 Hz) a 44100 Hz de sample rate.

El cálculo del biquad quedaba con polos fuera del círculo unitario:

```
w0 = 2·π·30000/44100 = 4.27 rad   (> π, por encima de Nyquist)
cos(w0) = -0.447
sin(w0) = -0.894
alpha = sin(w0)/(2·0.707) = -0.632
a0 = 1 + alpha = 0.368
a2 = (1 - alpha)/a0 = (1 - (-0.632))/0.368 = 4.43
```

**`|a2| = 4.43 > 1`** → polos FUERA del círculo unitario → **filtro INESTABLE** → auto-oscilación → buzz/hiss/ruido persistente.

Este filtro inestable también **invertía y amplificaba los agudos** en lugar de atenuarlos, explicando el reporte anterior de "demasiados agudos".

### Corrección

Reducir el cutoff del LPF a **15000 Hz** (seguro bajo Nyquist para 44.1k y 48k):

```js
// ANTES (inestable)
calc_biquad(0, 30000, 0.707);

// DESPUÉS (estable)
calc_biquad(0, 15000, 0.707);
```

Verificación de estabilidad del nuevo filtro:
```
w0 = 2·π·15000/44100 = 2.137 rad   (< π, bajo Nyquist)
a2 = (1 - 0.597)/1.597 = 0.252     (< 1 → estable) ✓
```

### Regla Importante

> **El cutoff de un LPF biquad NUNCA puede exceder Nyquist (sr/2).**
> A 44100 Hz, el máximo cutoff seguro es ~20000 Hz.

---

## Fix Adicional: `my_tanh()` Desbordamiento → NaN

### Síntoma

Con señales fuertes, `exp(2*x)` puede desbordar a `inf`, produciendo `inf/inf = NaN` en el denominador, inyectando ruido/parásitos en la señal.

### Corrección

Versión numéricamente estable usando `exp(-2|x|)` (nunca desborda), restaurando el signo después:

```js
// ANTES (puede desbordar)
function my_tanh(x) (
  _et = exp(2 * x);
  (_et - 1) / (_et + 1);
);

// DESPUÉS (estable)
function my_tanh(x) (
  _et = exp(-2 * abs(x));
  _t = (1 - _et) / (1 + _et);
  x < 0 ? -_t : _t;
);
```

---

## Archivos Modificados

| Archivo | Cambios |
|---------|---------|
| `Fairchild670.jsfx` | LPF 30k→15k (4 lugares), my_tanh estable |

---

## Verificación

- [x] Sin buzz/hiss/ruido al 1% de wet mix
- [x] Filtros estables (|a2| < 1)
- [x] Agudos atenuados correctamente (LPF funciona)
- [x] Sin NaN en la señal

---

## Versiones Anteriores

- v1.0.2: Fixes de compatibilidad EEL2 (tanh, $srate, if, time_start, ternarios)
- v1.0.1: Fix threshold dB, fix sr
