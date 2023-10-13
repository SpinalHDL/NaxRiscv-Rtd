.. role:: raw-html-m2r(raw)
   :format: html

Coherency
============================

The memory coherency is implemented the following way on the CPU : 

- Blocks of memory (in the data cache) have permission levels (unloaded < shared < unique[dirty])
- Shared permission only allows read-only access
- Unique permission allows read/write accesses
- When the data cache want to upgrade its permission, he will issue an acquire request (on the read bus)
- When the data cache want to free-up space, he will send a release request (on the write bus)
- The interconnect can downgrade the data cache permission on a given address (on the probe bus)
- Probe requests are handled by reusing the data cache store pipeline
- To allow the CPU to do atomic load/store access, a locking interface allows to prevent a given address permission downgrade

While NaxRiscv native data cache interface to the memory is custom, it is made to be bridged into Tilelink.

.. image:: /asset/image/coherency_l1.png

From a SoC perspective, you can pick either a L2 coherent cache or a cacheless coherency hub to connect multiple masters.
