---
title: Test, Build, Deploy
subtitle: A CI/CD Framework for Open-Source Hardware Designs
author: Calvin Deutschbein, **Aristotle Stassinopoulos** 
--- 

# Motivation

::::{.columns}

:::{.column width="50%"}

> None of us is as smart as all of us.

- Amber Huffman, Principal Engineer, Google Cloud

:::

:::{.column width="50%"}

<div style="width:100%;border-radius: 50%">
<img width="100%" src="https://www.semi.org/sites/semi.org/files/styles/webp/public/inline-images/biophoto-gsmc-_0000_amber-huffman.png.webp"/>
</div>

:::

::::

## Problem Statement

- With the collapse of MOSFET scaling circa 2006, higher hardware performance became tightly coupled with higher hardware complexity due to due heat and power constraints.
    - Also called "The Heat Wall"
- An explosion of hardware complexity drove major advances in Hardware Design Languages (HDLs).
    - In 2009 when IEEE 1364 Verilog standard expanded to inlcude SystemVerilog for software-driven simulation and testing.
- The 2009 "DevOps" formulation of Continuous Integration/Continuous Deployment can be applied to **specifications** of hardware in HDLs.

## State of Play

- Major advances in open source hardware.
    - RISC-V open source CPUs
    - OpenTitan/Caliptra open source root of trust (RoT)
- Major advances in CI/CD formulations for open source hardware
    - GitHub Actions for CI/CD
    - Hardware Specification mining for HDLs
- We provide:
    - A framework to build specifications from HDLs
    - A container package to automate the process within workflows.

# Outline

1. Methodology
1. Implementation
1. Evaluation
1. Conclusion

# Methodology

## Requirements

Testbench
: An HDL "script" to be executed in simulation by a hardware design
: A way of testing hardware pre-silicon

Simulator
: A piece of software that compiles HDL and testbench into a machine executable format

Trace
: A transcription of the value of every hardware register at every timepoint while executing some instructions
: Often stored as a "value change dump" only recording *changes* for brevity

Translator
: A custom script to convert a hardware trace into a simulated software trace, for software quality tools

Miner
: A software quality tool that automatical generates specifications from traces

## Specification

- Specify hardware design as *rules* over *registers* 

::::{.columns}

:::{.column width="50%"}


```email
(* Extended Backus–Naur *)

rule = test | isin | mult | line

test = reg, eqs, reg
     | reg, eqs, nat

isin = reg, "∈ {", nat, ",", nat, "}"
     | reg, "∈ {", nat, ",", nat, ",", nat "}"

mult = reg, "≡ 0 (mod", reg, ")"

line = reg, "+", reg, "×", reg, 
            "-", reg, "+", nat, "= 0"

reg : string
nat : natural
eqs = "=" | "≠" | "<" | "⋜" | ">" | "⋝"
```

:::

:::{.column width="50%"}

```email
# Examples

wr = -1
wr < trap
wr ⋜ imem
trap ⋝ imem
trap ≠ csr
out ∈ { -1, 0 }
status ∈ { -1, 0, 6144 }
clk % imem = 0
clk + 4 * csr - 2 * pc + 3 = 0
```

- `-1` denotes a non-numeric hardware value (for uninitalized registers)

:::

::::

## Graph Representation

![](myrtha.svg)

# Implementation

:::{.hidden}

We containerized the entire workflow on Ubuntu image with best-effort minimization.

Testbench
: We used designer-provided testbenchs. 

Simulator
: We used Icarus Verilog (`iverilog`) which helpfully follows the IEEE standard.

Translator
: We adapted a custom Python script from prior work.

Miner
: We used the Daikon invariant detector which imposes a JVM dependency.

:::

## Icarus Verilog or `iverilog`

- `iverilog` operates much like a compiler over hardware design languages.
    - Similar to testing library code, basically.
    - Designs almost always have testbenchs.
- `iverilog` is often used for "small projects" but makes IEEE 1364 Value Change Dump (VCD) files.
- ASCII encoded description of binary numbers representing numerical values of registers.
    - Hundreds of registers in our minimal examples.
    - We containerized, trimmed build tools but maintained dependencies, and didn't cache trace data after making specifications.

## Translation Algorithm 

How do we go from value changes to traces?

1. Traverse to the enumeration of registers.
1. Instantiate a data structure to store registers and values.
1. Write registers to a Daikon "declarations" file. (`.decls`)
1. Traverse to value changes.
1. For each time point:
    1. Read each line, updating current register value.
    1. Append current values to a Daikon "trace" file. (`.dtrace`)


## Daikon

- We regard Daikon as an example of a state-of-the-art specification miner.
- As [doi:10.1016/j.scico.2007.01.015](http://dx.doi.org/10.1016/j.scico.2007.01.015), Daikon has ~1600 citations.
- Intended for use in Java
    - Model registers as software level variables
    - Model time slices (usually some number of nanoseconds) as methods.

::::{.columns}

:::{.column width="50%"}

```v
`timescale 1 ns / 1 ps

module acw #
(
  //-- base address
  input wire[31 : 0] r_base_addr_wire,
  input wire[31 : 0] w_base_addr_wire,
  //-- num transaction
  input wire[15 : 0] r_num_trans_wire,
  input wire[15 : 0] w_num_trans_wire,
  ...
```

:::

:::{.column width="50%"}

```java
public class Main {

 static void acw() {

  //-- base address
  long r_base_addr_wire; // long to get 64 bits
  long w_base_addr_wire; // (32 data + room for -1)
  //-- num transaction
  int r_num_trans_wire; 
  int w_num_trans_wire; 
  ...
```

:::

::::

## Podman and GitHub Actions

- We fully automate the whole process with Makefiles, a CI/CD container, and GitHub actions.
- Specifications are generally generated in around 40 seconds over our 100-1000 register size designs on commit.
    - Around 4 seconds locally - so most time is spent spinning up our image.
    - Around ~70 KB in size.
- Makefiles tend to be 1-10 lines of code
- The Containerfile, Python script, and GitHub YAML are 10-100.
- Our container was ~1.7 GB.

# Results

- We used 3 designs.
    - One for development.
    - Two for test.
- We were mostly interested in:
    - Developer time to get the process working on a new design.
    - Cloud costs in data and compute.
    - Specification size.

## PicoRV32

- A 32 bit processor implementing the RV32I (RISC-V Integer) instruction set architecture (ISA) standard with optional Multiply and Divide (M) and Compressed (C) extensions.
- Basically a minimal full-featured processor design suitable for FPGA or ASIC.

::::{.columns}

:::{.column width="50%"}

- PicoRV32 size
    - 232 registers 
    - 3049 lines of Verilog
- Testbench size
    - 2201 cycles 
    - 86 lines of Verilog

:::

:::{.column width="50%"}

- Trace and specification size

| |lines|words|bytes|
|-|-----|-----|-----|
|.vcd 	    |30356 |	46200 |	269184|
|.decls 	|2219 |	4434 |	39059|
|.dtrace 	|2936134 |	2931732 |	17474090|
|spec|1846 | 5780| 71988|

:::

::::


## Holdouts

- To evaluate (1) Generalizability and (2) Performance we checked our framework on two other designs.
    - NERV - Another RISC-V core
    - AKER - An AXI access control module.
- We had to modify one line in the Containerfile (and rebuild for 5 minutes) for AKER and add one line to our Makefile to support SystemVerilog extensions but made no other changes.
- We regard this as a successful proof of concept, and are now seeking partners with larger designs and more powerful compute resources to evaluate scaling.

> We believe specification generation time scales linearly with the product of trace length and number of registers, and specification size scales with the log of the product.

## Performance


::::{.columns}

:::{.column width="50%"}

- NERV size
    - 549 registers 
    - 1267 lines of Verilog
- Testbench size
    - 19 cycles 
    - 155 lines of SystemVerilog


- Trace and specification size

| |lines|words|bytes|
|-|-----|-----|-----|
|.vcd 	    |30356 |	46200 |	269184|
|.decls 	|2219 |	4434 |	39059|
|.dtrace 	|2936134 |	2931732 |	17474090|
|spec|1846 | 5780| 71988|

:::

:::{.column width="50%"}


- AKER size
    - 432 registers 
    - 2002 lines of Verilog
- Testbench size
    - 1055 cycles 
    - 527 lines of SystemVerilog


- Trace and specification size

| |lines|words|bytes|
|-|-----|-----|-----|
|.vcd 	 |   6290 |	10990 	|53007|
|.decls  |   3519 |	7034 |	62829|
|.dtrace |	2230270|2228160 |	13840288|
|spec|5594  |17986 |181149

:::

::::


# Summary

# Future Work

