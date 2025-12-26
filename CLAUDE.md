# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the x0xb0x project - an open-source TB-303 clone synthesizer. The repository contains three main components:

1. **firmware/** - AVR C code for the ATmega162 microcontroller
2. **c0ntr0l/** - Python GUI application for controlling the x0xb0x via serial
3. **bootloader/** - AVR bootloader for firmware updates
4. **hardware/** - Eagle PCB design files (not typically modified in software work)

## c0ntr0l Python Application

### Running the Application

The Python application is being ported from Python 2 (wxPython) to Python 3. Current work is on the `dev.msa.port-c0ntr0l-python3` branch.

```bash
# Set up virtual environment
python3 -m venv venv
source venv/bin/activate  # or venv\Scripts\activate on Windows

# Install dependencies (not fully documented yet - check setup.py)
pip install pyserial wxPython

# Run the application
python c0ntr0l/c0ntr0l.py
```

### Architecture - Model-View-Controller Pattern

The c0ntr0l application strictly follows the MVC pattern:

- **Model** (`model.py`): Handles serial communication, pattern storage, firmware upload
- **View** (`view.py`, `GraphicalInterface.py`): wxPython GUI components
- **Controller** (`controller.py`): Mediates between Model and View, handles configuration

**Critical MVC rule**: Model and View NEVER communicate directly. All messages pass through the Controller via defined interface methods (see `c0ntr0l/architecture.txt` for the complete protocol).

### Serial Communication Layer

The serial protocol is packet-based with CRC-8 error checking:

- **DataLink** (`communication.py`): High-level protocol implementation
- **Packet** (`packet.py`): Packet parsing and validation
- **Pattern** (`pattern.py`): 16-note pattern data structure

Packet structure: `[Type (1 byte)][Size (2 bytes)][Body (N bytes)][CRC (1 byte)]`

Message types include: read/write patterns, tempo control, sequencer start/stop, etc. See `c0ntr0l/serial_protocol.txt` for complete protocol specification.

### Key Python Modules

- `AvrProgram.py` - Bootloader protocol for uploading firmware via serial
- `IntelHexFormat.py` - Parse Intel HEX files (.hex) for firmware
- `PatternFile.py` - Load/save pattern banks to disk
- `PatternEditGrid.py` / `PatternPlayGrid.py` - Grid UI components for pattern editing/playback
- `NotificationCenter.py` - Event notification system
- `DataFidelity.py` - CRC-8 checksum implementation

### Pattern Data Structure

Patterns are 16 notes with attributes:
- Note pitch (MIDI-style, 0x17-0x23 for C to C')
- Accent, slide, rest flags
- Stored in EEPROM: 16 banks × 8 locations = 128 patterns total

Constants in `Globals.py`:
- `NOTES_IN_PATTERN = 16`
- `NUMBER_OF_BANKS = 16`
- `LOCATIONS_PER_BANK = 8`
- `DEFAULT_BAUD_RATE = 19200`

## Firmware (AVR C)

### Building Firmware

```bash
cd firmware
make              # Build x0xb0x.hex
make clean        # Remove build artifacts
make program      # Flash to device via avrdude (requires hardware programmer)
```

The Makefile is configured for AVR-GCC with ATmega162 target at 16MHz.

### Firmware Architecture

- `main.c` - Main loop, initialization, UART communication
- `compcontrol.c/h` - Computer control protocol (serial packet handler)
- `synth.c/h` - Analog synth control (CV, gate, filter)
- `pattern_edit.c/h`, `pattern_play.c/h`, `track_edit.c/h` - Sequencer modes
- `keyboard.c/h` - Front panel button/knob scanning
- `led.c/h` - LED display control
- `midi.c/h` - MIDI I/O
- `dinsync.c/h` - DIN sync clock I/O
- `eeprom.c/h` - Pattern storage in EEPROM
- `switch.c/h` - Switch matrix scanning

Sync modes: `INTERNAL_SYNC`, `DIN_SYNC`, `MIDI_SYNC`, `NO_SYNC`

Tempo range: 20-300 BPM (stored in EEPROM)

### Bootloader

```bash
cd bootloader
make program      # Flash bootloader + set fuses
make verify       # Verify flash contents
```

The bootloader occupies upper 2KB of flash (0x3E00-0x3FFF) and allows firmware updates via serial without a hardware programmer.

## Python 2 to Python 3 Porting Notes

This codebase was originally Python 2 with wxPython (Classic). When porting:

1. **Print statements**: Change `print 'foo'` to `print('foo')`
2. **Exception syntax**: Change `except Exception, e:` to `except Exception as e:`
3. **String/bytes**: Serial data is bytes in Python 3. Use proper encoding/decoding
4. **wxPython**: Import is now `import wx` instead of `from wxPython.wx import *`
5. **Dict methods**: `.has_key()` removed - use `in` operator
6. **Integer division**: `/` is float division, use `//` for integer division

## Serial Port Auto-Detection

On Linux/macOS: Searches `/dev/cu.*`, `/dev/ttyUSB*`, `/dev/tts/ttyUSB*`
On Windows: Tries COM1-COM10

Selected port stored in wx.Config preference file.

## Important Constants

- Serial: 19200 baud, 2 second timeout
- Pattern format: 16 bytes per pattern (see `file_formats.txt`)
- Packet CRC: CRC-8 polynomial (see `DataFidelity.py`)
- Firmware flash address: Bootloader starts at 0x3E00
