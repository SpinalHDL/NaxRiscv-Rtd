.. role:: raw-html-m2r(raw)
   :format: html


===============
Simulation
===============


Introduction
===============


You can find some documentation about how to use the simulator directly in https://github.com/SpinalHDL/NaxRiscv/blob/main/src/test/cpp/naxriscv/README.md

Also note that the simulator support Gem5 / Konata traces which can show the execution of instruction through the different stage of the CPU.

Here is a trace of the NaxRiscv going though a linked list until it find a node with some specific values. 

.. image:: /asset/image/konata.png


Spike
===============

The simulator is running Spike (RISC-V ISA simulator) in parallel to the DUT in a lock-step manner to ensure the DUT isn't diverging.

It is done by compiling Spike as a shared object, which is then interfaced in the Verilator simulation.

See the following git issue for more discution for this usage of Spike : 

https://github.com/riscv-software-src/riscv-isa-sim/issues/911
