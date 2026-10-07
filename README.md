# Acoustic Camera

**Four-microphone sound localisation on an STM32, with a live webcam overlay.**

Course project for EEE 416 – Microprocessor and Embedded Systems Laboratory, Department of Electrical and Electronic Engineering, BUET (January 2026).

Make a sound — a clap, speech, a door slam — and a marker appears on the live webcam picture at the place the sound came from.

Four INMP441 digital MEMS microphones sit at the corners of a 112 mm square, with a webcam in the centre. Sound reaches each microphone a few microseconds apart. An STM32 NUCLEO-F446RE measures those tiny time differences and turns them into a direction (azimuth and elevation). A Python program on the laptop draws that direction on the camera image.

The method is purely physics-based: no training data, no per-sound tuning. No audio is recorded or sent — only two angles leave the board.

## How it works

1. **Capture.** Two I²S peripherals run side by side: I²S2 as master and I²S3 as slave on the same clock. Each carries two microphones on one data line, selected by the L/R pin, so the board receives four channels at about 32 kHz (31.914 kHz from the I²S PLL). Circular DMA fills 1024-sample blocks with no CPU involvement.
2. **Sound gate.** The firmware tracks the background level of each microphone and only measures when the signal rises at least 2× above it (and above a fixed floor). A quiet room produces no false directions.
3. **GCC-PHAT.** For each microphone pair, the 1024-point FFTs (CMSIS-DSP `arm_rfft_fast_f32`, Tukey window) are combined into a cross-spectrum, band-limited to 300 Hz – 6 kHz, and normalised to unit magnitude (the PHAT weighting). Its inverse FFT has a sharp peak at the time delay between the two microphones, searched within ±12 samples. The largest physical delay across 112 mm is about 10.4 samples.
4. **Sub-sample delay.** One sample is 31 µs, too coarse for good angles. The firmware refines each peak by evaluating the correlation directly from the spectrum at fractional delays, first in 0.05-sample steps and then in 0.004-sample steps.
5. **Quality check.** Each pair gets a quality score: the peak height divided by the mean correlation in the search window. Measurements below the threshold (default 1.8) are reported as `WEAK` and not used.
6. **Direction.** The two horizontal pairs and the two vertical pairs are averaged, and the delays are converted to azimuth and elevation using the array spacing and the speed of sound (343 m/s). A result is sent over UART every 200 ms.
7. **Self-recovery.** If the slave I²S slips out of step with the master, the firmware detects it and re-synchronises automatically. Press `x` to force a re-sync.

## Hardware

| Item | Qty |
|---|---|
| STM32 NUCLEO-F446RE | 1 |
| INMP441 I²S MEMS microphone module | 4 |
| USB webcam (Havit) | 1 |
| 100 kΩ resistor (data-line pull-downs) | 2 |
| 120 Ω resistor (2 in parallel = 60 Ω, WS jumper) | 2 |
| 1 µF ceramic capacitor (one per mic) | 4 |
| 10 µF capacitor (3V3 bulk) | 1 |
| Breadboard, jumper wires, 3D-printed frame | — |

### Wiring

All microphones: **VDD → 3V3**, **GND → GND**, **SCK → PB13**, **WS → PB12**.

| Mic | Position | SD pin | L/R |
|---|---|---|---|
| L1 (m0) | top-left | PC3 | GND |
| L2 (m1) | bottom-left | PC3 | 3V3 |
| R1 (m2) | top-right | PC12 | GND |
| R2 (m3) | bottom-right | PC12 | 3V3 |

Slave clock jumpers on the NUCLEO: **PB13 → PC10** and **PB12 → PA4** (through 60 Ω).
100 kΩ pull-down from **PC3 → GND** and **PC12 → GND**. 1 µF across VDD–GND at every microphone.

## Building the firmware

1. Open `acoustic_camera.ioc` in **STM32CubeMX** and click *Generate Code* (toolchain: MDK-ARM).
2. Copy `main_sound_gated.c` into `Core/Src/` as `main.c`, replacing the generated one.
3. Open `acoustic_camera.uvprojx` in **Keil µVision**.
4. Enable **CMSIS → DSP** in *Project → Manage → Run-Time Environment*.
5. In *Options for Target*: Arm Compiler 6, optimisation `-O2`, *Use MicroLIB*, *Single Precision FPU*.
6. Build (F7) and flash (F8).

Key CubeMX settings: HCLK 180 MHz, PLLI2S = 96 MHz (M = 8, N = 192, R = 2), I²S2 master RX / I²S3 slave RX, Philips, 16-bit on 32-bit frame, 32 kHz, DMA circular, **PB12/PB13 GPIO speed = Very High**, USART2 115200 8N1.

## Running the overlay

```bash
pip install numpy opencv-python pyserial
python sound_overlay.py --port COM4 --hfov 62
```

- Replace `COM4` with your board's port (`/dev/ttyACM0` on Linux).
- `--hfov` is your webcam's horizontal field of view in degrees.
- **Close PuTTY first** — only one program can use the COM port.

**Overlay keys:** `m` mirror · `t` trail · `c` clear · `f` freeze · `s` save PNG · `q` quit

## Serial output and commands

Open the port in PuTTY at **115200 baud** to see what the board reports:

```
QUIET   lvl   112   bg   190   needs   380      ← no sound
WEAK    q 1.42 < 1.80   lvl 401 (x2.1)          ← sound, but not trusted
* AZ  +12.4   EL  -3.1   q 3.42   lvl 287       ← a measurement
ANGLE az=12.40 el=-3.10                         ← line read by the overlay
```

| Key | Action |
|---|---|
| `+` / `-` | less / more sensitive |
| `q` / `Q` | raise / lower the quality threshold |
| `z` / `Z` | set current direction as zero / clear |
| `v` | verbose (per-pair lags) |
| `c` | detailed pair-quality dump |
| `h` | show / hide QUIET lines |
| `x` | force I²S re-sync |

## Known limitations

- **Azimuth is less reliable than elevation.** The left–right pairs combine one microphone from each I²S peripheral, and the slave's clock travels over jumper wires, so they are not perfectly synchronised. The top–bottom pairs stay within one peripheral and work well.
- A flat array cannot tell front from back.
- Pure tones (beeps) are ambiguous — use broadband sounds such as claps or speech.
- One dominant sound source at a time.

## Repository contents

| File | Purpose |
|---|---|
| `acoustic_camera.ioc` | STM32CubeMX configuration (clocks, I²S, DMA, UART) |
| `acoustic_camera.uvprojx` | Keil µVision project |
| `main_sound_gated.c` | Firmware: capture, sound gate, GCC-PHAT, direction, serial output |
| `sound_overlay.py` | Python/OpenCV webcam overlay that reads the angles over serial |

## References

- C. H. Knapp and G. C. Carter, "The generalized correlation method for estimation of time delay," *IEEE Trans. ASSP*, 1976.
- InvenSense, *INMP441 datasheet*.
- STMicroelectronics, *RM0390 STM32F446 reference manual*.
- Arm, *CMSIS-DSP library*.

## Author

Designed, built and programmed by **Jarif Shahriar Ahmed** ([@archnoid98](https://github.com/archnoid98)), EEE, Bangladesh University of Engineering and Technology (BUET).
