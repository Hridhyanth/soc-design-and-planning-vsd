# soc-design-and-planning-vsd
Code to Chip: An open-source RTL-to-GDSII journey with OpenLANE &amp; Sky130 PDK | VSD SoC Design and Planning Workshop
# Digital VLSI SoC Design and Planning — RTL to GDSII

> Over two weeks, this workshop — organised by VSD (VLSI System Design) and NASSCOM — walked me through the entire RTL-to-GDSII flow
> for digital VLSI SoC design.
> This repo captures my daily lab work,
> outputs, and the most important things I learned.  

## Day 1 — Inception of Open-Source EDA, OpenLANE & Sky130 PDK

#### Understanding the Chip Package

What most people call "the chip" on a circuit board is actually its package — a protective shell wrapped around the real silicon underneath. The actual die sits inside, and it talks to the outside world through wire bonding: a set of extremely fine wires linking the die's pads to the metal pins of the package.

#### Inside the Chip: Core, Pads, and Die

Strip away the package and look at the silicon itself: every signal entering or leaving the chip passes through pads arranged around its edge. The space inside those pads — where the actual logic lives and does its work — is called the core. Core plus pads together make up the die, which is the basic physical unit that comes off a fabrication line.
- **Foundry** —  fabrication facility that actually manufactures the silicon.
- **Foundry IPs** — that demand deep, process-specific know-how to design correctly (e.g., PLLs, SRAMs)
- **Macros** — are reusable digital logic blocks that don't carry that same process-level complexity.

#### From Software to Silicon — The ISA Bridge

From Code to Hardware — Bridging Software and Silicon

A C program doesn't run on silicon directly — it passes through several layers of translation first:

1. A compiler turns the C source into assembly instructions targeting a specific instruction set, such as RISC-V.
2. An assembler converts that assembly into raw machine code: the actual 0s and 1s the hardware understands.
3. For that machine code to mean anything, the processor itself must have an RTL implementation of the instruction set baked into its logic.
4. That RTL is then synthesized and carried through the full place-and-route flow until it becomes an actual physical layout.

In short, the operating system, compiler, and assembler form the bridge connecting what a programmer writes to what the hardware ultimately executes.

#### Why Open-Source EDA Matters

- A fully open-source ASIC design flow needs three pieces in place: openly available RTL source (drawn from places like opencores.org), a complete EDA toolchain covering synthesis, place-and-route, and verification, and PDK data — the process-specific design rules and standard cell libraries tied to a given fabrication node.

- For most of the industry's history, PDKs sat behind NDAs and proprietary licensing, putting real chip design out of reach for students and hobbyists. That changed in June 2020, when Google teamed up with SkyWater Technology to release Sky130 as the first fully open-source PDK — a genuine turning point for accessible VLSI design.

Historically, PDKs were proprietary and distributed only under NDAs, making chip design inaccessible to most people. This changed in **June 2020**, when Google collaborated with SkyWater Technology to release the **Sky130 PDK** as the world's first open-source process design kit — a massive milestone for the VLSI community.

#### OpenLANE and the Automated RTL to GDSII Flow

**OpenLANE** is an open-source flow built on top of multiple EDA tools that automates the journey from an RTL netlist all the way to the final GDSII layout file. It uses:

| Stage | Tool(s) Used |
|---|---|
| Synthesis | Yosys, ABC |
| Floorplan & PDN | OpenROAD |
| Placement | OpenROAD |
| CTS | TritonCTS |
| Routing | FastRoute, TritonRoute |
| SPEF Extraction | OpenRCX |
| GDS Streaming | Magic, KLayout |
| Timing Analysis | OpenSTA |
| DRC & LVS | Magic, Netgen |

### Lab — Running OpenLANE for `picorv32a`


#### Setting Up and Invoking OpenLANE

The very first step is to navigate to the OpenLANE working directory and launch the tool in **interactive mode**, which lets us run each stage step-by-step.

- **Step 1** :
  
```bash
cd /home/vscode/Desktop/OpenLane
make mount
./flow.tcl -interactive
package require openlane 1.0.2
```

- **Step 2** :
  
#### Preparing the Design

Before running synthesis, we prepare the design to merge the cell LEF and technology LEF files, and set up the run directory.

```tcl
prep -design picorv32a
```

- **Step 3** :
  
#### Running Synthesis

```tcl
run_synthesis
```

- **Step 4** :
  
After synthesis completes, we can calculate the **flop ratio** — a useful sanity check:

```
Flop Ratio = (No. of D Flip-Flops) / (Total No. of Cells)
           = 1613 / 15762
           ≈ 0.1023  →  ~10.23%
```

