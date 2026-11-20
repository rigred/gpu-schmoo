# gpu-schmoo

### Fake NVIDIA GPUs, counterfeit VBIOS identification, and recovering usable graphics cards

**gpu-schmoo** is a reverse-engineering and repair project documenting counterfeit, relabelled, and incorrectly configured graphics cards—especially older NVIDIA GPUs sold as newer GeForce models. The aim is to identify the *real* silicon and physical memory, explain how the video BIOS misrepresents or misconfigures the board, and recover a working configuration where the hardware permits it.

A fake GTX 960 cannot be turned into a genuine GTX 960 by flashing firmware. But the real GPU underneath may be salvageable as a correctly identified, correctly configured older card.

> **Read this before flashing:** VBIOS modifications can prevent a card from initializing, leave the display black, or make it impossible to recover without an external SPI programmer. Always preserve an untouched backup and validate the hardware against the ROM. The examples here are reverse-engineering evidence, not universal flash recipes.

## Contents

- [Why fake graphics cards exist](#why-fake-graphics-cards-exist)
- [The original fake GTX 960 investigation](#the-original-fake-gtx-960-investigation)
- [What the VBIOS actually controls](#what-the-vbios-actually-controls)
- [Identifying the real hardware](#identifying-the-real-hardware)
- [VRAM capacity and memory geometry](#vram-capacity-and-memory-geometry)
- [Using NVIDIA VBIOS Workbench](#using-nvidia-vbios-workbench)
- [GF116 fake-capacity repair research](#gf116-fake-capacity-repair-research)
- [Choosing or building a repaired BIOS](#choosing-or-building-a-repaired-bios)
- [Flashing and validation](#flashing-and-validation)
- [Beyond fake cards: reconfigured and damaged boards](#beyond-fake-cards-reconfigured-and-damaged-boards)
- [Limitations and research status](#limitations-and-research-status)
- [References](#references)

## Why fake graphics cards exist

Many inexpensive "GTX 960", "GTX 1050 Ti", and similarly branded cards on online marketplaces are not what their labels claim. Common examples repurpose genuine, much older GPUs on refurbished or newly assembled PCBs. Their firmware may contain altered PCI identifiers, model strings, configuration tables, or claimed VRAM capacities. The heatsink and retail listing can complete the illusion.

There are several distinct problems, and they require different fixes:

| Problem | Typical evidence | What may fix it |
|---|---|---|
| Spoofed GPU model | Name or PCI identity conflicts with GPU architecture and decoded firmware | Restore the appropriate identity and compatible board firmware |
| Inflated VRAM capacity | Software reports more VRAM than the populated DRAM can provide | Correct controller geometry/capacity configuration, then test for aliasing |
| Incorrect timings or memory type | Memory initialization failures, corruption, instability | Restore compatible RAMCFG/timing records for the installed DRAM |
| Different or depopulated PCB | Missing DRAM chips, changed memory bus population | Match initialization to actual wiring and available controller partitions |
| Defective memory or board | Repeatable memory errors, crashes, artifacts | Hardware repair; sometimes a *validated* reduction of active resources |
| Entirely different hardware | Different GPU generation, absent units, missing display circuitry | Cannot be transformed into the advertised product by VBIOS editing |

A modified PCI ID or marketing string is not proof of a working conversion. The GPU's actual shader units, memory controllers, internal architecture, and physical wiring remain determined by silicon and the PCB.

## The original fake GTX 960 investigation

The investigation began with coverage of suspicious low-cost cards, including a **PhilsComputerLab fake GTX 960 4 GB** video:

**[Watch: Fake GTX 960 graphics card eBay scam](https://www.youtube.com/watch?v=Ved84d_6occ)**

The original card investigated in this repository was marketed as a GTX 960, but the GPU package was identified as **GF116-200-KA-A1**, from NVIDIA's older **Fermi** family—not the **GM206 Maxwell** GPU used in a genuine GTX 960.

The visible DRAM population also contradicted the advertised capacity. Four **2-Gbit** memory devices provide, in total:

```text
4 chips × 2 Gbit/chip = 8 Gbit
8 Gbit ÷ 8 = 1 GiB = 1,024 MiB
```

This assumes those four devices are the complete population and their part markings correctly identify their density. **Gbit is not GB.** A chip advertised as 2 Gbit provides 256 MiB, not 2 GB.

The fascinating part is that the system could still report the counterfeit GTX 960 identity. That is possible because different diagnostic fields come from different places: some from software-visible identity or VBIOS records, and others from hardware architecture and controller registers. Cross-checking them exposes inconsistencies that a simple model-name lookup misses.

A repair should recover the GF116 card as whatever actual board configuration it supports—not attempt to unlock Maxwell capabilities that do not exist in Fermi silicon.

## What the VBIOS actually controls

An NVIDIA video BIOS is more than a name string in a flash chip. Depending on GPU generation and board design, its contents include:

- PCI/option-ROM headers, legacy image structures, and sometimes UEFI GOP images.
- NVIDIA BIT/BMP directories and firmware initialization scripts.
- PCI identification metadata and, on supported older designs, device-selection straps.
- Memory-configuration selection, initialization register writes, timings, and operating frequencies.
- Clock/performance states, voltage references, and recognized power-limit records.
- Display connector routing, DCB data, GPIO assignments, and fan/thermal configuration.
- Board-specific setup that cannot safely be transferred just because two cards share a GPU name.

Many values are repeated in different structures. Changing a printable device name, PCI option-ROM header, or a memory-size field in isolation is **not** equivalent to updating the complete hardware initialization.

Likewise, the locations of fields change between VBIOS generations and table versions. Historic hex offsets in [`vbios_structure_notes.md`](vbios_structure_notes.md) are **observations from individual ROMs**, not reliable offsets for every NVIDIA VBIOS. Use a parser and a ROM-specific byte diff rather than blindly changing a byte copied from a forum post.

## Identifying the real hardware

**Do this before editing anything.** The goal is to separate *what the ROM claims* from *what is physically fitted*.

1. **Photograph the card.** Record front and back, board revision and component population. If removing the cooler, use appropriate ESD and thermal-handling precautions.
2. **Read the GPU package marking.** Record the exact die/package designation; compare it with architecture documentation.
3. **Read every DRAM marking.** Look up manufacturer part numbers and calculate per-chip density, organization, data width, and installed total capacity.
4. **Identify the PCB.** Note memory footprints, missing/unused chip positions, power circuitry, display connectors, and SPI flash part/capacity.
5. **Capture the live device identity and diagnostic information.** Record PCI vendor/device/subsystem IDs, GPU revision, reported memory capacity and width, and accessible memory-controller topology.
6. **Dump the original VBIOS—preferably more than once.** Compare dump hashes and retain the untouched originals off-card.
7. **Compare against possible donor ROMs.** Match GPU generation, board circuitry, memory IC family, bus population, voltage controller, display wiring, and flash layout. A matching marketed model is not enough.

On Linux, useful read-only starting points include:

```bash
lspci -nn -vv
lspci -nnk
sha256sum original-vbios.rom
```

Use a tool appropriate to the card and platform to acquire the ROM; `lspci` does **not** dump VBIOS flash. A board may also expose an option-ROM copy through sysfs, but that should not be assumed to be a complete physical SPI-flash backup. For important recoveries, verify against an external programmer where practical.

### Capacity vs bus width: two different questions

**DRAM capacity** depends on chip density and chip count. **Memory bus width** depends on how the chips are connected to controller channels and on the width of each device. Four 2-Gbit chips give 1 GiB *in aggregate*; that fact by itself does **not** prove a 128-bit or 192-bit bus.

Also check whether the memory controller is configured to expose the physical capacity without mirroring or wrapping addresses. A simple device-properties display is not an adequate memory test.

## VRAM capacity and memory geometry

A DRAM address is interpreted through a combination of memory-controller settings and the organization of the actual chips. Depending on generation and the available firmware documentation, relevant parameters include:

| Parameter | Why it matters |
|---|---|
| Device density | Physical bits of storage per DRAM package |
| Device I/O width | Number of data lines per package and channel population |
| Row/column geometry | How address bits select locations within DRAM |
| Banks / bank groups | Internal DRAM organization; interpretation depends on memory generation |
| Rank / chip select | Selection of independently populated device groups, where implemented |
| Channels / partitions | Which parts of the GPU memory interface and PCB are active |
| Timing / training | Whether the controller can access the devices reliably at a given speed |
| RAMCFG strap selection | Which table entry and initialization profile firmware chooses |

These are **not interchangeable**. Setting a reported capacity to 1 GiB does not configure the memory chips to have the right row and column organization, and copying timings alone will not repair a mismatched controller geometry.

### Example: four versus six 2-Gbit devices

| Physical DRAM population | Aggregate physical capacity |
|---|---:|
| 4 × 2 Gbit | 1,024 MiB |
| 6 × 2 Gbit | 1,536 MiB |

Both arrangements still need the correct wiring-specific memory initialization, not merely a new number in an information table.

## Using NVIDIA VBIOS Workbench

The companion **NVIDIA VBIOS Workbench 1.2** is an offline browser-based editor for NVIDIA PCI option ROMs. The workbench archive is distributed separately; it is **not currently part of this repository's published source tree**. Obtain a copy of the workbench before following these steps.

To launch an extracted copy:

```bash
cd NVIDIA_VBIOS_Workbench
python3 app.py
```

Python 3.10+ is required. The local interface runs on loopback without a cloud service. Its staged edits remain in memory until you export a ROM. The supplied package includes an `nvbios` build for Linux x86-64, plus source material for building it on compatible systems.

The important workbench areas for fake-card investigation are:

| Workbench area | Useful functionality |
|---|---|
| **Board & topology** | Inspect ROM image layout, BIT firmware information and board identity |
| **PCI identity** | Inspect/edit supported identity fields and documented legacy strap mappings |
| **Memory geometry** | Inspect Fermi RAM initialization, RAMCFG routing, guarded controller operands, and supported partition masks |
| **Memory and timings** | Edit recognized timing records and selected existing RAMCFG information |
| **Clocks & P-states** | Inspect/edit supported clock domains, profiles and compatible donor records |
| **SM configuration** | Read supported topology data and inspect a limited set of SM/partition-related initialization controls |
| **Review & export** | Audit byte-level changes, restore/undo edits, perform supported structural/checksum repairs and export the staged image |

**Important limits:** This is a research editor, not a universal NVIDIA factory configurator. Decoded fields and writable structures are generation- and ROM-dependent. Row, column, bank, rank, lane and fuse controls are **not** universally decoded or safely editable. Nor can the tool infer an arbitrary working DRAM controller word from just a chip part number or desired capacity.

For background on the firmware formats, see [envytools](https://github.com/envytools/envytools) and [Nouveau](https://nouveau.freedesktop.org/).

## GF116 fake-capacity repair research

The Workbench 1.2 documentation describes a **specific guarded repair** for a family of counterfeit or misconfigured GF116 ROMs with **2-Gbit DRAM devices**. It is more precise than simply editing the reported memory size.

### The observed register pattern

The relevant Fermi boot initialization writes target memory-controller registers:

```text
0x10F204
0x110204
```

For the matching archived ROM layouts, the repair sets **bits 23:16 to `0x55`** in the eight applicable operands for each register while preserving other bits. These are *controller configuration words*, not generic PCI-device identification fields.

**Do not apply `0x55` to arbitrary NVIDIA boards.** The interpretation and applicability require the matching GF116 firmware layout, RAMCFG selection, fitted memory population, and actual board routing. A file named "4GB" is not evidence that 4 GiB is physically fitted.

### Reported ROM comparisons

The projected is based on the following archived comparisons:

| Input ROM | Compared reference | Reported layout-byte differences |
|---|---|---:|
| `original.bin` (192-bit layout) | `550Ti_Speshul_unlock_patch3.rom` | 14 |
| `fake_gtx960_6x_2g_HynixAFR_GTS450.bin` | `fake_gtx960_6x_2g_HynixAFR_GTS450_patched.bin` | 1 |
| `phils_fake_960.rom` | `216372.rom` | 16 |
| `gt440_super.rom` (128-bit layout) | `GF666.rom` | 16 |

**What this establishes:** The compared memory-controller operands in those archived images match the proposed corrected geometry pattern. **What it does not establish:** That every reference ROM is a safe full-image donor, or that these changes have passed real-board POST and memory-stress testing. References can differ separately in their identities, display tables, clocks, and other board initialization.

### Applying the guarded Workbench edit

1. Open your own untouched VBIOS image.
2. Go to **Memory geometry → Decode memory geometry**. Review the silicon identified from BIT firmware, the PCIR identity, RAMCFG translation, and the decoded register-write groups. Spoofed headers may not agree with the underlying firmware.
3. Confirm the chip markings and board population really correspond to the supported 2-Gbit GF116 case.
4. If the ROM and its guarded writes match the supported layout, select **Stage paired geometry repair**.
5. Inspect all staged writes and the byte-level diff; ensure no unrelated identity, timing, voltage, or display edits were silently introduced.
6. Export a separate candidate ROM, leaving the source unchanged. The workbench also repairs the recognized legacy checksum for the supported patch.
7. Validate on the actual card, first with a recoverable boot path, then with full-range VRAM read/write testing.

The Workbench reports ROM-parser and paired-reference comparisons for this work. **No physical GPU was available for the packaged verification**, so the repair should be treated as a well-defined candidate patch until hardware tests confirm it.

### What if my card is different?

Do **not** force the above repair. Instead, collect the original ROM, memory markings, PCB photos, a controller topology capture, and a close donor ROM. Workbench can inspect additional RAM-grouped init words, edit a supported masked CFG1 byte, and transplant certain same-silicon/same-register geometry records when the layouts match. Such transfers still require timing, training, voltage, topology and hardware validation.

## Choosing or building a repaired BIOS

There are three broad approaches, listed in preferred order when applicable:

**1. Restore a verified original board ROM.** Best when the exact PCB revision, GPU, memory chips, power controller, and connectors match. This avoids inventing an untested initialization sequence.

**2. Patch the existing ROM conservatively.** Often safer when the card is an unusual Chinese PCB with no exact reference board. Preserve its working display, voltage and power initialization while correcting only verified identity, memory geometry, and timing defects.

**3. Selectively import from a compatible donor.** Use the Workbench's supported donor controls where appropriate—for example, recognized same-family clock/power-state or timing structures, or a single matched geometry word. Never assume wholesale table imports are compatible merely because the memory capacity is similar.

Check the chip's SPI ROM capacity and legacy/EFI image chain, too. Checksums are necessary for supported ROM formats but do **not** make a donor compatible or bypass firmware signatures.

### Decision guide

```text
Suspicious GPU
  |
  +-- Verify actual GPU package and memory ICs
  |      |
  |      +-- Fake branding only? --> restore honest compatible identity
  |      |
  |      +-- Wrong installed capacity/geometry?
  |      |       --> identify RAMCFG + controller init + board population
  |      |       --> apply verified matched correction
  |      |
  |      +-- Timing/configuration mismatch?
  |      |       --> compare with chip-compatible initialization
  |      |
  |      +-- Physical defect? --> diagnose hardware; no generic ROM fix
  |      |
  |      +-- Different silicon? --> cannot obtain missing hardware features
  |
  +-- Save original --> export candidate --> recoverable flash
                                    --> verify every address region
```

## Flashing and validation

**Flash only after the candidate ROM passes static review and a recovery plan is in place.** The appropriate flashing method varies by generation, ROM protection, board layout, and whether the card can still initialize. Some unsupported or heavily spoofed cards are safer to program off-board with a suitable external SPI programmer than through software flashing.

Minimum checklist:

- [ ] Original flash image saved, hashed, and verified against a second read.
- [ ] Real GPU silicon, DRAM parts and PCB wiring documented.
- [ ] Candidate ROM verified against the actual flash size and board electrical configuration.
- [ ] Differences between original and candidate ROM reviewed and explained.
- [ ] A second working GPU, blind-flash procedure, or external programmer available for recovery.
- [ ] Cold POST and all display outputs needed for operation tested.
- [ ] PCI identity and GPU architecture checked after boot.
- [ ] VRAM capacity and bus width checked against physical hardware.
- [ ] **VRAM reads/writes exercised across the entire claimed space**, with patterns intended to detect aliasing and corruption, including near the highest addresses.
- [ ] Memory tests repeated warm, after a cold boot, and under sustained load; clocks, voltages, thermals and power stability checked.

A card that boots and displays a plausible VRAM number is **not yet fixed**. Memory may wrap around and overwrite earlier addresses, or only fail when temperature or load rises. Failure to install an NVIDIA driver can also be caused by unsupported/altered PCI identity or an incompatible driver generation rather than by ROM checksums alone.

## Beyond fake cards: reconfigured and damaged boards

The same techniques have legitimate applications in board repair and hardware experimentation:

- Replace DRAM packages with compatible parts of a different density or organization **when the GPU controller and PCB support that configuration**.
- Study different DRAM populations and the RAMCFG entries selected by straps.
- Compare equivalent timings and clock/power states between known-good board ROMs.
- Investigate whether a faulty memory partition or controller can be excluded on a particular GPU, accepting reduced capacity and/or bandwidth.
- Determine whether an apparently locked hardware unit is only software-disabled or is blocked by actual silicon fuses.

Workbench 1.2 provides guarded Fermi partition/controller mask operations involving `0x0225C8` and `0x0225D4` for compatible ROMs. **This is not a universal "disable bad VRAM chip" function.** The physical-to-logical partition mapping needs live board evidence, and the effective configuration can be constrained by factory fuses and later initialization. Chip-level, lane-level and rank-level repairs are not demonstrated by those mask edits.

In the supplied Workbench, Linux topology capture can be run separately from the browser UI:

```bash
python3 topology.py
sudo python3 topology.py 0000:01:00.0 > nvidia-topology.json
```

Substitute the card's actual PCI address. The helper performs read-only register collection for supported hardware. Review the helper before running with elevated privileges. A register-derived configured capacity is not proof that all addresses correspond to distinct working physical memory cells.

## Limitations and research status

**Original repository:** Observational reverse engineering of fake NVIDIA boards, with old byte-offset notes and an unfinished repair guide.

**Workbench 1.2 package reviewed for this update:** Offline ROM editing with structured decoding, staged geometry changes, donor matching, diff review, checksum/structure checks, and selected live topology inspection. The package documents successful parser and reference-ROM comparisons, **not** universal on-card validation.

| Area | Status |
|---|---|
| Distinguish fake model identity from underlying silicon | Practical diagnostic workflow |
| Calculate physical capacity from readable DRAM markings | Established arithmetic, subject to identifying the full population |
| Decode supported Fermi RAMCFG and memory init writes | Implemented for recognized formats |
| Paired GF116 2-Gbit geometry repair | Supported as a guarded ROM patch; hardware validation needed |
| Copy certain same-silicon memory geometry records | Conditional, matched-layout operation |
| Edit recognized performance states and timing data | Version- and format-dependent |
| Disable selected Fermi controller/partition resources | Experimental board-specific operation |
| Arbitrary row/column/bank/rank geometry synthesis | **Not established** |
| Universal factory-level fuse or SM configuration editing | **Not available** |
| Make old Fermi silicon operate as Maxwell/Pascal | **Impossible** |

### Contributing useful evidence

Issues and pull requests are most helpful when they include the original ROM hash, the **real** GPU package marking, DRAM part markings, both PCB sides, reported and tested capacity, PCI identifiers, the exact edits made, and before/after boot and full-memory test outcomes. Do not upload ROMs or board identifiers you do not have permission to share.

Prefer reproducible cases over universal recipes. A failed experiment with a clean backup and full byte diff can be as valuable as a successful one.

## References

- [Original gpu-schmoo repository](https://github.com/rigred/gpu-schmoo)
- [Original VBIOS structure observations](vbios_structure_notes.md)
- [PhilsComputerLab — Fake GTX 960 graphics card eBay scam](https://www.youtube.com/watch?v=Ved84d_6occ)
- [envytools / nvbios](https://github.com/envytools/envytools)
- [Nouveau open-source NVIDIA driver research](https://nouveau.freedesktop.org/)
- [Nouveau RAMCFG code](https://linux.googlesource.com/linux/kernel/git/tip/tip/+/refs/heads/perf/urgent/drivers/gpu/drm/nouveau/nvkm/subdev/bios/ramcfg.c)
- [Linux Nouveau Fermi FBPA probing](https://codebrowser.dev/linux/linux/drivers/gpu/drm/nouveau/nvkm/subdev/fb/ramgf100.c.html)

---

**Project principle:** Recover the hardware that physically exists. Don't trust the sticker, the device name, or a VRAM figure until the silicon, ROM and real memory all agree.
