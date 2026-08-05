# Fairchild 670 Vari-Mu Compressor

JSFX plugin emulating the legendary Fairchild 670 tube compressor for REAPER.

> "El Fairchild 670 es probablemente el compresor de tubo más jamás construido." - Walter E. Sear (1959)

[![GitHub release](https://img.shields.io/github/release/superdandi/fairchild-670-jsfx.svg)](https://github.com/superdandi/fairchild-670-jsfx/releases)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![REAPER](https://img.shields.io/badge/REAPER-6.0+-blue.svg)](https://www.reaper.fm/)

![Fairchild 670 UI](https://raw.githubusercontent.com/superdandi/fairchild-670-jsfx/main/screenshot.png)

---

## Features

- **Variable-Mu Gain Stage**: Authentic 6386 remote-cutoff tube emulation with 1024-point LUT
- **Sidechain Detector**: 6AL5 soft rectifier with RC envelope follower
- **Time Constants**: 6 positions including dual-branch auto-release modes
- **Transformer Emulation**: Input/output transformers with biquad filters + tanh saturation
- **Stereo Processing**: Stereo Linked, L/R Independent, Lateral-Vertical (M/S)
- **Graphics Interface**: VU meters, knobs, gain reduction display, transfer curve
- **Factory Presets**: 7 reference presets for different applications
- **Optimized DSP**: Anti-denormal protection, CPU monitoring, <3% per instance

---

## Installation

### Opción 1: Instalación Manual

#### Windows

1. **Descarga** el archivo `Fairchild670.jsfx` desde [Releases](https://github.com/superdandi/fairchild-670-jsfx/releases/latest)

2. **Copia** el archivo a la carpeta de Effects de REAPER:
   ```
   %APPDATA%\REAPER\Effects\
   ```
   
   > **Nota**: Puedes acceder rápidamente escribiendo `%APPDATA%\REAPER\Effects\` en la barra de direcciones del Explorador de Archivos.

3. **Reinicia REAPER** o actualiza la lista de efectos:
   - Ve a **Options** → **Preferences** → **VST** → **Re-scan**
   - O simplemente cierra y vuelve a abrir REAPER

#### macOS

1. **Descarga** el archivo `Fairchild670.jsfx` desde [Releases](https://github.com/superdandi/fairchild-670-jsfx/releases/latest)

2. **Copia** el archivo a la carpeta de Effects de REAPER:
   ```
   ~/Library/Application Support/REAPER/Effects/
   ```
   
   > **Cómo llegar ahí**:
   > - Abre **Finder**
   > - Presiona `Cmd + Shift + G`
   > - Pega la ruta: `~/Library/Application Support/REAPER/Effects/`
   > - Pega el archivo

3. **Reinicia REAPER**

#### Linux (Ubuntu/Debian/Mint/Fedora)

1. **Descarga** el archivo `Fairchild670.jsfx` desde [Releases](https://github.com/superdandi/fairchild-670-jsfx/releases/latest)

2. **Copia** el archivo a la carpeta de Effects de REAPER:
   ```bash
   # Para un solo usuario
   cp Fairchild670.jsfx ~/.config/reaper/Effects/
   
   # O con interfaz gráfica:
   # Navega a ~/.config/reaper/Effects/ y pega el archivo
   ```

3. **Reinicia REAPER**

---

### Opción 2: Instalación con ReaPack (Recomendada)

ReaPack es el gestor de paquetes de REAPER. Con él puedes instalar y actualizar plugins fácilmente.

#### Instalar ReaPack (si no lo tienes)

1. Descarga ReaPack desde: https://forum.cockos.com/showthread.php?t=212174
2. Instala el `.ReaPack` en REAPER (arrastra y suelta)
3. Reinicia REAPER

#### Agregar este repositorio

1. En REAPER, ve a **Extensions** → **ReaPack** → **Import Repositories**
2. Pega esta URL:
   ```
   https://github.com/superdandi/fairchild-670-jsfx/raw/main/index.xml
   ```
3. Haz clic en **OK**
4. ReaPack descargará el paquete automáticamente
5. Haz clic en **Apply** para instalar

#### Actualizar (futuras versiones)

Ve a **Extensions** → **ReaPack** → **Check for updates** → **Apply**

---

### Opción 3: Clonar el Repositorio

Para usuarios avanzados o desarrolladores:

```bash
# Clonar el repositorio
git clone https://github.com/superdandi/fairchild-670-jsfx.git

# Copiar el plugin a la carpeta de REAPER
# Linux:
cp fairchild-670-jsfx/Fairchild670.jsfx ~/.config/reaper/Effects/

# macOS:
cp fairchild-670-jsfx/Fairchild670.jsfx ~/Library/Application\ Support/REAPER/Effects/

# Windows (PowerShell):
Copy-Item fairchild-670-jsfx\Fairchild670.jsfx "$env:APPDATA\REAPER\Effects\"
```

---

## Uso

### Insertar el Plugin

1. En REAPER, crea o selecciona una pista
2. Haz clic en **FX** (o presiona `Ctrl+Shift+F` / `Cmd+Shift+F`)
3. Busca **"Fairchild 670"** en la lista de efectos
4. Haz clic en **Add** para insertarlo

### Controles Principales

#### Controles por Canal (L/R)

| Control | Rango | Descripción |
|---------|-------|-------------|
| **Input Gain** | -60 a +12 dB | Ganancia de entrada (simula el input level control) |
| **Threshold** | 0-100% | Umbral de compresión |
| **Time Constant** | 6 posiciones | Attack/Release presets |
| **Output Gain** | 0-100 dB | Makeup gain de salida |

#### Time Constants (Posiciones del Hardware Original)

| Posición | Attack | Release | Modo |
|----------|--------|---------|------|
| 1 | 0.2s | 0.3s | Fixed |
| 2 | 0.4s | 0.8s | Fixed |
| 3 | 0.6s | 2.0s | Fixed |
| 4 | 2.0s | 5.0s | Fixed |
| 5 | 0.2s | Auto-Fast | Program-dependent |
| 6 | 0.2s | Auto-Slow | Program-dependent |

> **Auto-Release**: Los modos 5 y 6 usan un sistema dual-branch que adapta automáticamente el tiempo de release según la señal. El modo Fast es ideal para transitorios (batería), mientras que el Slow es mejor para material sostenido (voces, cuerdas).

#### Controles Globales

| Control | Opciones | Descripción |
|---------|----------|-------------|
| **Mode** | Stereo Linked | Ambos canales usan el mismo CV |
| | L/R Independent | Cada canal tiene su propio sidechain |
| | Lateral-Vertical | Procesamiento M/S (para vinilo) |
| **Mix** | 0-100% | Dry/Wet (compresión paralela) |
| **Bypass** | Off/On | Activar/desactivar el plugin |
| **Preset** | 7 presets | Carga presets de fábrica |

### Factory Presets

| Preset | Descripción | Uso Recomendado |
|--------|-------------|-----------------|
| **Default** | Configuración estándar | Punto de partida |
| **Vinyl Mastering** | Compresión suave en M/S | Mastering para vinilo |
| **Vocal** | Attack rápido, makeup gain | Voces |
| **Drums** | Auto-Fast, paralelo | Batería |
| **Mastering** | Configuración equilibrada | Mastering general |
| **Gentle** | Compresión muy suave | Leveling suave |
| **Aggressive** | Compresión fuerte | Efecto más pronounced |

---

## Ejemplos de Uso

### Compresión para Voz

```
Input Gain: -6 dB
Threshold: 60%
Time Constant: 1 (0.2s/0.3s)
Output Gain: +6 dB
Mode: Stereo Linked
Mix: 100%
```

### Batería (Paralela)

```
Input Gain: -3 dB
Threshold: 70%
Time Constant: 5 (Auto-Fast)
Output Gain: +3 dB
Mode: Stereo Linked
Mix: 80%
```

### Mastering

```
Input Gain: -3 dB
Threshold: 45%
Time Constant: 3 (0.6s/2.0s)
Output Gain: +3 dB
Mode: Stereo Linked
Mix: 100%
```

### Vinilo (M/S)

```
Input Gain: 0 dB
Threshold: 40%
Time Constant: 4 (2.0s/5.0s)
Output Gain: 0 dB
Mode: Lateral-Vertical
Mix: 100%
```

---

## Technical Details

### Architecture

```
Input → [Input Transformer] → [Variable-Mu Stage] → [Tube Saturation] → [Output Transformer] → Output
                                    ↑
                              [Sidechain]
                                    ↑
                           [6AL5 Rectifier]
                                    ↑
                          [Envelope Follower]
                                    ↑
                           [Time Constants]
```

### Componentes Emulados

| Componente | Modelo | Parámetros |
|------------|--------|------------|
| **Tubo 6386** | Koren model con LUT 1024-point | x=1.0 (remote-cutoff), k=8.0 |
| **6AL5 Rectificador** | Soft knee parabólico | Knee=0.25, Full-wave |
| **Envelope Follower** | RC circuit model | Variable time constants |
| **Auto-Release** | Dual-branch blend | BLEND_SPEED=0.005 |
| **Transformers** | Biquad cascade | HPF 30Hz + LPF 30kHz, Q=0.707 |
| **Saturación** | tanh soft clipping | Drive variable |

### CPU Usage

- **44.1 kHz**: ~2.5% por instancia
- **48 kHz**: ~2.7% por instancia
- **96 kHz**: ~5.2% por instancia
- **192 kHz**: ~10.1% por instancia

> Puedes monitorizar el CPU en tiempo real en la interfaz gráfica del plugin.

---

## Troubleshooting

### El plugin no aparece en la lista de efectos

1. Verifica que el archivo esté en la carpeta correcta
2. Reinicia REAPER completamente
3. Ve a **Options** → **Preferences** → **VST** → **Re-scan**

### No hay audio / Plugin en bypass

1. Verifica que el **Bypass** esté en "Off"
2. Revisa que el **Mix** esté en 100%
3. Ajusta el **Input Gain** si la señal es muy silenciosa

### Errores de CPU

1. Reduce la cantidad de instancias
2. Usa el modo **L/R Independent** en lugar de **Stereo Linked**
3. Aumenta el buffer size en **Preferences** → **Audio** → **Device**

---

## Version History

| Versión | Cambios |
|---------|---------|
| **1.0.0** | Release inicial con todas las features |
| **0.9.0** | Optimización + CPU monitoring |
| **0.8.0** | Serialización + presets |
| **0.7.0** | Interfaz gráfica |
| **0.6.0** | Stereo Link + Matriz L/V |
| **0.5.0** | Emulación de transformadores |
| **0.4.0** | Time constants + auto-release |
| **0.3.0** | Sidechain detector |
| **0.2.0** | Variable-Mu gain stage |
| **0.1.0** | Template passthrough |

---

## Credits

- **Diseño original**: Walter E. Sear (Fairchild Recording Equipment Corp., 1959)
- **Emulación JSFX**: Dandi
- **Inspirado por**: Los diseños DIY de la comunidad de audio

---

## Contribuir

Las contribuciones son bienvenidas. Por favor:

1. Haz fork del repositorio
2. Crea un branch para tu feature (`git checkout -b feature/nueva-feature`)
3. Commit tus cambios (`git commit -m 'Add nueva feature'`)
4. Push al branch (`git push origin feature/nueva-feature`)
5. Abre un Pull Request

---

## Licencia

MIT License - Ver [LICENSE](LICENSE) para detalles.

---

## Links Útiles

- [REAPER](https://www.reaper.fm/) - DAW
- [ReaPack](https://forum.cockos.com/showthread.php?t=212174) - Gestor de paquetes
- [JSFX Documentation](https://www.reaper.fm/sdk/js/js.php) - Documentación oficial
- [Forum](https://forum.cockos.com/) - Comunidad REAPER

---

## Donaciones

Si este plugin te es útil, considera apoyar el desarrollo:

[![PayPal](https://img.shields.io/badge/PayPal-donar-blue.svg)](https://paypal.me/tuusuario)
[![Ko-fi](https://img.shields.io/badge/Ko--fi-donar-red.svg)](https://ko-fi.com/tuusuario)

---

**¡Gracias por usar el Fairchild 670 Vari-Mu Compressor!**
