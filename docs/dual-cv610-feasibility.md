# Dual-SoC Thinker redesign: feasibility and architecture brief

*Prepared 2026-10-08 for the `Thinker338Q` repo. Written for a reader who is new to PCB design.*

**Your idea:** rebuild the OpenIPC Thinker Base around **two HiSilicon Hi3516CV610 ("CV610") chips** joined by an internal Ethernet link. SoC #1 sends live video over RF (WFB-ng through a USB Wi-Fi adapter). SoC #2 records high-bitrate video to an SD card. A USB-C port to a PC replaces the old Ethernet connector.

**Labels used in this brief**

| Label | Meaning |
|---|---|
| **[V] Verified** | Checked directly in the repo files, in source code, or in an official document. |
| **[L] Likely** | From a credible second-hand source: a search-engine snippet of a datasheet, a distributor listing, or a blog. |
| **[I] Inferred** | My own reasoning, arithmetic or estimate. Treat it as a starting point, not a fact. |
| **[U] Uncertain / unknown** | Sources disagree, or nothing public was found. |

Numbers in brackets such as (S12) point to the source list in section 10.

**Glossary (one line each)**

| Term | Meaning |
|---|---|
| SoC | "System on chip": the main processor. Here it also contains the camera pipeline, video encoder and memory. |
| ISP / encoder | Image signal processor (turns raw sensor data into a picture) / the hardware that compresses it to H.264/H.265. |
| MIPI CSI-2 | The high-speed serial camera interface between the sensor and the SoC. |
| SDIO | The bus used to talk to an SD card. |
| NOR flash | The small SPI memory chip the SoC boots its firmware from. |
| MAC / PHY | The Ethernet logic inside the SoC / the analog circuit that drives the cable. Both are built into these SoCs. |
| LAN transformer ("magnetics") | A small transformer normally placed between a PHY and an Ethernet cable. |
| ESD / TVS | Protection parts that absorb static discharges / voltage spikes. |
| Buck / LDO | Switching voltage regulator (efficient) / linear regulator (simple, quiet, wastes heat). |
| USB gadget / device mode | The SoC acts as a USB device (like a phone plugged into a PC) instead of a USB host. |
| NAT / routing | Ways to pass network traffic from one interface to another. |
| OSD | On-screen display (flight data drawn over the video). |
| WFB-ng | The open "WiFi broadcast" FPV video link OpenIPC uses. |
| Majestic | OpenIPC's video streamer program. It also does SD recording. |
| DVR | Recording the received video at the ground station. |
| BOM / PCBA | Bill of materials / PCB assembly (the fab solders the parts on). |
| QFN / BGA | Chip packages with pads on the edges / a grid of balls underneath. |

---

## 1. Bottom line

- **It can be built, but it is not a modification.** The idea is workable in principle. It is not a change to the Thinker, though. It is a new and much harder board, and most of what you want can be done with one SoC.
- **One SoC can probably already record and stream at the same time.**
  - A single CV610 encodes a high-bitrate main stream and a low-bitrate sub-stream together [L] (S19).
  - On the SSC338Q already on your Thinker Base, OpenIPC's Majestic streamer has the settings to record one channel to SD while sending another channel out [V] (S15).
  - Whether that works with the FPV mode switched on, at your frame rate and bitrate, is untested [U]. The Stage 0 bench test answers it.
- **USB-C to a PC needs no new board today.** Plug the Thinker's included Ethernet cable into any off-the-shelf USB-C Ethernet dongle. OpenIPC's own setup guide lists a USB-Ethernet dongle as a valid PC connection [V] (S3, S45).
- **The biggest problem is software.** OpenIPC's CV610 firmware currently has:
  - no Wi-Fi stack and no FPV/WFB-ng build [V] (S5, S7);
  - no USB device mode [V] (S5);
  - no driver for the camera sensors the Thinker uses [V] (S8).
- **The hardware problems come right after:**
  - One MIPI camera cannot be wired to two SoCs [L] (S30).
  - The CV610 has no video output to pass video along [V] (S13).
  - Two SoCs will not fit on 25×25 mm [I].
  - HiSilicon's hardware design guide is not public [U] (S22).
- **What I recommend:**
  1. First, run the Stage 0 bench test on the Thinker Base you already own, and use the Stage 0b dongle for USB-C. Both cost almost nothing.
  2. Decide **what USB-C must do** and **what you want to record**: resolution, frame rate and Mbps (section 9, questions 1–2).
  3. Compare against **buying** a small action camera instead (option 0, section 5).
  4. If you still want two SoCs:
     - **one camera:** a new CV610 recorder board that sends its live sub-stream to an unmodified Thinker radio (**option B3**);
     - **two cameras:** an unmodified Thinker plus a new CV610 recorder with its own camera (**option D**), growing later into a two-board stack (**option C**).
  5. Learn on three or four simpler boards first.

> **Do this next (no PCB needed)**
> 1. Measure the Wi-Fi header's 5 V pin with a multimeter, then run the Stage 0 test: record one stream to SD while streaming the other.
> 2. Get USB-C to your PC today: the Thinker's J2-to-RJ45 cable plus a USB-C Ethernet dongle (Stage 0b).
> 3. Write down your recording target (resolution, fps, Mbps) and what USB-C must do.
> 4. Buy a 3.3 V USB-UART adapter, a multimeter and a current-limited bench supply. You will use them at every later stage.

---

## 2. What "CV610" is, and how it compares with the SSC338Q

I assume "CV610" means the **HiSilicon Hi3516CV610**. OpenIPC treats it and its sibling, the CV608, as one family called `hi3516cv6xx` [V] (S6).

### 2.1 The variants matter: the suffix changes memory, package and PCB

The datasheet, as quoted in OpenIPC's ipctool docs, says: *"Hi3516CV610-10B/20S/20G uses the QFN package, while Hi3516CV610-00S/00G uses the TFBGA package"* [V] (S14). The -20B and -00B appear only in the product brief, secondary summaries and distributor listings.

| Variant | Package | Memory | Max video | Comment |
|---|---|---|---|---|
| **-20S** | QFN 9×9 mm [V] (S14), 0.35 mm pitch [L] (S21, S23) | 128 MB (1 Gb) DDR3/3L in the package [L] (S21) | 4K@20 or 6 MP@30 [L] (S21) | **Best fit.** No external DDR routing. It is in the datasheet's QFN list, it is the variant OpenIPC has run (it reports itself as "CV613") [V] (S14), and OpenIPC publishes a boot image for it [V] (S52) |
| -20B | QFN-88, 9×9 mm, 0.35 mm pitch [L] (S23) | 128 MB DDR3/3L in the package [L] (S21) | 4K@20 or 6 MP@30 [L] (S21) | Brief and distributor listings only, **not in the datasheet's package list** [U] (S14). No OpenIPC boot image is named for it [V] (S52). Use it only if OpenIPC confirms support |
| -10B | QFN 9×9 mm [V] (S14) | 64 MB DDR2 in the package [L] (S21) | 5 MP@30 (2880×1620) [L] (S21) | Too little RAM for two streams plus WFB-ng [I] |
| -20G | QFN 9×9 mm [V] (S14) | not found [U] | not found [U] | Adds features FPV does not need [V] (S8) |
| -00S / -00G | TFBGA 12×13.3 mm, 0.65 mm pitch [V] (S14) | **External** DDR3/3L, up to 4 Gb [L] (S21) | 4K@20 [L] | **Avoid**: brings back DDR routing |
| -00B | TFBGA [L] (S21) | External DDR [L] (S21) | 4K@20 [L] | Brief/distributor only, not in the datasheet list [U]. **Avoid** |
| CV608 (sibling) | QFN 9×9 mm [V] (S14) | 64 MB DDR2 [L] (S21) | about 103 Mpix/s, 40 Mbps [V] (S13) | Too small |

### 2.2 CV610-20S compared with the Thinker's SSC338Q

| | **Hi3516CV610-20S** | **SigmaStar SSC338Q (current Thinker)** |
|---|---|---|
| CPU | 2× Cortex-A7. Clock reported as about 950 MHz by a blog and 1.2 GHz by the device tree [U] (S21, S10) | 2× Cortex-A7, 1.0–1.2 GHz [L] (S25) |
| NPU | 1 TOPS [L] (S21) | 0.5 TOPS [L] (S25) |
| RAM | 128 MB, in the package [L] (S21) | In the package, DDR3 at 1.5 V [V] (S1). About 256 MB [L] (S25, S18) |
| Package | QFN-88, 9×9 mm, **0.35 mm pitch**, about 6.4 mm exposed pad [L] (S23) | QFN-128, 12.3×12.3 mm [V] (S1) |
| Encoder | H.264/H.265, max 3840×2160 [L] (S19). The SDK's encoder limit is 4096×4096 [V] (S13). Above 3200 pixels wide the capture pipe cannot run "online" (`OT_VI_PIPE_ONLINE_MAX_WIDTH = 3200`) [V] (S13), so 4K raw frames must go through the 128 MB DDR first. That adds latency and memory pressure [I] | 4K@20 official spec [L] (S25) |
| Multi-stream (CV610 brief) | 4K@20 + 720p@20, or 3200×1800@30 + 720p@30 [L] (S19) | — |
| Bitrate limits | Brief: 80 Mbps max output [L] (S19). SDK per-channel ceiling: `OT_VENC_MAX_BITRATE` = 160×1024 kbit/s, about 164 Mbit/s [V] (S13) | Unknown [U]. Measure it in Stage 0 with Majestic's `venc0_bitrate_kbps` / `venc1_bitrate_kbps` metrics (S16) |
| **FPV track record** | **None** | **Proven in OpenIPC FPV**: 1080p@120 with IMX335, 1080p@90 with IMX415, 720p@120 [V] (S4) |
| Camera input | One MIPI receiver with 4 lanes total: 1×4-lane or 2×2-lane, so up to 2 sensors [V] (S13) | 4-lane MIPI on connector J1 [V] (S1) |
| Video **output** | **None**: no MIPI-TX, HDMI or BT.656/1120 out [V] (S13, S10) | Not used on the Thinker |
| Ethernet | One 10/100 MAC with an internal PHY. The device tree uses `internal-phy`, `phy-mode = "mii"` [V] (S10). The pin-mux table has no RMII/MII/MDIO function at all [V] (S14) | One 10/100 internal PHY. The pairs go straight to J2 with no magnetics or other parts [V] (S1), and OpenIPC's J2-to-RJ45 cable "can be connected directly to a computer" [V] (S3) |
| USB | **One** USB 2.0 port, dual-role (host or device) [V] (S11) | One USB 2.0, used as host for the Wi-Fi adapter [V] (S1, S3). OpenIPC's SigmaStar kernel builds it host-only (no gadget, no OTG) [V] (S50). Whether the hardware can act as a USB device is undocumented [U] |
| UARTs | Three (UART0–UART2) [V] (S10, S14) | UART0 console on J2, flight-controller UART on J3 [V] (S1) |
| SD | 2× SDIO 2.0 controllers, 4-bit, 50 MHz High-Speed. The device tree declares only `cap-sd-highspeed`, with no `sd-uhs-*` modes [V] (S10). Both SDIO ports share pins with other functions (eMMC, UART1/UART2, SPI, I2S, PWM, JTAG, sensor reset) [V] (S13, S14) | Thinker SD I/O fixed at 3.3 V, so High-Speed only [V] (S1) |
| Real-time clock | Not researched [U] | No 32 kHz crystal (RTC crystal pins tied off), AVDD_RTC fed from 3V3, no backup cell [V] (S1) |
| Core / IO / DDR voltage | Core about 0.9 V (some sources say 1.0 V) [U]. IO 1.8/3.3 V, DDR 1.5/1.35 V [L] (S19, S20). Core trimmed by an "SVB-PWM" signal [L] (S10) | Core about 0.9 V (annotated 0.825/0.9/1.05 V), DDR 1.5 V, 3.3 V, 1.8 V [V] (S1) |
| Typical power | 1043 mW per a brief snippet [L] (S19). HiSilicon's page says "TBD" [L] (S20) | Not documented [U] |
| Temperature | −20 to 70 °C [L] (S19) | Not checked |
| OpenIPC support | IP-camera builds only [V] (S5, S6, S7, S8, S9). See the list below | Official FPV target, including this Thinker [V] (S7, S3) |
| Documentation | Chinese product brief is public. An unofficial 768-page English datasheet is on Scribd. The hardware design guide and reference schematics appear to be SDK/NDA only [U] (S22) | This repo's schematic and Gerbers [V] (S1, S2) |
| Sourcing | JLCPCB/LCSC list 20S and 20B. Stock and price are unverified, and one listing showed 0 [U] (S23) | — |

What OpenIPC's CV610 builds lack today [V] (S5, S6, S8, S9):
- `CONFIG_WIRELESS` is off.
- There is no WFB-ng/FPV build.
- `CONFIG_USB_GADGET` is off and USB is built host-only.
- `CONFIG_BRIDGE`, connection tracking (`NF_CONNTRACK`) and iptables are off, so neither bridging nor NAT is possible without a kernel change.
- Only the "lite" version of Majestic is published.
- There is no IMX335/IMX415 driver.
- An open bug has a sensor that never streams.

**Takeaway:** the CV610 is newer and has a stronger NPU. But it has **half the RAM**, a **much finer-pitch package**, **no video output** and **no FPV software**. For FPV it is not an upgrade over the SSC338Q today [I].

---

## 3. Does this need two SoCs?

**Short answer:**
- To fly with a 720p/1080p live feed and record a higher-bitrate copy to SD: **probably no**. One SoC should do that. Stage 0 confirms it.
- To get high-frame-rate FPV and 4K/6 MP recording at the same time, from two cameras: **yes**.

### 3.1 Evidence that one SoC can stream and record together

1. **Hardware.** The CV610 brief's own multi-stream modes are "4K@20 + 720p@20" and "3200×1800@30 + 720p@30". That is a high-resolution main stream plus a 720p sub-stream from one chip [L] (S19).
2. **Software.** Majestic, OpenIPC's streamer, already has the settings for this split [V] (S15):
   - The `records:` section writes `video0` to SD, or `video1` if you set `records.substream`.
   - Each `outgoing` destination picks `channel: main|sub`.

   Whether this works on the Thinker with `fpv.enabled` at your frame rate is [U].
3. **The use case is already proposed.** OpenIPC's FPV FAQ suggests a board with USB and SD that would "broadcast with minimal latency in 720p, while simultaneously recording the original in 1080p" [V] (S17). That is a suggestion in the FAQ, not a tested result.
4. **SD bandwidth is not the bottleneck.** 80 Mbit/s is 10 MB/s. The SD High-Speed bus tops out at about 25 MB/s (50 MHz × 4 bits) [I, from S10 and S19]. Sustained write speed depends on the card, so use a card rated for continuous video [I].
5. **Your Thinker Base already has the hardware:** an SD slot plus the USB Wi-Fi port [V] (S3).

**Caveat for the SSC338Q.** On SigmaStar chips, Majestic's packet-loss and FPV encoder features (reference structure, intra refresh, noise level, ROI) apply to `video0` only, and they need `fpv.enabled: true` [V] (S16, S15). `fpv.enabled` also switches off automatic exposure and white balance at start-up [V] (S16). So you must choose:
- keep the live link on `video0` and record `video1`; or
- record `video0` and send `video1` to the radio without the FPV tuning.

Which trade-off is acceptable needs a bench test (Stage 0 in section 8) [U].

### 3.2 When a second SoC genuinely helps

| Your goal | Two SoCs needed? | Why |
|---|---|---|
| 720p/1080p FPV plus a high-bitrate 1080p recording of the same view | **No** | Main + sub stream on one chip (S19, S15) |
| USB-C to a PC while the Wi-Fi adapter is plugged in | **No** | Today: the stock J2 cable plus a USB-C Ethernet dongle [V] (S3, S45). Later: an on-board USB-Ethernet bridge chip (section 5, option A) [I]. A USB mux would need device mode, which OpenIPC's SigmaStar kernel does not enable [U] (S50) |
| 1080p@60+ FPV **and** 4K recording at the same time | **Yes, and two cameras** | One CV610 has an encoder budget of about 185–200 Mpix/s [I, arithmetic below] |
| Keep a stalled SD write from ever affecting the RF link | **Helps** | Separate CPUs [I]. Test whether stalls actually hurt the link in Stage 0 first |

**Arithmetic [I], based on S19:**

| Mode | Throughput |
|---|---|
| 4K@20 | 3840×2160×20 = 166 Mpix/s |
| 3200×1800@30 + 720p@30 | ≈ 200 Mpix/s |
| 1080p@60 | 124 Mpix/s |
| 4K@20 + 1080p@60 | ≈ 290 Mpix/s, more than one CV610 can do |
| SSC338Q's demonstrated 1080p@120 | ≈ 249 Mpix/s (S4) |

**Important:** a second SoC only adds encoding power if it has **its own camera** (section 4).

---

## 4. The camera problem: getting video to SoC #2

**Verified constraints:**
- **MIPI CSI-2 is point-to-point.** It runs from one transmitter to one receiver. The receiver switches its line termination between high-speed and low-power modes, so you cannot passively wire one camera to two SoCs [L] (S30).
- **The CV610 has one MIPI receiver** [V] (S13). On the QFN package its parallel (BT.1120) input shares pins with that MIPI receiver [V] (S13).
- **The CV610 has no video output of any kind** [V] (S13, S10). SoC #1 cannot "pass the camera through" to SoC #2.

**The options:**

| Option | How it works | Verdict |
|---|---|---|
| **(a) Two cameras** | Each SoC owns its own sensor: I²C, clock, reset, ISP and encoder. | **Simplest hardware**, and each camera runs its own mode. Downsides: two slightly different viewpoints and independent exposure/colour. It also adds a sensor, an FPC cable, a mount, weight and sensor-driver work. Frame sync through the sensors' slave-mode pins exists in the pinmux but is unproven [U]. |
| **(b) The camera SoC encodes both streams and sends one over Ethernet** | The link is 100 Mbit/s per direction, about 90 Mbit/s usable over TCP [I, measure it]. | If the **recording** crosses the link, it is capped and SoC #2 is just a file server. If the **live stream** crosses instead, it works, but one encoder still does everything [I]. |
| **(c) Active MIPI splitter** | Lattice CrossLink LIF-MD6000 with Lattice's "1 to N duplicator" reference design: a 36-ball WLCSP of about 2.5×2.6 mm, roughly $8.6–8.7 on a reel, about 20-week lead time [L] (S31). CrossLink-NX for up to 4 outputs [L] (S31). DS90UB954 / MAX9296A are automotive SerDes parts: the DS90UB954 replicates only 1–2 lanes and needs a serializer at the camera [L] (S32). | **Not for a beginner.** Needs an FPGA toolchain, a bitstream and extra supply rails. Only one SoC can control the sensor, and both SoCs get the same resolution and frame rate [I]. |
| **(d) Two cameras into one CV610** | The MIPI receiver can be split 2×2-lane [V] (S13). | Possible, but both cameras share one encoder budget, so it does not solve the 4K + high-fps problem [I]. |

**Sensor support is a second camera problem.** Whatever camera the CV610 uses must be a sensor OpenIPC's `cv6xx` package supports, or you port a driver yourself [V] (S8). Supported today: sc4336p, gc4023, os04d10, sc450ai, sc500ai, sc431hai, imx307, os02m10. The Thinker's usual modules are IMX335/IMX415, and those are **not** on the list [V] (S8, S4). Even a supported sensor has an open "never streams" bug on one CV610 board [V] (S9).

---

## 5. Architecture options

The complexity scores are my estimates on a 1–10 scale [I]. As a rough anchor for a beginner working evenings: **1** = an evening, **4** = a few weeks, **7** = several months, **9** = a year or more, likely with outside software help.

### Option 0 — Buy, don't build (baseline)

Keep the stock Thinker for FPV and record with something you buy.

- **Ground-station DVR** already exists. For example, the Radxa ground-station image records the received stream to a `.ts` file [V] (S49). That copy is only as good as the live link, and lost frames show up as jumps [V] (S49).
- **A small action or FPV recording camera** mounted next to the Thinker records on its own. Many models offer higher frame rates at 4K than the CV610 and add stabilisation or gyro data [I, check a specific model]. Expect tens of grams and a separate battery or power lead [I].

**What a custom board must beat.** The CV610 tops out at 4K@20 or 3200×1800@30 and 80 Mbps [L] (S19), and 20 fps looks choppy for flight footage [I]. So a custom recorder wins on integration rather than raw image quality: one power feed, files synced to telemetry and time, PC access through one port, and possibly weight and cost [I].

**Complexity:** **1/10**.

### Option A — One SoC does both (dual stream)

- **A1:** your existing Thinker Base (SSC338Q).
- **A2:** a new single-CV610 board.

```
                  +------------------- one SoC --------------------+
                  |  (SSC338Q on today's Thinker, or one CV610)    |
 Camera --MIPI--->|  ISP -> encoder -+- main stream (high rate) ---+--SDIO--> microSD
                  |                  +- sub stream (low rate)      |
                  |                        |                       |
                  |                     WFB-ng --> USB2 host ------+--SH-4--> USB Wi-Fi --> RF
                  |  UARTs ----------------------------------------+--SH-6--> flight controller
                  |  Ethernet PHY pairs --+                        |
                  +-----------------------|------------------------+
                                          v  (J2: no parts on the Thinker)
   TODAY:  stock J2-to-RJ45 cable --> USB-C Ethernet dongle --> PC
   LATER:  LAN transformer --> USB-Ethernet chip (e.g. RTL8152B) --> USB-C --> PC
           (optional on-board version; replaces the old Ethernet connector)
```

**USB-C to PC without a second SoC:**
- **Today, no PCB:** the Thinker kit's J2-to-RJ45 cable "can be connected directly to a computer" [V] (S3). OpenIPC's setup guide says the PC's Ethernet port "can be [a] USB-Ethernet dongle" [V] (S45). So a USB-C Ethernet dongle gives you USB-C to the PC now [I].
- **Later, on the board:** a USB-to-Ethernet chip behind a USB-C connector, wired to the SoC's Ethernet pairs.
  - The RTL8152B is a QFN-24, 4×4 mm USB 2.0 to 10/100 controller with a built-in PHY [L] (S35). The PC sees a USB network adapter, and **no SoC kernel change is needed**.
  - Copy the topology that already works: Thinker pairs wired direct, and a **LAN transformer on the RTL8152B side**, just as the PC side has one today [I].
  - OpenIPC's own SSC338Q-based Mario AIO already does this. Its "USB debug port" appears on the PC as a "USB 2.0 to Fast Ethernet adapter" (CoreChip SR9900 driver) [V] (S44). Its exact circuit is not published [U].
- **A USB 2.0 analog switch** that hands the SoC's USB port to USB-C is **not** a safe A1 path. It needs USB device mode, OpenIPC's SigmaStar kernel builds USB host-only with the gadget framework off [V] (S50), and whether the SSC338Q hardware supports device mode is undocumented [U].

| | |
|---|---|
| **Pros** | A1 runs today's proven firmware. Recording plus streaming needs **no new hardware**. USB-C works today with a dongle. Smallest and lightest. |
| **Cons** | One encoder budget for both streams. On the SSC338Q the FPV encoder features apply to `video0` only. A2 needs a full FPV software port. |
| **Complexity** | A1 config plus dongle: **1/10**. A1 plus an on-board USB-C network adapter: **4/10**. A2: **7/10** for hardware, plus a large software port. |

### Option B — Two SoCs, one camera

This is your idea as you stated it. There are three ways to split the work.

**B1: camera on the RF SoC; the recording crosses the link**

```
 Camera --MIPI--> SoC#1 (RF) ---- encodes BOTH streams
                   |  sub stream  --> WFB-ng --> USB host --> Wi-Fi --> RF
                   |  main stream --> Ethernet PHY
                   |                      ||   (<= ~90 Mbit/s usable)
                   |                  Ethernet PHY
                   +--- UART/GPIO --- SoC#2 (recorder) --SDIO--> microSD
                                         +--USB device--> USB-C --> PC
```

**B2: camera on the recorder SoC; only the live stream crosses the link**

```
 Camera --MIPI--> SoC#2 (recorder) ---- encodes BOTH streams
                   |  main stream --SDIO--> microSD
                   |  USB device  --> USB-C --> PC
                   |  sub stream  --> Ethernet PHY
                   |                      ||   (a few Mbit/s)
                   |                  Ethernet PHY
                   +--- UART/GPIO --- SoC#1 (RF) --> WFB-ng --> USB host --> Wi-Fi --> RF
```

**B3: B2 with an unmodified Thinker as the radio (one camera, stepping-stone version)**

```
 Camera B --MIPI--> NEW CV610 recorder board ---- encodes BOTH streams
                     |  main stream --SDIO--> microSD
                     |  USB device  --> USB-C --> PC
                     |  sub stream  --> Ethernet PHY --> ESD --> LAN transformer
                     |                                               ||  SH-6 cable to J2 (+ separate GND wire)
                     +--- UART (1k series) --------------------> stock Thinker (SSC338Q; its own camera unused)
                                                                    WFB-ng --> USB host --> Wi-Fi --> RF
```

**Pros**
- B2 and B3 neatly give each SoC's single USB port one job: Wi-Fi host on one, PC device on the other.
- In B2 and B3 the recording never crosses the link.
- B1 matches how you described the idea.
- B3 matches your **one-camera** goal and keeps the proven radio software. Only one new board is needed.

**Cons**
- In all three, **one encoder still does all the encoding**, so the video capability is no better than option A [I].
- B1: the recording is capped by the 100 Mbit link (realistically about 60–80 Mbit/s) [I], and SoC #2 becomes a file server.
- B2/B3: the live stream takes an extra network hop. The added latency is small but unmeasured [U].
- B2/B3: FPV quality depends on **low-latency encoding on the CV610, which is unproven** [U]. On HiSilicon, Majestic has no FPV module. It has only the per-channel packet-loss settings (`refEnhance`, `refPred`) [V] (S16).
- B3: the stock air-side script starts `wfb_tx` with a local-only video input (`-U rtp_local`) [V] (S51). Feeding it from the network needs a script change, and whether that works is [U]. The Thinker's own camera and Majestic sit idle.
- All: double the power, area, BOM and software stacks.

**Complexity:** B1/B2 **8/10**. B3 **7/10**.

### Option C — Two SoCs, two cameras (a true split)

Built either as a single board of about 36×36 mm with 30.5×30.5 mm mounting holes, or as a stack of two 25×25 mm boards with 20×20 mm holes.

```
 +------------------ BOARD 1: RF (25x25, 20x20 holes) ------------------+
 |  Camera A --MIPI--> SoC#1 --USB host--> SH-4 --> Wi-Fi adapter --> RF |
 |                     SoC#1 --UART-----> SH-6 <--> flight controller    |
 |  VCC 2-6S (SH-6) --> own buck --> 5V_RF --> SoC#1 rails, Wi-Fi port   |
 |                Ethernet PHY     UART0   RESET     VCC    GND          |
 +---------------------||------------|-------|--------|-----|------------+
                       ||   board-to-board stack connector   |
 +---------------------||------------|-------|--------|-----|------------+
 |       LAN transformer / caps     UART    GPIO    VCC in   GND         |
 |  Camera B --MIPI--> SoC#2 --SDIO--> microSD                           |
 |                     SoC#2 --USB device--> USB-C --> PC                |
 |  VCC --> own buck --> priority mux <-- USB-C VBUS --> recorder rails  |
 +---------------- BOARD 2: RECORDER (25x25, 20x20 holes) ---------------+
   "||" = two differential pairs, TX crossed to RX, through a LAN transformer
          (default) or series capacitors (experiment)
```

**Pros**
- Each SoC gets its full ISP and encoder, so you get high-fps FPV **and** 4K/6 MP recording.
- Each USB port has one job.
- The link carries only control, telemetry, preview and file transfers.
- The stack keeps the 20×20 mounting pattern.

**Cons**
- Two cameras, each needing a supported sensor.
- The most area, power, weight and cost.
- A stack traps heat between the boards.
- Needs a board-to-board connector.
- Two SoC software stacks, including a WFB-ng port if SoC #1 is a CV610.

**Complexity:** **9/10**.

### Option D — Stepping stone towards C: unmodified Thinker plus a new CV610 recorder board

```
 +--- Existing Thinker Base (unchanged: SSC338Q, stock OpenIPC FPV) ---+
 |  Camera A -> SSC338Q -> USB -> H1 (SH-4) -> Wi-Fi adapter -> RF      |
 |  J3 (SH-6): VCC, GND, flight-controller UART                        |
 |  J2 (SH-6): 1 UART_RX0, 2 UART_TX0, 3/4 TX+/-, 5/6 RX+/-  (NO GND)  |
 +-------------|-------------------------------------------------------+
               |  6-wire SH cable, plus GND (and VCC) from the power harness
 +-------------|------------- New CV610 recorder board ----------------+
 |  SH-6 pairs -> ESD -> LAN transformer -> CV610 Ethernet PHY         |
 |  SH-6 UART  -> 1k series resistors   -> CV610 UART                  |
 |  Camera B -> CV610 -> microSD                                       |
 |              CV610 USB device -> USB-C -> PC                        |
 |  VCC in -> own buck -> priority mux -> rails  (USB-C VBUS = backup) |
 +---------------------------------------------------------------------+
```

**Watch out for J2's ground.** The six J2 contacts carry only UART0 and the two Ethernet pairs. The connector's MP1/MP2 tabs are mounting pads, not cable contacts [V] (S1). A common ground therefore has to come from the power harness, for example J3's GND pins [V] (S1).

**Pros**
- The radio side runs proven firmware from day one.
- The CV610 side needs **no Wi-Fi stack and no WFB-ng**, only camera, encoder, SD, USB device mode and networking [I]. That removes the biggest software blocker.
- The link copies a topology that is already known to work: the Thinker's bare PHY pairs into a transformer, exactly as with a PC today [V] (S1, S3).
- It proves both the inter-SoC link and your first SoC board at half the risk.
- It can also be the **end state**: a permanent SSC338Q radio plus CV610 recorder.

**Cons**
- Still needs a supported sensor on the CV610, plus USB device mode and recording on the CV610. Whether the "lite" Majestic supports SD recording is not verified [U].
- A second camera, which you did not originally ask for. B3 is the one-camera alternative.
- A cable instead of a stack connector, so the pairs need ESD protection. The transformer also takes board area.

**Complexity:** **6–7/10**.

### Summary

Weights exclude the Wi-Fi adapter and antennas. The Thinker weighs 13.4 g with its heatsink [V] (S3). The other weights are my estimates [I].

| Option | SoCs | Cameras | Board | Gain over today | Main blocker | Weight / height [I] | Complexity [I] |
|---|---|---|---|---|---|---|---|
| 0 | 1 + bought camera | 2 | existing + bought | independent high-quality recording | separate device, no integration | 13.4 g + camera (tens of g) | 1 |
| A1 | 1 (SSC338Q) | 1 | existing | record + stream; USB-C via dongle | bench test only | 13.4 g | 1 (+3 for on-board USB-C) |
| A2 | 1 (CV610) | 1 | new 25×25 | same as A1 | full CV610 FPV software port | ~13 g | 7 |
| B1 / B2 | 2 | 1 | new, larger | separate USB jobs (B2) | one encoder; link cap (B1) | ~18–22 g | 8 |
| B3 | SSC338Q + CV610 | 1 | Thinker + new 25×25 | your one-camera idea with a proven radio | CV610 low-latency encoding; `wfb_tx` input change | ~24–28 g + cable | 7 |
| **C** | 2 | 2 | 36×36 or 2-board stack | **high-fps FPV + 4K record** | software on both SoCs, area, docs | ~25–30 g; stack adds ~8–12 mm height | 9 |
| **D** | SSC338Q + CV610 | 2 | Thinker + new 25×25 | same as C, lower risk | CV610 sensor + USB device mode | ~25–30 g + cable | 6–7 |

---

## 6. Long-term target (only if Stage 0 fails and the Stage 4 gate passes)

**Decision rule:**
- If the Stage 0 bench test (section 8) shows the existing Thinker can record the quality you want while streaming, **stop at option A1**. Use the Stage 0b dongle for USB-C, or later build the on-board USB-C network adapter.
- If an action camera (option 0) meets your recording needs, buy one.
- If you want one camera with a dedicated recorder SoC, build **option B3**.
- If you need something one chip cannot do, such as high-fps FPV plus 4K recording, build **option D first**, and only later the full **option C** two-board 25×25 stack.

Everything below describes the recorder board (D or B3) first, then the full dual-SoC target (C).

### 6.1 Block diagrams

**First build: the CV610 recorder board (option D; B3 is the same board without Camera A's role)**

```
            NEW RECORDER BOARD  (25x25, 20x20 holes, CV610-20S on the OUTER face)
 +-------------------------------------------------------------------------------+
 | SH-6 to Thinker J2: TX+/-, RX+/- --ESD--> LAN transformer --> CV610 PHY        |
 |                     UART_RX0/TX0 --1k series--> CV610 UART (Thinker console)   |
 | GND + VCC from the power harness --TVS--> buck (>=36 V) --> 5V_BAT             |
 | 5V_BAT --> priority power mux (battery wins) <-- USB-C VBUS (PC)  --> 5V_REC   |
 | 5V_REC --> 3V3 / DDR / core (SVB-PWM) / 1V8 regulators, each with EN + PG      |
 |                                                                                |
 | Camera B (FPC) --MIPI 4-lane--> CV610-20S <--SPI--> NOR flash; 24 MHz crystal  |
 |                                   |-- SDIO0 --> microSD (pull-ups, ESD, switched VDD)
 |                                   |-- USB2 device --> ESD --> USB-C (5.1k on CC1, 5.1k on CC2)
 |                                   |-- GPIO <-- VBUS divider (cable detect)     |
 |                                   |-- GPIO --> "recording" LED                 |
 |                                   |-- UART0 console --> test pads              |
 +-------------------------------------------------------------------------------+
```

**Later: the full option C stack**

```
                 BOARD 1: RF  (SoC on the OUTER face, heatsink on it)
 +-------------------------------------------------------------------------------+
 | SH-6 (VCC 2-6S, GND, FC UART) --TVS--> buck (>=36 V rated) --> 5V_RF           |
 |                                                                                |
 | Camera A (FPC) --MIPI 4-lane--> SoC#1 <--SPI--> NOR flash                      |
 |                                  |  24 MHz-class crystal, reset                |
 |                                  |-- USB2 host --> current-limit --> SH-4 --> Wi-Fi adapter
 |                                  |-- UART --> SH-6 (flight controller, MSP)    |
 |                                  |-- Ethernet PHY (TX+/-, RX+/-) -----------+  |
 |                                  |-- UART0 console <--[1k]----------------+ |  |
 |                                  |-- RESET (pulled inactive) <--[DNP]---+ | |  |
 +-----------------------------------------------------------------------|-|-|--+
        stack connector: 2 diff pairs, UART, RESET, GPIO, VCC, many GND  | | |
 +-----------------------------------------------------------------------|-|-|--+
 |                      LAN transformer (default) or series caps <-------|-|-+  |
 |                                  |-- Ethernet PHY (TX/RX crossed)     | |    |
 |                                  |-- UART --[1k]----------------------|-+    |
 |                                  |-- GPIO, open-drain (SoC#1 RESET) --+      |
 | Camera B (FPC) --MIPI 4-lane--> SoC#2 <--SPI--> NOR flash                     |
 |                                  |-- SDIO0 --> microSD (pull-ups, ESD, switched VDD)
 |                                  |-- USB2 device --> ESD --> USB-C (5.1k on CC1, 5.1k on CC2)
 |                                  |-- GPIO <-- VBUS divider (cable detect)    |
 | VCC (stack) --> own buck --> priority mux --> 5V_REC <-- USB-C VBUS           |
 +-------------------------------------------------------------------------------+
                 BOARD 2: RECORDER  (SoC on the OUTER face)
```

**Pin budget check (do this before drawing a schematic).**
- Each CV610 has only three UARTs (UART0–UART2) [V] (S10, S14).
- Both SDIO ports share pins with eMMC, UART1/UART2, SPI, I2S, PWM, JTAG and a sensor reset [V] (S13, S14).

Make a table that fits everything one SoC needs:
- UART0 console;
- flight-controller UART (RF SoC, or the recorder too if it gets its own telemetry feed, section 6.7);
- inter-SoC UART;
- SD on SDIO0;
- sensor clock, reset and I²C;
- USB;
- GPIOs for reset, record trigger, LED and VBUS detect.

If it does not fit, drop the inter-SoC UART and use the network [I].

### 6.2 The inter-SoC link

**What it is.** The two SoCs' **internal 100BASE-TX PHYs** are wired to each other ("PHY-to-PHY"). A direct MAC-to-MAC connection (RMII) is not possible: the CV610 pin-mux table has no RMII/MII/MDIO function, and the device tree uses its internal PHY [V] (S14, S10).

**Coupling: start from what is proven.**
- The Thinker drives a PC's LAN transformer with no parts at all on its own side [V] (S1, S3). A PHY with a current-mode driver needs bias from a local transformer centre tap, so the SSC338Q PHY is very likely voltage-mode or self-biased [I].
- **Default (options D and B3):** put **one LAN transformer on the CV610 side** and wire the Thinker side directly. That copies the Thinker-to-PC topology and HiSilicon's usual IP-camera layout. OpenIPC's open Hi3516EV200 board ties its transformer centre taps to GND through 10 nF [V] (S39).
- **Default (option C):** put one small LAN transformer between the two PHYs. Bias each winding's centre tap the way its PHY needs, which is unknown for the CV610 [U] (S33). Add footprints for bias and termination resistors [I].
- **Experiment only:** capacitor-only coupling, one series capacitor in each of the 4 lines. Application notes use about 33–100 nF [L] (S33). Whether the CV610 PHY tolerates this is unknown [U].
- Cross TX± to RX± and the reverse **on the PCB** (or in the cable). Do not rely on auto-MDIX; whether it is supported is unknown [U].

**Link mode.** Force 100 Mbit full duplex with auto-negotiation off, as TI suggests for transformerless links [L] (S33). With a transformer, normal auto-negotiation should work [I].

**Layout:**
- 100 Ω differential pairs, as short as possible, length-matched within each pair, over solid ground [L] (S33).
- Keep them away from the buck converter's switch node and from the MIPI/SDIO lines [I].
- Put many GND pins in the stack connector.

**ESD and compliance.**
- Any pair that reaches a connector or a cable handled separately (options D and B3, the Stage 3 test board) needs low-capacitance ESD parts. A transformer also gives isolation [I].
- Only a fixed stack connector inside one enclosure can skip them [I].
- A transformerless link is not IEEE 802.3 compliant. It is acceptable only inside one enclosure on one shared ground [L] (S33).

**Throughput.** 100 Mbit/s per direction, about 90 Mbit/s over TCP. That is an estimate; measure it with `iperf3` [I]. In this architecture the link carries control, telemetry, a preview stream, file transfers and the PC's traffic to SoC #1. **It never carries the recording.**

**IP addressing plan [I].** Every OpenIPC unit falls back to 192.168.1.10 when no DHCP server answers [V] (S3), so two SoCs on one link will clash. Plan it, for example:
- **Internal link:** a small private subnet, e.g. 10.10.0.1 (radio) and 10.10.0.2 (recorder). Avoid 10.5.0.x, which the WFB-ng tunnel uses [V] (S51).
- **USB gadget link:** a separate subnet, e.g. 192.168.7.1 (recorder) and 192.168.7.2 (PC), with the recorder serving DHCP.
- **Reaching the radio SoC from the PC** (web UI, SSH, firmware updates, OpenIPC Companion): kernel IP forwarding on the recorder plus a static route on the PC, or user-space port forwarding. NAT and bridging are both off in OpenIPC's `cv6xx` kernel (`NF_CONNTRACK`, iptables, `CONFIG_BRIDGE`) [V] (S5). In option D, the Thinker sits behind the recorder, so its updates must go through this route.

**Side-band signals:**
- **UART:** SoC #2's UART goes to SoC #1's console. This is your recovery path when SoC #1's software is broken [I].
- **Reset:** a SoC #2 GPIO can drive SoC #1's reset. **This is a flight hazard**: a recorder crash, reboot or floating GPIO during boot could reset the radio mid-flight [I]. To prevent it:
  - pull the line to "inactive" on SoC #1's side;
  - drive it open-drain through a series resistor;
  - add a solder jumper or do-not-fit part so it is disconnected for flight.
- **Record-trigger/sync GPIO.**
- **Back-powering.** When only USB-C powers the recorder, these lines can feed current into the unpowered radio SoC through its I/O protection diodes. That can cause odd boots or latch-up [I]. Fit series resistors of about 1 kΩ, or buffers rated for powered-off inputs (74LVC1G125-class). The transformer-coupled Ethernet pairs are not a problem [I].

**Software:**
- Plain IP routing (above) is simpler than bridging. Bridging needs `CONFIG_BRIDGE`, which is off in OpenIPC `cv6xx` [V] (S5).
- If SoC #1 is a CV610, it also needs `CONFIG_WIRELESS`, Wi-Fi drivers and WFB-ng ported [V] (S5, S7).

### 6.3 USB-C to the PC (on the recorder SoC, #2)

**Hardware checklist:**

| Item | Detail | Source |
|---|---|---|
| Connector | USB 2.0-only, 16-pin USB-C receptacle | [L] (S36) |
| CC pins | **Separate** 5.1 kΩ pull-down on CC1 **and** on CC2. Never share one resistor (the Raspberry Pi 4 bug), and never fit the pull-up meant for hosts | [L] (S36) |
| Data lines | Join A6/B6 (D+) and A7/B7 (D−) at the connector; route as 90 Ω differential | [L] (S36) |
| ESD | Low-capacitance array on D+/D−/CC, e.g. TPD4E05U06 (0.5 pF, 5.5 V); a TVS on VBUS | [L] (S37) |
| Cable detect | VBUS through a resistor divider to a SoC #2 GPIO | [I] |
| Power path | A **priority power mux**, e.g. TI TPS2121 (2.8–22 V, up to 4.5 A, automatic priority with manual override) [L] (S53). Battery-derived 5 V wins whenever present. VBUS feeds **only the recorder's local rail**. A plain ideal diode would let the PC's VBUS (4.75–5.25 V) carry the whole drone whenever it sits above the buck's 5.0 V [I] | [L] (S53), [I] |
| RF side on USB power | The RF board and the Wi-Fi current-limit switch stay disabled when only USB power is present | [I] |
| Power budget | A USB 2.0 port allows 100 mA until the device is configured and 500 mA (2.5 W) after [L] (S54). Budget the recorder at about 2–2.5 W or less in "PC mode". Drawing more than 100 mA while booting is out of spec, though most PC ports tolerate it [I] | [L] (S54), [I] |

**Software:**
- The CV610's USB controller is dual-role, and the vendor default is device mode [V] (S11).
- OpenIPC's `cv6xx` kernel builds it **host-only with USB gadget off**, so you need a custom kernel config [V] (S5).
- The vendor's own kernel config does enable the gadget framework with ACM (serial), RNDIS, mass storage, UAC1 and UVC. NCM is not enabled by default [V] (S12).

**What the PC should see:**

| Function | Notes |
|---|---|
| **CDC-NCM network adapter** | Binds automatically on Windows 11. Windows 10 needs the driver picked manually. Works on Linux [L] (S38) |
| **ACM serial console** | Standard serial port |
| RNDIS | Only if Windows 10 plug-and-play matters. It is deprecated. Linux maintainers have proposed disabling it since 2022 (re-pushed in December 2024, refreshed again in 2026), but this is not merged: mainline still has both the RNDIS host and gadget drivers [V] (S55). The vendor CV610 config enables `USB_CONFIGFS_RNDIS` [V] (S12) |
| Mass storage (optional) | SoC #2 must unmount the SD card while it is exported, or files get corrupted. Prefer copying files over the network [I] |
| UVC webcam (optional) | The SDK ships a `hi_uvc` module [V] (S13) |

**Copying footage is slow over this link [I].** A 10-minute recording at 80 Mbit/s is about 6 GB. At about 11 MB/s over a 100 Mbit link that takes roughly 9 minutes. Taking the microSD out and using a card reader is much faster. If "get footage off" is the main reason for USB-C, pulling the card may be the right answer.

**Recovery.**
- Whether a CV610 can be flashed over USB is unknown [U].
- OpenIPC's `defib` UART-recovery tool now resolves boot images for CV610 variants 00g, 00s, 10b, 20g and 20s [V] (S52). The change was not tested on hardware [U].
- Keep UART test pads and boot-mode access for **both** SoCs.

### 6.4 Power tree sketch

```
 VCC 2-6S (Thinker label "8~26V"; 6S full = 25.2 V)
   |
  TVS
   |
   +--> RF board buck (>= 36 V, 5.0 V, >= 3 A) --> 5V_RF --+--> current-limit switch --> Wi-Fi adapter 5V
   |                                                       +--> SoC#1 rails (as below)
   |
   +--> recorder buck (>= 36 V, 5.0 V) --> 5V_BAT --> [priority mux: 5V_BAT wins] --> 5V_REC --> SoC#2 rails
                                                               ^
                                  USB-C VBUS (PC, 500 mA) -----+   (feeds the recorder only)

 Rails per SoC (from 5V_RF or 5V_REC):
   3V3 buck        DDR buck            core buck            1V8 LDO       camera rails
   IO, NOR, SD,    1.5 V (DDR3) or     ~0.9 V, ONE PER      IO domains    1V15 buck,
   PHY, USB        1.35 V (DDR3L)      SoC, trimmed by                    3.0 V LDO, 1V8
                                       SVB-PWM                            (per camera)

 Enable / power-good chaining in the order the CV610 requires (UNKNOWN: needs the hardware design guide)
```

**Notes:**
1. **Use a higher-rated input buck.** The Thinker's TPS54335A is rated 4.5–28 V and 3 A [V] (S26). A full 6S pack at 25.2 V leaves under 3 V of headroom for spikes on the drone's power line. Use a buck rated 36–40 V or more, plus a TVS [I].
2. **Make sure the "5V" really is 5 V.**
   - In the schematic, the rail named "5V" is **3.29 V as drawn**, because resistor R37 is not fitted. With R37 = 0 Ω it becomes 4.99 V. The silkscreen reads "NC=3V3 / CL=5V" [V] (S1, S2).
   - OpenIPC's docs tell you to de-solder the "Vd voltage select resistor" for 3.3 V modules, which implies units ship at 5 V [V] (S3).
   - Measure yours. A USB Wi-Fi adapter needs true 5 V.
3. **Check the current budget [I].** Two SoC boards draw about 2.5–4 W without Wi-Fi, and the Wi-Fi adapter adds about 1.5–8 W. That is up to about 12 W, about 80% of the 15 W a 3 A, 5 V buck can supply, leaving little margin for inrush and transmit peaks. A separate buck on each board, each fed from VCC, avoids this. That is also the simplest arrangement for option D [I].
4. **Do not reuse the Thinker's small regulators.** The Thinker's IM6001 converters have no datasheet I could find [U] (S1), and every regulator's enable pin is tied to its input, so there is **no power sequencing** [V] (S1). For the CV610, use regulators with enable and power-good pins so the start-up order can be controlled [I].
5. **Use one core regulator per SoC.** The CV610 trims its own core voltage through an SVB-PWM signal [L] (S10). On a single larger board, the 3.3 V, DDR and 1.8 V rails could be shared between the two SoCs [I].
6. **The LDOs can be reused.** The RS3236 (500 mA LDO) is fine for 1.8 V and the camera's 3.0 V [L] (S27).
7. **USB power feeds the recorder only**, through the priority mux in section 6.3. The RF board stays off.

### 6.5 What carries over from the Thinker

| Thinker block (S1, S2) | In the new design | Notes |
|---|---|---|
| Input buck TPS54335A + its enable/under-voltage network | **Modify** | Same topology, but a part rated ≥36 V. Fix the output at 5 V |
| IM6001 3.3 V / 1.5 V / 0.9 V regulators | **Replace** | No datasheet [U]; CV610 needs sequencing and core trimming |
| RS3236 LDOs (1.8 V, 3.0 V) | **Reuse** | One set per camera / per SoC |
| SSC338Q (U2) | **Remove**, or **keep as the RF SoC** (options D and B3) | — |
| W25Q128JV boot flash (16 MB) | **Duplicate** | One per SoC. Check the size against OpenIPC `cv6xx` images: the larger "ultimate" builds already fail size checks [V] (S6) |
| 24 MHz crystal + reset RC network | **Duplicate** | The CV610 device tree's system timer runs at 24 MHz, which points to a 24 MHz reference [L] (S10). Reset polarity unverified [U] |
| No RTC crystal or backup (32 kHz pins tied off, AVDD_RTC on 3V3) [V] (S1) | **Consider adding on the recorder** | A 32.768 kHz crystal plus a supercap keeps file times across reboots (section 6.7). CV610 RTC pins unverified [U] |
| microSD socket J8 (Molex 104031-0811) | **Move to the recorder and improve** | Add CMD/DAT pull-ups, a CLK series resistor, ESD and a switched card supply (the Thinker leaves its SD power-enable unused). 1.8 V signalling is **not** needed, because the CV610 device tree enables no UHS modes [V] (S10) |
| Camera connector J1 (Hirose DF56C-26S, 26-pin) + 1V15 / 3.0 V / 1V8 rails | **Reuse, one per camera** | The module must carry a sensor OpenIPC supports [V] (S8) |
| Ethernet pairs to J2 | **Become the internal link** | Bare pairs on the Thinker side, LAN transformer on the far side. J2 is used as-is in options D/B3 and removed in option C |
| H1 Wi-Fi port (SH-4: 5V/DM/DP/GND) | **Keep on the RF SoC** | True 5 V, plus current limiting and ESD [I] |
| M1 footprint (BL-M8731BU internal Wi-Fi) | **Remove** | It adds a stub to the USB lines [V] (S1) |
| J3 (SH-6: VCC, flight-controller UART, GND) | **Keep on the RF board** | — |
| UART0 debug console | **Test pads per SoC, plus the cross-link** | — |
| MEMS mic | **One SoC only** | OpenIPC's `cv6xx` package ships an internal audio-codec module (`ot_acodec.ko`), so an analog mic input is likely [L] (S8) |
| LEDs D7 (power) / D8 (Ethernet link) | **Keep D7; repurpose D8** as a "recording" LED on SoC #2 | — |
| Under-chip thermal path (pad with a 3.1 mm hole, 5 mm pad on the back) | **Duplicate per SoC** | Whether the hole is plated is unknown [U] (S2) |

### 6.6 Software work this implies

The effort ranges are my estimates for a beginner [I]. Two SoCs also mean two firmware images, two web UIs and passwords, and two update procedures to maintain.

| Task | Skill needed | Effort [I] | Dev-board check (Stage 4) |
|---|---|---|---|
| Build OpenIPC from source on a Linux PC | Linux command line, Buildroot | days | Your own image boots |
| Kernel options: USB gadget (configfs, NCM, ACM), IP forwarding | Kernel configuration | days to weeks | The PC sees a network adapter |
| configfs gadget script (NCM + serial) | Shell scripting | days | The PC gets an IP address and a console |
| Glue scripts: record trigger, SD-full handling, LED, time sync, routing | Shell scripting | weeks | Recording starts and stops on command |
| Sensor driver port (for example IMX335/IMX415), if the camera is unsupported | C, the vendor SDK, sensor register tables (often under NDA) | months; possibly impossible without vendor tables. **Likely the hardest item** | Your chosen sensor streams |
| Low-latency encoder tuning on the CV610 (B2/B3 only) | Encoder settings, latency measurement | weeks to months | Measured glass-to-glass latency |
| Wi-Fi driver + WFB-ng port (only if SoC #1 is a CV610) | C, Linux Wi-Fi drivers | months | Monitor mode and packet injection work |

### 6.7 Time sync, telemetry and record control

**Time [V/I].**
- The Thinker has no RTC crystal or backup cell [V] (S1), so its clock likely resets at every boot [I]. There is no internet time in the field.
- OpenIPC's per-frame sensor timestamps are supported only on EV200, EV300, GK7205V200, AV300 and CV500, not on the CV610 or SSC338Q [V] (S48).
- Without a fix, files will be misdated and two cameras' files will not line up [I]. Fixes [I]:
  - make one SoC an NTP/chrony server for the other over the link;
  - take absolute time from the flight controller's GPS (via MSP/MAVLink) or from the ground station;
  - log a GPIO record-start pulse on both SoCs;
  - consider an RTC crystal plus a supercap on the recorder.
- Good timing is also needed to stabilise footage in Gyroflow using the flight controller's gyro log [I].

**Telemetry, OSD and arming [I].**
- The flight-controller UART goes only to the RF SoC. On its own, the recorder cannot know when the drone is armed, cannot burn in the OSD and cannot log telemetry.
- Forward MSP/MAVLink over the link by UDP, or give the recorder its own flight-controller UART (check the UART budget in section 6.1).
- Define an arm-to-record rule, and decide whether recordings should carry the OSD.

---

## 7. Board and PCB implications

### 7.1 Size

These are my estimates [I], calibrated against the Thinker's renders (S2).

| Layout | Fill of usable two-sided area |
|---|---|
| Today's Thinker, one SSC338Q, 25×25 | about 79% |
| Dual CV610 on one 25×25 board | **101–106%: does not fit before any routing** |
| Dual CV610 on one 36×36 board (30.5×30.5 holes) | 45–48% |
| Dual CV610 as a stack of two 25×25 boards | 52–55% |

Two 9×9 mm QFNs take about the same package footprint as one 12.3 mm SSC338Q: 2 × 81 = 162 mm² against 151 mm². The overflow comes from duplicating everything around each SoC: flash, crystal, regulators, decoupling and connectors [I].

### 7.2 Layers and fabrication rules

- **The Thinker itself** is 6 layers, through-hole vias only, 0.2 mm minimum drill, 0.10–0.127 mm minimum trace and mostly 0.15 mm traces [V] (S2).
- **JLCPCB's standard 6-layer service covers these rules:** 0.09 mm trace/space, 0.15 mm minimum via, controlled impedance, and free filled via-in-pad [L] (S42).
- **No HDI (microvias) is needed.** The QFN's pins are on its edges and escape directly on the top layer [L] (S39).
- **6 layers is recommended** (signal / GND / signal-power / power / GND / signal) so every differential pair has a solid ground reference [I]. OpenIPC's open Hi3516EV200 board shows that 4 layers can work for **one** SoC in this package class (QFN-88, 9×9 mm, 0.35 mm) [V] (S39).
- **Under each SoC**, put an array of thermal vias in the exposed pad, filled or plugged so solder does not wick away [L] (S42).

### 7.3 Controlled-impedance routing

| Interface | Target | Notes |
|---|---|---|
| MIPI CSI-2 | 100 Ω differential | Match lengths within each pair and to the clock pair. The camera connector must be on the same board as its SoC [I] |
| USB 2.0 | 90 Ω differential | Low-capacitance ESD at the connector [L] (S37) |
| Internal Ethernet | 100 Ω differential | LAN transformer (default) or series capacitors; TX/RX crossed [L] (S33) |
| SD (50 MHz, 4-bit) | relaxed | Keep it short and add CLK damping [I] |

Rough outer-layer geometry, assuming about 0.1 mm prepreg [I]:
- 0.15 mm trace / 0.15 mm gap gives roughly 90–95 Ω.
- 0.12 mm trace / 0.15 mm gap gives roughly 100–105 Ω.

Always use the fab's stackup calculator for the real values.

### 7.4 Heat, RF and regulations

- **A heatsink is mandatory.** The Thinker weighs 8.8 g bare and 13.4 g with its heatsink [V] (S3).
- **In a stack, put each SoC on an outer face**, because the gap between boards traps heat [I].
- Expect roughly 0.6–1.2 W per CV610 [I] (S24 quotes 668 mW for a CV608-class solution).
- Switching regulators and two clock domains near a 5.8 GHz antenna feed can hurt receive sensitivity. Plan placement and shielding [I].
- **Transmit power is your legal responsibility.** OpenIPC says: "Set power level accordingly to laws in your country, stock 25mW is legal in majority of countries" [V] (S4). The stock air-side script sets the "world" regulatory domain (`iw reg set 00`) and a fixed transmit power [V] (S51), so the firmware will not enforce your local rules. Check band, power limit and any licence requirement for your country [I].
- **A board you share or sell** needs EMC testing (FCC/CE or your local equivalent) [I].

### 7.5 Assembly, tools and cost

- **Professional assembly is required.** Fine-pitch (0.35 mm) QFNs with exposed pads, on both sides of the board, need it. Rework needs hot air plus a microscope [I].
- **JLCPCB "Standard" PCBA fees** (September 2025) [L] (S43):
  - $50 setup for double-sided assembly
  - $15.72 stencil
  - $1.50 per component feeder
  - $3.50 for hand soldering
- **The CV610's price is unknown**, and it may need global sourcing or be supplied by you [U] (S23).

| Item | Rough cost |
|---|---|
| Bare 6-layer PCB, 5 pcs, 50×50 mm or smaller | about $2 (promo) to about $60 [L] (S42) |
| Fixed assembly fees per design | about $120–150 [I] |
| One board design, first assembled run | $250–500 [I] |
| Two-board stack, first run | $450–900 [I] |
| Beginner total including 2–3 re-spins and tools | $1,000–3,000 [I] |

**Tools you will need [I]** (prices are rough):

| Tool | Why | Rough price |
|---|---|---|
| Current-limited bench power supply | Safe first power-on of every new board | $40–80 |
| Multimeter | Rail voltages, the Wi-Fi 5 V pin, continuity | $20–50 |
| Bought 3.3 V USB-UART adapter | Console access without depending on your own Stage 1 board | $5–10 |
| SOIC-8 test clip + SPI-NOR programmer (check it drives 3.3 V) | Unbricking the boot flash | $10–20 |
| Hot-air station, flux, tweezers, magnifier or microscope | Rework of small parts | $80–250 |
| Cheap 8-channel logic analyser | Debugging UART, I²C and boot problems | $10–20 |

### 7.6 Warnings about the Thinker files in this repo

- **No editable CAD exists**, only PDFs and Gerbers. Everything must be redrawn by hand. Gerbers can be imported into KiCad only as reference graphics [V] (S1, S2).
- **The Gerbers are a 3×3 panel**, not a single board.
  - The outline is 97×97 mm and the boards repeat every 28 mm [V] (S2).
  - Each copy has all four 2.05 mm mounting holes, at (2.5, 2.5), (22.5, 2.5), (2.5, 22.5) and (22.5, 22.5) mm. The panel rails carry 8 more tooling holes [V] (S2).
  - The 3.10 mm hole under the SoC sits at about (11.0, 9.75) mm on each copy [V] (S2).
  - Crop the import to one 25×25 copy before using it as a reference [I].
- **Revision mismatch:** schematic sheet 1 is dated June 11, 2025. Sheets 2–4 are dated October/November 2024, and the Gerber files are dated November 5, 2024. The copper may not match sheet 1 [V] (S1, S2).

---

## 8. Staged plan for a beginner

Each stage produces something useful even if you stop there. Costs and durations are my estimates [I]. Each fabrication, assembly and shipping round takes about 2–4 weeks. Reaching option D is realistically **6–18 months of evenings** [I].

Use **one CAD tool** throughout. I suggest **KiCad** [I]: it is free, widely taught, and OpenIPC's Wi-Fi carrier boards are KiCad projects (S39). The only EasyEDA design you need (the EV200 board) can be imported, with the checks described in Stage 5.

Before **every** order, post your schematic and layout for an OpenIPC community design review [I].

| Stage | Build | What you learn | Rough cost | Duration [I] |
|---|---|---|---|---|
| **0** | **Bench test: no PCB** | Whether you need a second SoC at all | $0–20 (an SD card) | 1–2 weekends |
| **0b** | **USB-C to PC today:** stock J2 cable + USB-C Ethernet dongle | Network access to the Thinker | $10–20 | an evening |
| **1** | **First PCB:** USB-C to UART debug adapter | KiCad, USB-C, ESD, ordering | $20–60 | 3–6 weeks |
| 2 | 25×25 power board: 2–6S to 5 V / 3 A | Switching-regulator layout, heat | $30–80 | 1–2 months |
| 3 | Link test board (for option C) + optional on-board USB-C network adapter | Controlled impedance, PHY coupling | $60–150 | 1–2 months |
| 4 | CV610 dev board (software, in parallel) | Whether CV610 software is ready; real power figures | dev board price [U] | 1–3 months, in parallel |
| 5 | First SoC board, based on the open EV200 design | QFN-88 SoC bring-up | $250–500 | 2–4 months |
| 6 | CV610 recorder board → option D or B3 | The actual recorder | $250–500 | 3–6 months |
| 7 | CV610 RF board → option C, or a merged 36×36 board | The full dual-SoC design | $250–500+ | 6+ months |

**Stage 0 — bench test on your Thinker Base (do this first).**

This stage assumes you already own a Thinker Base, a USB Wi-Fi adapter, a ground station and a display. If not, those come first; their prices were not researched [U].

1. **Check the Wi-Fi port voltage.** Measure the Wi-Fi header's 5 V (Vd) pin with a multimeter. The port is configurable for 5 V or 3.3 V [V] (S3). In the schematic the resistor that sets it is R37 ("NC=3V3 / CL=5V") [V] (S1, S2).
2. **Turn on recording.**
   - Start with the "onboard recording" toggle on the Camera tab of OpenIPC Companion [V] (S45).
   - Then edit Majestic's settings by hand: point the WFB-ng output at one channel with `outgoing … channel: main|sub`, and try both ways round with `records.substream` [V] (S15).
   - The SigmaStar FPV encoder features apply to `video0` only, and `fpv.enabled` also turns off automatic exposure and white balance [V] (S16).
3. **Measure:**
   - live latency, glass to glass: point the camera at a phone showing a millisecond stopwatch, film both the phone and the ground-station screen with a second camera, and subtract [I];
   - recording and live bitrates: `curl -s -u root:<password> http://<camera>/metrics | grep bitrate_kbps` [V] (S16);
   - dropped frames, CPU load, SD write speed and temperature.
4. **Test SD stalls.** This is the main real argument for a second SoC.
   - Record onto a slow card while watching the live link for hiccups.
   - Read `records_fragments_dropped_total` and `records_fsync_us_max` from `/metrics/records` [V] (S46).
   - Run OpenIPC's (destructive) fake-card check on your card [V] (S47).
5. **Test a power cut.** Majestic writes fragmented MP4. `records.syncSeconds` (default 30) bounds what a power cut loses, and `majestic --repair` trims damaged clips [V] (S46).
   - Pull the battery mid-recording and check that the clip plays.
   - Try a lower `syncSeconds`.
   - Whether the Thinker's FPV build of Majestic has these features is [U].

**Pass/fail [I].** Pass if all of these hold:
- the recording reaches your target bitrate from question 2 in section 9;
- `records_fragments_dropped_total` stays at 0;
- live latency and frame rate do not change noticeably with recording on;
- the temperature stays stable.

**If it passes, you do not need a second SoC.** Use Stage 0b for USB-C.

**Stage 0b — USB-C to your PC today.**
1. Plug the Thinker's J2-to-RJ45 cable into any USB-C Ethernet dongle.
2. Set the PC's adapter to a static address such as 192.168.1.11/24.
3. Open http://192.168.1.10 or connect with OpenIPC Companion [V] (S3, S45).

**Stage 1 — your first PCB: a USB-C to UART debug adapter.** Buy a ready-made 3.3 V USB-UART adapter first, so you never depend on this board. Then design a small 2-layer board with:
- a USB 2.0 Type-C receptacle with **two separate 5.1 kΩ resistors on CC1 and CC2** [L] (S36);
- a low-capacitance ESD array on D+/D− (e.g. TPD4E05U06) [L] (S37);
- a USB-UART bridge chip from JLCPCB's parts library;
- an SH 1.0 mm lead to the Thinker's console: adapter TX goes to **J2 pin 1 (UART_RX0)**, and adapter RX goes to **J2 pin 2 (UART_TX0)** [V] (S1).

Because J2 has no ground contact, add a separate GND wire, for example to a J3 GND pin [V] (S1). Confirm that the console's logic level is 3.3 V before connecting [I].

This board teaches you KiCad, USB-C, ESD and ordering.

**Stage 2 — 25×25 power board.**
- 2–6S input to 5 V / 3 A, with a buck rated ≥36 V, a TVS and 20×20 mounting holes [I].
- Measure efficiency and temperature.
- The OpenIPC Wi-Fi carrier KiCad projects are good layout references [V] (S39).

**Stage 3 — link test and optional on-board USB-C (4-layer, controlled impedance).**
- **(a) Link test (needed before option C):** a board that joins two Thinkers' J2 connectors with a common GND, ESD footprints, a LAN transformer footprint (default), and series-capacitor and bias-resistor footprints for the capacitor-only experiment. This proves a two-SoC link with real PHYs [L] (S33). Options D and B3 do not need this test, because their link copies the Thinker-to-PC topology.
- **(b) Optional on-board USB-C adapter:** USB-C → RTL8152B → LAN transformer → J2. This is a smaller, permanent version of Stage 0b and needs no SoC software change [I] (S35, S44).

**Stage 4 — de-risk the CV610 software on a dev board, in parallel from Stage 0.**
- Research named iTOP/Xunwei, Neardi, Haitu and LKD3516CV610 dev boards [U] (S22). A dev board is also the most likely legal route to the hardware design guide.
- **Go/no-go gate.** Before any CV610 PCB, confirm all of these on the dev board (see the table in section 6.6):
  - OpenIPC boots, and you can build it yourself;
  - your chosen sensor streams;
  - recording to SD works at your target bitrate;
  - USB device mode (NCM) works after a kernel rebuild;
  - for B3: the sub-stream's glass-to-glass latency is acceptable;
  - real power draw is measured.

**Stage 5 — first SoC board.** Use OpenIPC's open Hi3516EV200 board [V] (S39):
- same package class (QFN-88, 9×9 mm, 0.35 mm);
- 4 layers;
- editable EasyEDA source on oshwhub [V] (S39).

Steps:
1. First order the published EV200 design **unchanged**, with professional assembly, and get it to boot.
2. Then redraw it in KiCad. KiCad 8 and later can import EasyEDA projects [V] (S41). Check footprints, nets and copper pours against the original before trusting the import [I].
3. **First power-on method [I]:**
   - fit 0 Ω links or solder jumpers on every rail, plus test points;
   - power the regulators from the current-limited supply with the SoC rails disconnected;
   - check each voltage and the start-up order;
   - only then connect the SoC.

**Stage 6 — CV610 recorder board.**
- 25×25 mm, 6 layers, 20×20 holes.
- Contents: SPI NOR, microSD, USB-C device port with a priority power mux, camera FPC, LAN transformer and ESD for the link.
- It plugs into your unmodified Thinker through J2. That is **option D**, or **B3** if it sends its sub-stream to the Thinker's radio.
- Use the same first power-on method as Stage 5.

**Stage 7 — the full design.** Only if WFB-ng has been ported to the CV610: build a CV610 RF board and stack it (option C). Otherwise, keep D or B3 as the end state, or merge both halves into one 36×36 board.

---

## 9. Open questions to answer before committing

**Blocking**

1. **What must USB-C do?** Configuration, firmware updates, copying recordings off, a live webcam view, or powering the board? Each leads to a different design. If the answer is "get footage off", pulling the microSD into a card reader may be the right answer (section 6.3) [I].
2. **What exactly do you want to record?** Write down the resolution, frame rate and bitrate (Mbps). Is it the same view at higher quality (one SoC is probably enough), or a different, higher-resolution camera (two SoCs plus two cameras)?
3. **Must it stay one camera?** If yes, B3 is the dual-SoC route; D and C add a second camera [I].
4. **Can you legally get the CV610 datasheet, hardware design guide and reference schematic?** These cover power sequencing, the SVB core-voltage circuit, boot straps, the PHY circuit and the QFN land pattern [U] (S22).
5. **Which CV610 SKU?** -20S is recommended. It is in the datasheet's QFN list and OpenIPC has a boot image for it [V] (S14, S52). Confirm the pin count and exposed-pad size from HiSilicon's package drawing [L] (S23).
6. **Are you prepared to port software?** That means the Wi-Fi drivers and WFB-ng (if SoC #1 is a CV610), USB device mode and a sensor driver (section 6.6). Or is a hybrid acceptable (SSC338Q radio + CV610 recorder)? [V] (S5, S7, S8)
7. **Is 25×25 / 20×20 a hard requirement** (which forces a stack), or is a 30.5×30.5 board acceptable? [I]

**Technical unknowns to verify on hardware**

8. **The CV610 PHY:** voltage-mode or current-mode driver, auto-MDIX support, and does capacitor-only coupling work? [U] (S33). For the SSC338Q this is largely answered: it already drives a PC's transformer with no parts on the board [V] (S1, S3).
9. **CV610 power:** the rail list, per-rail currents and start-up order. HiSilicon lists typical power as "TBD" [L] (S20).
10. **CV610 core voltage (0.9 V or 1.0 V) and CPU clock (950 MHz or 1.2 GHz).** Sources disagree [U] (S10, S21).
11. **High frame rates and 4K:** can the CV610 run high-fps sensor modes and low-latency encoding? This was not found publicly [U]. How much do 4K modes suffer from the required offline capture path? [I] (S13)
12. **SSC338Q dual-stream:** does it work with `fpv.enabled` and a sub-stream at the same time, and what is its bitrate ceiling? This is the Stage 0 bench test [U] (S16).
13. **Recording on the CV610:** does the published "lite" Majestic for `cv6xx` support SD recording? [U] (S6)
14. **Flashing:** can a CV610 be flashed over USB? Over UART, OpenIPC's `defib` now resolves CV610 boot images but is untested on hardware [U] (S52).
15. **The PC side:** which operating systems must work plug-and-play (Windows 10 needs a manual NCM driver step), and should USB-C power the board? [L] (S38)
16. **Time and telemetry:** where should absolute time come from (flight-controller GPS or ground station)? Should recording start on arming, and should files carry the OSD? [I] (S48)
17. **Thinker hardware details:**
    - What does the Wi-Fi header's 5 V pin measure on your board (is R37 fitted)? [V] (S1, S3)
    - Is the 3.1 mm hole under the SoC plated? [U] (S2)
18. **Sourcing:** stock, price and minimum order for C46034082 (-20S) [U] (S23).
19. **Ownership:** do you already have the Stage 0 equipment (Thinker Base, Wi-Fi adapter, ground station, display)? [U]
20. **GitHub `BaktiyarBr/ssc338Q`** ("redesign from ssc338q") might be an editable SSC338Q design, but it could not be accessed [U].

---

## 10. Sources

**Repo files (local)**
- **S1** — Schematic [`schematic/338THNIKERV1.pdf`](../schematic/338THNIKERV1.pdf), sheets 1–4 (J2 pinout, R37 option, power tree, SD, USB, camera, RTC pins).
- **S2** — PCB files [`PCB/thinker338_gerber.zip`](../PCB/thinker338_gerber.zip), [`PCB/top.pdf`](../PCB/top.pdf), [`PCB/bottom.pdf`](../PCB/bottom.pdf) (parsed for layers, drills, trace widths, panel layout and mounting holes).

**OpenIPC**
- **S3** — Thinker AIO docs: https://docs.openipc.org/hardware/openipc/vtx/fpv-thinker-aio/ (repo copy: https://github.com/OpenIPC/docs/blob/main/src/content/docs/hardware/OpenIPC/VTX/fpv-thinker-aio.md)
- **S4** — WFB-ng quick start: https://github.com/OpenIPC/docs/blob/main/src/content/docs/use-cases/fpv/wfb-ng/-quick-start.md
- **S5** — `cv6xx` kernel config: https://github.com/OpenIPC/firmware/blob/master/br-ext-chip-hisilicon/board/hi3516cv6xx/hi3516cv6xx.generic.config
- **S6** — `cv6xx` defconfigs and Makefile:
  - https://github.com/OpenIPC/firmware/blob/master/br-ext-chip-hisilicon/configs/hi3516cv6xx_lite_defconfig
  - https://github.com/OpenIPC/firmware/blob/master/br-ext-chip-hisilicon/configs/hi3516cv6xx_ultimate_defconfig
  - https://github.com/OpenIPC/firmware/blob/master/Makefile
- **S7** — Builder device list: https://github.com/OpenIPC/builder/tree/master/devices
- **S8** — `cv6xx` sensor and driver package (including `files/kmod/ot_acodec.ko`): https://github.com/OpenIPC/firmware/tree/master/general/package/hisilicon-osdrv-hi3516cv6xx
- **S9** — https://github.com/OpenIPC/firmware/issues/2218 ; https://github.com/OpenIPC/firmware/pull/2461
- **S10** — CV610 device tree and drivers: https://github.com/OpenIPC/linux/blob/hisilicon-hi3516cv6xx/arch/arm/boot/dts/hi3516cv610.dtsi (plus `drivers/vendor/mmc/platform/sdhci_hi3516cv610.c` and `drivers/net/ethernet/vendor/eth/hleth.c` on the same branch)
- **S11** — https://github.com/OpenIPC/linux/blob/hisilicon-hi3516cv6xx/arch/arm/boot/dts/hi3516cv610_family_usb.dtsi
- **S12** — https://github.com/OpenIPC/linux/blob/hisilicon-hi3516cv6xx/arch/arm/configs/hi3516cv610_defconfig
- **S13** — https://github.com/OpenIPC/openhisilicon (`kernel/mipi_rx/hi3516cv6xx/.../hal_mipirx_custom.h`, `kernel/sys_config/hi3516cv6xx/pin_mux.c`, `kernel/include/hi3516cv6xx/ot_defines.h`, `kernel/mipi_tx/`, `hi_uvc`) ; https://github.com/OpenIPC/openhisilicon/pull/228
- **S14** — https://github.com/OpenIPC/ipctool/blob/master/docs/padmux.md ; https://github.com/OpenIPC/ipctool/blob/master/src/reginfo.c ; https://github.com/OpenIPC/ipctool/pull/207
- **S15** — https://github.com/OpenIPC/wiki/blob/master/en/majestic-config.md
- **S16** — https://github.com/OpenIPC/wiki/blob/master/en/majestic-encoder-tuning.md
- **S17** — https://github.com/OpenIPC/wiki/blob/master/en/fpv.md
- **S18** — https://github.com/OpenIPC/wiki/blob/master/en/memory-tuning.md

**HiSilicon CV610 (mostly second-hand)**
- **S19** — CV610 product brief v03, read via search snippets only: https://www.ickey.cn/static-pf/datasheet/d4/a2/f6aa/d4a2f6aab518a824a197df634a3e1276.pdf
- **S20** — HiSilicon product page, read via snippets: https://www.hisilicon.com/cn/products/smart-vision/public-security/ipc/hi3516cv610
- **S21** — Secondary summaries:
  - https://gitcode.csdn.net/6aa0dbfc1a35784c84207a7e.html
  - https://gitcode.csdn.net/6a1554d1662f9a54cb772c96.html
  - http://haitutech.com/product/116.html
  - https://blog.csdn.net/szhorsway/article/details/140460297
  - https://www.sohu.com/a/890001392_122391449
  - https://bbs.16rd.com/thread-627152-1-1.html
  - https://damodev.csdn.net/6a435bc010ee7a33f284915f.html
- **S22** — Documentation access:
  - https://www.scribd.com/document/967253082/Hi3516CV610-UHD-Smart-Vision-SoC-Data-Sheet (not read)
  - https://www.bilibili.com/read/cv46937974
  - http://haitutech.com/product/116.html
- **S23** — JLCPCB/LCSC listings:
  - https://jlcpcb.com/partdetail/HISILICON-Hi3516CV61020S/C46034082
  - https://jlcpcb.com/partdetail/JLCPCBAssembly-HI3516CV61020S/C9900186868
  - https://jlcpcb.com/partdetail/HISILICON-HI3516CV61020B/C22396752
  - https://easyeda.com/component/8de1f8b0aa31404a98a21f311afd8a62
- **S24** — CV608-class 668 mW figure: https://blog.csdn.net/2403_89813604/article/details/145159139

**SigmaStar**
- **S25** — https://www.sigmastarsemi.com/uploadfile/file/20220419/20220419031720_62035.pdf ; https://www.cnx-software.com/2021/05/05/sigmastar-ssc33x-camera-soc-pin-to-pin-compatible-hisilicon-hi3516-hi3518/

**Parts and interfaces**
- **S26** — TPS54335A: https://www.ti.com/product/TPS54335A ; https://www.ti.com/lit/gpn/TPS54335A
- **S27** — RS3236: https://www.run-ic.com/en/details/161/37.html
- **S28** — Connectors: Molex 104031-0811 https://heilind.com/mol104031-0811.html ; Hirose DF56C https://lcsc.com/product-detail/_HRS-Hirose-_C5338332.html
- **S29** — BL-M8731BU: https://www.lb-link.com/M8731BU4-Q-1T1R-802-11a-b-g-n-WiFi-Module-pd569675168.html ; https://docs.openipc.org/use-cases/fpv/net-cards/rtl8731bu/
- **S30** — MIPI D-PHY is point-to-point: https://www.analog.com/en/resources/app-notes/an-1337.html ; https://scdn.rohde-schwarz.com/ur/pws/dl_downloads/dl_application/application_notes/1sl410/1SL410_0e_MIPI_D-PHY_debugging_and_compliance_testing.pdf
- **S31** — Lattice MIPI duplicator and LIF-MD6000:
  - https://www.latticesemi.com/products/designsoftwareandip/intellectualproperty/referencedesigns/referencedesigns05/1ton_mipi_csi2_dsi_duplicator
  - https://www.latticesemi.com/products/designsoftwareandip/intellectualproperty/ipcore/ipcores04/1inputto2outputmipicsi2camerasplitterbridge
  - https://suntsu.com/product/integrated-circuits/field-programmable-gate-array-fpga/lif-md6000-6uwg36itr1k/
  - https://www.mouser.in/new/lattice-semiconductor/lattice-crosslink-video-bridging/
- **S32** — SerDes replication:
  - https://www.ti.com.cn/document-viewer/cn/DS90UB954-Q1/datasheet/GUID-8829528F-8C7B-45D0-A0CB-B42CDE739968.html
  - https://e2e.ti.com/support/interface-group/interface/f/interface-forum/969917/ds90ub954-q1-replicate-mode-on-port1
  - https://atta.szlcsc.com/upload/public/pdf/source/20220810/75AA1D3B85D263291249875EF82149C5.pdf
- **S33** — Capacitor-coupled PHY-to-PHY Ethernet:
  - https://microchip.com/en-us/application-notes/micrelan120capacitivecouplinge
  - https://www.intel.com/content/dam/doc/application-note/8255x-fast-ethernet-controllers-without-magnetics-appl-note.pdf
  - https://wiki.analog.com/resources/eval/user-guides/adin1300-and-adin1200/cap-coupling
  - https://e2e.ti.com/support/interface-group/interface/f/interface-forum/612659/dp83848i-transformerless-connection
  - https://e2e.ti.com/support/interface-group/interface/f/interface-forum/1430972/dp83826e-transformer-less-application
  - https://e2e.ti.com/support/interface-group/interface/f/interface-forum/1603498/dp83822i-conencting-two-dp83822i-to-eachother
- **S34** — KSZ8863 (switch, rejected): https://www.utmel.com/components/ksz8863rll-ic-ethernet-sw-3port-48-lqfp-datasheet-pinout-and-features?id=1428
- **S35** — USB-Ethernet chips: AX88772C https://asix.com.tw/en/product/USBEthernet/High-Speed_USB_Ethernet/AX88772C ; RTL8152B https://www.lcsc.com/product-detail/C500393.html and https://www.armdesigner.com/download/RTL8152B-VB-CG.pdf
- **S36** — USB-C sink CC resistors: https://people.kernel.org/bleung/how-to-design-a-proper-usb-c-power-sink-hint-not-the-way-raspberry-pi-4
- **S37** — TPD4E05U06: https://www.ti.com/product/TPD4E05U06
- **S38** — USB gadget NCM/RNDIS:
  - https://www.bktechnologies.com/service-portal/assets/images/BKSB-1099.pdf
  - https://docs.kernel.org/6.12/usb/gadget_configfs.html
  - https://www.neowin.net/news/linux-removing-an-outdated-insecure-microsoft-usb-network-protocol-thats-still-on-windows/
  - https://postmarketos.org/edge/2023/10/29/rndis-ncm

**Reference designs and tools**
- **S39** — OpenIPC open hardware: https://github.com/OpenIPC/hardware (`board_hi3516ev200`, BL-R8812AF1 carrier projects) ; editable EV200 source https://oshwhub.com/suikunlin/openipc-hi3516
- **S40** — OLIMEX S3-OLinuXino (open KiCad IP-camera board): https://github.com/OLIMEX/S3-OLinuXino
- **S41** — KiCad EasyEDA import: https://www.kicad.org/blog/2024/02/Version-8.0.0-Released/ ; https://www.kicad.org/blog/2026/01/KiCad-9.0.7-Release/
- **S42** — JLCPCB 6-layer capabilities: https://jlcpcb.com/6-layer-pcb ; https://jlcpcb.com/resources/6-layer-pcbs
- **S43** — JLCPCB assembly pricing: https://jlcpcb.com/help/article/pcb-assembly-price ; https://jlcpcb.com/help/article/pcb-assembly-faqs

**Added in this revision**
- **S44** — OpenIPC Mario AIO (SSC338Q; USB debug port appears as a USB-to-Fast-Ethernet adapter): https://github.com/OpenIPC/docs/blob/main/src/content/docs/hardware/OpenIPC/VTX/fpv-mario-aio.md
- **S45** — OpenIPC FPV advanced setup guide (USB-Ethernet dongle on the PC; Companion "onboard recording" toggle): https://github.com/OpenIPC/docs/blob/main/src/content/docs/use-cases/fpv/wfb-ng/advanced-setup.mdx
- **S46** — Majestic streamer: recording, power cuts, `majestic --repair`, `/metrics/records`: https://github.com/OpenIPC/wiki/blob/master/en/majestic-streamer.md
- **S47** — SD card diagnostics (fake-card check): https://github.com/OpenIPC/wiki/blob/master/en/sd-card-diagnostics.md
- **S48** — Timestamps and time sync (supported SoCs): https://github.com/OpenIPC/wiki/blob/master/en/timestamps-and-time-sync.md
- **S49** — Radxa Zero 3W ground station (DVR to `.ts`): https://github.com/OpenIPC/docs/blob/main/src/content/docs/use-cases/fpv/wfb-ng/groundstation-radxa-zero-3w.md
- **S50** — OpenIPC SigmaStar infinity6e kernel config (`CONFIG_USB_GADGET` and `CONFIG_USB_OTG` not set): https://github.com/OpenIPC/firmware/blob/master/br-ext-chip-sigmastar/board/infinity6e/infinity6e-ssc012b.config
- **S51** — OpenIPC wifibroadcast-ng air-side script (`wfb_tx -U rtp_local`, `iw reg set 00`, `wfb_tun` 10.5.0.10/24): https://github.com/OpenIPC/firmware/blob/master/general/package/wifibroadcast-ng/files/wifibroadcast
- **S52** — OpenIPC defib PR #117 (CV6xx boot images for 00g/00s/10b/20g/20s; merged 2026-07-28; no hardware test stated): https://github.com/OpenIPC/defib/pull/117
- **S53** — TI TPS2121 priority power mux, read via search snippets: https://www.ti.com/product/TPS2121 ; https://www.mouser.in/new/texas-instruments/ti-tps2120-tps2121-multiplexers/
- **S54** — USB 2.0 unit load before configuration (spec §7.2.1 as quoted on LKML): https://lkml.iu.edu/hypermail/linux/kernel/0606.0/0080.html ; https://lkml.iu.edu/hypermail/linux/kernel/0606.0/0203.html
- **S55** — RNDIS status in mainline Linux:
  - https://github.com/torvalds/linux/blob/master/drivers/net/usb/Kconfig
  - https://github.com/torvalds/linux/blob/master/drivers/usb/gadget/Kconfig
  - https://www.phoronix.com/news/RNDIS-Disable-Linux-2026-Patch

*Access note: during research only github.com (and its raw file host) could be fetched directly. HiSilicon, JLCPCB, CSDN, TI, Phoronix and Scribd facts come from search-engine snippets, so they are marked [L] or [U].*