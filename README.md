# Moonlight PS3

Moonlight PS3 is an open-source PlayStation 3 client for NVIDIA GameStream and Sunshine game streaming servers. Built using the open-source PSL1GHT SDK toolchain, Tiny3D graphics engine, FreeType 2 vector font renderer, `cellVdec` hardware video decoding pipeline, and a dedicated SPU Opus audio offloading architecture, Moonlight PS3 brings modern low-latency game streaming to physical PlayStation 3 hardware (CFW / PS3HEN) and the RPCS3 emulator.

---

## Key Features

- **Hardware-Accelerated H.264 Video Decoding (`cellVdec`)**: Direct hardware decoding utilizing Sony's Cell Broadband Engine `libvdec` hardware subsystem (allocating 5 dedicated SPE cores for CAVLC/CABAC entropy parsing and parallel macroblock reconstruction), mapped directly to RSX graphics memory via zero-copy IOMMU mappings.
- **Dedicated SPU Opus Audio Offloading (`spu_opus`)**: Offloads Opus multistream audio decoding and 16-to-32-bit float PCM vector conversion to a dedicated Synergistic Processing Unit (SPE) worker thread. Paired with 5 SPEs for video decoding, all 6 usable GameOS SPEs are fully utilized with 0 host CPU audio overhead.
- **Clock Drift Elimination & Host Pacing**: Resolves the exact $60.000\text{ Hz}$ PC host vs $59.940\text{ Hz}$ PS3 NTSC clock frequency delta ($1\text{ extra frame}$ every $16.68\text{ seconds}$) across three synchronized tiers:
  - Protocol-level refresh negotiation (`clientRefreshRateX100 = 5994` for NTSC / `5000` for PAL), locking Sunshine/GFE host capture timers directly to the PS3 display hardware.
  - Zero-latency RSX presentation drain: snaps immediately to the freshest decoded frame on VBlank when multiple frames are queued, permanently locking display input latency to 0–16 ms without breaking H.264 GOP references.
  - Sized depacketizer queue (`decodeUnitQueue = 4`) preventing multi-second buffer bloat and reclaiming ~4.8 MB of heap RAM.
- **Cell PPE Hardware Flush-to-Zero (`FPSCR[NI]`) Optimization**: Hardware Flush-to-Zero (FTZ) enabled on the Cell PPE main and audio threads (`mtfsb1 29`), eliminating high-latency microcode assist interrupt traps when audio signals decay to silence or floating-point statistics approach subnormal thresholds ($0 < |x| < 1.17 \times 10^{-38}$).
- **Elastic 6-Buffer RSX Texture Pool**: Expanded frame buffer queue (`VDEC_FRAME_COUNT = 6`) providing elastic jitter absorption between VDEC decoding and RSX presentation, accompanied by clean `DR_NEED_IDR` recovery on hardware queue saturation.
- **Adaptive RSX Scaling Engine (AMD CAS & 4-Tap Area Box)**:
  - **AMD Contrast-Adaptive Sharpening (CAS)**: 5-tap unsharp mask shader for pristine upscaling (e.g. 720p -> 1080p) without edge ringing or halos.
  - **4-Tap Bilinear Box Downsampler**: 16-texel area-averaging shader suppressing moire and flicker on high-contrast text when downscaling (e.g. 1024x768/800x600 PC resolutions -> 480p/576p CRT modes).
  - **Native 4:3 Display Support**: Automatic hardware aspect ratio detection (`VIDEO_ASPECT_4_3` vs `VIDEO_ASPECT_16_9`) with filtered resolution menus and anamorphic aspect ratio correction.
- **Full USB & Bluetooth Keyboard and Mouse Support**:
  - **Dual Mouse Modes**: Supports both **Game Mode** (unbounded relative delta motion for 3D camera control) and **Desktop Mode** (high-precision 1:1 absolute coordinate motion for desktop navigation), configurable in the Stream Settings menu.
  - **Complete Mouse Input**: Full support for Left, Right, Middle, Side X1, and Side X2 buttons, alongside high-precision vertical scroll wheel input.
  - **Universal Keyboard Translation**: Accurate Win32 Virtual Key (VK) mapping for standard US layout keyboards (A-Z, 0-9, F1-F12, navigation cluster, arrows, numpad, and punctuation) with discrete modifier tracking (Ctrl, Shift, Alt, Meta/Win).
- **Interactive Host Game & Application Selection Menu**: Automatically queries and parses `/applist` XML from your Sunshine/GameStream server, displaying all available host games and applications in a scrollable Material UI list with active selection highlight, index counter `[ X / Y ]`, and one-click launching.
- **Native PS3 GameOS System Dialogs**: Official `<sysutil/msg.h>` pop-up confirmation modals ("Do you want to quit Moonlight and return to the PS3 XMB?") preventing accidental exits.
- **Persistent Configuration Engine**: Automatically saves and restores Sunshine Host IP, target FPS (30/60), bitrate (2.5, 5.0, 10.0, 12.5, 15.0 Mbps), scaling mode (Adaptive Shader / Legacy), mouse mode (Game/Desktop), RSX VSync mode, HUD stats visibility, and verbose logging across console reboots in `/dev_hdd0/game/MNLT00001/USRDIR/config.ini`.
- **High-Fidelity FreeType 2 Font Rendering**: Dynamically loads official PlayStation 3 console vector fonts (`SCE-PS3-RD-R-LATIN.TTF`, `SCE-PS3-SR-R-LATIN.TTF`) from `/dev_flash/data/font/` with subpixel anti-aliasing, accompanied by custom procedural vector PlayStation button glyphs (Cross ✕, Circle ◯, Triangle △, Square ◻, D-Pad ↕).
- **Zero-Overhead RSX Hardware VSync & Telemetry HUD**: Flexible flip mode configuration (`GCM_FLIP_VSYNC` for tear-free 60Hz or `GCM_FLIP_HSYNC` for ultra-low latency) paired with an on-screen Performance Stats HUD that introduces zero CPU/GPU overhead when disabled.
- **Universal NPDRM Retail Package Build Pipeline**: Native `ppu-strip`, `sprxlinker`, and `make_self_npdrm` integration generating retail-signed standalone PKG files compatible with both RPCS3 emulator and physical PS3 consoles (CFW / HEN).

---

## Controls & Key Bindings

| Action | Controller | Keyboard / Mouse |
| :--- | :--- | :--- |
| **Navigate Menus** | D-Pad Up / Down / Left / Right | Arrow Keys |
| **Select / Confirm / Launch** | Cross (✕) | Enter / Left Click |
| **Back / Exit to XMB (with Confirmation)** | Circle (◯) | Escape / Right Click |
| **Open On-Screen Keyboard (OSK)** | Cross (✕) on Host IP row | Physical Keyboard Direct Entry |
| **Toggle Settings Values** | Left / Right / Cross | Left / Right / Enter |
| **Emergency Stream Abort** | `Select + Start + L3 + R3` | `Ctrl + Shift + Alt + Q` |

---

## Building from Source (Retail Release Package)

### 1. Clone the Repository
```bash
git clone https://github.com/Cruslan/PS3-Moonlight
cd PS3-Moonlight
```

### 2. Prepare the PS3 SDK Environment

Moonlight PS3 requires a PSL1GHT v2 toolchain with critical RSX command buffer and memory alignment patches. Running `make prepare` automatically downloads and extracts the pre-built, patched SDK and PSL1GHT toolchain from [Cruslan/ps3dev](https://github.com/Cruslan/ps3dev) (`nightly-2026-11-05`) for your host operating system (macOS ARM64/x64 or Linux x64) directly into the project root directory (`./ps3dev`):

```bash
make prepare
```

#### Why a Patched SDK?
The toolchain distributed via [Cruslan/ps3dev](https://github.com/Cruslan/ps3dev) integrates vital stability fixes necessary for reliable hardware streaming:
- **`PSL1GHT` RSX Fixes**: Fixes a pointer scaling overflow in `rsxSetupContextData`, wraps 64-bit procedure descriptors in `__get_opd32` for 32-bit callback loaders, sets the required `allowTex` (bit 15 / `0x8000`) control bit in `LoadFragmentProgramLocation` (preventing RSX GPU lockups), and bounds FIFO spin loops in `ResetCommandBuffer`.
- **`Tiny3D` Hardware Alignment Fix**: Enforces `-mstrict-align` in Tiny3D to prevent GCC 13+ from emitting unaligned 64-bit `stdx` doubleword stores into RSX I/O memory, eliminating fatal Cell PPE hardware alignment traps (black screen DSI / Machine Check crashes).

#### Using an Existing or Custom SDK (Manual Patching)
If you already have a toolchain installed or prefer building upstream from source ([ps3dev/ps3dev](https://github.com/ps3dev/ps3dev)), you can apply these standalone patchsets manually from the [Cruslan/ps3dev patches layer](https://github.com/Cruslan/ps3dev/tree/main/patches):
```bash
# In your PSL1GHT repository checkout:
git apply /path/to/ps3dev/patches/PSL1GHT/*.patch

# In your ps3libraries repository checkout:
git apply /path/to/ps3dev/patches/ps3libraries/*.patch
```
After building and installing the patched SDK into your preferred prefix, simply export `PS3DEV` before compiling:
```bash
export PS3DEV=/path/to/ps3dev
```

### 3. Compile the Retail Package
Build the optimized standalone Retail Release package:
```bash
make
# or explicitly:
make pkg
```

### 4. Output Package
Upon completion, the build outputs the standalone signed package in the `build/` directory:
- `build/moonlight-ps3-<TIMESTAMP>.pkg`: Signed Retail NPDRM package ready for installation on any physical PS3 (CFW or PS3HEN) and RPCS3 emulator.

*(To clean build artifacts safely, run `make clean`).*

---

## Installation & Pairing

1. Copy the generated `moonlight-ps3-*.pkg` file to the root of a FAT32-formatted USB drive.
2. Plug the USB drive into your jailbroken PlayStation 3 console (running CFW or PS3HEN).
3. On the PS3 XMB, navigate to **Game > Package Manager > Install Package Files > Standard** and select the package to install.
4. Launch **Moonlight PS3** from your XMB Game column.
5. Select the **Host IP** row and enter your Sunshine/GameStream host PC's IP address using the native PS3 On-Screen Keyboard.
6. Select **Connect / Pair to Host**. A dedicated modal will display a 4-digit PIN.
7. Open the Sunshine Web UI on your PC, navigate to the **PIN** tab, and enter the 4-digit PIN shown on your PS3 screen to authorize the connection.
8. Once paired, browse and launch your host games or desktop from the interactive App List menu!

---

## Configuration Settings

Settings are persisted across reboots in `/dev_hdd0/game/MNLT00001/USRDIR/config.ini`:

| Setting | Default | Options | Description |
| :--- | :--- | :--- | :--- |
| **Host IP** | `192.168.1.X` | IPv4 Address | Target Sunshine / GameStream host IP. |
| **Resolution** | 720p / 1080p | 16:9 (720p, 1080p) / 4:3 (XGA, SVGA) | Filtered dynamically based on active display aspect ratio. |
| **Frame Rate** | 60 FPS | 30 FPS / 60 FPS | Target stream frame rate. |
| **Bitrate** | 10.0 Mbps | 2.5, 5.0, 10.0, 12.5, 15.0 Mbps | Optimal throughput ceiling tailored for PS3 Ethernet stability. |
| **Scaling Mode** | Adaptive Shader | Adaptive Shader (CAS / Box) / Legacy | High-quality shader filtering vs direct Tiny3D texture blit. |
| **Mouse Mode** | Game Mode | Game Mode / Desktop Mode | Relative raw delta motion (3D camera) vs absolute 1:1 pointer. |
| **RSX VSync** | VSync On | VSync (60 Hz) / HSync (Uncapped) | Hardware presentation sync mode. |
| **Stats HUD** | Off | On / Off | Real-time on-screen telemetry overlay (0 overhead when off). |

---

## Credits & Acknowledgments

- **[Moonlight-QT](https://github.com/moonlight-stream/moonlight-qt)**: Core client architecture, GameStream/Sunshine protocol handling, icon/graphics assets, and streaming implementation are adapted from Moonlight-QT.
- **[Opus Interactive Audio Codec](https://opus-codec.org/)**: Audio decoding functionality is powered by the Opus codec library.
- **[Moonlight Common C](https://github.com/moonlight-stream/moonlight-common-c)**: Common GameStream client protocol library.
- **[PS3DEV / PSL1GHT SDK](https://github.com/ps3dev/ps3dev)**: Enormous thanks to the open-source PS3 toolchain and PSL1GHT SDK developers whose foundational work, cross-compilers, hardware header definitions, and libraries made this PlayStation 3 port possible.
- **[Mohammed Asif (mohasi)](https://codeberg.org/mohasi)**: Special thanks for technical guidance, problem-solving support, and project inspiration.
- **AcidNT3.1**: Testing, validation, and feedback during development.
- **SyrianClippy**: Testing, validation, and feedback during development.
- **Okeanos**: Testing, validation, and feedback during development.

---

## License

This project is licensed under the GNU General Public License v3.0 (GPLv3).
