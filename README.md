# 🤖 J.A.R.V.I.S. — ESP32-S3 Edge-AI Conversational Voice Assistant

A low-latency, full-duplex conversational voice assistant engineered on the **ESP32-S3 (Xtensa Dual-Core LX7, 8MB Octal PSRAM)** using **ESP-IDF v6.1** and **FreeRTOS**.

Features 100% on-device offline wake-word detection, full-duplex WebSocket streaming, an animated OLED facial expression engine (LVGL v9), custom DSP audio enhancements, and a J.A.R.V.I.S. British AI persona with multi-LLM backend support (DeepSeek V4 / GPT-5 / Qwen) and RAG.

> **Attribution & Acknowledgement:**  
> This project builds upon the open-source **[Xiaozhi (小智) ESP32 Framework](https://github.com/708xiaozhi/xiaozhi-esp32)**. The foundational networking protocols, WebSocket bridge, and base state machine are provided by the Xiaozhi open-source community. This repository contains custom board-level porting, physical pinout remapping, FreeRTOS power-on timing stabilization, I2C clock derating, acoustic DSP audio pre-amplification/clamping, and the integrated procedural OLED Face Engine customized for breadboard hardware.

---

## ⚡ System Architecture

- **Microcontroller:** ESP32-S3-DevKitC-1 N16R8 (240MHz Dual-Core, 16MB Flash, 8MB Octal PSRAM)
- **Audio Input:** INMP441 MEMS Microphone (I2S0 RX @ 16kHz, 24-bit PCM)
- **Audio Output:** MAX98357A 3W Class-D Audio Amplifier (I2S1 TX @ 24kHz, 32-bit slot) in Bridge-Tied Load (BTL) configuration
- **Acoustic Transducer:** 5cm (50mm) 4Ω/8Ω Hi-Fi Dynamic Speaker
- **Display:** SSD1306 0.96" OLED (128x64) running procedural animated eyes and mouth (`FaceEngine`)
- **Cloud AI Pipeline:** Streaming ASR (SenseVoice) → DeepSeek V4 / GPT-5 LLM (RAG Vector Knowledge Base) → Neural TTS

---

## 🔊 Speaker & Acoustic Engineering

Driving clean, room-filling sound from a small embedded microcontroller presents several physical and electrical challenges:

### 1. Bridge-Tied Load (BTL) Output Drive
The MAX98357A amplifier outputs audio across an H-bridge in **Bridge-Tied Load (BTL)** configuration directly across the speaker terminals:
$$P_{\text{RMS}} = \frac{V_{\text{DD}}^2}{2 \times R_{\text{load}}}$$
- Powered from the **3.3V rail**: Max output into a $4\Omega$ load is only $\approx \mathbf{1.36\text{ Watts}}$ (heavy clipping, low volume).
- Powered from the **5.0V VBUS rail**: Delivers up to $\approx \mathbf{3.125\text{ Watts}}$ clean power ($\approx 2.3\times$ higher acoustic power headroom) without causing voltage dips on the 3.3V LDO regulator.
- The GAIN pin is held at 5V via a **68kΩ pull-up resistor**, establishing a **15 dB analog gain boost** optimized for soft speech synthesis.

### 2. Acoustic Short-Circuiting (Phase Cancellation Physics)
A speaker cone radiates sound simultaneously from its front and rear faces $180^\circ$ ($\pi$ radians) out of phase:
$$p_{\text{net}}(t) = A\sin(\omega t) + A\sin(\omega t + \pi) = 0$$
In free air without a baffle, low-frequency pressure waves wrap around the edge of the speaker frame, causing destructive phase cancellation (resulting in thin, quiet, tinny audio). Placing the 5cm speaker inside a sealed chamber isolates the rear radiation, boosting effective Sound Pressure Level (SPL) by **$3\text{ dB} - 6\text{ dB}$** with rich vocal resonance.

---

## 🔧 Key Engineering Challenges & Solutions

1. **Signal Integrity & I2C Bus Stabilization:**  
   The OLED display previously locked up during startup. Breadboard jumper parasitic capacitance (~100pF) slowed rise times, violating the 400kHz Fast-Mode spec ($t_r \le 300\text{ ns}$). Derated the clock to 100kHz Standard Mode and added a 100ms startup delay (`vTaskDelay`) for internal charge pump stabilization and GPIO 46 strapping pin evaluation.

2. **Dual-Core FreeRTOS Task Pinning:**  
   Separated processing pipelines across cores to eliminate audio buffer underruns:
   - **Core 0:** Network I/O (TLS 1.3), persistent WebSockets, and I2S DMA interrupts.
   - **Core 1:** On-device neural keyword spotting (ESP-SR WakeNet9) and LVGL v9 animated graphics rendering.

3. **Audio DSP & Saturation Clamping:**  
   Replaced quadratic volume attenuation with linear gain scaling, applied a 3x digital pre-amp multiplier for cloud TTS headroom, and implemented mathematical saturation clamping (`INT32_MAX`/`INT32_MIN`) to eliminate integer overflow audio pops.

4. **Power Rail Isolation:**  
   Separated sensitive logic (ESP32/Mic on 3.3V LDO) from the high-transient 3W amplifier (connected to 5V VBUS) to prevent brownout resets during simultaneous Wi-Fi packet bursts and loud audio transients.

---

## 🚀 Hardware Pinout

| Peripheral | Board Pin | ESP32-S3 GPIO | Bus / Protocol | Role |
| :--- | :--- | :--- | :--- | :--- |
| **INMP441 (Mic)** | WS | GPIO 4 | I2S0 Word Select | 16 kHz Frame Clock |
| | SCK | GPIO 5 | I2S0 Bit Clock | 1.024 MHz Shift Clock |
| | SD | GPIO 6 | I2S0 Serial Data In | 24-bit PCM Audio Input |
| **MAX98357A (Amp)** | DIN | GPIO 7 | I2S1 Serial Data Out | PCM Audio Output |
| | BCLK | GPIO 15 | I2S1 Bit Clock | 1.536 MHz DAC Clock |
| | LRC | GPIO 16 | I2S1 Word Select | 24 kHz Frame Clock |
| | SD_MODE | GPIO 8 | Digital Output | Amp Enable / Shutdown |
| | GAIN | 68kΩ to 5V | Analog Config | 15 dB Gain Mode |
| **SSD1306 (OLED)** | SDA | GPIO 18 | I2C0 Serial Data | Display Commands & Framebuffer |
| | SCL | GPIO 46 | I2C0 Serial Clock | 100 kHz Standard Mode |
| **5cm Speaker** | +/- | Amp Terminals | BTL Analog | Differential Audio Output |

---

## 🛠️ Build & Flash Instructions

Requires **ESP-IDF v6.1**:

```powershell
# 1. Build firmware with animated Face Engine & Jarvis wake word
python scripts/build.py bread-compact-wifi --name bread-compact-wifi-128x64 --language en-US --wake-word wn9_jarvis_tts

# 2. Flash and monitor output
idf.py erase-flash flash monitor
```

---

## 📜 Credits & Acknowledgements
- **[Xiaozhi ESP32 Community](https://github.com/708xiaozhi/xiaozhi-esp32):** For the core open-source IoT voice assistant framework and cloud protocol design.
- **[LVGL Community](https://lvgl.io):** For the light and versatile embedded graphics engine.
- **[Espressif Systems](https://github.com/espressif):** For the ESP-IDF framework, FreeRTOS SMP port, and ESP-SR speech recognition algorithms.

---

## 👨‍💻 Author
**Saad** — 2nd Year Computer Engineering Student  
*Aspiring Firmware & Embedded Systems Engineer*  
GitHub: [@saadtambe35-bits](https://github.com/saadtambe35-bits)
