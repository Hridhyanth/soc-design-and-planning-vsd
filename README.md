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

1. Choosing the starting substrate (typically lightly-doped p-type)
2. Defining active regions through field oxidation and a nitride mask
3. Forming the n-well and p-well via ion implantation
4. Growing the gate oxide
5. Depositing the polysilicon gate
6. Implanting source/drain regions (including LDD and halo implants)
7. Adding contacts and metal interconnect layers
8. Applying the final passivation layer

### Lab — Cloning and Characterising a Custom Inverter Cell

#### Cloning the Standard Cell Repository

```bash
git clone https://github.com/nickson-jose/vsdstdcelldesign.git
```

```bash
magic -T sky130A.tech sky130_inv.mag &
```

#### Extracting SPICE Netlist from Magic

Inside the tkcon console:

```tcl
extract all
ext2spice cthresh 0 rthresh 0
ext2spice
```

#### Running ngspice Simulation

```bash
ngspice sky130_inv.spice
```

```ngspice
plot y vs time a
```

From the waveform, measure rise time, fall time, and propagation delay values.

##### Rise transition time calculation

**Rise transition time** = **Time taken for output to rise to 80%** - **Time taken for output to rise to 20%**

20% of output = 660 mV

80% of output = 2.64 V

##### Fall transition time calculation

**Fall transition time** = **Time taken for output to fall to 20%** - **Time taken for output to fall to 80%**

20% of output = 660 mV

80% of output = 2.64 V

## Day 4 — Pre-Layout Timing Analysis and Clock Tree Synthesis

#### LEF Requirements for Custom Standard Cells

Before OpenLANE can use a custom cell, it needs a LEF file spelling out the cell's physical outline, where its pins sit, and which metal layers it occupies. Two rules govern valid port placement:

- Every input and output port has to sit exactly where a horizontal and a vertical routing track cross
- The cell's width must be an odd multiple of the horizontal track pitch, and its height an odd multiple of the vertical track pitch

#### Core Concepts in Static Timing Analysis

Setup slack is calculated as the data-required time minus the data-arrival time, and it needs to stay at or above zero for the design to meet timing. A few sources of uncertainty get folded into this calculation:

- OCV (On-Chip Variation) accounts for process, voltage, and temperature variation using derating factors
- Clock uncertainty adds margin for jitter and skew along timing paths
- CRPR (Clock Reconvergence Pessimism Removal) strips out artificial pessimism that appears when the launch and capture paths share common clock buffering

#### Building the Clock Tree

Clock tree synthesis inserts a balanced network of buffers so the clock signal reaches every flop across the chip with minimal skew. Once CTS finishes, two things need rechecking: hold timing (since the new buffers add real delay) and setup timing (since the clock paths themselves have changed).

### Lab — Custom Cell Integration and STA with OpenSTA

#### Editing `config.tcl` to Include Custom Cell

```tcl
set ::env(LIB_SYNTH)      "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(LIB_FASTEST)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__fast.lib"
set ::env(LIB_SLOWEST)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__slow.lib"
set ::env(LIB_TYPICAL)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(EXTRA_LEFS)     [glob $::env(OPENLANE_ROOT)/designs/$::env(DESIGN_NAME)/src/*.lef]
```

#### Running OpenSTA (Pre-CTS Timing)
Newly created pre_sta.conf for STA analysis in openlane directory

```bash
sta pre_sta.conf
```

#### Running CTS

```tcl
run_cts
```

#### Command to run OpenROAD tool
openroad

**Reading lef file:**
read_lef /OpenLane/designs/picorv32a/runs/24-03_10-03/tmp/merged.nom.lef

**Reading def file:**
read_def /OpenLane/designs/picorv32a/runs/24-03_10-03/results/cts/picorv32a.def

Creating an OpenROAD database to work with:
write_db pico_cts.db

Loading the created database in OpenROAD:
read_db pico_cts.db

Read netlist post CTS:
read_verilog /OpenLane/designs/picorv32a/runs/24-03_10-03/results/synthesis/picorv32a.v

Read library for design:
read_liberty $::env(LIB_SYNTH_COMPLETE)

Link design and library:
link_design picorv32a

Read in the custom sdc we created:
read_sdc /OpenLane/designs/picorv32a/src/my_base.sdc

Setting all cloks as propagated clocks:
set_propagated_clock [all_clocks]

Check syntax of 'report_checks' command:
help report_checks

Exit to OpenLANE flow
exit
---

## Day 5 — Final RTL to GDSII using TritonRoute & OpenSTA

#### Routing in Two Passes

Routing happens in two stages. Global routing (handled by FastRoute) splits the chip into regions and works out an approximate path for every net, keeping layer assignments and congestion in mind. Detailed routing (handled by TritonRoute) then takes those rough guides and turns them into exact wire segments, vias, and metal tracks — all while respecting the process's DRC rules.

#### Extracting Parasitics and Final Timing Sign-off

Once routing is complete, the actual resistance and capacitance of every wire gets extracted into a SPEF (Standard Parasitic Exchange Format) file. Those parasitics are back-annotated onto the netlist, and STA is run one more time to produce the final, sign-off-quality timing numbers.

### Lab — Power Distribution, Routing

#### Generating Power Distribution Network

```tcl
gen_pdn
```
#### Running Routing

```tcl
run_routing
```

Common violations to look out for:

- *Min spacing violations* – two wires too close on the same layer
- *Antenna violations* – long metal segments accumulating charge during etch (can damage gate oxide)
  - Fix: insert antenna diodes or use jumper vias to a higher layer

-----

## Tools & Environment

| Tool | Purpose |
|---|---|
| **OpenLANE** | RTL-to-GDSII automation flow |
| **Yosys** | RTL synthesis |
| **OpenROAD** | Floorplan, Placement, CTS, Routing |
| **Magic** | Layout editor, DRC, LVS |
| **OpenSTA** | Static Timing Analysis |
| **ngspice** | SPICE simulation |
| **TritonRoute** | Detailed routing |
| **Netgen** | LVS (Layout vs Schematic) |
| **Sky130 PDK** | SkyWater 130nm open-source PDK |

---

## Key Learnings

- Open-source toolchains can genuinely replace commercial EDA flows. This workshop showed that an entirely open-source stack — OpenLANE, Magic, ngspice, OpenSTA,
and the Sky130 PDK — can carry a design all the way from RTL to a finished GDSII layout. Going in, I assumed professional chip design tools were locked behind
expensive licenses, but watching every stage run on free, community-maintained software changed how I think about access to this field.

- The die is more than just "the chip." Understanding how the package, die, core, and pads relate to each other gave me a much clearer mental model of what's
physically happening inside any embedded board I look at. Foundry IPs, macros, and the core/pad boundary stopped being abstract textbook terms and started feeling
like deliberate engineering decisions I could actually point to.

- Floorplanning decisions ripple through the entire flow. Choosing a utilisation factor or aspect ratio isn't a cosmetic step — it directly affects how much room
is left for buffers, how congested routing gets later, and whether the design closes on timing at all. Seeing pre-placed cells, decoupling capacitors, and power
mesh/ring structures laid out made it clear that many downstream problems are either prevented or created right here.

- Library cell characterization connects transistor-level behavior to digital signoff. Writing a SPICE deck, extracting a netlist from a Magic layout, and
measuring rise time, fall time, and propagation delay in ngspice showed me exactly how a "simple" standard cell's analog behavior gets distilled into the timing
numbers that STA tools depend on later. It closed a gap between transistor-level intuition and digital-level abstraction that lecture slides alone hadn't.

- Timing closure is an ongoing process, not a single checkpoint. Concepts like setup slack, OCV derating, clock uncertainty, and CRPR made it clear that "meeting
timing" has to be re-verified after every major change — especially after clock tree synthesis inserts new buffers that shift both hold and setup paths. This made
it obvious why STA gets run repeatedly throughout a real flow rather than just once at the end.

- Routing has to satisfy geometry and electrical constraints at the same time. Watching global routing hand off to detailed routing, and then seeing parasitics 
get extracted into a SPEF file for final sign-off STA, showed that a layout being "DRC clean" and a layout being "timing clean" are two separate hurdles that both
need clearing. Tracking down spacing and antenna violations was one of the most concrete, satisfying parts of the whole flow.

- Documenting the workflow taught me as much as running the labs did. Writing up each day's concepts in my own words, instead of just executing commands and
moving on, forced me to understand why each step mattered rather than just how to perform it. It surfaced gaps in my understanding that I likely wouldn't have
noticed otherwise, and it's a habit I plan to keep for future projects.

---

## Acknowledgements

A huge thank you to *Kunal Ghosh* (Co-founder, VSD Corp. Pvt. Ltd.) and *Nickson P Jose* (Physical Design Engineer, Intel) for putting together such a well-structured and genuinely practical workshop. Running a real CPU from RTL to GDSII using nothing but open-source tools is something I didn’t expect to be possible — and yet here we are.
- **Kunal Ghosh** — Co-founder, VSD (VLSI System Design)
- **Nickson Jose** — for the `vsdstdcelldesign` repository used in Day 3 labs
- **NASSCOM** — for facilitating this workshop program

---

## References

1. **OpenLANE** — https://github.com/The-OpenROAD-Project/OpenLane.
   This is the automated RTL-to-GDSII flow used throughout the workshop, tying together synthesis, 
   floorplanning, placement, clock tree synthesis, routing, and GDSII generation into a single scriptable pipeline. 

2. **Sky130 PDK** — https://github.com/google/skywater-pdk.
   This is the open-source process design kit jointly released by Google and SkyWater Technology, providing the standard cell libraries and process design rules      that every layout and simulation in this workshop was built against.

3. **Magic VLSI Layout Tool** — http://opencircuitdesign.com/magic/.
   Magic was used for viewing and editing physical layouts, extracting SPICE netlists from custom cells, and checking designs against the Sky130 design rules         during the library characterization labs.

4. **OpenSTA** — https://github.com/The-OpenROAD-Project/OpenSTA.
   OpenSTA is the static timing analysis engine used both before and after clock tree synthesis to check setup and hold timing throughout the flow.

6. **VSD (VLSI System Design)** — https://www.vlsisystemdesign.com/.
   This is the organization that designed and delivered the workshop itself, and their site has additional course material and information about related programs.
8. **OpenROAD Project** — https://theopenroad.readthedocs.io/.
   The OpenROAD project underlies much of the automation inside OpenLANE, and its documentation was a useful reference for understanding what was happening at        each stage of the flow in more depth.

---
