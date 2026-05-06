<p align="center">
  <img src="docs/assets/sponsor_logos_header.png" alt="Perovskia and BM Labs" width="520">
</p>

<p align="center">
  <a href="https://perovskia.solar/"><strong>Perovskia</strong></a> — customizable printed perovskite solar cells for indoor and outdoor energy harvesting.<br>
  <a href="https://bmsemi.io/"><strong>BM Labs</strong></a> — neuromorphic and in-memory-compute semiconductor IP, including the Neuromorphic X1 macro used in this design.
</p>

# Secure Logger Controller – ReRAM-Based Medical Event Logger

**ChipFoundry Application Design Contest Submission**  
Designed for the [ChipFoundry Application Design Contest](https://chipfoundry.io/challenges/application), this project is a complete reference-design concept for a secure, low-power medical event logger on the **Caravel** platform. It combines BM Labs’ **Neuromorphic ReRAM NVM** with a custom secure logging controller to preserve critical medical events under power loss, intermittent connectivity, and safety-critical operating conditions.

**Designer:** Juan Carlos Aquino Hernández  
**Institution:** Universidad Tecnológica de Nayarit (UTNAY)

---

## Project Overview

This repository contains the RTL, integration scripts, and documentation for a **Secure Logger Controller** that augments the open-source [Caravel](https://github.com/efabless/caravel) System-on-Chip with non-volatile medical-event logging.

The design is aimed at applications where medical data must remain trustworthy even when:

- power is intermittent,
- connectivity is unavailable,
- device handling must be auditable, or
- radiation exposure can disturb digital state.

Key platform blocks include:

Key platform blocks include:
Secure Logger Controller: validates, encrypts, and writes medical-event payloads.
Neuromorphic ReRAM NVM (Neuromorphic_X1): retains event history through power cycles.
Caravel Chip Zone: provides the Wishbone bus, GPIO, and IRQ infrastructure for control and observability.
Selective TMR hardening: protects the critical logging path for radiation-tolerant medical workflows.
---
**Neuromorphic X1 status for this tape-out**

The original concept included the BM Labs Neuromorphic X1 ReRAM macro as both non‑volatile storage and an in‑memory compute block. However, for this tape‑out the neuromorphic macro is **not included in the hardened `user_project_wrapper`** due to power–domain incompatibilities between the macro (`VSS`, `VDDA`, `VDDC1`, `VDDC2`, `VDDA2`) and the Caravel/ChipFoundry wrapper rails (`vccd1`, `vccd2`, `vdda1`, `vdda2`, `vssd1`).  

In the current iteration, the silicon implementation focuses on the **secure logging controller, AES‑based encryption pipeline, and TMR‑protected ReRAM logging path** within the standard Caravel power rails. Neuromorphic X1 remains part of the **system‑level vision and simulations**, and will be reconsidered in a future revision using a dedicated power‑integration strategy.

## Key Innovation Points

- **Fail-safe persistent logging:** sensor or therapy events are validated, encrypted, and stored in non-volatile memory so records survive resets and power interruptions.
- **Error detection via CRC-8:** every event is checked with a CRC-8 polynomial (0x07) before commit.
- Secure writes: payloads are packed into a 128‑bit AES block and encrypted with a dedicated AES‑128 core (`aes_sbox`, `aes_key_expansion`, `aes_round`, `aes128_core`, `encryptor`) before storage, replacing the earlier XOR placeholder.
- **Selective Triple Modular Redundancy (TMR):** critical logging-path state can be triplicated with majority voting to mask radiation-induced upsets without changing the external product behavior.
- **Complete application fit:** the architecture supports silicon, packaging, NFC-based user interaction, and indoor-light energy harvesting in a practical medical product concept.

---

## Technical Highlights

| Metric | Target / Behavior | Description |
|---|---|---|
| **Event data width** | 8-bit sensor / status events | Low-bandwidth medical samples, flags, handling events, or dose-state indicators. |
| **Integrity check** | CRC-8 (poly 0x07) | On-chip CRC computed before committing an event. |
| **Encryption block** |Encryption block  128-bit AES-128 block (implemented in RTL)
| **Power-failure handling** | Fails closed (fail flag asserted) | Any power-fail condition blocks writes and sets an error flag. |
| **Caravel integration** | Wishbone + GPIO + IRQ | Logger controlled via Wishbone and observable via GPIO / IRQ. |
| **TMR hardening** | Triplicated critical state + voter | Masks a single upset in the protected logging path. |
| **Energy strategy** | Indoor-light harvesting ready | Suitable for short NFC bursts and low-duty-cycle medical logging. |

---
Physical implementation notes

The design is implemented as a Caravel `user_project_wrapper` with the secure logging logic and AES pipeline synthesized and placed‑and‑routed using OpenLane on SKY130. The OpenLane configuration defines `wb_clk_i` as the primary clock with a 25 ns target period (~40 MHz) and uses Caravel’s standard power rails: `vccd1`, `vccd2`, `vdda1`, `vdda2` for VDD and `vssd1`, `vssa1`, `vssd2`, `vssa2` for ground. [file:164]  

The power delivery network is generated on `met4`/`met5` with rails enabled and a conservative placement density (around 18%) to preserve routability and IR‑drop margin around the secure logger and ReRAM‑interface blocks. [file:164]
## Architecture
![Arquitectura](https://github.com/user-attachments/assets/5bf1885d-a2a1-470f-8a1d-809667c5cd96)



System-level block diagram showing integration of:

- **Caravel Management SoC (RISC-V core)** providing Wishbone control, debug and system management.  
- **Secure Logger Controller (CRC-8 + AES)** validating sensor events, encrypting payloads, and generating status/IRQ.  
- **Neuromorphic ReRAM NVM (BM Labs Neuromorphic_X1_wb)** used as non-volatile storage and analog in-memory compute macro, connected via 32-bit Wishbone and analog bias pins.

All IPs communicate through the standard **32-bit Wishbone Bus**, with GPIO and Logic Analyzer lines used to inject sensor events, CRC references, power-fail test signals, and to observe encrypted outputs and status flags.
## Getting Started

### 1. Clone the Repository

```bash
git clone https://github.com/BMsemi/Secure-Edge-IoT-Event-Logger-on-Caravel.git
cd Secure-Edge-IoT-Event-Logger-on-Caravel
```

### 2. Prepare Your Environment

Install the required Caravel harness, management core, OpenLane, and SKY130 PDK support:

```bash
make setup
```

### 3. Install ChipFoundry IPM and Neuromorphic X1 IP

```bash
pip install cf-ipm
ipm install Neuromorphic_X1_32x32
```

After installation, replace the behavioral model in the IP directory:

```bash
cd ip/Neuromorphic_X1_32x32
mv hdl hdl_original
mv hdl_replace_inside_ip hdl
# rename supporting folders
mv gdss gds
mv leff lef
mv libb lib
```

### 4. Run Testbenches

Use Cocotb to verify correct operation of the logger controller and ReRAM interface:

```bash
make cocotb-verify-ram_word-rtl
```

### 5. Harden the Design

Generate the physical implementation of the user project wrapper using OpenLane:

```bash
make user_project_wrapper
```

---
Verification status and coverage

Verification focuses on both block‑level correctness and integration maturity:

| Block / level                   | Strategy                                   | Current status |
|---------------------------------|--------------------------------------------|----------------|
| AES‑128 core + encryptor        | RTL testbench with known‑answer vectors    | PASS           |
| CRC‑8 + parity checker          | RTL unit tests over representative payloads| PASS           |
| ReRAM NVM + TMR wrapper         | Behavioral model with fault‑injection sims | PARTIAL        |
| Secure logger controller (FSM)  | System‑level testbench, event scenarios    | PARTIAL        |
| Caravel user_project_wrapper    | Smoke tests for Wishbone, GPIO, IRQ        | PARTIAL        |

The goal for the next iteration is to expand integration‑level testbenches (e.g., full event capture → AES encrypt → ReRAM commit → readback) and report aggregate pass/fail counts and coverage metrics in the verification section and CI logs.
## Application Scenarios

### Secure, Fail-Safe Medical Logging

This IP targets medical and clinical products where local truth matters:

- **Medication adherence devices** that must record the last dose taken and adherence history.
- **Medical wearables** that must preserve alarms, threshold crossings, and symptom markers.
- **Smart therapeutic packaging** that must provide a tamper-evident handling log.
- **Radiation-tolerant clinical accessories** that must keep trustworthy records near radioactive therapies.

It is also suitable for industrial safety black-box logging and other ultra-low-power edge-monitoring applications.

### Secure Logger IP Description

**Secure Logger IP** is an ultra-low-power hardware event-logging core. It guarantees that critical events are never silently lost, preserving data across power failures, resets, and communication gaps. Events are captured locally, validated, encrypted, and committed to ReRAM. NFC provides a simple authenticated readout path for phones, readers, or clinical tools.

---

## Customer Story: Secure Adherence Cap

The Secure Logger platform can power smart packaging. The **Secure Adherence Cap** looks like a normal cap but behaves like a trusted dose diary. After each use, the cap or dock stores a local event record and later returns a simple phone-tap view: last use, adherence history, and freshness / storage status.

![Secure Adherence Cap](docs/assets/secure_adherence_cap.png)

**Key value:**

- **Non-volatile record** survives power interruptions.
- **Validated + encrypted history** is available through NFC.
- **Indoor-light energy harvesting** supports maintenance-friendly operation.

---

## Secure Adherence Cap – Nuclear Medicine Edition

**Secure Adherence Cap – Nuclear Medicine Edition** is a radiation-tolerant smart cap for radioactive therapies such as **I-131 treatment**. It keeps the original secure-logger architecture intact — local event capture, encrypted non-volatile history, NFC readout, and indoor-light energy harvesting — but adds **TMR** to harden the critical logging path against radiation-induced upsets.

The result is a trusted dose diary and chain-of-custody logger for **earth-based radiation-oncology and nuclear-medicine workflows**.

### What TMR does here

**Triple Modular Redundancy (TMR)** means the most critical logging logic is implemented three times, and a **majority voter** decides the final result. If radiation flips one copy of the state or control path, the other two copies still agree, so the system can continue logging correctly.

In this design, TMR is intended for the **critical logging path**, such as:

- write-enable / commit decisions,
- event-valid and status-state tracking,
- key control state in the secure logger FSM,
- error and fail-safe signaling that must remain trustworthy.

This keeps the architecture practical: the product remains compact and low power, while the most safety-relevant logic gets extra protection.

### Why TMR matters for medical use

TMR helps this product serve multiple medical needs with a single secure-logger platform:

| Medical need | Example product / workflow | Value of Secure Logger + TMR |
|---|---|---|
| **Radiation-tolerant dose logging** | I-131 therapy cap / container | Preserves dose and handling history even when radiation can upset digital state. |
| **Medication adherence** | Daily oral therapy smart cap | Maintains a trusted last-dose and adherence diary without depending on the cloud. |
| **Clinical chain of custody** | Pharmacy-to-patient or ward-to-home handoff | Records openings, handoffs, and events in encrypted non-volatile memory. |
| **Storage / freshness verification** | Light- or time-sensitive therapies | Logs status checks and makes them available over NFC at the point of care. |
| **Wearable medical event logging** | Cardiac, glucose, or respiratory edge devices | Keeps local alerts and event markers persistent across low-power interruptions. |

---

## Energy Strategy: Perovskite Solar Integration

Intermittent operation is enabled through printed **perovskite solar cells** that conform to caps or wearable patches. Energy is harvested under indoor lighting, stored in a thin-film cell or supercapacitor, and used for short NFC bursts. Stable rails and burst buffering help keep the Caravel ASIC and ReRAM macro operational in realistic medical usage.

![Perovskite Solar Integration](docs/assets/perovskia_energy_architecture.png)

### Benefits

- Continuous low-power energy harvesting from indoor light.
- No always-on wireless required — energy is saved for short NFC bursts.
- Maintenance-friendly operation optimized for intermittent usage.

---

## Use Case Summary

| Feature | Value Delivered |
|---|---|
| Non-volatile logging | No silent data loss |
| Secure encryption | Trusted, auditable records |
| NFC interface | Simple clinician / patient interaction |
| Energy harvesting ready | Reduced battery dependence |
| Selective TMR hardening | Better tolerance to radiation-induced upsets |
| Event-driven design | Ultra-low-power operation |

---

## Why This Design Fits the Application Contest

- **Complete system story:** this is not only a chip block; it maps naturally to a deployable smart medical product with silicon, packaging, NFC interaction, and energy harvesting.
- **Real medical relevance:** the same secure-logger core can address medication adherence, nuclear medicine, wearable event logging, and clinical chain-of-custody needs.
- **Practical reliability strategy:** CRC, encryption, fail-safe behavior, non-volatile memory, and selective TMR together make the design more compelling for safety-sensitive workflows.
- **Open-source reproducibility:** the architecture is compatible with Caravel-based prototyping and a documented open-source integration flow.

---
Hardware and BOM concept

While this README focuses on the Caravel‑based ASIC integration, the intended deployment includes a concrete hardware stack:

- A **Sky130 ASIC** implementing the secure logger controller, AES‑128 pipeline, CRC‑8, parity checking, FIFO buffering, and TMR‑protected ReRAM interface inside the Caravel `user_project_wrapper`.  
- A **non‑volatile ReRAM device** (standalone NVM compatible with the `re_ram_nvm` behavioral model and `tmr_re_ram_nvm_wrapper` integration) providing persistent storage for encrypted medical events.  
- A **host MCU or SoC** (e.g., low‑power RISC‑V or Cortex‑M0+) acting as a Wishbone master or bridged controller to configure the logger, trigger captures, and read back logs in clinic or field settings.  
- **Perovskia perovskite solar cells** for indoor‑light energy harvesting and a small energy buffer (supercapacitor or thin‑film battery) to sustain intermittent Caravel and ReRAM operation during NFC readout and event bursts.  

This combination provides a realistic bill of materials for commercial and edge‑IoT medical products, aligning the ASIC with deployable devices rather than a purely conceptual platform.
Deployment scope and adjacent applications

Beyond the core medical adherence use case, the secure logger architecture naturally extends to:

- **Cold-chain logging**: encrypted, non‑volatile logging of temperature and handling events for pharmaceutical transport and storage, with NFC readout at any handoff point.  
- **Trial compliance logging**: trusted, local records of medication intake, device activations, and visit‑related events in clinical studies, even when connectivity is intermittent.  
- **Assisted‑care medication monitoring**: smart caps and dispensers in home‑care or assisted‑living settings that locally retain adherence and handling history, and expose it securely to clinicians via NFC.  

These adjacent paths are consistent with the secure, low‑power, edge‑IoT profile and the radiation‑tolerant variant for nuclear‑medicine workflows.

README update based on ChipFoundry review

This README has been updated in response to the ChipFoundry Application Design Contest feedback to:

- Replace the original XOR placeholder description with the current **AES‑128 encryption implementation** and document the AES RTL modules used in the design.  
- Add an explicit **hardware/BOM concept** describing how the Caravel ASIC, ReRAM, MCU and energy‑harvesting elements combine into a deployable product.  
- Provide a concise **verification status table** with block‑level and integration‑level pass/partial status.  
- Clarify the **neuromorphic IP status** for this tape‑out, noting the power‑domain incompatibility between the existing neuromorphic macro and the Caravel/ChipFoundry wrapper rails.
## Documentation

- Neuromorphic ReRAM IP: [Neuromorphic X1 documentation](https://github.com/BMsemi/Neuromorphic_X1_32x32)
- Caravel User Project and Wrapper: [Caravel user project docs](https://caravel-user-project.readthedocs.io)
- Contest details: [ChipFoundry Application Design Contest](https://chipfoundry.io/challenges/application)

---

## License

This project is licensed under the **Apache 2.0** License — see the `LICENSE` file for full terms.

