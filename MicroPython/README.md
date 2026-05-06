# Comprehensive MicroPython Guide: ESP32-S3 & RP2040

This guide provides a complete workflow for establishing a MicroPython development environment and implementing advanced embedded systems concepts, specifically tailored for the **Raspberry Pi RP2040** and the **Espressif ESP32-S3**.

## Table of Contents
1. [Environment Setup (Thonny IDE)](#1-environment-setup-thonny-ide)
2. [Flashing MicroPython Firmware](#2-flashing-micropython-firmware)
3. [Library & Package Management](#3-library--package-management)
4. [Tutorial 1: WS2812 (NeoPixel) Control](#4-tutorial-1-ws2812-neopixel-control)
5. [Tutorial 2: PID Controller](#5-tutorial-2-pid-controller)
6. [Tutorial 3: Digital Signal Processing with ulab](#6-tutorial-3-digital-signal-processing-with-ulab)
7. [Official Sources & References](#7-official-sources--references)

---

## 1. Environment Setup (Thonny IDE)

Thonny is the officially recommended IDE for MicroPython development. It handles serial connections, file transfer, and basic package management out of the box.

### Windows Setup
1. Navigate to the official [Thonny Website](https://thonny.org/).
2. Download the Windows installer (`.exe`).
3. Run the installer and follow the standard prompts.
4. **Driver Note:** Windows 10/11 natively supports the USB CDC Serial drivers required for the RP2040 and ESP32-S3. No additional drivers are usually necessary.

### Linux Setup (Debian/Ubuntu/Fedora)
You can install Thonny via your package manager or `pip`. Using `pip` ensures you get the latest version.
```bash
# Install via pip
pip3 install thonny

# CRITICAL: Serial Port Permissions
# You must add your user to the 'dialout' group to access /dev/ttyACM0 or /dev/ttyUSB0 without sudo.
sudo usermod -aG dialout $USER

# Log out and log back in (or restart your computer) for the group change to take effect.
```

---

## 2. Flashing MicroPython Firmware

Before writing code, the microcontrollers must be flashed with the MicroPython interpreter.

### RP2040 (Raspberry Pi Pico / Custom Boards)
The RP2040 features a UF2 bootloader in ROM, making flashing as simple as dragging and dropping a file. This process is identical on **Windows and Linux**.

1. Download the latest `.uf2` firmware file from the [Official MicroPython RP2040 Release Page](https://micropython.org/download/RPI_PICO/).
2. Hold down the **BOOTSEL** button on your RP2040 board.
3. While holding the button, plug the board into your computer via USB.
4. Release the button. A mass storage drive named `RPI-RP2` will appear on your system.
5. Drag and drop the `.uf2` file onto the `RPI-RP2` drive. The board will automatically reboot and run MicroPython.

### ESP32-S3
Unlike the RP2040, the ESP32-S3 requires a serial flasher to write to its flash memory. Thonny has this built-in, making it easy on both OS ecosystems.

1. Download the latest `.bin` firmware file from the [Official MicroPython ESP32-S3 Release Page](https://micropython.org/download/ESP32_GENERIC_S3/).
2. Connect your ESP32-S3 to your computer via USB.
3. Open Thonny. Navigate to **Run > Configure interpreter...**
4. Select **MicroPython (ESP32)** from the dropdown.
5. Click the **Install or update MicroPython** link in the bottom right corner.
6. Select your target port (e.g., `COM3` or `/dev/ttyACM0`), browse for the `.bin` file you downloaded, and click **Install**.
    * *Alternative CLI Method:* Advanced users can use the [esptool.py command-line utility](https://docs.espressif.com/projects/esptool/en/latest/esp32/).

---

## 3. Library & Package Management

MicroPython uses a package manager called `mip` (MicroPython Install Package). Thonny provides a GUI for this.

### Installing Libraries via Thonny (Windows & Linux)
1. Ensure your board is connected and the Thonny REPL (shell) is active.
2. Go to **Tools > Manage packages...**
3. Search for a library (e.g., `ssd1306` or `simple-pid`).
4. Click **Install**. Thonny will automatically create a `/lib` folder on your microcontroller and download the `.py` files into it.

### Installing Libraries via Code (`mip`)
If your ESP32-S3 or Pico W is connected to Wi-Fi, you can install packages programmatically:
```python
import mip
# Install from the official micropython-lib repository
mip.install("requests")
# Install directly from a GitHub raw URL
mip.install("github:username/repo/module.py")
```

---

## 4. Tutorial 1: WS2812 (NeoPixel) Control

The `neopixel` module is baked into the standard MicroPython firmware. On the RP2040, it utilizes the hardware PIO for perfect timing. On the ESP32-S3, it utilizes the RMT (Remote Control) peripheral.

**Wiring:** Connect the Data-In (DI) of your WS2812 strip to **GPIO 16**. Ensure the strip shares a common ground (GND) with the microcontroller.

```python
import machine
import neopixel
import time

# Configuration
PIN_NUM = 16
NUM_LEDS = 12

# Initialize the NeoPixel object
# Pin 16 is valid for both ESP32-S3 and RP2040
pin = machine.Pin(PIN_NUM, machine.Pin.OUT)
np = neopixel.NeoPixel(pin, NUM_LEDS)

def color_chase(color, delay_ms):
    """Animates a single color moving across the LED strip."""
    for i in range(NUM_LEDS):
        np.fill((0, 0, 0)) # Turn all pixels off
        np[i] = color      # Set the target pixel (R, G, B)
        np.write()         # Push data to the strip
        time.sleep_ms(delay_ms)

# Run a continuous chase animation (Red, Green, Blue)
try:
    while True:
        color_chase((255, 0, 0), 50)
        color_chase((0, 255, 0), 50)
        color_chase((0, 0, 255), 50)
except KeyboardInterrupt:
    np.fill((0,0,0))
    np.write()
    print("Animation stopped.")
```

---

## 5. Tutorial 2: PID Controller

A Proportional-Integral-Derivative (PID) controller is essential for closed-loop robotics and heating systems. 

**The Mathematical Model:**
$$u(t) = K_p e(t) + K_i \int_{0}^{t} e(\tau) d\tau + K_d \frac{de(t)}{dt}$$

This example provides a robust, discrete-time PID class that can be easily dropped into any project.

```python
import time

class PID:
    def __init__(self, Kp, Ki, Kd, setpoint=0):
        self.Kp = Kp
        self.Ki = Ki
        self.Kd = Kd
        self.setpoint = setpoint
        
        self._last_time = time.ticks_ms()
        self._last_error = 0
        self._integral = 0
        self.output_limits = (None, None)

    def compute(self, measurement):
        now = time.ticks_ms()
        dt = time.ticks_diff(now, self._last_time) / 1000.0 # Convert to seconds
        
        if dt <= 0.0:
            return 0
            
        error = self.setpoint - measurement
        
        # Proportional term
        P = self.Kp * error
        
        # Integral term
        self._integral += error * dt
        I = self.Ki * self._integral
        
        # Derivative term
        derivative = (error - self._last_error) / dt
        D = self.Kd * derivative
        
        # Compute total output
        output = P + I + D
        
        # Apply output limits (Anti-windup strategy)
        min_out, max_out = self.output_limits
        if min_out is not None and output < min_out:
            output = min_out
        elif max_out is not None and output > max_out:
            output = max_out
            
        # Save state for next loop
        self._last_error = error
        self._last_time = now
        
        return output

# --- Implementation Example ---
# Simulated system: A heater trying to reach 50.0 degrees
pid = PID(Kp=2.0, Ki=0.5, Kd=1.0, setpoint=50.0)
pid.output_limits = (0, 100) # Output is 0% to 100% power

current_temp = 20.0 # Starting temperature

for i in range(15):
    # Compute the required heater power
    power = pid.compute(current_temp)
    
    # Simulate the heater warming up the system
    # (In reality, 'current_temp' would be read from an ADC or sensor)
    current_temp += (power * 0.1) 
    
    print(f"Step {i:02d} | Temp: {current_temp:.2f} °C | Heater Power: {power:.2f}%")
    time.sleep_ms(100)
```

---

## 6. Tutorial 3: Digital Signal Processing with `ulab`

Executing complex mathematics using standard Python lists is incredibly slow and fragments RAM. MicroPython solves this with `ulab` (Micro-lab), a C-compiled NumPy equivalent. 

*Note: `ulab` is pre-compiled into the official firmware releases for both the RP2040 and ESP32-S3. You do not need to install it via `mip`.*

```python
import gc
import math
from ulab import numpy as np

print("ulab successfully imported!")

# 1. High-Speed Array Generation
# Create a contiguous 1D array of 10 points from 0 to 2*Pi
time_steps = np.linspace(0, 2 * math.pi, num=100)

# 2. Vectorized Math (No Python loops required)
# Generate a sine wave based on the time steps
signal = np.sin(time_steps)

# 3. Fast Array Manipulation
# Add a random noise floor to the signal
rng = np.random.Generator(123456)
noise = rng.random(size=(100,)) * 0.1
noisy_signal = signal + noise

# 4. Statistical Analysis
max_val = np.max(noisy_signal)
min_val = np.min(noisy_signal)
mean_val = np.mean(noisy_signal)

print("-" * 30)
print(f"Signal Max:  {max_val:.3f}")
print(f"Signal Min:  {min_val:.3f}")
print(f"Signal Mean: {mean_val:.3f}")
print("-" * 30)

# 5. Filtering (Moving Average via convolution)
# Create a simple 3-point moving average kernel
kernel = np.array([1/3, 1/3, 1/3])
# Convolve the noisy signal with the kernel to smooth it out
smoothed_signal = np.convolve(noisy_signal, kernel)

print(f"First 5 raw values:      {noisy_signal[:5]}")
print(f"First 5 smoothed values: {smoothed_signal[:5]}")

# Always collect garbage after large array operations
gc.collect()

```

---

## 7. Official Sources & References

To delve deeper into the architectures and software discussed in this guide, refer to the following official documentation:

*   **Thonny IDE:** [thonny.org](https://thonny.org/)
*   **MicroPython Downloads:**
    *   [RP2040 Firmware Releases](https://micropython.org/download/RPI_PICO/)
    *   [ESP32-S3 Firmware Releases](https://micropython.org/download/ESP32_GENERIC_S3/)
*   **MicroPython Core Documentation:**
    *   [Package Management (`mip`)](https://docs.micropython.org/en/latest/reference/packages.html)
    *   [NeoPixel Library API](https://docs.micropython.org/en/latest/library/neopixel.html)
*   **ulab (NumPy for MicroPython):** [micropython-ulab.readthedocs.io](https://micropython-ulab.readthedocs.io/en/latest/)
*   **Hardware Datasheets:**
    *   [Raspberry Pi RP2040 Datasheet (PDF)](https://datasheets.raspberrypi.com/rp2040/rp2040-datasheet.pdf)
    *   [Espressif ESP32-S3 Technical Reference (PDF)](https://www.espressif.com/sites/default/files/documentation/esp32-s3_technical_reference_manual_en.pdf)
