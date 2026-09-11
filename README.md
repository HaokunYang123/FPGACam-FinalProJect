# FPGA Camera and Image-Processing System

A CS122A course project by Haokun Yang: an **OV7670 camera → iCESugar Pro FPGA → SDRAM → TFT LCD**, with a **Pico 2 W → Wi-Fi/TCP → laptop** path for sending a frozen image. The FPGA applies four selectable pixel filters before storing the image.

This repository preserves the completed course project so it can be rebuilt after the hardware is taken apart. A future sensor-to-token-to-VLM research project will reuse the hardware in a separate development effort. **This version ends at the laptop image receiver; it needs no GPU, model weights, or API key.**

**First-time sequence:** clone → install OSS CAD Suite → build/program FPGA → configure/build/program Pico → start laptop receiver → capture, freeze, and send.

- [Original report and wiring diagram](hyang243_custom_lab_report.pdf): operation on page 2, wiring on page 4, system diagram on page 5, acknowledgements on pages 6–7.
- [Original demonstration video](https://youtu.be/cr1FI0NDv3g)
- [Saved example image](laptopRecevier/frame.png)

## Contents

- [Hardware and software](#hardware-and-software)
- [1. Clone the repository](#1-clone-the-repository)
- [2. Install OSS CAD Suite](#2-install-oss-cad-suite)
- [3. Build and program the FPGA first](#3-build-and-program-the-fpga-first)
- [4. Set up and build the Pico firmware](#4-set-up-and-build-the-pico-firmware)
- [5. Program the Pico second](#5-program-the-pico-second)
- [6. Start the laptop receiver](#6-start-the-laptop-receiver)
- [7. Capture, freeze, and send](#7-capture-freeze-and-send)
- [How the pieces work together](#how-the-pieces-work-together)
- [Troubleshooting](#troubleshooting)
- [Preserving and returning to this version](#preserving-and-returning-to-this-version)
- [Verification and scope](#verification-and-scope)
- [References and acknowledgements](#references-and-acknowledgements)

## Hardware and software

| Item | Configuration used here |
| --- | --- |
| FPGA | **iCESugar Pro**, Lattice **ECP5 LFE5U-25F**, CABGA256 package, speed grade 6; onboard SDRAM and iCELink programmer |
| Camera | **OV7670**, configured for **320 × 240 RGB565**, with an 8-bit parallel data connection |
| LCD | **4.3-inch 480 × 272 parallel RGB TFT**, connected as in the report; not an SPI-only display |
| Microcontroller | **Raspberry Pi Pico 2 W** (`pico2_w`, RP2350 with Wi-Fi) |
| Controls | Three external buttons: capture/live view, filter mode, and Pico image-send trigger |
| Connections | Board breakout/carrier, wires, resistors, and USB data cables as in the report |
| Debugging | The original setup included a debug probe. The Pico BOOTSEL/UF2 method below does not require one. |
| Computer | Git, terminal, VS Code, Python 3, and a network connection reachable by the Pico |

Use the report for physical wiring. FPGA package locations in `top.lpf` and Pico GPIO numbers are different numbering systems, not interchangeable connector pin numbers. Power down before reconnecting hardware, use a common ground, and observe the boards' 3.3 V signal levels.

The build instructions were checked on **Apple Silicon macOS**. Terminal examples use a POSIX shell (macOS/Linux), with Windows alternatives where relevant. Linux/Windows hardware operation has not been separately verified for this project.

### Recorded tool versions

| Tool | Recovery reference |
| --- | --- |
| OSS CAD Suite | Installed bundle `VERSION`: **20260331** |
| Yosys | **0.63+184**, git `240439bdb-dirty` |
| nextpnr-ecp5 | **0.10-15-g77ccf518** |
| Pico SDK | **2.2.0** |
| Arm GNU toolchain | **14_2_Rel1** |
| Picotool | **2.2.0-a4** |
| Pico extension-generated configuration | CMake **3.31.5**, Ninja **1.12.1** |

The Pico versions are recorded in the checked-in CMake/VS Code configuration. Prefer these for reproduction; verify any later tool upgrades separately. The fresh Pico build for this README used host CMake **4.3.2** with the recorded SDK/compiler and Ninja.

## 1. Clone the repository

```sh
git clone https://github.com/HaokunYang123/FPGACam-FinalProJect.git
cd FPGACam-FinalProJect
```

This is the **repository root**. Start each numbered terminal section here unless otherwise stated. In a new terminal, navigate back to your clone first.

The clone contains source, the report, and an example image. It does **not** contain OSS CAD Suite, the Pico SDK, generated bitstreams/firmware, or your Wi-Fi credentials. A new clone will not have the old `build/` directories.

## 2. Install OSS CAD Suite

Download an archive matching your OS and CPU from [YosysHQ's official OSS CAD Suite releases](https://github.com/YosysHQ/oss-cad-suite-build/releases). The suite includes **Yosys**, **nextpnr-ecp5**, and **ecppack**; Yosys alone is not the complete build toolchain.

Extract it outside the repository, for example to `~/tools/oss-cad-suite`. On Apple Silicon select `darwin-arm64`; use the matching Intel macOS, Linux, or Windows build elsewhere. For reproduction, look for the recorded 2026-03-31 bundle in release history.

On macOS, follow the upstream instructions to allow the downloaded tools to run; the extracted suite provides an `activate` script for this purpose. Then initialize the FPGA build terminal:

```sh
source "$HOME/tools/oss-cad-suite/environment"
yosys --version
nextpnr-ecp5 --version
```

Substitute your extraction path. Activate the environment again in each new FPGA terminal. Use a separate terminal for Pico/Python work to avoid mixing tool environments.

**Windows PowerShell:** initialize the suite with:

```powershell
. "$HOME\tools\oss-cad-suite\environment.ps1"
```

See the [upstream installation instructions](https://github.com/YosysHQ/oss-cad-suite-build#installation) for other shells and OS requirements.

## 3. Build and program the FPGA first

### Synthesize, place and route, and pack

From the repository root, with OSS CAD Suite active:

```sh
cd CustomProject-FPGA
mkdir -p build
yosys -p 'read_verilog -sv -I src src/top.sv; synth_ecp5 -top top -json build/top.json'
nextpnr-ecp5 --25k --package CABGA256 --speed 6 --json build/top.json --textcfg build/top.cfg --lpf top.lpf --freq 65
ecppack --svf build/top.svf build/top.cfg build/top.bit
cd ..
```

**Run each command only after the previous command succeeds.** On PowerShell, replace `mkdir -p build` with `New-Item -ItemType Directory -Force build`; the three tool commands can remain on single lines as above.

| Stage | Purpose | Output in `CustomProject-FPGA/build/` |
| --- | --- | --- |
| Yosys | Synthesize the SystemVerilog hardware description into an ECP5 netlist | `top.json` |
| nextpnr-ecp5 | Place logic and route connections, using `top.lpf` pin constraints | `top.cfg` |
| ecppack | Package the routed design for programming | **`top.bit`**, plus `top.svf` for suitable JTAG workflows |

These explicit commands are the checked build path. The Makefile also includes an ECP5 `.bit` rule and an iCE40 `.bin` rule. **iCESugar Pro uses ECP5: do not use the iCE40 `top.bin` target.** The explicit Yosys command provides the include directory and selects `top`. If maintaining the Makefile later, note that its dependencies do not enumerate all included RTL files.

`--freq 65` preserves the Makefile's default timing target; it is not camera FPS and does not make every clock 65 MHz. The PLL generates the SDRAM and LCD clocks separately. Review timing results and errors rather than suppressing them to obtain a bitstream.

### Program using iCELink

1. Connect the **iCESugar Pro's iCELink USB connection** using a data-capable cable.
2. Find the USB drive named **iCELink**.
3. Copy **`CustomProject-FPGA/build/top.bit`** onto it.
4. Wait for programming to finish before disconnecting/resetting. Check the programmer's error indication if programming fails.

The board vendor documents this [drag-and-drop programming method](https://github.com/wuxx/icesugar-pro#icelink). The iCELink drive is different from the Pico BOOTSEL drive. Do not copy a `.json`, `.cfg`, or Pico `.uf2` onto iCELink.

With the report's wiring restored, hold the capture button to check the camera/LCD path before proceeding. An image is not guaranteed before the first capture: SDRAM image contents do not survive power loss.

## 4. Set up and build the Pico firmware

### Install the official Raspberry Pi Pico extension

1. Install [VS Code](https://code.visualstudio.com/).
2. Install **Raspberry Pi Pico**, published by **Raspberry Pi** ([extension](https://marketplace.visualstudio.com/items?itemName=raspberry-pi.raspberry-pi-pico), [official guide](https://github.com/raspberrypi/pico-vscode)). This project uses the **C/C++ Pico SDK**, not MicroPython.
3. Use the extension's existing-project import/open workflow for **`CustomProject-PICO`**, the folder containing `CMakeLists.txt`. Select **Pico 2 W** and SDK **2.2.0**, and allow its SDK/compiler/CMake/Ninja/Picotool setup to complete.
4. For the first build, open **`CustomProject-PICO` itself as the workspace**, not the entire course folder. The checked-in `.vscode` paths assume that workspace root. Preserve the existing C source rather than generating a blank project over it.

The firmware targets `pico2_w`. A non-wireless Pico 2 cannot perform the Wi-Fi step; a Pico W/RP2040 is not the recorded target either.

### Configure Wi-Fi and the laptop address

From the repository root, copy the template **once**:

```sh
cp CustomProject-PICO/wifi_config.h.example CustomProject-PICO/wifi_config.h
```

Edit `CustomProject-PICO/wifi_config.h` to replace these example values:

```c
#define WIFI_SSID "your-network-name"
#define WIFI_PASSWORD "your-network-password"
#define LAPTOP_IP "192.168.1.100"
```

- Use a **2.4 GHz network** compatible with the firmware's WPA2 password-based connection.
- Find the laptop's **local IPv4 address** in its active network adapter settings. `LAPTOP_IP` is not `127.0.0.1`, a website, or the Pico's address.
- Both devices must be able to reach each other. Guest/client-isolated networks may block communication even when both have internet access.
- The Pico and receiver both use **TCP port 4242**. Allow the receiver through the laptop firewall on your trusted network.
- After changing network credentials or laptop IP, **rebuild and reprogram the Pico**.

The real header is Git-ignored. Do not overwrite it with the example during every rebuild or upload credentials. A UF2 built with real credentials contains them too; keep that firmware backup private.

### Compile

Configure the imported project and use the Pico extension's **Compile Project** action. Verify that it selects `pico2_w` and the Arm compiler rather than the laptop compiler.

Alternatively, after the extension installs its tools, use a terminal with CMake, Ninja, and the Arm toolchain available. From the repository root on macOS/Linux, the recorded default installation layout is:

```sh
export PICO_SDK_PATH="$HOME/.pico-sdk/sdk/2.2.0"
export PICO_TOOLCHAIN_PATH="$HOME/.pico-sdk/toolchain/14_2_Rel1"
export PATH="$PICO_TOOLCHAIN_PATH/bin:$HOME/.pico-sdk/cmake/v3.31.5/bin:$HOME/.pico-sdk/ninja/v1.12.1:$PATH"
cmake -S CustomProject-PICO -B CustomProject-PICO/build -G Ninja -DPICO_BOARD=pico2_w
cmake --build CustomProject-PICO/build --parallel 4
```

Substitute actual installation paths if your extension uses a different layout. On Windows, the extension's configure/build actions handle those paths.

Successful compilation produces:

- **`CustomProject-PICO/build/CustomProject-PICO.uf2`** — firmware to copy to the Pico.
- `CustomProject-PICO/build/CustomProject-PICO.elf` — for debugger-based loading/debugging.
- `CustomProject-PICO/build/compile_commands.json` — compiler/include information for the editor.

After moving the project or changing SDK/compiler versions, configure from a fresh build directory. Preserve any working UF2 first, then rename the old `build` folder to an unused backup location outside the repository and configure again. Do not reuse a CMake cache from another machine.

## 5. Program the Pico second

1. Disconnect the Pico's USB cable.
2. Hold its onboard **BOOTSEL** button while connecting USB, then release it.
3. Its USB boot drive appears, normally **RP2350** for Pico 2 boards.
4. Copy **`CustomProject-PICO/build/CustomProject-PICO.uf2`** onto that drive.
5. The Pico reboots into the application and attempts to connect to Wi-Fi.

BOOTSEL is not the external GPIO22 image-send button. These steps program the Pico; the FPGA should already have its separate `top.bit` programmed.

Open the Pico's **USB serial port** in a serial monitor for diagnostics. USB stdio is enabled and UART stdio is disabled in `CMakeLists.txt`. A 115200 monitor setting is conventional for the USB console, not the SPI rate. Expect `Connecting to Wi-Fi...` followed by `Connected.`. Repeated connection attempts indicate a network/configuration issue before image transfer begins.

The original setup included a debug probe, but the checked-in `Flash` task uses an `openocd.exe` path and is not portable unchanged to every OS. BOOTSEL/UF2 avoids the probe/OpenOCD dependency for initial programming. See [Raspberry Pi's board and programming documentation](https://www.raspberrypi.com/documentation/microcontrollers/pico-series.html).

## 6. Start the laptop receiver

Python's standard library can save raw bytes. Install **Pillow** to also generate a PNG. In a fresh terminal, starting at the repository root on macOS/Linux:

```sh
python3 -m venv "$HOME/.venvs/fpgacam-receiver"
source "$HOME/.venvs/fpgacam-receiver/bin/activate"
python -m pip install Pillow
cd laptopRecevier
python receiver.py
```

On later runs, just activate the environment, enter `laptopRecevier`, and start the script. The folder is intentionally referenced with its existing spelling: **`laptopRecevier`**.

Windows PowerShell equivalent, starting at the repository root:

```powershell
py -3 -m venv "$HOME\.venvs\fpgacam-receiver"
& "$HOME\.venvs\fpgacam-receiver\Scripts\python.exe" -m pip install Pillow
Set-Location laptopRecevier
& "$HOME\.venvs\fpgacam-receiver\Scripts\python.exe" receiver.py
```

Expected startup message:

```text
Listening on 0.0.0.0:4242 ... waiting for the Pico.
```

`0.0.0.0` means the receiver listens on the laptop's interfaces. Keep the laptop's actual LAN address in the Pico's `LAPTOP_IP`. Start the receiver **before pressing the image-send button**.

Each complete transfer saves in the receiver's current working directory:

- **`frame.bin`** — 153,600 raw bytes: 320 × 240 × 2.
- **`frame.png`** — decoded image, if Pillow is installed.

Each transfer replaces the previous output files. Running from `laptopRecevier` therefore overwrites its tracked sample PNG; save images elsewhere when comparing experiments. A short transfer skips PNG conversion, so an old PNG can remain: check the new byte-count message before treating the file as the latest capture. Stop the receiver with **Ctrl+C** when finished.

## 7. Capture, freeze, and send

With both boards programmed, the report's wiring restored, Wi-Fi connected, and the receiver waiting:

| Control | Behavior |
| --- | --- |
| **Capture/live view** — FPGA `Capturing` | Hold to request repeated captures. Release to stop requesting new captures; an in-progress frame can finish, leaving the last capture in SDRAM. |
| **Filter mode** — FPGA `buttonInput` | Press/release to cycle original → grayscale → warm posterized/cartoon-like → inverted → original. Hold capture to see freshly filtered pixels. |
| **Send image** — Pico GPIO22 | Active-low input with pull-up. Hold for about one second (or until the console prints `sending 0x01`), then release. Pico requests the image over SPI and sends it by TCP. Holding the button can initiate another transfer when it returns to idle. |

First demonstration:

1. Point the camera at a recognizable object and hold capture until the LCD updates.
2. While capturing, cycle through the four filters.
3. Release capture and allow the current frame to finish. Moving the object should no longer change the stored image.
4. Hold the Pico send button for about one second, then release. Leave capture released during transfer.
5. Look for `sending 0x01`, `read done`, and `wifi send OK` in the Pico console.
6. Confirm the laptop reports **153600 bytes (expected 153600)** and open the new `frame.png`.

The camera image occupies only part of the 480 × 272 LCD; this version does not scale to fill it or provide a touchscreen UI. LCD and SPI share SDRAM read access, so do not expect uninterrupted live display updates during SPI readout.

SPI is initialized at **1 MHz**. The image payload alone takes approximately **1.23 seconds** at that rate, before software gaps and Wi-Fi/TCP time. This is snapshot transfer, not 30-FPS wireless video.

## How the pieces work together

```text
OV7670: 8-bit data + PCLK / VSYNC / HREF
  ↓
FPGA capture: assemble two bytes into one RGB565 pixel
  ↓
FPGA filter: original / grayscale / posterized / inverted
  ↓
Asynchronous FIFO: camera clock → SDRAM controller clock
  ↓
SDRAM frame storage
  ├─ LCD read requests → display FIFO → RGB LCD + HSYNC / VSYNC / DE
  └─ SPI read requests → pixel register / serializer → Pico receive buffer
                                                        ↓
                                                    Wi-Fi / TCP
                                                        ↓
                                                Python receiver → PNG
```

### Details worth remembering

- **Pixels:** RGB565 uses 5 red, 6 green, and 5 blue bits. The 8-bit camera bus carries a 16-bit pixel in two transfers; bus width and pixel size differ.
- **Capture timing:** after a request, the state machine waits for VSYNC high, then collects bytes with VSYNC low and HREF high. It stops after 76,800 pixels. The FIFO carries words rather than frame-start markers, so downstream addressing relies on its counters staying aligned.
- **Filters:** `top.sv` changes pixels **before** the first FIFO. Selecting another mode does not reprocess a frozen image already in SDRAM.
- **Buffers:** each FIFO is 512 × 16 bits (1,024 bytes), not a complete frame. SDRAM stores the image. Pico separately holds `uint16_t Storage[76800]`, or 153,600 bytes.
- **Clocks:** the board supplies 25 MHz. The PLL produces 100 MHz for SDRAM logic and about 9.09 MHz for LCD pixel timing. The camera supplies `PCLK`; clock frequency is not frame rate.
- **Image layout:** SDRAM addressing uses a 480-pixel row stride. After the 320 camera pixels, `+161` skips 160 unused positions to the next row. SPI read addressing makes the same skip; data sent to the laptop is packed 320 × 240 without LCD padding.
- **Memory control:** requests/acknowledgements pace access. The LCD asks for refills at 64 or fewer FIFO words. A multiplexer selects LCD or SPI read addresses/requests; the current RTL still connects returned read data to the video FIFO during SPI mode.
- **SPI:** Pico is controller/master and FPGA is peripheral/slave. Pico writes 8-bit command `0x01`, then performs 16-bit reads while supplying SCK. FPGA sends pixel bits MSB-first on MISO. The application uses `spi_read16_blocking`, not application-configured DMA for this read.
- **TCP:** after the entire SPI read, Pico sends its memory bytes using lwIP callbacks. The little-endian buffer becomes byte pairs reconstructed as `lo | (hi << 8)` in Python. Each connection carries one fixed-size raw frame without a custom image header.

### Source map

| File | Purpose |
| --- | --- |
| [`top.sv`](CustomProject-FPGA/src/top.sv) | Top-level integration, filters, input FIFO, SDRAM write addressing |
| [`OV7670cameraSM.sv`](CustomProject-FPGA/src/OV7670cameraSM.sv) | Capture state machine, timing gates, byte assembly |
| [`OV7670MasterCommand.sv`](CustomProject-FPGA/src/OV7670MasterCommand.sv), [`OV7670_configROM.sv`](CustomProject-FPGA/src/OV7670_configROM.sv) | Camera configuration sequence and register values |
| [`asych_fifo.sv`](CustomProject-FPGA/src/asych_fifo.sv) | Separate read/write clocks, pointers, full/empty flags |
| [`SDRAM.sv`](CustomProject-FPGA/src/SDRAM.sv) | Memory initialization, refresh, reads, and writes |
| [`lcd_fb.sv`](CustomProject-FPGA/src/lcd_fb.sv) | Display FIFO, PLL connection, LCD/SPI read selection |
| [`lcd_timing.sv`](CustomProject-FPGA/src/lcd_timing.sv), [`ecspll.sv`](CustomProject-FPGA/src/ecspll.sv) | Display timing signals and generated clocks |
| [`SPI_MISO(slave).sv`](CustomProject-FPGA/src/SPI_MISO%28slave%29.sv) | SPI command reception and pixel serialization |
| [`top.lpf`](CustomProject-FPGA/top.lpf) | FPGA package pin assignments and constraints |
| [`CustomProject-PICO.c`](CustomProject-PICO/CustomProject-PICO.c) | `Init → SendingSignal → Receiving → WifiSend`, image buffer, TCP callbacks |
| [`CMakeLists.txt`](CustomProject-PICO/CMakeLists.txt) | Pico board, SDK versions, USB console, linked libraries |
| [`receiver.py`](laptopRecevier/receiver.py) | Fixed-length TCP receive and RGB565-to-PNG conversion |

## Troubleshooting

Check in order: **FPGA build/programming → camera/LCD → Pico Wi-Fi → SPI → TCP → PNG**. Compilation success is not physical wiring verification.

| Symptom | First checks |
| --- | --- |
| FPGA tools not found | Activate the correct OSS CAD Suite environment in this terminal. |
| FPGA include/output-directory errors | Work in `CustomProject-FPGA`, create `build`, and retain `-I src`. |
| Parser error with a different FPGA tool | Compare against the tested Yosys version/flow. The preserved camera port list contains a trailing comma that stricter parsers may reject. |
| No iCELink or Pico boot drive | Check the USB data cable and correct board connector. BOOTSEL applies to the Pico. |
| Missing `wifi_config.h` | Copy the tracked `.example` once, set local values, rebuild. |
| All Pico includes show red squiggles | Open `CustomProject-PICO` itself, configure/build, and check SDK/compiler paths plus `build/compile_commands.json`. Regenerate a cache with old paths after moving folders; reload VS Code or reset C/C++ IntelliSense if necessary. |
| No image before first capture | Hold capture; the camera can run while FPGA capture is waiting. |
| Noise, tint, rolling, or repeated images | Check report/LPF wiring, camera format, byte order, frame/line alignment, and the 480-pixel memory stride. The report records a previous P7/R7 pin-label mistake; do not retune registers to compensate for wiring. |
| Mode button appears ineffective | Capture new pixels while changing modes; frozen data retains its previous filter. |
| Wi-Fi retries forever | Check credentials, 2.4 GHz availability, and authentication compatibility. |
| SPI returns empty/wrong pixels | Capture first, release capture during readout, and check SPI wiring/format/common ground. |
| No laptop connection | Start receiver first; check compiled laptop IP, TCP 4242, firewall, and client isolation. Reflash after IP changes. |
| Shorter than 153600 bytes | Incomplete transfer; inspect Pico logs/network. The previous PNG may remain. |
| Raw data but no PNG | Install Pillow into the Python environment running the receiver. |
| Port 4242 already in use | Stop your earlier receiver with Ctrl+C, or identify the listener before changing anything. |

## Preserving and returning to this version

The source baseline is **`c335431c206c87dd2e1b8783ee51657f02f0a6b2`** (`Restore August 7 project version`). This README expansion changes documentation only. The original report/demo are historical course-project evidence, separate from a fresh build or the particular binary currently on a board.

Before dismantling a working setup, keep private copies of the actual working **FPGA `.bit`**, **Pico `.uf2`**, local Wi-Fi header, a successful image, and any photos supplementing the wiring diagram. Record the source commit and tool versions beside the binaries. Newly built files should not be assumed to be identical to the binaries currently flashed on the hardware.

To recover that source without resetting active work, use a separate clone:

```sh
git clone https://github.com/HaokunYang123/FPGACam-FinalProJect.git FPGACam-legacy-recovery
cd FPGACam-legacy-recovery
git switch --detach c335431c206c87dd2e1b8783ee51657f02f0a6b2
```

That historical commit has the older README; keep this expanded guide open from GitHub when rebuilding it. To improve the recovered source, first create a branch with `git switch -c my-camera-experiment`. Keep the upcoming sensor-to-token/VLM work in a separate branch or repository so this course-project reference stays recoverable.

When returning, follow the numbered setup steps, capture one image, compare LCD and laptop results, and only then change one part at a time.

## Verification and scope

The author reports that the physical course project worked and was passed. The report also records image noise/dots, an approximate cartoon filter, and unfinished ideas.

README verification on **2026-09-10**, in a temporary export of the unchanged source baseline:

- **FPGA:** Yosys synthesis, nextpnr place-and-route, and ecppack completed and produced `top.bit`. Reported clock timing checks passed under the supplied constraints. Existing tri-state and unused-address connection warnings were not removed by changing source.
- **Pico:** a fresh CMake/Ninja build with SDK 2.2.0 and the recorded Arm toolchain produced ELF/UF2 using **example network settings**. This verifies compilation, not Wi-Fi connectivity.
- **Hardware:** no boards were reprogrammed and no live camera/LCD/SPI/Wi-Fi demonstration was rerun for the documentation update. Timing-tool success is not a complete clock-domain-crossing audit.

The existing [`top_tb.sv`](CustomProject-FPGA/tb/top_tb.sv) is an exploratory simulation scaffold. It lacks an SDRAM behavioral model, and vendor primitives need simulation support. It is not a complete automated end-to-end test; `make top.sim` is not proof of a working LCD read path.

Implemented: four per-pixel modes, capture/freeze, LCD display, and button-triggered snapshot transfer. Planned but unfinished: ultrasonic triggering and LVGL/touchscreen controls. The cartoon effect is arithmetic/bit operations, not generative AI. Continuous Wi-Fi video, DMA-based image reception, learned visual tokens, and VLM inference are not implemented here.

## References and acknowledgements

- [Original report](hyang243_custom_lab_report.pdf), including full references and AI-use statement.
- [UCR CS122A resources](https://github.com/UCR-CS122A); FIFO/SDRAM source comments acknowledge Allan Knight's reference code.
- [iCESugar Pro board and programming documentation](https://github.com/wuxx/icesugar-pro)
- [YosysHQ OSS CAD Suite](https://github.com/YosysHQ/oss-cad-suite-build)
- [Official Pico VS Code extension](https://github.com/raspberrypi/pico-vscode) and [Pico SDK](https://github.com/raspberrypi/pico-sdk)
- [OV7670 configuration reference](https://github.com/westonb/OV7670-Verilog) and [additional camera reference](https://github.com/amsacks/OV7670-camera)
- [Nandland SPI slave](https://github.com/nandland/spi-slave) and [FPGA4Fun SPI](https://www.fpga4fun.com/SPI2.html)

The project combines course/reference modules with project-specific integration. As disclosed in the report, AI assisted with Wi-Fi/TCP code, the Python receiver, the cartoon-style filter, debugging, and learning. Use the report and source history to distinguish adapted components from independently developed work.
