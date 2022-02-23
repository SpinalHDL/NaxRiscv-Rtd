.. role:: raw-html-m2r(raw)
   :format: html




====================
Performance and Area
====================


Performance
===============

A few things to keep in mind : 

- You can trade FMax IPC Area
- There is better IPC xor FMAX xor Area configs  

For the following configuration : 

- RV32IMASU, dual issue,OoO, linux compatible
- 64 bits fetch, 2 decode, 2 issue, 2 retire
- Shared issue queue with 32 entries
- Renaming with 64 physical registers
- 2 execution unit (1\*Int/Shift, 1\*Branch/load/store/mul/div/csr/env)
- LSU with 16 load queue, 16 store queue
- Load hit predictor (3 cycles load to use delay)
- Store to load bypass / hazard free predictor
- I$ 16KB/4W, D$ 16KB/4W 2 refill 2 writeback slots
- MMU with ITLB 6 way/192 entries, DTLB 6 way/192 entries
- BTB 1 way/512 entries, GSHARE 1 way/4KB, RAS 32 entries

Performance : 

- Dhrystone   : 2.51 DMIPS/Mhz    (-O3 -fno-common -fno-inline)
- Coremark    : 4.22 Coremark/Mhz (-O3 and so many more random flags)
- Embench-iot : 1.33 baseline     (-O2 -ffunction-sections)

On Artix 7 speed grade 3 :

- 13.1 KLUT, 9.3 KFF, 13 BRAM, 4 DSP
- 145 Mhz

Performance :

- Dhrystone   : 2.59 DMIPS/Mhz    1.42 IPC (-O3 -fno-common -fno-inline)
- Coremark    : 4.70 Coremark/Mhz 1.19 IPC (-O3 and so many more random flags)
- Embench-iot : 1.55 baseline     1.32 IPC (-O2 -ffunction-sections)

On Artix 7 speed grade 3 :

- 15.0 KLUT, 9.5 KFF, 13 BRAM, 4 DSP
- 145 Mhz

To go further, increasing the GSHARE storage or implementing something as TAGE should help.

Here are a pipeline representation of the two above configurations : 

.. image:: /asset/image/pipeline_simple.png

Also notes that the NaxRiscv simulator support gem5 / konata logs, allowing to visualise the execution flow.


Notes
===============

Here is a few notes collected durring the developpement : 

- An out of order CPU without branch prediction is will perform realy bad ^^
- Avoiding store having to wait for the store data in the IQ can realy help avoiding bad load speculation.
- Some tests were made with two cycle latency ALU (in prevision of RV64 timing relaxation) seems to show "little" impact on the overall performances (~15%, need to verify on more benchmarks)
- Adding more and more execution units seems to goerealy fast into diminushing returns lands



