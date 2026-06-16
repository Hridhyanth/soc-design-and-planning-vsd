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

## Day 2 — Floorplanning and Introduction to Library Cells

#### Chip Floorplanning — Core Area and Utilisation

Floorplanning is essentially the process of deciding how the chip's real estate gets allocated before anything is placed in detail. Two numbers anchor this decision:

- **Utilisation Factor** = **area taken up by the netlist** ÷ **total core area**. Most designs aim for somewhere around 0.5–0.6, leaving enough breathing room for buffers, routing channels, and later optimization.
- **Aspect Ratio** = **core height** ÷ **core width**. A value of 1 gives a perfect square; anything else stretches the core into a rectangle.

#### Pre-Placed Cells and Decoupling Capacitors

Certain blocks — memories, PLLs, and other complex IP — get locked into position manually before the automated placement tool ever runs. Their location is chosen by hand based on how they connect to the rest of the design and how power needs to reach them.

Around these pre-placed blocks, designers add decoupling capacitors, which act like small local reservoirs of charge. They smooth out the voltage dips caused by nearby switching activity, keeping the power supply to these sensitive blocks clean and stable.

#### Power Planning: Rings and Mesh

A solid power distribution strategy combines two structures: power rings running around the core's perimeter, and a power mesh spreading across the whole chip. By running multiple VDD and VSS rails across several metal layers, every standard cell ends up close to a power tap — which keeps IR drop and electromigration risk under control.

#### Pin Placement and I/O Blockage

I/O pins are arranged along the chip's outer boundary, and where exactly they sit isn't random — pins tied to logic buried deep in the core get positioned closer to that logic to keep wire lengths short. The strip of space between the core and the die edge (the I/O ring area) is deliberately walled off from automatic placement, reserved instead for things like pin buffers and ESD protection cells.

### Lab — Floorplan and Placement

#### Running Floorplan

```tcl
run_floorplan
```

After this completes, we can inspect the DEF file that was generated :

```bash
cd results/floorplan/
less picorv32a.def
```

#### Viewing the Floorplan in Magic

```bash
magic -T /home/vsduser/Desktop/OpenLane/designs/picorv32a/sky130A/libs.tech/magic/sky130A.tech \
      lef read ../../tmp/merged.nom.lef \
      def read picorv32a.def &
```

#### Running Placement

```tcl
run_placement
```

Standard cells are legally placed .

## Day 3 — Design and Characterisation of Library Cells using Magic & ngspice

#### Characterising a Standard Cell with SPICE

To properly characterise any standard cell, you start by writing a SPICE netlist that captures its PMOS and NMOS transistors, their width-to-length ratios, the supply voltage, the input waveform driving the cell, and the capacitive load it's expected to drive.

#### From the resulting simulation, three numbers matter most:

- Rise  time — how long the output takes to climb from 20% to 80% of its swing
- Fall time — how long it takes to drop from 80% down to 20%
- Propagation delay — measured from the 50% point of the input to the 50% point of the output


#### A Quick Look at the 16-Mask CMOS Process

Fabricating a chip on silicon follows roughly sixteen distinct masking steps, starting with substrate prep and ending in passivation:

