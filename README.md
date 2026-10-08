# Thinker338Q Hardware

Hardware files for the SSC338Q-based OpenIPC Thinker FPV air unit.

This repository contains hardware documentation, PCB fabrication outputs, and a mechanical model. Firmware, software, build tools, and device configuration are outside its scope.

## Hardware overview

The [official OpenIPC Thinker documentation](https://docs.openipc.org/hardware/openipc/vtx/fpv-thinker-aio/) describes a compact air unit with an SSC338Q processor, 2-6S power input, a 3 A BEC, an analog MEMS microphone, and MIPI camera, UART, USB, and Ethernet interfaces. The documented PCB measures 25 x 25 mm, with a 20 x 20 mm mounting pattern.

The product family includes a Base edition with an SD card slot and external Wi-Fi, and a Tiny edition with integrated Wi-Fi and no SD card slot. These are product-family specifications; this file collection does not establish coverage of both editions. Check the applicable board revision before using the fabrication files or mechanical model.

## Repository contents

| Path | Contents |
| --- | --- |
| [schematic/338THNIKERV1.pdf](schematic/338THNIKERV1.pdf) | Schematic PDF |
| [PCB/top.pdf](PCB/top.pdf) | Top-side PCB PDF |
| [PCB/bottom.pdf](PCB/bottom.pdf) | Bottom-side PCB PDF |
| [PCB/thinker338_gerber.zip](PCB/thinker338_gerber.zip) | PCB fabrication archive with Gerber and drill files |
| [3D/ThinkerV1SSC338.stp](3D/ThinkerV1SSC338.stp) | STEP mechanical model |
| [docs/dual-cv610-feasibility.md](docs/dual-cv610-feasibility.md) | Feasibility and architecture brief for a dual Hi3516CV610 redesign |

No editable schematic or PCB CAD project, standalone bill of materials, or pick-and-place file is included in this collection.

## Using the files

- Open the schematic and PCB views in a PDF viewer.
- Load the Gerber archive in a Gerber viewer or a PCB manufacturer's preview tool. Confirm fabrication requirements and board revision before ordering.
- Import the STEP file into a compatible mechanical CAD tool for enclosure and integration work.

For connector pinouts, wiring, power precautions, cooling, and camera options, consult the [official hardware installation guide](https://docs.openipc.org/hardware/openipc/vtx/fpv-thinker-aio/#hardware-installation). Firmware installation and configuration instructions are maintained in the same upstream documentation.
