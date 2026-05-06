# Programmable I/O (PIO) on the RP2040: Comprehensive Architecture Guide

**Standardized Reference for C3I (Université de Sherbrooke) Embedded Projects**

The following document is formatted for direct integration into the C3I GitHub repository. It provides a rigorous technical overview of the RP2040 Programmable I/O (PIO) subsystem, instruction set architectures, and implementation examples using MicroPython.

---

## 1. Architectural Overview

The RP2040 microcontroller (found on the Raspberry Pi Pico) deviates from traditional ARM Cortex architectures by including a dedicated hardware subsystem known as Programmable I/O (PIO). Instead of relying strictly on fixed hardware peripherals (like a standard silicon I2C or SPI block) or utilizing CPU cycles for bit-banging, the PIO provides a highly deterministic, dedicated processing unit for serial data.

**Hardware Specifications:**
*   **PIO Blocks:** 2 independent hardware blocks (PIO0 and PIO1).
*   **State Machines (SM):** 4 independent state machines per block (8 total).
*   **Instruction Memory:** 32-word instruction memory per block, shared among its 4 state machines.
*   **FIFOs:** Dedicated 4-word Transmit (TX) and Receive (RX) First-In-First-Out buffers for CPU-to-SM communication.
*   **Clock Scaling:** Fractional clock dividers allowing execution frequencies from the system clock down to low-kilohertz ranges.

## 2. Strategic Advantages and Disadvantages

### Advantages
*   **Deterministic Execution:** Each PIO instruction consumes exactly one clock cycle unless explicitly configured to delay or wait. This guarantees nanosecond-level timing precision without interrupt jitter.
*   **CPU Offloading:** Core 0 and Core 1 are completely relieved of bit-shifting operations. The CPU only needs to push/pull data to the DMA or FIFOs.
*   **Protocol Flexibility:** PIO can implement non-standard, legacy, or custom protocols (e.g., DVI video, WS2812, custom encoder interfaces) that lack dedicated silicon.
*   **Hardware Expansion:** If a project requires 6 UARTs and the RP2040 only has 2 hardware UARTs, the PIO can instantiate the remaining 4.

### Disadvantages
*   **Strict Memory Constraints:** The 32-instruction limit per block requires highly optimized assembly logic. Complex protocols must be heavily abstracted.
*   **Limited Computation:** The state machines possess no Arithmetic Logic Unit (ALU). They cannot perform multiplication, division, or floating-point operations. Logic is restricted to bit-shifting and basic decrements.
*   **Learning Curve:** Requires paradigm shifts from procedural C/Python to cycle-accurate assembly state management.

## 3. Tooling and Environment Prerequisites

To compile and deploy these PIO state machines via MicroPython, ensure the C3I development environments are correctly configured.

*   **Linux (Fedora / Wayland):** Accessing the serial port via Thonny or `mpremote` requires `dialout` group permissions. Execute `sudo usermod -aG dialout $USER`. Udev rules must be configured for the `2e8a:0005` vendor/product ID.
*   **Windows 10/11:** Ensure the standard USB CDC driver is active in Device Manager. Use PowerShell (`Get-PnpDevice`) to verify COM port allocation if Thonny fails to connect.

## 4. The PIO Instruction Set Architecture (ISA)

The PIO operates on a reduced instruction set comprising exactly nine operations.

### `in(source, bit_count)`
Shifts `bit_count` number of bits (1 to 32) from the `source` (pins, X/Y scratch registers, null, or status) into the Input Shift Register (ISR).

### `out(destination, bit_count)`
Shifts `bit_count` number of bits out of the Output Shift Register (OSR) into the `destination` (pins, X/Y registers, program counter, or execution state).

### `push(iffull, block/noblock)`
Pushes the contents of the ISR into the RX FIFO (towards the main CPU). If `block` is set, the state machine halts until the CPU reads the FIFO. Auto-push can be configured to execute this automatically when the ISR reaches a specific bit threshold.

### `pull(ifempty, block/noblock)`
Pulls a 32-bit word from the TX FIFO (from the main CPU) into the OSR. If `block` is set, the state machine halts until the CPU writes new data. Auto-pull is also available.

### `mov(destination, source)`
Copies a 32-bit value from the `source` to the `destination`. Used heavily for moving data between the OSR, ISR, and the X/Y scratch registers.

### `set(destination, data)`
Writes an immediate 5-bit integer (0 to 31) into the `destination` (pins, X/Y registers, or pin directions). Excellent for setting initial pin states or loop counters.

### `jmp(condition, label)`
Transfers execution to the `label`. Conditions include `not_x`, `x_dec` (decrement X and jump if not zero), `pin` (jump if the JMP pin is high), or no condition (unconditional jump).

### `wait(polarity, source, index)`
Stalls the state machine until the specified `source` (a GPIO pin or an IRQ flag) matches the `polarity` (0 or 1). Essential for synchronizing with external hardware clocks.

### `irq(clear/set/wait, index)`
Manipulates interrupt flags. Used to trigger interrupts on the main CPU or to synchronize multiple PIO state machines across the RP2040.

---

## 5. Implementation Examples (MicroPython)

The following examples utilize the `@rp2.asm_pio` decorator, the standard method for inline PIO assembly in MicroPython.

### Example A: WS2812B (NeoPixel) Driver
The WS2812 protocol requires strict 800 kHz timing with specific duty cycles for logical 0s and 1s. This example uses `.side_set` to toggle the pin concurrently with instruction execution, ensuring zero cycle-slip.

```python
import rp2
import machine

@rp2.asm_pio(sideset_init=rp2.PIO.OUT_LOW, out_shiftdir=rp2.PIO.SHIFT_LEFT, autopull=True, pull_thresh=24)
def ws2812_driver():
    T1 = 2
    T2 = 5
    T3 = 3
    wrap_target()
    label("bitloop")
    out(x, 1)               .side(0) [T3 - 1] 
    jmp(not_x, "do_zero")   .side(1) [T1 - 1] 
    jmp("bitloop")          .side(1) [T2 - 1] 
    label("do_zero")
    nop()                   .side(0) [T2 - 1] 
    wrap()

# Instantiation:
# sm = rp2.StateMachine(0, ws2812_driver, freq=8000000, sideset_base=machine.Pin(16))
# sm.active(1)
```

### Example B: Hardware Stepper Motor Control (Step/Dir)
Generating high-frequency square waves for stepper drivers can bog down a CPU. This PIO program accepts a pulse count from the CPU and generates the exact number of square waves independently.

```python
import rp2

@rp2.asm_pio(set_init=rp2.PIO.OUT_LOW)
def stepper_pulse_generator():
    pull(block)             # Wait for CPU to send pulse count
    mov(x, osr)             # Move pulse count into X register
    label("pulse_loop")
    set(pins, 1)      [15]  # Step Pin HIGH, delay 15 cycles
    set(pins, 0)      [15]  # Step Pin LOW, delay 15 cycles
    jmp(x_dec, "pulse_loop")# Decrement X, repeat until 0
```

### Example C: Simple SPI Transmitter
While the RP2040 has hardware SPI, the PIO can create additional SPI ports. This is a Mode 0 (CPOL=0, CPHA=0) transmitter. Side-set controls the Clock (SCK), while standard `out` controls the Master Out (MOSI).

```python
import rp2

@rp2.asm_pio(out_shiftdir=rp2.PIO.SHIFT_LEFT, autopull=True, pull_thresh=8, sideset_init=rp2.PIO.OUT_LOW)
def spi_tx_mode0():
    wrap_target()
    out(pins, 1)      .side(0) [1] # Output data bit, SCK remains low
    nop()             .side(1) [1] # SCK goes high, device samples data
    wrap()
```

### Example D: Unidirectional I2C (Simplified Write)
Full I2C implementation requires handling clock stretching and open-drain topologies. This snippet demonstrates the structural approach to generating an I2C Start Condition using PIO.

```python
import rp2

@rp2.asm_pio(set_init=(rp2.PIO.OUT_HIGH, rp2.PIO.OUT_HIGH), out_init=(rp2.PIO.OUT_HIGH, rp2.PIO.OUT_HIGH))
def i2c_start_condition():
    # Assumes set_base is SDA (Pin 0) and SCL (Pin 1)
    set(pindirs, 0b11)      # Set both SDA and SCL as outputs
    set(pins, 0b10)   [7]   # SDA goes LOW, SCL remains HIGH (Start Condition)
    set(pins, 0b00)   [7]   # Both go LOW, ready for data transmission
    # Data transmission logic would follow here...
```

---

## 6. Official Resources and Datasheets

To ensure rigorous compliance with silicon specifications, consult the following foundational documents:

*   **RP2040 Microcontroller Datasheet:** The definitive hardware specification, covering the PIO subsystem hardware registers in Chapter 3.
    *   *Link:* [RP2040 Datasheet (Raspberry Pi Foundation)](https://pip.raspberrypi.com/documents/RP-008371-DS-rp2040-datasheet.pdf)
*   **Raspberry Pi Pico Python SDK:** Official documentation for implementing the `rp2` module, decorators, and state machine class methods.
    *   *Link:* [Pico Python SDK (Raspberry Pi Foundation)](https://pip.raspberrypi.com/documents/RP-008388-KB-raspberry-pi-pico-python-sdk.pdf)
*   **Raspberry Pi Pico C/C++ SDK:** Even when coding in MicroPython, the C/C++ SDK contains the most exhaustive explanations of PIO logic, DMA integration, and state machine instruction timing.
    *   *Link:* [Pico C/C++ SDK](https://pip.raspberrypi.com/documents/RP-009085-KB-raspberry-pi-pico-c-sdk.pdf)
*   **Wokwi PIO Emulator:** An online tool for testing and debugging PIO assembly code cycle-by-cycle without flashing hardware.
    *   *Link:* [Wokwi PIOASM](https://wokwi.com/tools/pioasm)
