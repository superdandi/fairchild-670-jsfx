# Fairchild 670 Vari-Mu Compressor

JSFX plugin emulating the legendary Fairchild 670 tube compressor for REAPER.

## Features

- **Variable-Mu Gain Stage**: Authentic 6386 remote-cutoff tube emulation with LUT
- **Sidechain Detector**: 6AL5 soft rectifier with RC envelope follower
- **Time Constants**: 6 positions including dual-branch auto-release modes
- **Transformer Emulation**: Input/output transformers with biquad filters + saturation
- **Stereo Processing**: Stereo Linked, L/R Independent, Lateral-Vertical (M/S)
- **Graphics Interface**: VU meters, knobs, gain reduction display, transfer curve
- **Factory Presets**: 7 reference presets for different applications
- **Optimized DSP**: Anti-denormal, CPU monitoring, efficient algorithms

## Installation

### Manual Installation

1. Download `Fairchild670.jsfx`
2. Place in REAPER's JSFX folder:
   - Windows: `%APPDATA%\REAPER\Effects\`
   - macOS: `~/Library/Application Support/REAPER/Effects/`
   - Linux: `~/.config/reaper/Effects/`
3. Restart REAPER or refresh effects list

### ReaPack Installation

1. Open REAPER
2. Go to Extensions > ReaPack > Import repositories
3. Add: `https://github.com/yourusername/fairchild-670-jsfx/raw/main/index.xml`
4. Click "Apply" to install

## Usage

1. Insert Fairchild 670 on a track
2. Use the Preset slider to load factory presets
3. Adjust Input Gain, Threshold, and Time Constant as needed
4. Use Mix for parallel compression
5. Select Mode for stereo processing options

## Controls

### Channel Controls (L/R)
- **Input Gain**: -60 to +12 dB
- **Threshold**: 0-100%
- **Time Constant**: 0.3s to 5.0s + Auto modes
- **Output Gain**: 0-100 dB (makeup)

### Global Controls
- **Mode**: Stereo Linked / L/R Independent / Lateral-Vertical
- **Mix**: Dry/Wet blend (0-100%)
- **Bypass**: On/Off
- **Preset**: Factory preset selector

## Technical Details

- **Tube Model**: 6386 remote-cutoff with 1024-point LUT
- **Sidechain**: 6AL5 soft rectifier + RC envelope follower
- **Auto-Release**: Dual-branch system for program-dependent behavior
- **Transformers**: Biquad HPF (30Hz) + LPF (30kHz) + tanh saturation
- **CPU Usage**: <3% per instance at 44.1kHz

## Version History

- **1.0.0**: Initial release with all core features
- **0.9.0**: Optimization + CPU monitoring
- **0.8.0**: Serialization + presets
- **0.7.0**: Graphics interface
- **0.6.0**: Stereo Link + L/V matrix
- **0.5.0**: Transformer emulation
- **0.4.0**: Time constants + auto-release
- **0.3.0**: Sidechain detector
- **0.2.0**: Variable-Mu gain stage
- **0.1.0**: Pass-through template

## Credits

Based on the original Fairchild 670 hardware (1959) designed by Walter E. Sear.

## License

MIT License - Free to use and modify.
