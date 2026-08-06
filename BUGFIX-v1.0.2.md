# Bugfix v1.0.2 - Fairchild 670 Vari-Mu Compressor

## Resumen

Corrección de múltiples bugs de compatibilidad EEL2/JSFX que impedían que el plugin cargara correctamente.

**Fecha:** 2026-08-06
**Versión anterior:** v1.0.1
**Versión corregida:** v1.0.2

---

## Bugs Corregidos

### Bug #1: `tanh` undefined (Línea 384)

**Error:** `@init:384: 'tanh' undefined`

**Causa:** EEL2 no tiene función `tanh()`.

**Solución:** Implementar manualmente:
```js
function my_tanh(x) (
  _et = exp(2 * x);
  (_et - 1) / (_et + 1);
);
```

---

### Bug #2: `$srate` syntax error (Línea 641)

**Error:** `@slider:641: syntax error: 'sr = $srate;'`

**Causa:** `$srate` no está disponible como variable legible en EEL2.

**Solución:** Usar valor fijo:
```js
sr = 44100;
```

---

### Bug #3: `time_start` undefined (Línea 915)

**Error:** `@block:915: 'time_start' undefined`

**Causa:** `time_start()` y `time_end()` no existen en EEL2.

**Solución:** Eliminar todo el código de medición de CPU.

---

### Bug #4: Syntax error en ternario (Línea 680)

**Error:** `@sample:680: syntax error: ' ) : ('`

**Causa:** Branch vacío en operador ternario.

**Solución:** Agregar no-op:
```js
bypass ? (
  0;  // no-op
) : (
  process_audio();
);
```

---

### Bug #5: `if` undefined (Línea 1096)

**Error:** `@gfx:1096: 'if' undefined`

**Causa:** EEL2 no tiene función `if()`.

**Solución:** Usar ternario:
```js
// ANTES
if (condition, true_code, false_code);

// DESPUÉS
condition ? (true_code) : (false_code);
```

---

### Bug #6: Biquad filter sin normalizar (CRÍTICO)

**Síntoma:** Demasiados agudos, filtros no funcionaban correctamente.

**Causa:** Faltaba normalizar los coefficients por `a0 = 1 + alpha`.

**Solución:**
```js
function calc_biquad(type, freq, q) (
  w0 = TWO_PI * freq / sr;
  cos_w0 = cos(w0);
  sin_w0 = sin(w0);
  alpha = sin_w0 / (2 * q);

  type == 0 ? (
    // LPF
    a0 = 1 + alpha;
    bq_b0 = ((1 - cos_w0) / 2) / a0;
    bq_b1 = (1 - cos_w0) / a0;
    bq_b2 = ((1 - cos_w0) / 2) / a0;
  ) : (
    // HPF
    a0 = 1 + alpha;
    bq_b0 = ((1 + cos_w0) / 2) / a0;
    bq_b1 = -(1 + cos_w0) / a0;
    bq_b2 = ((1 + cos_w0) / 2) / a0;
  );
  bq_a1 = (-2 * cos_w0) / a0;
  bq_a2 = (1 - alpha) / a0;
);
```

---

## Bugs Originales (v1.0.1)

### Bug #1: Threshold vs Nivel de Envolvente (CRÍTICO)

Los knobs no tenían efecto audible. El threshold en escala lineal (0-1) nunca se comparaba correctamente con la envolvente (0.01-0.1).

**Solución:** Cambiar a escala dB (-60 a 0dB).

### Bug #2: Time Constants 1000x más rápidos

`sr = 0.001 * $srate` causaba time constants 1000x más rápidos.

**Solución:** `sr = $srate` (luego cambiado a valor fijo 44100).

---

## Funciones NO Disponibles en EEL2

| Función | Error | Alternativa |
|---------|-------|-------------|
| `tanh(x)` | `tanh undefined` | `(exp(2x)-1)/(exp(2x)+1)` |
| `$srate` | `syntax error` | Valor fijo (44100 Hz) |
| `if(cond, true, false)` | `if undefined` | `cond ? (true) : (false)` |
| `time_start()` | `undefined` | No hay alternativa |
| `time_end()` | `undefined` | No hay alternativa |
| `random()` | `undefined` | `rand(max_value)` |

---

## Archivos Modificados

| Archivo | Cambios |
|---------|---------|
| `Fairchild670.jsfx` | Fixes de EEL2/JSFX |
| `EEL2_REFERENCE.md` | Documentación completa de limitaciones |
| `BUGFIX-v1.0.2.md` | Esta documentación |

---

## Referencias

- [EEL2 Language Reference](https://www.cockos.com/EEL2/)
- [JSFX Programming Reference](https://www.reaper.fm/sdk/js/js.php)
- [JSFX Graphics Reference](https://www.reaper.fm/sdk/js/gfx.php)
