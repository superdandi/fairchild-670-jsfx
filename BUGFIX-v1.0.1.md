# Bugfix v1.0.1 - Fairchild 670 Vari-Mu Compressor

## Resumen

Corrección de 2 bugs críticos que impedían que los knobs del compressor afectaran el sonido.

**Fecha:** 2026-08-05
**Versión anterior:** v1.0.0
**Versión corregida:** v1.0.1

---

## Bug #1: Threshold vs Nivel de Envolvente (CRÍTICO)

### Síntoma

Los knobs de Input Gain, Threshold, Output Gain y Time Constant **no tenían efecto audible** en el procesamiento. El compressor no comprimía.

### Root Cause

El slider `Threshold` va de 0-100 y se mapeaba a 0-1 lineal:
```js
// ANTES (v1.0.0)
thresh_l = slider2 / 100;  // Slider 50 → thresh = 0.5
```

Pero el `envelope_to_cv` comparaba este valor lineal contra el nivel de la envolvente rectificada:
```js
// ANTES (v1.0.0)
function envelope_to_cv(env_level, threshold, ratio) (
  env_level > threshold ? (  // env_level típicamente 0.01-0.1, threshold = 0.5
    // ...
  ) : (
    cv = 0;  // SIEMPRE 0 → sin compresión
  );
);
```

**Problema:** La envolvente de audio real tiene valores de **0.01 a 0.1** (amplitud normalizada), mientras que el threshold era **0.5**. La condición `env_level > threshold` **nunca se cumple**, resultando en CV = 0 (sin compresión).

### Corrección

Cambiar el mapeo del threshold a escala **dB** (-60dB a 0dB) y comparar la envolvente también en dB:

```js
// DESPUÉS (v1.0.1)
// @slider
thresh_l = -60 + slider2 * 0.6;  // Slider 0 → -60dB, Slider 50 → -30dB, Slider 100 → 0dB

// envelope_to_cv
function envelope_to_cv(env_level, threshold_db) (
  env_level > 0.0001 ? (
    env_db = 20 * log(env_level) / log(10);  // Convertir envolvente a dB
    env_db > threshold_db ? (
      cv = (env_db - threshold_db) / 30;  // 30dB range → CV 0-1
      cv > 1 ? cv = 1;
    ) : (
      cv = 0;
    );
  ) : (
    cv = 0;
  );
);
```

### Verificación

1. Insertar Fairchild 670 en una pista
2. Reproducir audio
3. Mover el slider Threshold → debería escuchar compresión
4. Verificar que el GR meter se mueve

---

## Bug #2: Cálculo de `sr` (Time Constants 1000x más rápidos)

### Síntoma

Los time constants (attack/release) eran **1000x más rápidos** de lo especificado. El modo "Auto-Fast" se comportaba como "instantáneo".

### Root Cause

```js
// ANTES (v1.0.0)
sr = 0.001 * $srate;  // Para44100 Hz: sr = 44.1 (samples/ms)
```

La fórmula del coefficient es:
```js
attack_coeff = exp(-1 / (sr * attack_time_seconds));
```

Con `sr = 44.1` y `attack_time = 0.4s`:
```
exp(-1 / (44.1 * 0.4)) = exp(-0.0567) ≈ 0.945
```

Esto corresponde a un time constant de **~1ms**, no 400ms.

### Corrección

```js
// DESPUÉS (v1.0.1)
sr = $srate;  // Para44100 Hz: sr = 44100 (samples/sec)
```

Con `sr = 44100` y `attack_time = 0.4s`:
```
exp(-1 / (44100 * 0.4)) = exp(-0.0000567) ≈ 0.999943
```

Esto corresponde al time constant correcto de **400ms**.

### Verificación

1. Seleccionar Time Constant = 2 (0.4s attack / 0.8s release)
2. Señal con transitorios fuertes
3. El attack debería ser perceptible (~400ms), no instantáneo
4. El GR meter debería subir/bajar suavemente

---

## Cambios Adicionales

### Eliminación de `COMPRESSOR_RATIO`

La constante `COMPRESSOR_RATIO = 3.0` ya no se usa (el ratio ahora es implícito en la fórmula `cv = (env_db - threshold_db) / 30`).

### Transfer Curve Display

Actualizado para usar el threshold en dB directamente.

---

## Archivos Modificados

| Archivo | Cambios |
|---------|---------|
| `Fairchild670.jsfx` | Fix threshold, sr, envelope_to_cv, eliminación COMPRESSOR_RATIO |
| `BUGFIX-v1.0.1.md` | Esta documentación |

---

## Testing Checklist

- [ ] Los knobs de Input/Threshold/Output afectan el sonido
- [ ] El GR meter muestra ganancia de reducción
- [ ] Los 6 time constants suenan correctamente
- [ ] Auto-release funciona (program-dependent)
- [ ] Stereo Linked usa el canal más fuerte
- [ ] L/R Independent procesa separadamente
- [ ] Lateral-Vertical (M/S) funciona
- [ ] Presets cargan correctamente
- [ ] CPU usage < 5% por instancia
