# EEL2/JSFX Reference — Fairchild 670

## Referencias Oficiales

- **EEL2 Language**: https://www.cockos.com/EEL2/
- **JSFX Programming**: https://www.reaper.fm/sdk/js/js.php
- **JSFX Graphics**: https://www.reaper.fm/sdk/js/gfx.php

---

## Funciones Disponibles en EEL2

### Math Functions
| Función | Descripción |
|---------|-------------|
| `sin(x)`, `cos(x)`, `tan(x)` | Trigonométricas |
| `asin(x)`, `acos(x)`, `atan(x)` | Inversas |
| `atan2(x,y)` | Arcotangente de x/y |
| `sqr(x)` | Cuadrado (x*x) |
| `sqrt(x)` | Raíz cuadrada |
| `pow(x,y)` | Potencia (x^y) |
| `exp(x)` | e^x |
| `log(x)` | Logaritmo natural (base e) |
| `log10(x)` | Logaritmo base 10 |
| `abs(x)` | Valor absoluto |
| `min(x,y)`, `max(x,y)` | Mínimo/Máximo |
| `sign(x)` | Signo (-1, 0, 1) |
| `rand(x)` | Pseudo-aleatorio entre 0 y x |
| `floor(x)`, `ceil(x)` | Redondeo |
| `invsqrt(x)` | Inversa raíz cuadrada |

### Constants
| Variable | Valor |
|----------|-------|
| `$pi` | π (3.14159...) |
| `$phi` | φ (1.61803...) |
| `$e` | e (2.71828...) |

---

## JSFX Special Variables

| Variable | Descripción |
|----------|-------------|
| `$srate` | **NO DISPONIBLE en EEL2** - usar valor fijo (44100) |
| `$srate_inv` | **NO DISPONIBLE en EEL2** |
| `spl0`, `spl1`, ... | Audio samples (solo en @sample) |
| `slider1`, `slider2`, ... | Parámetros |
| `samplesblock` | Tamaño del bloque actual |

---

## Secciones JSFX Válidas

| Sección | Descripción |
|---------|-------------|
| `@init` | Ejecuta al cargar plugin |
| `@slider` | Ejecuta al cambiar parámetros |
| `@block` | Ejecuta antes de cada bloque de audio |
| `@sample` | Ejecuta por cada sample |
| `@serialize` | Guarda/carga estado |
| `@gfx` | Interfaz gráfica (~30fps) |

**NO EXISTE**: `@load`, `@update`, `@process`

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

## Funciones Gráficas (JSFX)

| Función | Sintaxis | Descripción |
|---------|----------|-------------|
| `gfx_set` | `gfx_set(r,g,b,a,mode,dest,a2)` | Establecer color |
| `gfx_line` | `gfx_line(x1,y1,x2,y2,aa)` | Dibujar línea |
| `gfx_lineto` | `gfx_lineto(x,y,aa)` | Línea desde posición actual |
| `gfx_rect` | `gfx_rect(x,y,w,h)` | Rectángulo relleno |
| `gfx_rectto` | `gfx_rectto(x,y)` | Rectángulo hasta posición |
| `gfx_circle` | `gfx_circle(x,y,r,fill,aa)` | Círculo (relleno o borde) |
| `gfx_arc` | `gfx_arc(x,y,r,start,end,aa)` | Arco |
| `gfx_triangle` | `gfx_triangle(x1,y1,x2,y2,x3,y3)` | Triángulo |
| `gfx_setfont` | `gfx_setfont(idx,name,size,flags)` | Establecer fuente |
| `gfx_drawstr` | `gfx_drawstr(str)` | Dibujar texto |
| `gfx_printf` | `gfx_printf(format,...)` | Texto formateado |
| `gfx_clear` | `gfx_clear = 0xRRGGBB` | Color de fondo |

### Variables Gráficas
| Variable | Descripción |
|----------|-------------|
| `gfx_w`, `gfx_h` | Tamaño de ventana |
| `gfx_x`, `gfx_y` | Posición actual |
| `gfx_r`, `gfx_g`, `gfx_b`, `gfx_a` | Color actual |

---

## Errores Comunes y Soluciones

### 1. Notación Científica
```js
// ❌ INCORRECTO (EEL2 no soporta)
TINY = 1e-18;

// ✅ CORRECTO
TINY = 0.000000000000000001;
```

### 2. Función random()
```js
// ❌ INCORRECTO (no existe en EEL2)
x = random();

// ✅ CORRECTO
x = rand(max_value);
```

### 3. $srate no disponible
```js
// ❌ INCORRECTO ($srate no funciona en ninguna sección)
sr = $srate;

// ✅ CORRECTO (usar valor fijo)
sr = 44100;
```

### 4. gfx_circle sin parámetro aa
```js
// ❌ INCORRECTO (falta parámetro aa)
gfx_circle(x, y, r, 1);

// ✅ CORRECTO
gfx_circle(x, y, r, 1, 1);  // fill=1, aa=1
```

### 5. sprintf incorrecto
```js
// ❌ INCORRECTO (value es número, no string)
gfx_drawstr(sprintf(value, "%.0f"));

// ✅ CORRECTO
_temp_str = #;
sprintf(_temp_str, "%.0f", value);
gfx_drawstr(_temp_str);
```

### 6. log(x)/log(10) innecesario
```js
// FUNCIONAL pero innecesario
db = 20 * log(x) / log(10);

// ✅ CORRECTO
db = 20 * log10(x);
```

### 7. if() no existe
```js
// ❌ INCORRECTO (if no existe en EEL2)
if (condition,
  true_code;
,
  false_code;
);

// ✅ CORRECTO (usar ternario)
condition ? (
  true_code;
) : (
  false_code;
);
```

### 8. Branch vacío en ternario
```js
// ❌ INCORRECTO (branch vacío causa error)
bypass ? (
  // pass-through
) : (
  process_audio();
);

// ✅ CORRECTO (agregar no-op)
bypass ? (
  0;  // no-op
) : (
  process_audio();
);
```

### 9. tanh() no existe
```js
// ❌ INCORRECTO (tanh no existe en EEL2)
y = tanh(x);

// ✅ CORRECTO (implementar manualmente)
function my_tanh(x) (
  _et = exp(2 * x);
  (_et - 1) / (_et + 1);
);
y = my_tanh(x);
```

### 10. time_start()/time_end() no existen
```js
// ❌ INCORRECTO (no existen en EEL2)
cpu_time_start = time_start();
// ... process ...
cpu_time_end = time_end();

// ✅ CORRECTO (no se puede medir tiempo en EEL2)
// Eliminar código de medición de tiempo
```

---

## Variables Locales en Funciones

```js
// Declarar variables locales con local()
function myFunc(x) local(temp1 temp2) (
  temp1 = x * 2;
  temp2 = temp1 + 1;
  temp2;
);
```

---

## Strings en EEL2

```js
// Strings literales (inmutables)
x = "hello";

// Strings temporales (#)
_temp = #;
strcpy(_temp, "hello ");
strcat(_temp, "world");
gfx_drawstr(_temp);

// Strings con slots fijos (0-1023)
x = 50; // slot 50
strcpy(x, "hello");

// Named strings
#myString = "hello";
#myString += " world";
```

---

## Loops

```js
// Loop con conteo fijo
loop(32,
  r += b;
  b *= 1.5;
);

// While loop
while(a < 1000) (
  a += b;
  b *= 1.5;
);
```

---

## Biquad Filter (LPF/HPF)

```js
// Calcular coefficients de biquad
// type: 0=LPF, 1=HPF
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

**IMPORTANTE**: Los coefficients deben ser normalizados dividiendo por `a0 = 1 + alpha`.

---

## Referencias

- [EEL2 Language Reference](https://www.cockos.com/EEL2/)
- [JSFX Programming Reference](https://www.reaper.fm/sdk/js/js.php)
- [JSFX Graphics Reference](https://www.reaper.fm/sdk/js/gfx.php)
