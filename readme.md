# Small OS for ARM-Based Microcontroller


---

## Overview

A minimal but functional operating system built from scratch in C (with selective Assembly) for the **STM32F103C8T6** microcontroller (ARM Cortex-M3, 72 MHz). The project demonstrates that core OS principles — scheduling, interrupt-driven I/O, DMA, and filesystems — can be realized on a resource-constrained embedded platform with only 64 KB Flash and 20 KB SRAM.

Total kernel size: **< 10 KB**.

---

## Hardware

| Component | Details |
|---|---|
| MCU | STM32F103C8T6 — ARM Cortex-M3 @ 72 MHz |
| SRAM / Flash | 20 KB / 64 KB |
| Display | ST7789 IPS, 135×240, 16-bit color, SPI |
| Storage | 8 GB microSDHC, SPI mode |
| Communication | UART @ 115200 baud (keyboard/serial monitor) |
| Power | HW-131 supply, 3.3 V & 5 V / 700 mA |

---

## Building
 
**Requires:** `arm-none-eabi-gcc` and `st-flash` (Linux/macOS) or `ST-LINK_CLI` (Windows) installed and on your PATH.
 
**Build** — compiles all sources and produces `firmware.bin`:
```bash
make
```
 
**Flash** — writes `firmware.bin` to the STM32 over ST-LINK USB:
```bash
make flash-unix      # Linux / macOS
make flash-windows   # Windows
```
 
**Disassemble** — dumps the full disassembly of `firmware.elf` to `dasm.txt`:
```bash
make dasm
```
 
**Clean** — removes all build artefacts (`firmware.elf`, `firmware.bin`, `dasm.txt`, object files):
```bash
make clean
```


## Kernel Components

### Boot & Clock
- Custom linker script places the vector table at `0x08000000` (Flash origin)
- Reset handler copies `.data` from Flash → SRAM and zeroes `.bss` before jumping to `main`
- PLL configured to multiply the 8 MHz HSE crystal × 9 → **72 MHz** system clock
- Flash wait states and APB1 prescaler set before clock switch to avoid hard faults

### Interrupt Handling (NVIC / IVT)
- Interrupt Vector Table expressed as a C array of function pointers
- NVIC enables per-peripheral IRQ lines; nested/prioritized interrupts supported
- Hardware auto-stacks `PC, PSR, LR, R0–R3, R12` on interrupt entry

### UART Driver
- USART1 on APB2 (72 MHz), pins PB6/PB7, 115200 baud, 8N1
- **Interrupt-driven** with a 32-byte circular (ring) buffer — no polling
- IRQ 37 → IVT entry 53; triggers on RXNE (receive) and TXE (transmit)

### SPI & SD Card Driver
- SPI1 @ 18 MHz (MOSI/MISO/SCLK/CS on PA4–PA7)
- SD card initialized in SPI mode: CMD0 reset → CMD8 voltage check → ACMD41 activation → CMD58 capacity detection → CMD16 block-size set
- Block read/write via CMD17 / CMD24 (512-byte blocks)

### DMA Controller
- DMA1 channels 2 (SPI RX) and 3 (SPI TX) used for SD card transfers
- **Asynchronous request queue** — tasks submit requests without blocking the CPU; DMA completion interrupt dispatches the next queued request
- Read requests put the calling task into **blocked** state; DMA ISR wakes it on completion

### ST7789 Display Driver
- SPI2 on PB12–PB15; DC pin distinguishes commands from pixel data
- Initialization sequence: software reset → sleep out → inversion on → pixel format (RGB565) → display on
- Capable of rendering images loaded from SD card

### Task Scheduler
- **Preemptive Round-Robin** using the SysTick timer (~1 000 000 tick quantum)
- Each task has its own isolated stack; task state: **Ready** or **Blocked**
- SysTick ISR sets the PendSV pending bit; **PendSV** (lowest priority) performs the actual context switch — avoiding preemption of higher-priority ISRs
- Context switch saves/restores R4–R11 in Assembly; hardware saves the rest automatically

### Synchronization
- **Spinlocks** implemented with ARM `LDREX`/`STREX` exclusive monitor instructions — atomic read-modify-write without disabling interrupts or locking the bus
- Prevents race conditions on shared resources between concurrently scheduled tasks

### Filesystem
- Custom **FAT-based** filesystem on the SD card partition
- Layout: Superblock → FAT table (4-byte chain entries) → Data blocks
- Supports `open`, `read`, `write`, `close`; path resolution from root directory
- File descriptors track current offset and block position for O(1) sequential access
- FAT32 chosen over inode-based EXT2 due to lower RAM overhead on the 20 KB SRAM target

---

## Key Design Decisions

- **ROM-resident kernel** — entire OS and task code lives in Flash; SRAM holds stacks and data only (RTOS pattern, not desktop-style load-into-RAM)
- **Memory-mapped I/O** — all peripherals accessed via C structs cast to base addresses; no special I/O instructions needed
- **No third-party libraries** — every driver written from scratch against STM32 and ARM reference manuals
- **PendSV for context switching** — defers the switch until all higher-priority interrupts are serviced

---