.. role:: raw-html-m2r(raw)
   :format: html


===============
Simulation
===============


Introduction
===============


You can find some documentation about how to use the simulator directly in https://github.com/SpinalHDL/NaxRiscv/blob/main/src/test/cpp/naxriscv/README.md

Also note that the simulator supports Gem5 / Konata traces which can show the execution of instructions through the different stages of the CPU.

Here is a trace of the NaxRiscv going though a linked list until it finds a node with some specific values.

.. image:: /asset/image/konata.png


Spike
===============

The simulator is running Spike (a RISC-V ISA simulator) in parallel to the DUT in a lock-step manner to ensure the DUT isn't diverging.

It is done by compiling Spike as a shared object, which is then interfaced in the Verilator simulation.

See the following git issue for more discussion for this usage of Spike :

https://github.com/riscv-software-src/riscv-isa-sim/issues/911

Multi core simulation
==============================

You can run multi-core simulation by using src/main/scala/naxriscv/platform/tilelinkdemo/SocSim.scala

It is a little NaxRiscv/tilelink SoC which can load bin/elf files and integrate rvls checking.

See SocSim.scala for usages.
