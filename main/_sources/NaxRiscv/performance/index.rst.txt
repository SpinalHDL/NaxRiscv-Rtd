.. role:: raw-html-m2r(raw)
   :format: html


====================
Performance and Area
====================


RV32
=========================

A few things to keep in mind : 

- You can trade FMax IPC Area
- There is better IPC xor FMAX xor Area configs  

For the following configuration : 

- RV32IMASU, dual issue,OoO, linux compatible
- 64 bits fetch, 2 decode, 3 issue, 2 retire
- Shared issue queue with 32 entries
- Renaming with 64 physical registers
- 3 execution unit (2\*Int/Shift/branch, 1\*load/store/mul/div/csr/env)
- LSU with 16 load queue, 16 store queue
- Load hit predictor (3 cycles load to use delay)
- Store to load bypass
- I$ 16KB/4W, D$ 16KB/4W 2 refill 2 writeback slots
- MMU with ITLB 6 way/192 entries, DTLB 6 way/192 entries
- BTB 1 way/512 entries, GSHARE 1 way/4KB, RAS 32 entries

Performance :

- Dhrystone   : 2.64 DMIPS/Mhz    1.48 IPC (-O3 -fno-common -fno-inline, 318 instruction per iteration)
- Coremark    : 4.70 Coremark/Mhz 1.19 IPC (-O3 and so many more random flags)
- Embench-iot : 1.59 baseline     1.35 IPC (-O2 -mcmodel=medany -ffunction-sections)

On Artix 7 speed grade 3 :

- 14.6 KLUT, 9.8 KFF, 12.5 BRAM, 4 DSP
- 145 Mhz

Reducing the number of int ALU to a single one and moving the branch to the shared pipeline will produce :


Performance : 

- Dhrystone   : 2.54 DMIPS/Mhz    (-O3 -fno-common -fno-inline)
- Coremark    : 4.19 Coremark/Mhz (-O3 and so many more random flags)
- Embench-iot : 1.39 baseline     (-O2 -mcmodel=medany -ffunction-sections)

On Artix 7 speed grade 3 :

- 13.2 KLUT, 9.7 KFF, 12.5 BRAM, 4 DSP
- 148 Mhz


To go further, increasing the GSHARE storage or implementing something as TAGE should help.

Here are a pipeline representation of the two above configurations : 

.. image:: /asset/image/pipeline_simple.png

Also notes that the NaxRiscv simulator support gem5 / konata logs, allowing to visualise the execution flow.


RV64
=========================

In a similar configuration than the above RV32 (2\*Int/Shift/Branch, 1\*/load/store/mul/div/csr/env)

Performance : 

- Dhrystone   : 2.54 DMIPS/Mhz    (-O3 -fno-common -fno-inline)
- Coremark    : 4.75 Coremark/Mhz (-O3, u32 as s32 and so many more random flags)
- Embench-iot : 1.70 baseline     (-O2 -ffunction-sections)

On Artix 7 speed grade 3 :

- 19.5 KLUT, 12.0 KFF, 12.5 BRAM, 16 DSP
- 140 Mhz

So overall, the RV64 support do not has too much of an impact compared to RV32, mostly because the current critical path are in the addresses and control paths, which stays relatively similar between the two (39 bits for RV64, 32 bits for RV32).



Notes
===============

Here is a few notes collected durring the developpement : 

- An out of order CPU without branch prediction is performing realy bad ^^
- Avoiding store having to wait for the store data in the IQ can realy help avoiding bad load speculation.
- Some tests were made with two cycle latency ALU (in prevision of RV64 timing relaxation) seems to show "little" impact on the overall performances (~15%, need to verify on more benchmarks)
- Adding more and more execution units seems to goerealy fast into diminushing returns lands


How to run the benchmark
==============================

First follow the steps in https://github.com/SpinalHDL/NaxRiscv/blob/main/src/test/cpp/naxriscv/README.md#how-to-setup-things to get a functional simulator.

Then dhrystone and coremark benchmark can be manualy run with : 

.. code:: shell

    obj_dir/VNaxRiscv --name dhrystone --output-dir output/nax/dhrystone --load-elf ../../../../ext/NaxSoftware/baremetal/dhrystone/build/rv32im/dhrystone.elf --start-symbol _start  --stats-print --stats-toggle-symbol sim_time
    obj_dir/VNaxRiscv --name coremark --output-dir output/nax/coremark --load-elf ../../../../ext/NaxSoftware/baremetal/coremark/build/rv32im/coremark.elf --start-symbol _start --pass-symbol pass  --stats-print-all --stats-toggle-symbol sim_time

To run embench, you have to clone https://github.com/SpinalHDL/embench-iot.git and then follow  the steps defined in config/riscv32/boards/naxriscv_sim/README.md
