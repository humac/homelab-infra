# 🚀 Hardware Optimization Runbook: Huy’s AI Garage
**Target Hardware:** Dell PowerEdge (Dual Xeon) | **GPUs:** Tesla P40 & Quadro P2000
**Environment:** Proxmox 9.x | **Last Updated:** January 2026

---

## Phase 1: BIOS & UEFI Configurations
*These settings ensure your server can properly address 256GB of RAM and 29GB of VRAM simultaneously.*

1.  **Reboot to BIOS (F2 on Dell).**
2.  **Integrated Devices:**
    * **Memory Mapped I/O above 4GB:** Set to **Enabled**.
    * **Slot Disablement:** Ensure the slots holding your GPUs are set to **Enabled**.
3.  **System Profile Settings:**
    * **System Profile:** Set to **Performance**. (This prevents the CPUs from downclocking during inference).
4.  **PCIe Settings:**
    * **MMIO Base:** Set to **56TB** (if available) or **12TB**. *Avoid the 2GB default.*
5.  **Boot Settings:**
    * **Boot Mode:** Ensure **UEFI** is selected and **CSM (Compatibility Support Module)** is **Disabled**.

---

## Phase 2: Host-Level NVIDIA Optimizations
*Run these commands on the **Proxmox Host Shell**. These optimize the way the driver talks to the Pascal-architecture Tesla P40.*

### 1. Enable Persistence Mode
This prevents the driver from unloading when the AI is idle, eliminating the 2-3 second "wake-up" lag.
```bash
nvidia-smi -pm 1
```

### 2. Disable ECC (Error Correction)
ECC is vital for database servers but eats ~10% of memory bandwidth. For LLMs, disabling it increases speed.
```bash
# Target GPU 1 (the Tesla P40)
nvidia-smi -i 1 -e 0
```
> **Note:** This requires a host reboot to take effect.

### 3. Lock Application Clocks
Tesla cards aggressively throttle to save power. Locking them to their maximum rated speed ensures consistent token generation.
```bash
# Set P40 (GPU 1) to Max Memory (3505) and Max Graphics (1531)
nvidia-smi -i 1 -ac 3505,1531
```

---

## Phase 4: Automation (The "Startup Script")
*To ensure these settings stick after a reboot, create a systemd service on the **Proxmox Host**.*

1.  **Create the script:** `nano /usr/local/bin/gpu-optimize.sh`
2.  **Paste the following:**
    ```bash
    #!/bin/bash
    # Enable Persistence
    nvidia-smi -pm 1
    # Lock P40 Clocks
    nvidia-smi -i 1 -ac 3505,1531
    # Force UVM initialization for LXC
    nvidia-modprobe -u -c0
    ```
3.  **Make it executable:** `chmod +x /usr/local/bin/gpu-optimize.sh`
4.  **Create a Crontab entry:**
    Run `crontab -e` and add:
    ```text
    @reboot /usr/local/bin/gpu-optimize.sh
    ```

---

## Phase 5: Software-Level "Quick Wins"
* **Quantization:** Always download **GGUF** models in **Q4_K_M** or **Q5_K_M** format.
* **Context Management:** In **Open WebUI**, limit `num_ctx` to **8192** for daily coding to keep the KV Cache entirely in VRAM. Increase to **32k** only when you need to analyze large files.
